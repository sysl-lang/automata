# automata

Cellular automata in [sysl](https://github.com/sysl-lang/sysl): a bit-packed grid that steps
64 cells per machine word, any rule written in B/S, Hensel or Larger than Life notation, Golly rule tables, RLE patterns, and a
[syslUI](https://github.com/sysl-lang/syslui) viewer.

![The Gosper glider gun after 60 generations](shot.png)

```
sysl run .
sysl run . -- --rule B36/S23 --pattern 1
sysl run . -- --pattern my-pattern.rle --size 400x300 --bounded
sysl run . -- --shot shot.png --size 96x72 --generations 60
```

## The engine

A row is as many `u64` words as its width needs, one cell per bit. A step never looks at one cell: the
eight neighbours of a word's 64 cells are eight words (the rows above, below and the row itself, each
shifted one place either way, with the bit that crosses a word boundary carried in), summed by
bit-sliced addition into four words holding a 4-bit count per cell. Comparing those counts with the
rule's birth and survival sets is a few ANDs per count.

Edges are a **torus** (the grid wraps) or **bounded** (dead cells beyond the edge that stay dead), and
any width works, not only multiples of 64: the bits past the width are kept at zero.

The tests hold a naive cell-by-cell step as the oracle, and the fast step must agree with it bit for
bit on random grids under six rules, six widths (1, 5, 63, 64, 65, 130) and both edge modes.

## Rules

A rule is a value, parsed from B/S notation wherever it comes from: `--rule`, the `rule =` field of an
RLE header, or the field under the board. `B3/S23`, `S23/B3` and the older `23/3` all read as Life;
`B2/S` (Seeds) has an empty half. A rule that does not read is refused with a diagnostic pointing at
the character that broke it, on stderr from the command line and in the status area in the viewer.

**Isotropic non-totalistic rules** are read in Hensel notation, as Golly and LifeWiki write it:
`B2-a/S12`, `B3/S2-i34q`, `B3aeij/S23-k`. Letters after a count name which of its arrangements it
means, `-` and letters mean all but those, and a bare count means all of them; letters may be either
case, and the status line shows the rule in Golly's canonical spelling. The 51 arrangements are
compiled into a 512-entry table (a cell's state and its eight neighbours), which the step reads once
per cell, sliding each cell's 3×3 block along the packed words a column at a time. The tests check
every arrangement against LifeWiki's drawing of it in all eight orientations, B/S rules spelled out
letter by letter against the B/S engine, random rules against a cell-by-cell oracle, and tlife against
the same rule written as a Golly `rotate4reflect` table.

## Larger than Life

```
sysl run . -- --pattern patterns/bosco.rle
sysl run . -- --pattern patterns/majority.rle --size 180x180
sysl run . -- --pattern patterns/gnarl.rle --size 400x300
sysl run . -- --rule "R10,C255,M1,S2..3,B3..3,NM" --pattern 3
```

Larger than Life counts the live cells in a neighbourhood of any range from 1 to 500, and gives birth
and survival as one range of counts each. The notation is
[Golly's](https://golly.sourceforge.io/Help/Algorithms/Larger_than_Life.html):
`R5,C0,M1,S34..58,B34..45,NM` is Bosco's Rule — range 5, two states, the middle cell counted, survival
on 34 to 58 live cells, birth on 34 to 45, the Moore square. Kellie Evans' five numbers
(`5,34,45,34,58`: range, birth range, survival range, Moore with the middle counted) read as the same
rule, and Golly's canonical spelling is what the status line shows. Letters may be either case.

- **`C`** above 2 makes a cell that fails to survive decay like Generations: state 2, 3, … and then
  dead, drawn fading toward the board. Only state 1 is counted. `C0`, `C1` and `C2` are all two states.
- **`M1`** counts the cell itself toward its own survival, so Life is `R1,C0,M0,S2..3,B3..3,NM` or
  `R1,C0,M1,S3..4,B3..3,NM`.
- **`N`** is any of Golly's neighbourhoods on a square grid: `M` Moore, `N` von Neumann, `C` circular
  (distance² ≤ r² + r), `2` Euclidean (≤ r²), `+` cross, `X` saltire, `*` star, `#` hash, `B`
  checkerboard, `D` aligned checkerboard, and `H` hexagonal, `A` asterisk and `3` tripod, which Golly
  runs on the same square array and shows skewed. Golly's triangular (`NL`), weighted (`NW`), Gaussian
  (`NG`) and custom (`N@`) neighbourhoods are refused by name, and so is a `:T`/`:P` grid suffix,
  since the grid is `--size` and `--bounded` here.
- Counts are checked as Golly checks them, against the neighbourhood's size: a rule asking for more
  cells than it holds, a range that ends below its start, or birth on 0 with more than two states is
  refused, pointing at the number.

A step pads the grid with `r` cells of what lies beyond each edge (the far side on a torus, dead when
bounded) and sums over it. **Moore** reads a summed-area table, four lookups a cell whatever the range.
**Every other shape** is a few runs per row — a whole row segment, or every second cell for the
checkerboards — and each run is one subtraction of a row's prefix sums (or of its prefix sums two
cells apart), so a cell costs about one subtraction per row of its range. On a 512×512 torus a step
takes about 1.6 ms at range 5 and 2.0 ms at range 50 under Moore; von Neumann, circular and
checkerboard take 4 ms at range 5 and 22 ms at range 50.

The bundled `bosco.rle`, `majority.rle` and `gnarl.rle` are Golly's, with their rule lines written in
Larger than Life notation (Golly ships them in HROT notation). The tests hold each neighbourhood as a
one-line test of an offset, taken from Golly's help and source, and check every shape's size against
Golly's formulas at ranges 1 to 9; step every shape against a cell-by-cell oracle at ranges 1, 2, 5 and
7 with and without the middle, on both edges and on a torus narrower than the range; check decay
against the oracle at 3, 7 and 256 states; step Life written as Larger than Life against B3/S23 bit for
bit; and run Bosco's bug through its 166-generation period, oracle and all, back to its own shape.

## Rule files

```
sysl run . -- --rule-file rules/WireWorld.rule --pattern patterns/diodes.rle --size 32x10
sysl run . -- --rule-file rules/Life.rule --pattern 4
```

`--rule-file` loads a rule in Golly's [`.rule` format](https://golly.sourceforge.io/Help/formats.html#rule).
Of its sections, four are read:

- **`@RULE`** names the rule, and the status line shows the name.
- **`@TABLE`**: `n_states` (2 to 256), `neighborhood` (`Moore` or `vonNeumann`; `hexagonal` and
  `oneDimensional` are refused by name), `symmetries` (`none`, `rotate4`, `rotate8`,
  `reflect_horizontal`, `rotate4reflect`, `rotate8reflect`, `permute`), `var` declarations (a set may
  name earlier variables, and a variable written twice in one transition is the same state both
  times), and transitions with commas or as bare digits. The first matching transition wins; a cell
  none matches keeps its state.
- **`@TREE`**: `num_states` (2 to 256), `num_neighbors` (8 for Moore, 4 for von Neumann),
  `num_nodes`, and the node lines `depth child…`, looked up in Golly's order — NW, NE, SW, SE, N, W,
  E, S, then the cell, or N, W, E, S, then the cell — with the root last. Every node's depth, child
  count, child numbers and states are checked. A file with both a `@TABLE` and a `@TREE` runs the
  one that comes first, as Golly does.
- **`@COLORS`**: `state r g b` lines and six-number gradient lines, over Golly's default red-to-yellow
  gradient.

Every other section — `@ICONS`, `@NAMES` — is skipped, as Golly skips one it does not know. A
malformed file is refused with a diagnostic at its line and column. A table is expanded once, when
the file is read, into an array indexed by a cell's state and its neighbours', so a step is one
lookup per cell; that limits a table to 2²⁴ entries, which is up to 6 states for Moore and 27 for
von Neumann. A tree has no such limit: one that fits is expanded into the same array, and a larger
one is walked for every cell, a step per neighbour.

`rules/` holds `Life.rule` (Life as a `permute` table, which the tests run against B3/S23 bit for
bit), `LifeTree.rule` (Life as a tree, which expands to exactly `Life.rule`'s table), `WireWorld.rule`,
and `Swirl.rule` (the eight-state Generations rule 23/34/8 as a tree — too many states for a table —
checked against Generations worked out cell by cell).

```
sysl run . -- --rule-file rules/Swirl.rule --pattern 4
```

## Patterns

RLE as Golly and LifeWiki write it: `#` lines (`#N` names the pattern), the `x = , y = , rule =`
header, runs of `b` and `o`, `$` and `!`. Multi-state patterns use Golly's letters: `.` for state 0,
`A`–`X` for 1–24, and `pA`…`yO` above them. A header whose rule is a name (`rule = WireWorld`) needs
the rule's file given with `--rule-file`. The reader is built on
[`parsing`](https://github.com/sysl-lang/parsing). Five patterns are bundled: glider, R-pentomino,
acorn, Gosper glider gun and pulsar.

## The viewer

| input | does |
|---|---|
| drag | pan |
| wheel | zoom about the pointer |
| click | toggle a cell |
| space | run / pause |
| `n` | one generation |
| `+` / `-` | faster / slower |
| `r` / `c` | random fill / clear |
| `t` | torus / bounded |
| `1`–`4` | Life, HighLife, Seeds, Day & Night |
| `5`–`9` | the bundled patterns |
| the rule field, then Return or **apply** | run a typed rule |

`--shot <png>` opens no window: it runs the pattern for `--generations` (default 100), lays out and
rasterizes one frame into a PlutoVG surface, and writes it out.

## Tests

```
sysl test .
```

## Licence

ISC — see `LICENSE`. `patterns/bosco.rle`, `patterns/majority.rle` and `patterns/gnarl.rle` are from
[Golly](https://golly.sourceforge.io)'s pattern collection, by the authors their `#O` lines name.
