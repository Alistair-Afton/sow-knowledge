# Geometry: `.mesh`, `.inst`, `.wldmsh`

> **Status:** **verified****
> **Provenance:** mesh/PRFB/world-mesh decode + renders — Andúril + sow-rs.

Three mesh-carrying formats. Skinned characters/props use MMSH;
prefab and region static geometry share one declaration-driven
scheme inside PRFB and MESH v5.

## The shared static-geometry scheme — CONFIRMED

Used by `PRFB` `.inst` mesh blobs and `MESH` v5 `.wldmsh` bodies
identically (Andúril confirmed byte-level on barrel/gate/region;
sow-rs on all 5,832 `.inst`):

```text
u32 count_or_flag, u32 vertex_bytes, u32 index_bytes
vertex_bytes  vertex data — per submesh block: all stream-0 records,
              then all stream-1 records
index_bytes   u16 triangle-list indices (no 0xFFFF restart markers)
u32 decl_count, decl_count × { u32 element_count, elements }
u32 submesh_count, submesh_count × 52-byte descriptors
```

### Vertex declarations — CONFIRMED

Element = packed u32, bytes `[stream, type, usage, usage_index]`:

| type | layout | | usage | semantic |
|---|---|---|---|---|
| 2 | f32 ×3 (12 B) | | 0 | position |
| 3 | f32 ×4 (16 B; lighting prefabs only) | | 1 | normal |
| 12 | f16 ×4 (8 B, one lane = constant 1.0 pad) | | 2 | tangent |
| 10 | f16 ×2 (4 B, UV) | | 3 | bitangent |
| 4 | u8 ×4 (4 B, colour) | | 4 | UV (by usage_index) |
|   | | | 5 | colour |

Stride = sum of element sizes per stream; block stride = s0 + s1.
Common pairs: `[pos f16x4 | uv]` 12 B; `[pos f16x4 | nrm, uv0, uv1,
tan, btan, col]` 52 B; `[pos f32x3 | uv]` 16 B; `[pos f32x3 | nrm,
uv, tan, btan, col]` 56 B.

**Position pad lane varies by file** (CONFIRMED): f16x4
positions carry the constant `1.0` in lane 1 on some files
(`{x,1,y,z}`) and lane 3 on others (`{x,y,z,1}`). Detect per block by
the lane with the strongest ≈1.0 prevalence; read the other three as
xyz. Reading the wrong lane produces a flat "y≈1.87" mess.

### Submesh descriptors (52 B) — CONFIRMED

A D3D9 `DrawIndexedPrimitive` parameter block:

```text
+0x00 u32 min_index   smallest stored index in this draw
+0x04 u32 nv          vertex count   (+0x08 duplicates it)
+0x0C u32 idx_off     StartIndex — element offset into index data
+0x10 u32 base        BaseVertexIndex, in units of the decl stride
+0x14 u32 ntri        triangle count (low 24 bits)
+0x18 u32 material    index into the prefab's material table
+0x24 u32 decl        declaration index
+0x28 u32 vert_off    cumulative vertex-data end (bytes)
+0x1C/+0x20/+0x2C/+0x30  undecoded
```

Verified relations: `idx_off` advances by `3×ntri`; block =
`[(base+min)·stride .. (base+min+nv)·stride)` ending at `+0x28`;
`(vert_off_k − vert_off_{k−1}) / nv_k` = decl stride; stored indices
∈ `[min, min+nv)` → subtract `min`; blocks can be shared across
descriptors (material/LOD twins); max `vert_off` = `vertex_bytes`,
max index end × 2 = `index_bytes`.

**Truncation quirk (CONFIRMED — Andúril):** the last block can end in
zero padding — reads must clamp to the real vertex/index boundary;
the missing stream-1 tail is virtual, not corrupt data.

### Materials for static meshes — VERIFIED

The prefab's `f`-byte table (header `c` paths) is NUL-terminated
`.mat` names padded to 4 bytes; descriptor `+0x18` indexes it.
`.wldmsh` material indices instead index the `.wldclnt`'s `.mat`
name list in blob order. `<name>_1.mat` / `proxy_*` shaders and
`artshared/.../zonlyprepass.mat` mark distance-proxy and depth-
prepass geometry — skip when drawing detail.

### UV and texture conventions — CONFIRMED

- f16 UVs (statics) range beyond [0,1] legitimately: tile-wrap AND
  **mirror-repeat** addressing both occur — characters store a
  mirrored UV half with `u ∈ [−1,0]` that must sample `|u|`
  (CONFIRMED on orc body: ~half the verts carry negative u; repeat
  wrap smears the whole side into stripes). samplers: mirrored-repeat.
- Normal maps are BC3 with **X in green, Y in alpha**; blue ≈ 255,
  red is a cavity-like map (see materials.md).
- Tangents follow +u, bitangents +v (v down); handedness is
  per-vertex `sign((N×T)·B)` after any mirror (VERIFIED, 2.9 M tris).

## PRFB v8 — `.inst` — CONFIRMED structure, residual flavors

Header 0x38 B: `"PRFB", u32 8, self_hash, node_count, b, c, d, e,
f, g, h, mesh_blob_size, string_count, string_bytes`. Then
`(node_count + e)` × 0x2C nodes, sized sub-tables, `d` × 0x50
instance records (`string_index, hash, 0, draw_dist, bbox min/max,
pos, quat xyzw, scale, flag, u8 flags`), `k` bytes of `j` child-prefab
paths. The mesh blob is the declared layout above.

- Nodes are the prefab's scene-graph records (name_hash, parent,
  type) — structure CONFIRMED, semantics PARTIAL.
- 0x50 records place **child prefabs**; region files also embed the
  coarse proxy mesh (Minas Ithil: 28 submeshes ~52.8k tris) and only
  ~24 prop placements — buildings come via `.wldclnt`.
- **Residual flavor warning (CONFIRMED):** some files' 52-byte tables
  satisfy every relation above yet decode to garbage —
  `fishandguts.inst` uses a different record layout where the
  window/min semantics differ. Andúril dual-decodes (declared +
  run-detection heuristic) and keeps the more coherent result;
  ~30/407 corpus files still show geometric outliers (large flags,
  ziplines, some fort towers). Do NOT assume a clean parse means a
  correct mesh — sanity-check edge statistics or render.

### `.wldmsh` (MESH v5) — VERIFIED

0x20 B header `"MESH", 5, 1, submesh_count, decl_count, 16, size, 0`
then the identical `[vdata][idata][decls][descriptors]` block — the
region's merged terrain/cliff geometry (Minas Ithil: 17 submeshes;
4 span kilometres). Reuse the same decoder.

## MMSH — `.mesh` — CONFIRMED (skinned meshes)

Two independent readers (Andúril, sow-rs) agree on the structure —
Andúril's byte-level notes are in its `docs/formats/mesh.md`:

- Header → LOD table; per LOD: declared vertex/index/group byte
  sizes checked to the byte (sow-rs: 348/348 files).
- Per submesh: **buffer-0 attributes then buffer-1 attributes**
  (positions in one stream, normals/tangents/uv/colour/skin in
  another).
- **UVs: `Vec2S16` = s16 / 32767** on characters (orc/Talion/
  Celebrimbor census: 669/754 orc meshes; f16 read gives ±65504
  garbage); `Vec2F32` appears in some layouts. Static prefabs use
  f16 instead — the encoding differs by asset class.
- Skin weights are u8s summing to 255; the skin palette maps mesh
  joints to skeleton bones in **Havok sibling-block order** → convert
  to source bone indices (VERIFIED; also CONFIRMED in Andúril's
  skel/anix path which skins captains correctly).
- Every vertex carries Normal/Tangent/Bitangent `Vec3F32` + u8x4
  colour — real smooth-shading data (CONFIRMED; Andúril uses it
  per-pixel).

### LOD groups = parts — VERIFIED

`lod_group_ids[g]` = name hash of skel LOD-group `g`; the connection
table lists each group's LOD chain (best first, `lod = −1` end). A
"LOD index" selects **one part at one detail level** — e.g. an orc
head has 11 groups (`Orc_Eye_L/R`, gorecaps, `Decal_2/3/5`,
`Beard_2/3/5`, `Orc_Buff_Head` LODs 12-14). The skel LOD table pairs
a distance threshold + a material index per level. A character =
each used group's LOD for the camera distance; variant choice
(which beard/decal) comes from Nemesis appearance data (OPEN how
the game picks — viewers take the first).

### Mesh → material binding — VERIFIED

`Model:<name>` record: `ModelFile` + `MaterialSets[0]` (`SolidMaterial`
/ `TranslucentMaterial` per LOD group). Select per LOD group: match
mesh LOD group to skel LOD group by name hash → selected level's
skeleton material index → `SolidMaterial` → load `.mat` →
`tDiffuseMap`. The material index **can differ between groups at the
same LOD** (verified on Flint pants: group indices 0,1,1,1,0 vs
others binding 15, 5/6, 2/3, 4). `Model.scale` multiplies every part.
Alpha handling is partially known: skip groups named `decal`, `scar`,
or `hair` (except helmets) when the selected material's shader name
contains `alpha`/`transparent`/`translucent` or its
`TranslucentMaterial` is nonempty — name/shader-token heuristic,
not real alpha-test/blend emulation (PARTIAL).

### `MeshIndexPairs` variant selection — PARTIAL

`Combat/InventorySetup` → items → `ItemBuilder` → `ArmorData` →
`ModelPicker` (`RequiredBodyType`) → `SimpleModelPiece` → `Model`.
`MeshIndexPairs` `MinIndex..MaxIndex` selects among numbered mesh
groups (`_mash1..4` suffixes, case-insensitive); tagged
`RequiredAppearanceTag`/`ExcludedAppearanceTag` gate eligibility by
the grunt's appearance tags. When no group matches the range,
sow-rs keeps the armor visible by loading all groups (PLACEHOLDER —
the game's actual fallback is unknown).
