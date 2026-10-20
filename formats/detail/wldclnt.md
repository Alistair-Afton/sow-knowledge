# `.wldclnt` — WDSC v9 world client descriptors

> **Status:** **verified**** — population descriptors decoded
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
baraddur `{3,5,1}`): `n_regions` AABBs verified (`seq` = `0..n-1`,
`flag` ∈ {0,1}) followed by a packed blob (~64 KB on MI, ~137 KB on
desert). In that blob every u32 lane satisfies `(x >> 19) <
placement_count` on desert (100%) — the stream is a sequence of
`{placement-index, 19-bit payload}` records grouped into sorted runs
(longest run: 1099 records on MI, all sharing idx 2 first). Exact bit
framing is still OPEN — the payload is not u16-aligned, not
u32-count-framed lists, not LEB128-varint clean; it contains f32
exponent bytes (0x45/0x46/0xc5/0xc6) so part of it may be float data
interleaved with index records. The `f2`/`f3` counts for large files
(5/6, 5/1, 6/2, 5/1) presumably size two further sub-tables — record
shapes undetermined. Preserved raw as `tail`.

## Verified

`0x2c + pool_size + count×0x4c ≤ len` on all 109 files; parser enforces
bounds. CLI: `wldclnt <path>` (summary), `wldclnt <path> strings`,
`wldclnt <path> placements`, `wldclnt scan` → 109 ok / 0 failed.

## Unknowns

- Header fields f08..f24 semantics (counts/types for tail sections?).
- `+0x20` placement field (mostly 1, occasionally 3 — flag?).
- `+0x2c` base constant's exact meaning (cull radius² vs streaming
  weight) — values are round numbers per prefab, −1 = never cull.
- Whether per-placement bound variance comes from parametric prefab
  content (hypothesis) or a different record semantic.
- Tail table structure (likely layer index lists keyed by header counts).
