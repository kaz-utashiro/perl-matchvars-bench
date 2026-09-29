# perl-matchvars-bench

Benchmark showing that reading `@-` / `@+` after a successful match
on a utf8 string is orders of magnitude slower than obtaining the
same information through `pos()`.

Each read of `$-[n]` / `$+[n]` converts the byte offset to a
character offset by walking the regexp's saved string (`subbeg`) from
its beginning with `utf8_length()` (`Perl_magic_regdatum_get()` in
mg.c).  Nothing is cached, so the cost of each read is proportional
to the match position, and a loop reading offsets on every match is
quadratic over the string.  `pos()` converts through the SV's UTF-8
position cache instead, which works fine.

Write-up in Japanese, from 2014 — twelve years and the behaviour is
unchanged:
[Perl の @- と @+ のペナルティが高すぎる](https://qiita.com/kaz-utashiro/items/2facc87ea9ba25e81cd9).

Related: [perl-substr-bench](https://github.com/kaz-utashiro/perl-substr-bench)
(the opposite conversion direction; perl/perl5#24531).

## Results

12k matches on a string mixing multibyte and ASCII characters,
ubuntu-latest, measured 2026-09-29
([full run](https://github.com/kaz-utashiro/perl-matchvars-bench/actions/runs/36517817055),
[blead run](https://github.com/kaz-utashiro/perl-matchvars-bench/actions/runs/36517823582);
see [bench.yml](.github/workflows/bench.yml)):

| perl | `@-`/`@+` (sec) | pos() (sec) | ratio |
|---|---:|---:|---:|
| 5.12.5 | 1.64 | 0.159 | 10x |
| 5.14.4 – 5.16.3 | 2.90 – 3.26 | 0.14 – 0.15 | ~20x |
| 5.18.4 | 6.47 | 0.051 | ~130x |
| 5.20.3 – 5.36.3 | 4.81 – 7.32 | 0.002 – 0.004 | ~2000x |
| 5.38.5 – 5.44.0 | 0.52 – 0.59 | 0.003 | ~170x |
| blead 5.45.4 (built from source) | 0.59 | 0.003 | 205x |

The ratios in the 5.20–5.36 band are noisy because the `pos()` side
is down at a few milliseconds; the `@-`/`@+` column is the meaningful
one.

The `pos()` path was also slow originally, was half-fixed in 5.18 and
fully fixed in 5.20, and has been fast ever since.  The match
variables never were fixed: they got slower in 5.14 and again in
5.18, improved ~12x in 5.38 (presumably from `utf8_length()` itself
getting faster, which changes the constant but not the complexity),
and still cost O(offset) per read today, 5.44.0 included.
