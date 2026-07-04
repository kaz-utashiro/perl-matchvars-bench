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

Related: [perl-substr-bench](https://github.com/kaz-utashiro/perl-substr-bench)
(the opposite conversion direction; perl/perl5#24531).

## Results

12k matches on a string mixing multibyte and ASCII characters,
ubuntu-latest
([full run](https://github.com/kaz-utashiro/perl-matchvars-bench/actions/runs/28723361695),
[blead run](https://github.com/kaz-utashiro/perl-matchvars-bench/actions/runs/28723362174);
see [bench.yml](.github/workflows/bench.yml)):

| perl | `@-`/`@+` (sec) | pos() (sec) | ratio |
|---|---:|---:|---:|
| 5.12.5 | 1.45 | 0.153 | 9x |
| 5.14.4 – 5.16.3 | 2.9 – 3.3 | 0.15 | 20x |
| 5.18.4 | 6.5 | 0.068 | 95x |
| 5.20.3 – 5.36.3 | 5.7 – 8.4 | 0.003 | ~2000x |
| 5.38.0 – 5.42.2 | 0.46 – 0.59 | 0.003 | 150 – 190x |
| blead 2026-07-05 (built from source) | 0.59 | 0.003 | 222x |

The `pos()` path was also slow originally, was half-fixed in 5.18 and
fully fixed in 5.20, and has been fast ever since.  The match
variables never were fixed: they got slower in 5.14 and again in
5.18, improved ~12x in 5.38 (presumably from `utf8_length()` itself
getting faster, which changes the constant but not the complexity),
and still cost O(offset) per read today.
