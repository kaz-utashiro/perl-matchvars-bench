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

This is not a regression; it has been this way for a long time (I
first measured it in the 5.12–5.16 era).  Related to but distinct
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

On 5.42.2 (ubuntu-latest):

```
match vars:   0.560 sec (12000 matches)
pos():        0.003 sec (12000 matches)
```

I benchmarked all releases from 5.12.5 to 5.42.2
([results and workflow](https://github.com/kaz-utashiro/perl-matchvars-bench)):

| perl | `@-`/`@+` (sec) | pos() (sec) | ratio |
|---|---:|---:|---:|
| 5.12.5 | 1.45 | 0.153 | 9x |
| 5.14.4 – 5.16.3 | 2.9 – 3.3 | 0.15 | 20x |
| 5.18.4 | 6.5 | 0.068 | 95x |
| 5.20.3 – 5.36.3 | 5.7 – 8.4 | 0.003 | ~2000x |
| 5.38.0 – 5.42.2 | 0.46 – 0.57 | 0.003 | 150 – 190x |
| blead (built from source) | 0.59 | 0.003 | 222x |

Some history visible in the numbers: the `pos()` path was also slow
originally, was fixed in 5.18/5.20, and has been fast ever since.
The match variables never were: they got slower in 5.14 and again in
5.18, improved ~12x in 5.38 (presumably from `utf8_length()` itself
getting faster, which changes the constant but not the complexity),
and still cost O(offset) per read today.

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
($-[0] then $+[0], then the next match's offsets, in increasing
order) would then cost amortized O(length) for a whole match loop,
the same as the pos() idiom.  Since that adds a field to the regexp
structure, it would be for the next development cycle.

## Real-world impact

Same background as #24531: [App::Greple](https://metacpan.org/dist/App-Greple)
avoids the match variables entirely and uses `pos()` + `${^MATCH}`
because of this, and has done so for over a decade.  Any code which
naively collects match positions with `@-`/`@+` over a large
multibyte string pays a quadratic cost without any indication of why.

## Perl configuration

Measured on ubuntu-latest with shogo82148/actions-setup-perl builds
(5.12.5 through 5.42.2); also reproduced on macOS/arm64 with Homebrew
perl 5.42.2 (threaded).

<details><summary>perl -V (Homebrew 5.42.2, macOS arm64)</summary>

```
（ここに perl -V の全文を貼る）
```

</details>
