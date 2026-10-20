# `.terrain` — `TERR` v9 world terrain manifest

> **Status:** **verified**** — manifest + heightmap tiles decoded
> **Provenance:** terrain manifest + heightmap tiles decoded on the real install.

The desert world ships one file (`worlds/retail/desert/desert.terrain`,
~135 KiB): the manifest for the world's heightmap tile grid. The actual
heights live in the referenced `desert.hm_01_<n>ter.tex` textures
(parseable via `anduril-tex`; `desert.hm_01_0ter.tex` is a raw DDS, not a
`TEXR` wrapper — `Tex::parse` accepts both).

## Canonical record structure (sow-rs — verified on minasithil.terrain)

The desert description below predates this cleaner model; both agree
where they overlap (height tiles R16 + 2px border, splat maps, str16
names):

```text
"TERR" u32 9, u32 terrain_count
per terrain:
  u32 4, u32 4                  (4×4 top-level grid)
  vec3 position (cm), quat xyzw, vec3 extent (cm)
  f32 400, u32 64, u32 128, u32 3, u32 32, f32 2.0, u32 1024, u32 10
  u32 node_count, u32 ?
  str16 material, str16 root height tile, u32 tile_size
  node_count × { u32 flags; if !(flags & 0x20): f32 lo, f32 hi,
                 u32 group_index }
  u32 group_count × { str16 height, str16 map_a, str16 map_b, u32 }
```

- Nodes are a **quadtree**: low 4 bits = child mask (children
  pre-order), `0x10` = has height group, `0x20` = empty cell (no more
  fields), `0x40` = top-level. Minas Ithil: 107 nodes over a 4×4
  top-level grid, 92 tile groups.
- Group tiles: height = **R16 with a 2-px border** (260×260 root =
  4×64 interior + border; 68×68 small patches — matches our desert
  tiles); `map_a` RGBA8 mask (black where terrain exists), `map_b`
  material weights. `y = pos.y + raw/65535 × extent.y`.
- **Child-mask bit order is CONFIRMED (Andúril)**: bits map
  **row-major** — bit0=(0,0) NW, bit1=(1,0) NE, bit2=(0,1) SW,
  bit3=(1,1) SE. Proven by scoring all 24 quadrant permutations
  against the real height tiles' 2-px shared-border strips: row-major
  has mean border mismatch 1024 vs ≥2375 for every other order and
  the most tile adjacencies; the assembled heightfield renders as one
  coherent landmass. Implemented as `Terrain::parse_quadtree` +
  `Terrain::quadtree_cells` in `anduril-terr`.
- A node can carry a group tile **and** children: the coarse parent
  tile layers under its finer children (e.g. MI's L1 cell (1,1) has
  all four L2 children tiled). `terrain <p> quad-render` paints
  parents first, children over them — 87 cells, 0 missing tiles.
- MI's terrain is only the *low ground outside the city* —
  plaza surfaces are prefab geometry; sow-rs checked no axis
  permutation fits OBJL object heights (best 24% within 1 m).
- Desert's larger file (913 records / 2,572 tile names) is the same
  family at a bigger grid; our record-table description below is the
  corresponding structure.

## Layout (Andúril verified — desert parses exactly to EOF)

```text
+0x00  char[4] "TERR"
+0x04  u32     version = 9
+0x08  u32×3   {1, 8, 8}
+0x14  f32     world min corner x  (-6786.4)
+0x18  u32     0
+0x1c  f32     world min corner z  (-17677.1)
+0x20  f32×3   ~7.9e-6 (inverse scale?)
+0x2c  u32     0x3f80000c
+0x30  f32×2   world extent        (130031.7 × 130031.7)
+0x38  f32     13000.0  (height scale — normalized*13000+500 = world y)
+0x3c  f32     500.0    (height base)
+0x40  u32×9   {64, 128, 3, 32, 0x40000000, 1024, 55, 920, 2660}
+0x60  u32     field (2660)
+0x64  {u16 len, name}        material (prefabs\...\desert_terrain.mat)
       {u16 len, name}        first tile name (…hm_01_0ter.tex)
       [record table]         14,592 B = 913 × 16B records
       {u16 len, name} ×2571  remaining tile names; every tile whose
        + [u32 trailer]       index is divisible by 3 carries a u32
                              (0x2469b ×856, 0x89229b ×1 — flags, not
                              positions)
```

- 2572 tile names total; names are u16-len UTF-8, trailing `\x00`
  counts in the length.
- Tiles per cell: **856 cells × 3 tiles** = one 68×68 R16 heightmap
  (`tile index = 3·cell + 1`) plus two 132×132 BGRA blend/splat maps
  (`3·cell + 0` and `+2`), and 2 extra overview tiles (`hm_01_0` is a
  256×64-header / 128×64-payload raw DDS — the whole-map silhouette).

## Record table — `cell_records()`

Each 16-byte record is `{f32 lo, f32 hi, u32 seq, u32 aux}` with the
field order **cyclically rotated between contiguous runs**. The decoder
detects which field is `seq` (monotonically increasing inside a run) —
records with `seq == 0` are group headers whose `lo/hi` are bounds over
the run's cells.

- `lo/hi` are the cell's normalized height bounds — they match the
  68×68 R16 tile `3·seq+1` min/max exactly (verified on the full desert
  corpus).
- `aux` is a small tag (values 16, 17, 81, 23…) — semantics unknown,
  correlates with nothing positional; likely flags/LOD id.
- Groups are ~20-cell spatial chunks; seq ordering follows a nested
  pattern (4×4-cell chunks → 2×2 blocks → 2×2 cells, row-major at each
  level) but is *not* a plain row-major grid — don't derive positions
  from `seq`.

## Cell positions — `assemble_grid()`

No explicit position table exists. Positions are recovered from the
height data itself: each 68×68 tile's 2-px border contains the
neighbour cell's interior samples, so every cell's 4 directional
neighbours resolve by exact 2-wide strip match. Verified: ~3,200 clean
links, desert assembles into

- a 659-cell full-resolution landmass (`positions`), and
- a ~187-cell **second layer** (`other`) covering the same landmass at
  lower density — its cells match 2× downsampled 2×2-cell patches of
  the main layer. This is a coarser LOD level, not separate geography;
  ignore it for rendering.

The CLI loads tiles through the VFS (they live inside `.embb` bundles),
assembles the grid, and emits:

- `terrain <p> render out.png` — grayscale heightfield (interior 64²
  samples per cell; borders dropped — they're shared with neighbours).
- `terrain <p> obj out.obj` — world-scale mesh (x/z = 50 u/sample, y =
  norm·13000+500), vertices shared across quads, step 4 samples/quad.

## World registration (verified against `desert.wldclnt`)

There is **no stored tile→world table**: positions come from the
edge-matched grid plus a world transform that must be fitted against
placement ground truth (`terrain <p> register <wldclnt>` scans all 8
dihedral symmetries × pitch × origin, scoring by placement-height
correlation — Pearson r of `pos.y` vs terrain height, which is immune
to the systematic offsets that sink a plain `|Δh|<ε` count).

Verified on desert (12,876 placements):

- **Orientation is identity**: grid x → world +x, grid y → world +z.
  r = 0.78 vs ≤ 0.63 for the best alternative; all z-flips are ≤ 0.
  Independently confirmed by the overview tile's land silhouette
  (IoU ≈ 1.0, no transpose/flip).
- **Cell pitch = 4096 world units** (64 u per height sample, 64
  samples/cell interior). Minimizes the residual on steep-ground
  placements (550 vs ≥580 for neighbors); the r curve is flat
  3960–4160 but 4096 is the steep-residual minimum.
- **Origin**: the assembled grid's min-occupied cell lands at about
  `(-59848, -84016)` — fitted, not stored. `assemble_grid` normalizes
  positions to the min occupied cell, so the nominal (0,0) offset
  inside the file's own frame is unknown (seq is *not* a Morton or
  nominal index — decoded offsets don't correspond).
- **Height fit**: flat-ground placements give
  `pos.y ≈ 1.017·h + (-426)` — slope ≈ 1 confirms `h = norm·13000+500`
  scale; the −426 intercept means either the base is ~74 rather than
  500, or prop anchors sit ~430 u embedded. Ambiguous, bounded.

## Overview tile

`tiles[0]` (`hm_01_0ter.tex`) is a **128×64 R16 whole-map silhouette**
— the DDS header misdeclares 64×64 RGB32 (payload is 16,384 B =
128×64 u16). Its land mask matches the assembled grid under the
fitted frame, confirming orientation; it does not pin the absolute
frame (the blob covers more area than the assembled cells).

## Known unknowns

- The header `(-6786.4, -17677.1)` + 130032 extent does **not** anchor
  the assembled grid directly (fitted origin is ~13–16 cells off that
  corner) — its frame is likely for the nominal (pre-crop) grid whose
  cell indices aren't recoverable from `seq`.
- `aux` tag semantics; the nine header u32s.
- Per-tile trailer u32 (`0x2469b` ×856, `0x89229b` ×1).
- `.wldmsh` is the region's **merged render geometry** (sow-rs: same
  mesh block as PRFB; cliffs/mountains/rock), not the TERR heightfield
  — its verts sit above the prop cloud. Do not use it as terrain
  ground truth either way.
