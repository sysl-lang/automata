# automata

Cellular automata in [sysl](https://github.com/sysl-lang/sysl): a bit-packed grid that steps
64 cells per machine word, any rule written in B/S, Hensel, Generations or Larger than Life notation,
Wolfram's elementary rules, MCell's cyclic, Vote for Life, Weighted Life, Margolus, 1-D binary and
1-D totalistic rules, Golly rule tables, HashLife, RLE, MCell and Golly macrocell patterns, and a
[syslUI](https://github.com/sysl-lang/syslui) viewer.

![The Gosper glider gun after 60 generations](shot.png)

```
sysl run .
sysl run . -- --rule B36/S23 --pattern 1
sysl run . -- --pattern my-pattern.rle --size 400x300 --bounded
sysl run . -- --pattern 2 --torus
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

## The unbounded plane

B/S rules, hexagonal ones among them, and isotropic (Hensel) rules run on an **unbounded plane** by
default. It holds the cells in
64×64 tiles of 64 words, the grid's packing, all in one buffer with a free list of slots and a map
from a tile's coordinates (64-bit, so a glider can travel for as long as you care to watch) to its
slot. A step runs every live tile, and a neighbour only where the border they share has live cells;
the word step is the grid's, its neighbourhood drawn across the tiles' borders. A tile whose next
generation is empty is given back, so memory follows the population: a glider runs for ever on at
most four tiles.

A rule with **B0** cannot run there (every empty cell of an endless plane would be born at once), and
runs on a torus instead; so does every other kind of rule — Generations, Wolfram, Larger than Life,
MCell's families and rule tables — which keep the grid. `--torus` or `--bounded` asks for the grid
under any rule (giving both is refused), and so does an MCell file's `#WRAP`. A pattern is placed
centred on the origin, which is where the grid's centre is: switching between the plane and the grid
with `t` keeps the picture where it was, and cells beyond the grid's region are dropped on the way
in (the viewer says how many). `--size` is the grid's size, and on the plane the size of the region
the first frame shows.

The tests run the plane beside a torus too big to wrap — Life, HighLife, Seeds, Day & Night and three
isotropic rules, from soups and from the R-pentomino, acorn and gun, centred on tile seams, at
negative coordinates and 2⁴⁰ cells out — and check they agree cell for cell; run a glider 10 000
generations in each of the four directions to exactly where it should be on at most four tiles; and
check tiles are given back and their slots reused.

## HashLife

`--hashlife`, or `h` in the viewer, runs the plane under **HashLife**, Gosper's algorithm
(`hashlife.sysl`). The universe is a quadtree whose equal subtrees are one node, looked up in a table
as they are made; its leaves are 8×8 squares, a word apiece, and a level-4 node's 16×16 square is
stepped through the plane's own word step. Each node keeps its **result** — its centre half
2^(k−2) generations on — so a region that recurs anywhere, at any time, is worked out once. A step
is 2^j generations: `--step <j>`, or `[` and `]` in the viewer, from 2^0 to 2^40, and the status line
says which. A node too big for the step takes its result at that pace instead, so any count of
generations is the sum of its binary digits' steps.

The rules are the plane's — B/S, hexagonal and isotropic, **without B0**: HashLife holds all of empty
space as one shared node, and B0 would make it change. Every other rule is refused with that reason.

Memory is bounded by a node limit (2^21 nodes). When a result is about to be computed and the table
is full, the nodes still needed — those the root reaches, and those every result in progress holds,
kept on a stack of pins — are marked, and the rest are swept onto a free list and out of the lookup
tables, mid-step as well as between steps. Node numbers never change, so a sweep needs no renumbering.

The board draws only the cells under it, copied from the universe onto the plane when the view or the
universe changes, so a universe of hundreds of millions of cells draws as fast as a glider. The Gosper
gun runs to generation 2^30 (179 million cells) in a single step in 0.04 s.

**Macrocell files** (`.mc`) are read and written as [Golly documents
them](https://golly.sourceforge.io/Help/formats.html#mc): the `[M2]` line, `#R` the rule, `#G` the
generation, then the tree children first, a leaf as 8 rows of `.` and `*` each ended by `$`, a node as
`level nw ne sw se` numbering the lines before it from 1, `0` the empty node, the last line the root,
whose south-east child's north-west cell is (0, 1). `--pattern <file.mc>` runs one under HashLife,
fitted to the view. Multi-state files (leaves `1 a b c d`), forward references, children of the wrong
size and rules HashLife cannot run are refused with a diagnostic at the offending text.

The bundled `patterns/logarithmic-width.mc` is Golly's: Jason Summers and Gabriel Nivasch's pattern
whose diameter grows logarithmically, with Dean Hickerson's analysis in its header.

The tests run HashLife beside the plane cell for cell — Life patterns and soups under five rules,
hexagonal and isotropic among them, at counts that are not powers of two; check the R-pentomino settles
at 116 cells at generation 1103, the Gosper gun's population at 2^20 against its period-30 glider count,
and a glider 2^28 cells away after a single 2^30 step; count the nodes of a field of 16384 identical
blocks; run under a 3000-node limit; check the logarithmic-width file against its own analysis — the
block 8n cells east at every g(n), to generation 5 527 200 — and against the plane; and read, write
and read macrocell files and their refusals.

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

## Generations

```
sysl run . -- --pattern 6
sysl run . -- --rule /2/3 --size 256x192
```

**Generations rules** are read as Golly reads them: `B2/S/C3` (births, survivals, and the number of
states after `C`), the same with `S` first, or MCell's survivals/births/states with no letters, `/2/3`
and `345/2/4`. Hensel letters narrow the counts as they do anywhere (`B2-a/S12/C5`). There are 2 to
256 states: 0 is dead, 1 alive, and a live cell that does not survive goes through 2, 3, … to
`states - 1` a generation at a time, whatever is around it, and is then dead. Only state 1 counts as a
neighbour. The status line writes the rule in Golly's survivals/births/states (`345/2/4`), and a
Generations rule of two states is the B/S rule of the same counts.

The step counts neighbours on a plane of state-1 bits with the word-at-a-time step a two-state rule
runs (or, with letters, the 512-entry table), and keeps each cell's state in a byte: every dying cell
changes every generation, so it needs its own write whatever the engine, and the bytes are what the
viewer and the rule tables already read. The viewer draws the dying states fading from the live
colour towards the board's. The tests run Brian's Brain, Star Wars, a lettered rule, a two-state
rule, one with `B0` and one of 256 states against a cell-by-cell oracle on six widths and both edges;
`23/34/8` against `rules/Swirl.rule`'s tree bit for bit; and `B2-a/S12/C5` against the same rule
written out by hand as a `rotate4reflect` table.

## Hexagonal rules

```
sysl run . -- --pattern patterns/hex-glider.rle
sysl run . -- --rule B2/S34/C3H
```

Under the second, `r` gives a random fill.

A final **`H`** makes a B/S or Generations rule hexagonal, as in Golly: `B2/S34H`, `S34/B2H`, `34/2H`,
`B2/S34/C3H`, `34/2/3H`. The counts are of six neighbours, so none is above 6. As in Golly, the grid
stays square and a cell's six neighbours are its Moore neighbours **without the north-east and
south-west corners** (NW, N, W, E, S, SE), so RLE from Golly's hexagonal collections loads unchanged.
The step is the B/S word step counting six of the nine neighbourhood words instead of eight. A
hexagonal B/S rule runs on the plane, unless it has B0; a hexagonal Generations rule runs on the
torus. Golly's hexagonal letters (`B2o/S2m34H`) are refused with a diagnostic, and so are Hensel
letters on a hexagonal rule: Hensel's table is built from eight neighbours.

The board draws hexagonal rules in **square cells**, as Golly does by default. A sheared lattice
would have to be built into the plane's drawing, the torus's, panning, the tap that toggles a cell,
and fitting the view, so it is not done.

The tests check which six of the eight cells count, on the grid and on the plane; step hexagonal
B/S and Generations rules against a cell-by-cell count of the six on five widths and both edges; run
two gliders from the [HROT glider database](https://github.com/jedlimlx/HROT-Glider-DB) (Peter
Naszvadi's B2/S34H glider, `patterns/hex-glider.rle`, and muzikbike's B245/S3H one) to the period and
displacement it lists; run the plane beside the torus; and run B2/S34H as a hexagonal rule table
beside the `H` rule.

## Wolfram's elementary rules

```
sysl run . -- --pattern 8
sysl run . -- --rule W30 --bounded
```

`W` and a number from 0 to 255 is one of Wolfram's elementary rules, as Golly names them: a line of
cells, each becoming bit `4l + 2c + r` of the number, `c` being the cell and `l` and `r` its west and
east neighbours. Golly runs one on the plane, a generation to a row, and so does this: **the grid is
the line's history**, and a step writes the row after the latest one from it, leaving every other row
alone. Once the bottom row is reached the history scrolls up, so the latest generation is always the
bottom row. The latest generation is the lowest row with a live cell in it when a run starts, and
again after the grid is edited or another rule has run; a pattern under a `W` rule is placed at the
top.

The grid's edges are the line's ends: a torus makes the line a ring, and bounded holds it between dead
cells. Top and bottom are only history.

**Odd numbers are allowed here, though Golly refuses them.** An odd rule turns a dead cell between dead
neighbours on, and Golly, which applies the rule to every row at once, would fill the empty plane below
the pattern, the problem `B0` rules have. Here only the row after the latest is ever written, so the
rows below cannot fill, and an odd rule is the elementary automaton it is defined to be: `W1` from an
empty bounded line alternates between all on and all off. The tests check every one of the 256 rules
against a cell-by-cell oracle on random lines of six widths on both edges, `W30` from one cell row by
row, and `W90` from one cell against Pascal's triangle mod 2. Wolfram's k-colour totalistic codes are
not read: Golly has no notation for them, and this program reads only existing rule languages.

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

## MCell's families

```
sysl run . -- --rule R1/T3/C3/NM
sysl run . -- --pattern patterns/bbm.mcl
sysl run . -- --pattern patterns/critters.mcl
sysl run . -- --pattern patterns/kites.mcl
sysl run . -- --rule R3,C0,M1,S3,S6,S7,B0,B1
```

Six of the rule families in Mirek Wójtowicz's [MCell](http://www.mirekw.com/ca/) are read in MCell's
own notation, wherever a rule is read:

- **Cyclic** (`R1/T3/C3/NM`, Griffeath's spirals; `R1/T1/C14/NN`): range 1 to 10, threshold, 2 to
  256 colours, and `NM` for Moore's square or `NN` for von Neumann's diamond. A cell moves on to the
  next colour, the last to 0, when at least the threshold of its neighbours already have it. With
  `/GH` after it (`R3/T5/C8/NM/GH`) it is Greenberg and Hastings' excitable medium: a resting cell is
  excited by that many excited neighbours, and every other colour moves on regardless. The colours
  are counted cell by cell, or for a large range colour by colour with Larger than Life's prefix sums.
  `r` in the viewer fills the board with every colour.
- **Vote for Life** (`46789`, `13579`): digits with no `/`, the counts of live cells among the nine — a
  cell and its neighbours — that make a cell live. Each is the B/S rule it amounts to (`46789` is
  B4678/S35678) and runs on the word-at-a-time step.
- **Weighted Life** (`NW1,NN2,NE1,WW1,ME0,EE1,SW1,SS1,SE1,HI0,RS2,RS3,RB3`): a weight from -256 to 256
  for each neighbour and the cell itself, the sums of live weights a live cell survives on (`RS`) and
  a dead one is born on (`RB`), and `HI`, the number of states when a failed live cell decays through
  them as in Generations, or 0. Keywords come in any order, and a place not given weighs 0.
- **Margolus** (`MS,D0;8;4;3;2;5;9;7;1;6;10;11;12;13;14;15`, the Billiard Ball Machine): the grid
  cut into 2×2 blocks whose corners are on even coordinates one generation and odd ones the next,
  each block — 1, 2, 4, 8 for its cells from the north-west across and down — replaced by the block
  the sixteen numbers name. A block that would run off a bounded grid or across a torus of odd size is
  left as it is.
- **1-D binary** (`R1,W6E`, `R2,WBF8A5CD8` Kites, `R2,WBF8A18C8,H5`): a range from 1 to 4, and the
  rule in hexadecimal, a bit for every arrangement of the `2r + 1` cells about a cell, read west to
  east as a binary number — at range 1 that is Wolfram's code, so `R1,W6E` is W110. An optional `H`
  above 2 is the number of states: a live cell the rule kills decays through them as in Generations.
- **1-D totalistic** (`R3,C0,M1,S3,S6,S7,B0,B1` Champagne, `R1,C0,M1,B1` Pascal's triangle): range 1
  to 10, `C` the number of states (history above 2, as `H` is), `M1` when the cell counts itself,
  then each total of live cells a live cell survives on (`S`) and a dead one is born on (`B`), each
  with its own letter, as MCell writes them.

Both 1-D families run as the Wolfram rules do: the grid is the line's history, a generation to a row,
scrolling once the bottom is reached, and a pattern is drawn at the top. `R`, a range and a comma is
1-D binary when `W` follows, Larger than Life when the rule has a `..` in it, and 1-D totalistic
otherwise.

`--pattern` reads MCell's `.mcl` files for these families and for Life, Generations and Larger than
Life: `#GAME` and `#RULE` (split over as many lines as it takes) give the rule, `#BOARD` and `#WRAP`
the board unless `--size` and `--bounded` say otherwise, `#L` lines the cells (`.`, `A`–`X`, and `a`–`j`
in front for states above 24), and the first `#D` line the name. `patterns/` holds the Billiard Ball
Machine, Critters and Tron (the "trip-a-tron"), from *Cellular Automata Machines*, and MCell's own 1-D
examples Pascal's Triangle and Kites. The tests step every family against a cell-by-cell oracle
written from MCell's description, on both edges and on grids as small as one cell; Vote for Life and
Weighted Life against the B/S rules they are; 1-D binary at range 1 against Wolfram's codes; each
reversible Margolus rule thirteen steps forward and back again under its inverse; and MCell's own
example file. MCell's Neumann binary, general binary and rule-table families are not read.

## Rule files

```
sysl run . -- --rule-file rules/WireWorld.rule --pattern patterns/diodes.rle --size 32x10
sysl run . -- --rule-file rules/Life.rule --pattern 4
```

### Rule names

```
sysl run . -- --rule QuadLife --pattern patterns/quadlife-soup.rle
sysl run . -- --pattern patterns/diodes.rle
sysl run . -- --rules-dir ~/my-rules --rule MyRule --pattern 4
```

As in Golly's [RuleLoader](https://golly.sourceforge.io/Help/Algorithms/RuleLoader.html), a rule
string that is not a notation is a rule name, and the rule is the table in `<name>.rule`. This holds
for `--rule` and for an RLE header's `rule =`, so each bundled pattern loads with `--pattern` alone.

- **A notation wins.** `B3/S23`, `W110` and every other notation read are never looked up as files.
- **Directories**, first match wins: `--rules-dir <dir>` if given, then `rules/` in the current
  directory, then `rules/` beside the executable. (Golly searches its temporary folder, then your rules
  folder, then the supplied Rules folder; the flag and the current directory are the user's, the one
  beside the executable is the supplied one.)
- **Names are case-sensitive**, and `<name>.rule` is spelled exactly. The file's `@RULE` line must
  spell the name exactly too, as Golly requires, or the file is refused with both names.
- **Names are letters, digits, `-` and `_`.** Golly's file format forbids spaces and
  colons in them, so `QuadLife:T10` is not a name and is refused as before; a `:` suffix is not read.
- **A name nobody has** is refused with the diagnostic drawn under it: `not a known notation, and
  no Nope.rule in rules, /usr/local/bin/rules`.

`--rule-file` loads a rule in Golly's [`.rule` format](https://golly.sourceforge.io/Help/formats.html#rule).
Of its sections, four are read:

- **`@RULE`** names the rule, and the status line shows the name.
- **`@TABLE`**: `n_states` (2 to 256), `neighborhood` (`Moore`, `vonNeumann` or `hexagonal`, whose
  transitions list C,N,E,SE,S,W,NW,C' on the six cells the `H` of a B/S rule counts; `oneDimensional`
  is refused by name), `symmetries` (`none`, `rotate4`, `rotate8`, `reflect_horizontal`,
  `rotate4reflect`, `rotate8reflect`, `permute`; for `hexagonal`, `none`, `rotate2`, `rotate3`,
  `rotate6`, `rotate6reflect` and `permute`, as Golly allows), `var` declarations (a set may
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
lookup per cell; that limits a table to 2²⁴ entries, which is up to 6 states for Moore, 10 for
hexagonal and 27 for von Neumann. A tree has no such limit: one that fits is expanded into the same array, and a larger
one is walked for every cell, a step per neighbour.

`rules/` holds `Life.rule` (Life as a `permute` table, which the tests run against B3/S23 bit for
bit), `LifeTree.rule` (Life as a tree, which expands to exactly `Life.rule`'s table), `WireWorld.rule`,
and `Swirl.rule` (the eight-state Generations rule 23/34/8 as a tree — too many states for a table —
checked against Generations worked out cell by cell).

```
sysl run . -- --rule-file rules/Swirl.rule --pattern 4
```

### Langton's ant and turmites

Golly runs Langton's ant and turmites as ordinary rule trees, von Neumann, with a state for each
colour and one for each colour × ant state × direction (an ant on colour `c` in state `s` facing N, E,
S or W is `colours + 4·(states·c + s) + d`), and so does this. `rules/` holds three of Golly's own
files: `Langtons-Ant.rule` (10 states), `LangtonsAnt_LLRR.rule` (the four-colour cardioid, 20) and
`Turmite_180121020081.rule` (a two-state turmite, 18). The tests step each against a turmite stepped
directly from the action table its name spells, and Langton's ant against the literature: from one
ant, 12 more black cells and two cells diagonally every 104 steps from step 9977, and not before.
Golly's larger turmites — the 39- and 203-state ones in its Rules folder — read and run too, walked
rather than expanded.

```
sysl run . -- --rule-file rules/Langtons-Ant.rule --pattern patterns/langtons-ant.rle --size 200x200
sysl run . -- --rule-file rules/LangtonsAnt_LLRR.rule --pattern patterns/langtons-ant-llrr.rle
sysl run . -- --rule-file rules/Turmite_180121020081.rule --pattern patterns/turmite.rle
```

### Coloured Life: Immigration and QuadLife

Immigration and QuadLife are Life, B3/S23, with colours: a survivor keeps its colour and a cell born
takes the colour most of its three parents have — in QuadLife, when all three differ, the fourth
colour. They run from LifeWiki's rule files, `rules/Immigration.rule` (3 states) and
`rules/QuadLife.rule` (5), as ordinary rule trees, so on a torus as every rule table does, and in
the colours their `@COLORS` give. The tests step both against coloured Life worked out cell by cell
over mixed soups, check that their live cells, colour aside, are exactly B3/S23's, and check births
worked by hand: a two-to-one majority, and in QuadLife three different parents giving the fourth.
`patterns/` has a 24 × 24 soup for each.

```
sysl run . -- --rule-file rules/Immigration.rule --pattern patterns/immigration-soup.rle
sysl run . -- --rule-file rules/QuadLife.rule --pattern patterns/quadlife-soup.rle
```

## Patterns

RLE as Golly and LifeWiki write it: `#` lines (`#N` names the pattern), the `x = , y = , rule =`
header, runs of `b` and `o`, `$` and `!`. Multi-state patterns use Golly's letters: `.` for state 0,
`A`–`X` for 1–24, and `pA`…`yO` above them. A header whose rule is a name (`rule = WireWorld`) loads
`WireWorld.rule`, as under [Rule names](#rule-names); one that names a `W` rule is that rule. The reader is built
on [`parsing`](https://github.com/sysl-lang/parsing). Eight patterns are bundled: glider, R-pentomino,
acorn, Gosper glider gun and pulsar; two Brian's Brain spaceships head on (`/2/3`); a Star Wars soup
(`345/2/4`); and Rule 110 from one cell.

## The viewer

| input | does |
|---|---|
| drag | pan |
| wheel | zoom about the pointer |
| click | toggle a cell |
| space | run / pause |
| `n` | one generation |
| `+` / `-` | faster / slower |
| `r` / `c` | random fill (every colour under a cyclic rule) / clear |
| `t` | plane → torus → bounded → plane (torus / bounded under a rule the plane cannot run) |
| `f` | fit the view to the pattern (on the grid, to the whole grid) |
| `h` | HashLife on / off (on the plane) |
| `[` / `]` | HashLife's step: half / twice as many generations |
| `1`–`4` | Life, HighLife, Seeds, Day & Night |
| `5`–`9` | the first five bundled patterns |
| `0` | the next of the others: Brian's Brain, Star Wars, Rule 110 |
| the rule field, then Return or **apply** | run a typed rule |

On the plane the status line gives the population and the box the live cells lie in, as its size and
its north-west cell.

`--shot <png>` opens no window: it runs the pattern for `--generations` (default 100), lays out and
rasterizes one frame into a PlutoVG surface, and writes it out.

## Tests

```
sysl test .
```

## Licence

ISC — see `LICENSE`. `patterns/bosco.rle`, `patterns/majority.rle` and `patterns/gnarl.rle` are from
[Golly](https://golly.sourceforge.io)'s pattern collection, by the authors their `#O` lines name.
`patterns/hex-glider.rle` is Peter Naszvadi's, from the HROT glider database.
`rules/Langtons-Ant.rule`, `rules/LangtonsAnt_LLRR.rule` and `rules/Turmite_180121020081.rule` are
Golly's own files, unchanged but for a description naming their source; Golly is Copyright © 2005–2026
The Golly Gang and free software under the GNU General Public License, version 2 or later, and these
three files are distributed under those terms, not ISC. `patterns/logarithmic-width.mc` is Golly's
own file too, unchanged (`Patterns/HashLife/Constructions/logarithmic-width.mc`), by Jason Summers
and Gabriel Nivasch with Dean Hickerson's analysis, as its `#C` lines say, and is likewise distributed
under the GNU General Public License, version 2 or later, not ISC. `rules/Immigration.rule` and
`rules/QuadLife.rule` are LifeWiki's (`Rule:Immigration` and `Rule:QuadLife` at conwaylife.com),
unchanged but for a description naming their source; LifeWiki's content is available under the GNU
Free Documentation License 1.2, and these two files are distributed under those terms, not ISC; the
licence's text is in `LICENSE-GFDL-1.2`.
