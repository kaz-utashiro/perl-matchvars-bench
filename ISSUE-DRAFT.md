# Title

Reading @- / @+ after a utf8 match is orders of magnitude slower than pos()

# Body

## Description

Reading `$-[n]` / `$+[n]` after a successful match on a string
containing multibyte characters takes time proportional to the byte
offset of the match: every single read walks the regexp's saved
string (`subbeg`) from its beginning with `utf8_length()`.  Nothing
is cached, so a loop which reads the match offsets on every iteration
— an ordinary way to collect match positions — becomes quadratic
over the string.

The equivalent information obtained through `pos()` is two orders of
magnitude cheaper, because `pos()` converts through the SV's UTF-8
position cache (`sv_pos_b2u`), which works fine.

This is not a recent regression; it has been this way for a long time
(I first measured it in the 5.12–5.16 era).  Related to but distinct
from #24531, which is about the opposite conversion direction.

## Steps to Reproduce

```perl
use Time::HiRes qw(time);
my $s = (chr(0x3042) . chr(0x3044) . chr(0x3046) . " abc ") x 12_000;
my($t, $c);

$t = time; $c = 0;
while ($s =~ /abc/g) { my($b, $e) = ($-[0], $+[0]); $c++ }
printf "match vars: %7.3f sec (%d matches)\n", time - $t, $c;

$t = time; $c = 0;
while ($s =~ /abc/gp) { my($b, $e) = (pos($s) - length(${^MATCH}), pos($s)); $c++ }
printf "pos():      %7.3f sec (%d matches)\n", time - $t, $c;
```

On 5.44.0 (ubuntu-latest):

```
match vars:   0.515 sec (12000 matches)
pos():        0.003 sec (12000 matches)
```

I benchmarked all releases from 5.12.5 to 5.44.0, plus blead
([results and workflow](https://github.com/kaz-utashiro/perl-matchvars-bench)):

| perl | `@-`/`@+` (sec) | pos() (sec) | ratio |
|---|---:|---:|---:|
| 5.12.5 | 1.64 | 0.159 | 10x |
| 5.14.4 – 5.16.3 | 2.90 – 3.26 | 0.14 – 0.15 | ~20x |
| 5.18.4 | 6.47 | 0.051 | ~130x |
| 5.20.3 – 5.36.3 | 4.81 – 7.32 | 0.002 – 0.004 | ~2000x |
| 5.38.5 – 5.44.0 | 0.52 – 0.59 | 0.003 | ~170x |
| blead 5.45.4 (built from source) | 0.59 | 0.003 | 205x |

(The ratios in the 5.20–5.36 band are noisy because the `pos()` side
is down at a few milliseconds; the `@-`/`@+` column is the meaningful
one.)

Some history is visible in the numbers: the `pos()` path was also
slow originally, was fixed in 5.18/5.20, and has been fast ever
since.  The match variables never were: they got slower in 5.14 and
again in 5.18, improved ~12x in 5.38 (presumably from `utf8_length()`
itself getting faster, which changes the constant but not the
complexity), and still cost O(offset) per read today.

## Analysis

`Perl_magic_regdatum_get()` in mg.c:

```c
if (RX_MATCH_UTF8(rx)) {
    const char * const b = RX_SUBBEG(rx);
    if (b)
        i = RX_SUBCOFFSET(rx) +
                utf8_length((U8*)b,
                    (U8*)(b-RX_SUBOFFSET(rx)+i));
}
```

Each read converts the byte offset to a character offset by counting
characters from the start of `subbeg`, every time.  The SV's UTF-8
position cache cannot help here because the conversion operates on
the regexp's saved copy, not on the original SV — which is exactly
why `pos()` (which does go through the SV's cache) is fast and this
path is not.

A possible fix is to keep a small (byte offset, char offset) cache in
the regexp structure, reset when a new match fills `subbeg`, and walk
incrementally from the cached position.  Typical access patterns
(`$-[0]` then `$+[0]`, then the next match's offsets, in increasing
order) would then cost amortized O(length) for a whole match loop,
the same as the pos() idiom.  Since that adds a field to the regexp
structure, it would be a change for the current (5.45) development
cycle rather than a maintenance branch.

## Real-world impact

Same background as #24531: [App::Greple](https://metacpan.org/dist/App-Greple)
avoids the match variables entirely and uses `pos()` + `${^MATCH}`
because of this, and has done so for over a decade.  Any code which
naively collects match positions with `@-`/`@+` over a large
multibyte string pays a quadratic cost without any indication of why.

## Perl configuration

Measured on ubuntu-latest with shogo82148/actions-setup-perl builds
(5.12.5 through 5.44.0) and blead 5.45.4 built from source; also
reproduced on macOS/arm64 with Homebrew perl 5.44.0 (threaded).

<details><summary>perl -V (Homebrew 5.44.0, macOS arm64)</summary>

```
Summary of my perl5 (revision 5 version 44 subversion 0) configuration:
   
  Platform:
    osname=darwin
    osvers=24.6.0
    archname=darwin-thread-multi-2level
    uname='darwin sequoia-arm64.local 24.6.0 darwin kernel version 24.6.0: fri feb 27 19:34:48 pst 2026; root:xnu-11417.140.69.709.8~1release_arm64_vmapple arm64 '
    config_args='-des -Dinstallstyle=lib/perl5 -Dinstallprefix=/opt/homebrew/Cellar/perl/5.44.0 -Dprefix=/opt/homebrew/opt/perl -Dprivlib=/opt/homebrew/opt/perl/lib/perl5/5.44 -Dsitelib=/opt/homebrew/opt/perl/lib/perl5/site_perl/5.44 -Dotherlibdirs=/opt/homebrew/lib/perl5/site_perl/5.44 -Dvendorlib=/opt/homebrew/lib/perl5/vendor_perl/5.44 -Dvendorprefix=/opt/homebrew -Dperlpath=/opt/homebrew/opt/perl/bin/perl -Dstartperl=#!/opt/homebrew/opt/perl/bin/perl -Dman1dir=/opt/homebrew/opt/perl/share/man/man1 -Dman3dir=/opt/homebrew/opt/perl/share/man/man3 -Duseshrplib -Duselargefiles -Dusethreads'
    hint=recommended
    useposix=true
    d_sigaction=define
    useithreads=define
    usemultiplicity=define
    use64bitint=define
    use64bitall=define
    uselongdouble=undef
    usemymalloc=n
    default_inc_excludes_dot=define
  Compiler:
    cc='cc'
    ccflags ='-fno-common -DPERL_DARWIN -DNO_THREAD_SAFE_QUERYLOCALE -DNO_POSIX_2008_LOCALE -DHAS_BROKEN_LANGINFO_CODESET -DNO_LOCALE_COLLATE -fno-strict-aliasing -pipe -fstack-protector-strong'
    optimize='-O3'
    cppflags='-fno-common -DPERL_DARWIN -DNO_THREAD_SAFE_QUERYLOCALE -DNO_POSIX_2008_LOCALE -DHAS_BROKEN_LANGINFO_CODESET -DNO_LOCALE_COLLATE -fno-strict-aliasing -pipe -fstack-protector-strong'
    ccversion=''
    gccversion='Apple LLVM 17.0.0 (clang-1700.6.4.2)'
    gccosandvers=''
    intsize=4
    longsize=8
    ptrsize=8
    doublesize=8
    byteorder=12345678
    doublekind=3
    d_longlong=define
    longlongsize=8
    d_longdbl=define
    longdblsize=8
    longdblkind=0
    ivtype='long'
    ivsize=8
    nvtype='double'
    nvsize=8
    Off_t='off_t'
    lseeksize=8
    alignbytes=8
    prototype=define
  Linker and Libraries:
    ld='cc'
    ldflags =' -fstack-protector-strong'
    libpth=/opt/homebrew/lib /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/17/lib /Library/Developer/CommandLineTools/SDKs/MacOSX15.4.sdk/usr/lib /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib /usr/lib
    libs=-lgdbm
    perllibs=
    libc=
    so=dylib
    useshrplib=true
    libperl=libperl.dylib
    gnulibc_version=''
  Dynamic Linking:
    dlsrc=dl_dlopen.xs
    dlext=bundle
    d_dlsymun=undef
    ccdlflags=' '
    cccdlflags=' '
    lddlflags='-bundle -undefined dynamic_lookup -fstack-protector-strong'


Characteristics of this binary (from libperl): 
  Compile-time options:
    HAS_LONG_DOUBLE
    HAS_STRTOLD
    HAS_TIMES
    MULTIPLICITY
    PERLIO_LAYERS
    PERL_COPY_ON_WRITE
    PERL_HASH_FUNC_SIPHASH13
    PERL_HASH_USE_SBOX32
    PERL_MALLOC_WRAP
    PERL_OP_PARENT
    PERL_PRESERVE_IVUV
    PERL_USE_SAFE_PUTENV
    USE_64_BIT_ALL
    USE_64_BIT_INT
    USE_ITHREADS
    USE_LARGE_FILES
    USE_LOCALE
    USE_LOCALE_CTYPE
    USE_LOCALE_NUMERIC
    USE_LOCALE_TIME
    USE_PERLIO
    USE_PERL_ATOF
    USE_REENTRANT_API
  Built under darwin
  Compiled at Jul 15 2026 11:53:55
  %ENV:
    PERLDOC="-MPod::Text::Termcap"
    PERL_BADLANG="0"
  @INC:
    /opt/homebrew/opt/perl/lib/perl5/site_perl/5.44/darwin-thread-multi-2level
    /opt/homebrew/opt/perl/lib/perl5/site_perl/5.44
    /opt/homebrew/lib/perl5/vendor_perl/5.44/darwin-thread-multi-2level
    /opt/homebrew/lib/perl5/vendor_perl/5.44
    /opt/homebrew/opt/perl/lib/perl5/5.44/darwin-thread-multi-2level
    /opt/homebrew/opt/perl/lib/perl5/5.44
    /opt/homebrew/lib/perl5/site_perl/5.44/darwin-thread-multi-2level
    /opt/homebrew/lib/perl5/site_perl/5.44
```

</details>
