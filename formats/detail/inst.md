# `.inst` — PRFB v8 prefab instances

> **Status:** **verified** — declared mesh layout decoded; residual flavors documented
> **Provenance:** PRFB geometry layout cross-validated against sow-rs world
> notes and confirmed byte-for-byte on retail samples (brokenbarrel02,
> gate_stnbrk, minasithil, fishandguts).

Prefab descriptors — per world (`worlds/retail/<w>/<w>.inst`), per
`sectionsandlayers/` layer, and 400+ `prefabs/**` definitions. ~830
top-level files plus thousands inside `.embb` bundles, 76 B – 14.7 MB.

## Header (verified on all files)

```text
+0x00  char[4] "PRFB"
+0x04  u32     version = 8
+0x08  u32     self_hash   name_hash of the file's own VFS path (backslashes)
+0x0c  u32     node_count
+0x10  u32     b
+0x14  u32     c           material path count in the f-table
+0x18  u32     d           instance-record count (0x50 records)
+0x1c  u32     e           extra node count (second 0x2C node set)
+0x20  u32     f           byte size of the material/string table
+0x24  u32     g
+0x28  u32     h
+0x2c  u32     i           mesh blob size (tail_size)
+0x30  u32     j           string count (child-prefab paths)
+0x34  u32     k           string byte size
+0x38  …        node table, then sub-tables, mesh blob, strings
```

Loader read order (from the exe's PRFB loader `FUN_14058a26c` —
Ghidra decompile, byte-verified): `version == 8` checked by
equality; `node_count` × 0x2C records → `e` × 0x2C extra nodes →
`b`-sized 0x18 record table → `d` × 0x50 instance records →
`i` bytes raw mesh blob → `j` **length-prefixed** strings in `k`
bytes (the loader steps by a read length, not NUL scans). One
combined buffer `(nodes + b*8)·0x10 + d·0x68 + 0x18-table` is
allocated up front — 0x2C file nodes expand to 0x20 runtime
records and 0x50 file instances to **0x68 runtime records**: the
loader computes `centre = (bbox_min+bbox_max)*0.5` and
`half = (max−min)*0.5` at load and stores them at +0x58..+0x64 —
centre/extent are derived fields, not stored data. Node `u16`
fields form a parent chain (`0xffff` terminator walked at load);
each instance's `{string_off, tag}` is interned via the name-intern
map (`FUN_14058a928`) into the `u16` at runtime +0x1a. The mesh blob
lives inside the region after the node records (our parser slices
it as `tail` = the last `i` bytes).

0x2C node record: `u32 name_hash, i32 parent(-1), u32 0, u32 0,
i32 -1, u32 0, u32 0, f32 -1.0, u32 type, u32 ?, u32 ?`.

0x50 instance record (confirmed on the quest-trigger prefab):
`+0x00 u32 string_index, +0x04 u32 hash, +0x08 u32 0,
+0x0C f32 draw_distance, +0x10 vec3 bbox_min, +0x1C vec3 bbox_max,
+0x28 vec3 position(cm), +0x34 quat xyzw, +0x44 f32 scale,
+0x48 u32 flag, +0x4C u8 flags`. `string_index` maps through the
string table to a child prefab path.

## Mesh blob — declared layout (verified)

When `i` (mesh_data_size) is nonzero the blob starts with:

```text
u32 submesh_count_or_flag, u32 vertex_bytes, u32 index_bytes
vertex_bytes   vertex data — per submesh block: all stream-0 records,
               then all stream-1 records
index_bytes    u16 indices, contiguous over all descriptors
u32 decl_count
decl_count × { u32 element_count, element_count × u32 element }
u32 submesh_count
submesh_count × 52-byte descriptors
material name strings (NUL-terminated, f bytes)
```

### Vertex declarations

Each element is a packed u32 read as bytes
`[stream, type, usage, usage_index]`:

| type | meaning |
|------|---------|
| 2    | f32 ×3 (12 B) |
| 3    | f32 ×4 (16 B) — only in `sectionsandlayers` lighting prefabs |
| 12   | f16 ×4 (8 B; one lane is a constant 1.0 pad) |
| 10   | f16 ×2 (4 B; UV pair) |
| 4    | u8 ×4 (4 B; colour) |

| usage | semantic |
|-------|----------|
| 0 | position |
| 1 | normal |
| 2 | tangent |
| 3 | bitangent |
| 4 | UV |
| 5 | colour |

Observed decls: `[pos f16x4 | uv f16x2]` (12 B stride) and
`[pos f16x4 | normal, uv0, uv1, tangent, bitangent, colour]` (52 B) on
props; `[pos f32x3 | uv]` (16 B) and `[pos f32x3 | normal, uv,
tangent, bitangent, colour]` (56 B) on region/f32 files.

### Submesh descriptors (52 B each — a D3D9-style indexed draw)

```text
+0x00  u32   min_index    smallest index in this draw's index range
+0x04  u32   nv           vertex count
+0x08  u32   nv2          (== nv in every observed record)
+0x0C  u32   idx_off      StartIndex — element offset into index data
+0x10  u32   base         BaseVertexIndex, in units of the decl stride
+0x14  u32   ntri         triangle count (low 24 bits)
+0x18  u32   material     index into the material name table
+0x1C  u32   ?            (flags/ordinal — undecoded)
+0x20  u32   ?            (base-pointer offset — undecoded)
+0x24  u32   decl         vertex-declaration index
+0x28  u32   vert_off     cumulative vertex-data end, in bytes
+0x2C  f32   ?
+0x30  f32   ?
```

Verified relations (checked on barrel, gate, region; consistent with
sow-rs's all-files claim):

- `idx_off` advances by `3 × ntri`; total indices × 2 = `index_bytes`.
- `(vert_off − previous vert_off) / nv` == the declaration's stride
  for every descriptor.
- Vertex block = `[(base + min) × stride .. (base + min + nv) × stride)`
  in the vertex data; it ends exactly at `+0x28`.
- Stored indices lie in `[min, min + nv)`; subtract `min` for
  zero-based local indices. Indices are a clean triangle list — no
  `0xFFFF` primitive-restart markers (checked on barrel + gate).
- Descriptors share vertex blocks / index ranges for material and LOD
  variants (e.g. wall column draws 955 of a 2237-vert block twice).
- Material index `+0x18` selects into the trailing material strings —
  drives `tDiffuseMap` resolution for textured renders.

### Position record quirk (verified on barrel + gate)

f16x4 position records carry a constant `1.0` pad in **one** lane,
but which lane varies by file (barrel/gate: lane 1 → `{x, 1, y, z}`;
older heuristics assumed lane 3 → `{x, y, z, 1}`). The decoder picks
the lane with the strongest `≈1.0` prevalence and reads the remaining
three lanes as xyz. Files where lane 3 is the pad also occur.

### Truncated tail data (verified on barrel + gate)

The largest descriptor's `vert_off` can exceed the physically stored
vertex data — the region between the real index-buffer base and
`vert_off` is zero padding (1838 B on the barrel). The last block's
stream-1 tail is virtual; the decoder clamps reads to the index-buffer
boundary and treats missing stream-1 attributes as absent (no UVs)
rather than failing.

### Decode strategy and residual issues

`mesh_sections()` runs the declared-layout decode and the legacy
run/window heuristic, then keeps whichever is more geometrically
coherent (worst section median edge; the declared result wins unless
it is >1.5× worse). This dual-decode is load-bearing: some files'
52-byte tables satisfy every structural relation yet still decode to
garbage under the declared model — `fishandguts.inst` uses a different
flavor (descriptor `min`/`base` semantics differ) where the heuristic
wins 35–100×.

**Known residuals (do not overstate coverage):**

- Per-block stream tails: the last ~1–5% of stream-0 records in some
  large blocks decode as attribute-like values (±1.8 magnitudes), not
  positions — likely a shorter stream-0 array than `nv` records.
  Currently tolerated as outlier verts.
- Several corpus files still show large edge outliers (terrain shells,
  flags, ziplines, some fort towers): structurally valid, geometrically
  imperfect. Some are legitimately sparse low-poly plates; a few are
  unresolved flavor variants.
- `+0x1C`, `+0x20`, `+0x2C`, `+0x30` descriptor fields undecoded.
- Whether `min`/`base` can be nonzero in ways the current block model
  misses (fishandguts-like flavors) — open.

## Region file note

`worlds/retail/<w>/<w>.inst` uses the same mesh-blob layout for the
region's coarse proxy geometry (28 submeshes for Minas Ithil, ~52,789
tris — road ring + block volumes), plus `d` = 24 placement records
(props only — the buildings come from `.wldclnt`, see wldclnt.md).

## Strings and material table

The `f` byte table after the blob is `c` NUL-terminated `.mat` paths,
each padded to 4 bytes; descriptor `+0x18` indexes it. A prefab's own
`<name>_1.mat` is typically a `proxy_*` shader (low-detail distance
proxy — `tProxyDiffuseMap` / `_proxd.tex`); `zonlyprepass` submeshes
are depth-prepass occluders. Both should be skipped when drawing
(sow-rs finding, consistent with our material census).

## Distribution

~200/838 top-level `.inst` files carry a decodable embedded mesh
(3.76 M verts, 11.5 M tris corpus-wide). Many referenced props only
exist inside `.embb` bundle payloads; `Vfs::read` expands them via the
`.bndl` manifest index.

## Verified

838/838 files parse through the VFS. CLI: `inst <path>` (summary),
`inst <path> strings`, `inst <path> sections`, `inst <path> obj
<out.obj>`, `inst <path> render <out.png>`, `inst scan`.

## Unknowns

- Header fields b/g/h exact table semantics.
- Node table's runtime role (prefab scene-graph nodes?) — structure
  parsed, semantics open.
- Descriptor `+0x1C`/`+0x20`/`+0x2C`/`+0x30` fields.
- fishandguts-style alternate descriptor flavors (different record
  layout that still passes structural checks).
- LOD selection: which descriptor set is drawn at which distance —
  all sets currently render/dedup together.
- Legacy heuristic knowledge (position-run detection, flavor A/B
  index spaces, uv attr-block layouts `stride−32`, f32-shell "flavor
  C") remains documented in git history and still applies where the
  declared layout fails; it is retained as the fallback decode.
