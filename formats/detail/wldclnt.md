# `.wldclnt` — WDSC v9 world client descriptors

> **Status:** **verified** — population descriptors decoded
> **Provenance:** population descriptors decoded on real `.wldclnt` files.

World/section population descriptors — one per world (`worlds/retail/<w>/`)
and per `sectionsandlayers/` layer, plus UI prefabs. 109 files.

## Layout (verified on all 109)

```text
+0x00  char[4] "WDSC"
+0x04  u32     version = 9
+0x08  u32     f08 (1|3)
+0x0c  u32     f0c
+0x10  u32     record_count   (placement_count — verified vs table)
+0x14  u32     f14 (0|1|5)
+0x18  u32     f18 (0|1)      wldmsh_count (sow-rs)
+0x1c  u32     f1c            mat_count (sow-rs)
+0x20  u32     f20
+0x24  u32     f24 (0|0x14)
+0x28  u32     string_pool_size (name blob)
+0x2c  u8[..]  NUL-separated asset paths (not 4-aligned)
       u32 mat_count, u32 0  (sow-rs: name-offset table header)
       (wldmsh_count + mat_count) × u32 name offsets — the .wldmsh
       and .mat names in blob order
       n × 0x4c records
tail   u8[..]  u32 layer/section index tables (undecoded)
```

String pool holds 2,729 `.inst`, 163 `.mat`, 51 `.wldmsh` references
corpus-wide — a wldclnt is the list of prefabs/materials/world-meshes a
world section pulls in.

## Canonical record layout (sow-rs verified on Minas Ithil)

The 76-byte placement record ends with the name reference — **the
name is trailing**, not leading (reading from the leading `u32 1`
pairs each name with the *next* record's transform; sow-rs proved the
trailing name by mesh-bbox fitting: it wins 1,104:8 where both
neighbours fit, and 6,263/6,556 records agree within 3 m):

```text
+0x00 vec3 bbox_min (world cm)   +0x0C vec3 bbox_max
+0x18 vec3 position              +0x24 quat xyzw
+0x34 f32 scale                  +0x38 u32 1
+0x3C u32 kind (0/1/3)           +0x40 u32 type_hash
+0x44 f32 draw_distance          +0x48 u32 name_offset (into blob)
```

`type_hash` selects the placement kind — 11 hashes share this layout
on Minas Ithil: `0xD238B9DD` 6,233 static prefabs, `0xD238B9DE` 225
terrain pieces, `0x00003249` 32 small props (lanterns, torches), and
eight more prop hashes at 1–20 records. Undecoded sections follow
the table (`u32 17, 5, 6` + float data on MI — probably keyed by the
header's remaining counts).

NOTE: our per-region field rotation table below may partly be
record-frame misalignment — the canonical frame (bbox-first) is the
reference layout; verify which frame each region uses before
interpreting rotated fields.

## Records (0x4c = 76 B) — Andúril multi-layout findings

Heterogeneous — at least two shapes:

- **Placement** (verified on desert, 12,876 of 12,877 records):

  | off | contents |
  |-----|----------|
  | +0x00 | f32 ×3 — world-space position |
  | +0x0c | f32 ×4 — unit quaternion (x,y,z,w; 499/500 norm=1.0±2%) |
  | +0x1c | f32 — **uniform scale** (verified: bounds of static prefabs
          scale exactly by it — rugfloor02 s=1.6 → extents ×1.600;
          haybalestack02 s=1.3 → ×1.30 on all axes; 0.64–1.8 range) |
  | +0x20 | u32 — 1 (mostly; 3 seen) |
  | +0x24 | u32 — 0 |
  | +0x28 | u32 — sequential object id (0xd238b9dd+n on desert) |
  | +0x2c | f32 — **s × per-prefab constant** (verified: f32/s is
          identical across all 67 scale-varying assets — round values
          9000, 35000, 400000, 700000, −1 on worldshells; authored
          cull/stream radius²) |
  | +0x30 | u32 — **string-pool byte offset → asset path** (`.inst`) |
  | +0x34 | f32 ×3 — world AABB min |
  | +0x40 | f32 ×3 — world AABB max |

  `WldClient::placements()` yields `Placement { position, rotation,
  asset, object_id, bounds }`; the detector requires a unit quaternion
  and a pool-offset that lands on a NUL boundary. Position scatter of
  all 12,876 desert placements draws the real map (settlements, road
  chains, bone-trail clusters).

  The AABB bounds the placed prefab's *full* content — for prefabs
  with internally-variable arrangements (haybalestack02, rope coils,
  navconnector ropes) extents differ per placement even at identical
  transform+scale, while static props (torchstanding01, brazierorc01)
  repeat one canonical extent that matches the `.inst` mesh AABB
  exactly at s=1.0.

- **Section/typed**: leading u32s — e.g. desert rec0 `{10,0,0, <ten
  u32 string-pool offsets to .mat names>, aabb}` is a material-section
  header listing terrain layer materials. Baraddur rec0 `{2,0,0,39,87}`
  similar shape. Kept raw via `Record::raw` / `u32()` / `f32()`.

### Alternate placement layouts (B/C/D)

Only desert's root file uses layout A. Other regions' records are the
same 76 bytes with fields in a different order; detection is per-record
(unit quaternion at the layout's offset + a pool-boundary asset offset).
`WldClient::placements()` normalizes all four into `Placement`.

| Field | A (desert) | B (snowmtn) | C (island `sectionsandlayers/*`) | D (minasithil) | E (`ol_*` layers) |
|-------|-----------|-------------|----------------------------------|----------------|-------------------|
| position | `+0x00` | `+0x30` | `+0x18` | `+0x00` | `+0x2c` |
| quaternion | `+0x0c` | `+0x3c` | `+0x24` | `+0x20` | `+0x38` |
| scale | `+0x1c` | `+0x00` | `+0x34` | `+0x30` | `+0x48` |
| object id | `+0x28` | `+0x0c` | `+0x40` | `+0x3c` | — (none seen) |
| asset off | `+0x30` | `+0x14` | `+0x3c` | `+0x44` | `+0x10` |
| AABB min | `+0x34` | `+0x18` | `+0x00` | *(unverified)* | `+0x14` |
| AABB max | `+0x40` | `+0x24` | `+0x0c` | *(unverified)* | `+0x20` |
| stream radius² | `+0x2c` | `+0x10` | `+0x44` | `+0x40` | `+0x0c` |

The radius word is `s × per-prefab constant` on all five layouts —
located by a corpus-wide audit requiring `word/scale` to be a single
value per asset (C: 7/11 assets const, D: 261/271, E: 90/90; values
are the same round constants 5000..700000 plus `-1` worldshells).

B/C files lead their record table with a string-offset index (record 0
is a run of pool offsets). Correlates with `header[1]`: 0x10=A, 0x18=B,
0x00=C, 0x11=D; E files carry `header[0]=2`. E records also point back
to the layer's `.wldmsh` at `+0x04` (pool offset). Coverage after
multi-layout support: desert 12,876, snowmtn 4,846, volcano 9,625,
minasithil 6,555 placements, plus all C-section and E `ol_*` flavor
layers. Island/mtnpass/baraddur/blackgates/lithlad decode but ship no
`spawncloset` assets — those regions' NPC spawn points use a different
mechanism (likely gamedb `AI/SpawnAmbient` or nav data, undecoded).

## Tail

The tail is a region spatial index over the placement table. Cross-file
analysis (all 110 retail files) shows two stored sub-formats:

```text
+0x00  u32  n_regions        — number of 32B region AABB records
+0x04  u32  f2               — secondary table count (0 in small files)
+0x08  u32  f3               — index-list count
```

**Small files** (`f2=0`, `f3=placement_count`, `n_regions=1`):
`f3 × u32` placement-index permutation of `0..pl` (a reordered list —
e.g. keepsmall `{1,2,0,3,4,5,6,7}`), then `n_regions × 32B` region
record(s) (all-zero when degenerate), then a 20-byte zero footer.

**Grouped files** (e.g. `flairindustrialworld` `{1,0,1}`): `n_regions`
groups each = `{vec3 min, vec3 max, u32 flag, u32 seq}` 32B AABB,
`{u32 lo, u32 cnt}` 8B range header, then `cnt × u32` placement indices
covering `0..pl` — a per-region membership list.

**Large worlds** (MI `{17,5,6}`, island `{8,5,1}`, desert `{16,6,2}`,
baraddur `{3,5,1}`, volcano `{27,5,5}`, …): **fully decoded** — two
sequential sections, verified byte-exact on desert (tail ends at
exactly the last leaf record + trailing index array):

```text
Section A — layer/group index   (FUN_1406b3cd0, ×f08 times)
  +0x00  u32 n_aabb, u32 n_veclists, u32 n_groups
  +0x0c  n_aabb × 32B   {vec3 min, vec3 max, u16, u8, 5B pad}
                        — region AABBs (desert: 16, all sane world
                        coords; entries may be sparse/zero)
  +...   n_veclists × {u8 n, n×12B vec3}   (desert 6 lists, 24 pts)
  +...   n_groups × {u32 id, u32 n, n×u32 placement_idx}
         — group id 0 holds ~all placements (desert 12,714/12,877,
         island/blackgates exactly all); other ids are name-hash-like
         layer groups (0xD9E….. family)

Section B — spatial tree   (FUN_1406b3664)
  +0x00  13 × u32 header — fields at positions {0,1,3,4,6,7,8,9,11}
         used; positions 2,5,10,12 are required zero-sentinels
         (desert: {1,1,0,1,1,0,0,207,207,80404,0,·,0})
  +...   field8(n_leaf) × leaf record:
           {f32 radius, u32 flags, u32 n_vec3, u32 n_a, u32 n_b}   20B
           n_vec3 × 12B vec3 positions
           n_a × u16   — index list
           n_b × u16   — index list
         — 207/207 leaves tile byte-exact on desert; radii are round
         draw/stream distances (20000–30000, −1 = unbounded)
  +...   field0 × interior node (FUN_141cd3e54):
           {60B header incl. AABB, u32 region_idx→72B runtime recs,
           u32 flags, u8 n_children} + n_children×16B child refs
           + u32 n, n×u32 → 96B recs + u32 m, m×u32 → 160B recs
  +...   field3 × index node (FUN_141cd401c):
           {20B header {u32 n,…}} + n×u32 leaf refs + 2×u32 →40B recs
         — the trailing "sequential u32 1..n" table seen in raw dumps
           is this leaf-index array
```

Semantics: Section B is a two-level bounding-volume / streaming
index — one interior root over `n_leaf` leaves, each leaf covering a
spatial cell of `n_vec3` positions + `n_a`/`n_b` placement-index
lists (u16 lanes seen in earlier byte censuses were the u16 index
arrays; the 0x44–0x48 "codes" were f32 radii). `f44` leading
placement records are skipped when the runtime region array is
bound (`records + f44×0x48`) — consistent with record 0 being a
material/section header, not a placement.

## Retail read path (Ghidra decompile, verified)

`ShadowOfWar.exe` headless analysis (`C:\tools\ghidra-proj\sow`,
331,366 functions) locates the loader via the `0x43534457` ("WDSC")
immediate — exactly one hit:

- `FUN_1406b3510` — file loader: checks `WDSC` + version 9, calls
  `FUN_140423e70` (the **namehash**, already byte-confirmed) on the
  file path, then `FUN_1406b3914` (body) + `FUN_1406b3664` (tree).
- `FUN_1406b3914` — main parser: reads the 9-u32 file header
  (+0x08..+0x2c — **all of f08..f28 are element counts/sizes**),
  reads the `f28`-byte string pool, `f0c` name-offset entries of
  `{u32 n_items, u32 name_off, u16}` each with `n_items × u32`
  sub-indices, then `f10` placement records via `FUN_141ccb150`,
  then `f08` Section-A sub-structures via `FUN_1406b3cd0`.
- `FUN_141ccb150` — placement reader: **0x48 (72 B) data + trailing
  u32 pool offset** = 76 B/record exactly; name offset is
  `namehash`'d and interned; contains a `u8` 2-bit flag field (the
  `&3` group split in Section A) and the f32 stream radius that the
  engine squares — byte-level proof of the "name is trailing" and
  "radius²" findings.
- `FUN_141cd4124` — leaf reader (20B hdr + vec3s + 2×u16 lists,
  `minps`/`maxps` bound accumulation); `FUN_141cd3e54` — interior
  node (60B hdr + child refs + index lists into 96B/160B records,
  linked via `+0x90/+0x98`); `FUN_141cd401c` — index node
  (20B hdr + u32 leaf-index list + 2×u32 refs into 40B records).

## Verified

`0x2c + pool_size + count×0x4c ≤ len` on all 109 files; parser enforces
bounds. CLI: `wldclnt <path>` (summary), `wldclnt <path> strings`,
`wldclnt <path> placements`, `wldclnt scan` → 109 ok / 0 failed.

## Unknowns

- Semantic role of tail Section A groups vs Section B leaves (named
  layer membership vs spatial streaming cells — partially modelled).
- Interior-node 60-byte header field order and flag byte meanings.
- `+0x20` placement field (mostly 1, occasionally 3 — flag?).
- Whether per-placement bound variance comes from parametric prefab
  content (hypothesis) or a different record semantic.
- Small-file tail variant ({1,0,pl} permutation + 32B record) — does
  it follow the same Section-A grammar with degenerate counts?
