# World data: `.wldclnt`, `.objres`, `.terrain`, `.wldmsh`, `.veggrid`

> **Status:** **verified**
> **Provenance:** region assembly evidence on Minas Ithil + desert — sow-rs + Andúril.

How a region is assembled. All VERIFIED by sow-rs on Minas Ithil
(`worlds/retail/minasithil/`, bundle 1.42 GB) unless noted; Andúril
has independent `.wldclnt`/`terrain` readers and renders the desert
region scene coherently (CONFIRMED at the structure level).

## `.wldclnt` — WDSC v9 — the region's placement table — VERIFIED

```text
"WDSC" u32 9, then u32 fields incl. placement_count (6,556 MI),
wldmsh_count+mat_count (1+15), name_blob_size, ...
0x2C: name blob — NUL .wldmsh/.mat/.inst paths
then u32 mat_count, u32 0,
  (wldmsh_count + mat_count) × u32 name offsets,
  placement_count × 76-byte records
```

**76-byte placement record — name is TRAILING** (the record ends
with the name offset; reading it from the leading `u32 1` pairs each
name with the *next* transform — sow-rs fixed this by bbox fitting:
trailing name fits 1,104:8 and 6,263/6,556 records agree within 3 m):

```text
+0x00 vec3 bbox_min (world cm)   +0x0C vec3 bbox_max
+0x18 vec3 position              +0x24 quat xyzw
+0x34 f32 scale                  +0x38 u32 1
+0x3C u32 kind (0/1/3)           +0x40 u32 type_hash
+0x44 f32 draw_distance          +0x48 u32 name_offset (into blob)
```

11 type hashes share the layout (MI): `0xD238B9DD` 6,233 static
prefabs, `0xD238B9DE` 225 terrain pieces, `0x00003249` 32 lanterns/
torches/props, eight more prop hashes 1–20 records each. Sections
after the table (`u32 17, 5, 6` + float data) are OPEN.
Andúril's CPU scene render confirms the placements compose a
coherent city.

## `.objres` — OBJR v13 — sectioned object resources — CONFIRMED

Header `"OBJR", u32 13, u32 hash, u32 section_count`; then
`section_count × {u32 tag_hash, u32 size, size bytes}` ending exactly
at EOF (independently verified by Andúril's declared-section walk —
tiles to EOF on `minasithil.objres`). **Note:** section tags are not
4-aligned in the file, so byte-scan tag surveys miss most sections —
always walk the declared frame. Minas Ithil's 14 sections:

| Tag | Bytes | Content |
|---|---|---|
| `KEYF` v1 | 15k | 9 key-frame tracks |
| `SFXS`, `SWFX` | 62k, 40k | FX paths / world FX |
| `ONLS`, `NLFE` | 488k, 39k | (undecoded) |
| `PFDG` | 1.5M | nav surface points + markup records — PARTIAL, below |
| (none) | 984k | clutter-prop `.mat` paths |
| `SCRB` | 173k | (undecoded) |
| `VOLM` | 3.5M | volumes |
| `CAWR` | 6.8k | CADB group-key lookup mirror: `{count, count×{key,0,key}}` — keys = `name_hash` of OBJL object names (CONFIRMED, Andúril) |
| `CADB` v3 | 9.5M | **context-action edge markup** — see traversal.md |
| `GADB` | 800k | per-region gamedb (`ObjectDB` records; names `minasithil.ss_*` stamp bundles) |
| (none) | 30.9M | object resource table: 740 `{u16 len, path}` (.inst/.fxp) each + `u32 id, f32 draw_dist, vec3 half_extent, vec3 centre` (bounds, not placements) |
| `OBJL` v5 | 4.7M | **object list** — the region's named objects |

### PFDG — nav surface — PARTIAL (Andúril)

`{u32 n1, u32 n2}` then `n1 + n2` × 12-byte vec3 slots of nav-surface
points (~2% sentinel slots skipped on the same grid), then 21-byte
markup records `{u32 seq, rgba, u32 a, u32 b, u8 flag, u32 -1}`.
Minas Ithil: 67,354 real points / 23,984 seq-flavoured records; the
point cloud plots as the city's street map — CONFIRMED nav-surface
semantics. Record stats (Andúril): `flag ∈ {0,1}` on real records;
`seq = 0xFFFFFFFF` on most — the 2,141 real seqs span
`25,250..33,607`, indexing the *first* vert partition (`n1`). The two
vert partitions differ in kind: p1 is clean positional, p2 is
**interleaved 24-byte cells** `{vec3 desc, vec3 pos}` (~17,593 cells —
alternation + even-parity desc slots confirmed). `pos` slots alone
render as the city's nav-cell centres; `desc` is a small quantized
triple (~10–300) that is mostly distinct per cell — per-cell extents
or packed data, exact semantics OPEN. `(a,b)` pairs are
618 **disjoint windows** spanning `100..35,169 ⊂ [0, n2)` — they
partition regions of that second array; records sharing a window are
markups grouped under it (`a` mostly nondecreasing). `rgba` runs the
rainbow spectrum — debug colours. Note: a no-seq `{rgba,a,b,flag,
-1,-1}` pattern also matches between records but yields garbage flags
— parser false positives, not a second record kind.

### OBJL v5 — VERIFIED

`"OBJL", u32 5, u32 1, u32 object_count, u32 type_bytes,
u32 name_bytes, u32 extra_bytes`; three NUL-terminated blobs (1,685
type names like `WorldModel`, `Prop`, `AINode*`, `StreamingStamp`),
then `u32 hash, u32 object_count`, then **50-byte objects**:
`u32 type_off, u32 name_off, u16 flags (0xFFFF for 87%), u32
gadb_record, vec3 pos, quat xyzw, u32, u32 hash`. Minas Ithil: 71,269
objects — AI nodes, spawners, quest points, lights; lighting-section
objres files use the same layout for `LightDirectional`,
`AtmosphereVolume`, `*Component`, `SkyCamera`, `SpecialFX` objects
whose properties live in the section's own GADB `ObjectRec`s (hashed
attribute names, recovered by dictionary).

## `.terrain` — TERR v9 — CONFIRMED structure incl. quadtree order

```text
"TERR" u32 9, u32 terrain_count
per terrain: u32 4, u32 4 (4×4 top-level grid)
  vec3 position(cm), quat xyzw, vec3 extent(cm)
  f32 400, u32 64, u32 128, u32 3, u32 32, f32 2.0, u32 1024, u32 10
  u32 node_count, u32 ?
  str16 material, str16 root height tile, u32 tile size
  node_count × { u32 flags; if !(flags&0x20): f32 lo, f32 hi, u32 group }
  u32 group_count × { str16 height_path, str16 map_a, str16 map_b, u32 }
```

- Nodes = quadtree: low 4 bits child mask (children pre-order),
  `0x10` has height group, `0x20` empty cell, `0x40` top-level.
- **Child-mask bit order CONFIRMED (Andúril)**: row-major quadrants —
  bit0=(0,0) NW, bit1=(1,0) NE, bit2=(0,1) SW, bit3=(1,1) SE. Scored
  all 24 permutations by 2-px border-strip continuity between
  same-level neighbours on the real MI tiles: row-major wins by >2.3×
  and renders as one coherent landmass (87 occupied cells).
- Nodes may carry a group tile **and** children (CONFIRMED Andúril):
  the parent's coarse tile underlays its finer children — paint
  pre-order (parent first, children over).
- Height tiles = R16 with 2-px border; `y = pos.y + raw/65535·extent.y`;
  `map_a` RGBA8 mask (black where terrain), `map_b` material weights.
- MI's terrain is the *low ground outside the city* — plaza floors
  are prefab geometry, not TERR (verified: no axis permutation fits
  objects).

## `.veggrid` — VEGG v1 — PARTIAL

`"VEGG" u32 1` + short `str16` region name; grid fields uninterpreted.

## Weather → lighting chain — VERIFIED

A region's sun/sky/fog live in a **lighting section bundle** chosen
by weather, not in the region files:

1. `Weather/Climates:<region>` → per-quest `QuestPatternEntry`
   patterns + `Bundle`.
2. `Weather/Patterns:<p>` → `Daytime`, `Section`
   `worlds/retail/<r>/sectionsandlayers/<section>.bndl`, `FXContexts`,
   `SkyBrightness`, fog `MaterialOverride` links.
3. Section bundle: `.wldclnt` (sky mesh + vista prefabs), `.objres`
   (light/atmosphere/fog/HDR/shadow objects + GADB properties),
   `.inst` (fog/light cards), light-probe volumes
   (`*.lso_mid_01.{x,y,z,n,p}.tex` + `.env.texa`), shadow tree
   (`*.osc_main.cres`), `.wldmsh` **sky dome** (~46 km radius around
   a `SkyCamera` ~52 km below the city; `tEmissiveMap` = BC6H HDR
   panorama).
4. `Weather/Lighting/Base:<region>` tints outdoor light per forecast.

Key component properties (recovered attribute names, VERIFIED):
`OLC_Main.direction` = Euler pitch-then-yaw (degrees, LH frame) for
the sun; `intensity` ~38,000; `diffusecolor`/`specularcolor` i32
ARGB; `FSC_Main` `skycolor`/`suncolor`/`sunintensity`/`skyextinction`
(~0.015/km — real Rayleigh-scale clear air)/Rayleigh+Mie terms/
`fogmaptexture`; `HDRSC_*` bloom/contrast/hdr-grey; `OSC_*` shadow
distances + `*.sst` shadow tree; `LD_*` projector fill lights;
`AtmosphereVolume` lists its components by object name.

## Hidden/auxiliary submeshes — VERIFIED

Skip when rendering: `proxy_*` + `*_1.mat` distance proxies,
`zonlyprepass` occluders, `translucent_lit\displacement\*` water
ribbons (the stored mesh is tessellation input for a displaced
surface — e.g. MI submesh 27, `mi_river.mat`, reads as two giant
"slabs" over the city if drawn naively; vertex colours are marker
data, not shading).
