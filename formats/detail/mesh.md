# `.mesh` — MMSH v17

> **Status:** **proven** — all 348 files decode; textured renders verified (minor INFO residue)
> **Provenance:** all 348 real `.msh*` files decoded; textured renders verified.

Cooked character/prop geometry. Verified against all **348** `.mesh`
files in the retail install (`anduril mesh scan`: 348 ok, 0 failed),
with primary RE on `characters/modelcharacters/baranor/heads/baranor_head/
player_baranor_head.mesh`.

Derived from the community `mesh.hexpat` (Project Nemesis vendored
imhex-patterns) — its section skeleton is correct; field-level details
below were re-derived and verified against real binaries. Where the
community spec is wrong or incomplete it is noted.

## Container

```text
+0x00  char[4]  "MMSH"
+0x04  u32      version = 17 (all 348 files)
+0x08  u32      desc_section_size
+0x0c  u32      info_section_size
+0x10  u32      data_section_size
```

All three section sizes **include** the section's own 4-byte id.
Sections follow in fixed order at `0x14`, `0x14+desc`, `0x14+desc+info`.

## DESC section — id `0xEBAEC3FA`

```text
u32 mat_count
u32 data_section_count        == lod_count (verified)
i32[mat_count] unknown_indices  mostly 3, sometimes 1 — UNVERIFIED
i32[mat_count] mat_indices      material slot or -1 (separators)
u32[ds] vert_data_sizes       vertex block bytes per LOD
u32[ds] face_data_sizes       u16 index bytes per LOD
u32[ds] vgroup_data_sizes     u8 vgroup bytes per LOD
```

`mat_indices` alternates value/−1 in the sample (e.g.
`0,-1,1,-1,2,-1,…`) — looks like per-connection material assignment
with gaps.

## INFO section — id `0x1A1541BC`

```text
u32 counts[10]: lod_group_count, connection_count, lod_count,
                mesh_count, unknown_ulong1, lod_settings_count,
                mesh_lod_count, shadow_lod_count, vbl_count,
                ulong_read_count
f32  unk_float0                  (400.0 in sample — draw distance?)
f32  unk_floats4[4]              (bounds-like values)
u32  bone_count
u32  bone_section_size           includes 4B id
u32  lod_group_ids[lod_group_count]      FNV-ish hashes
Connection[connection_count]     4×i32 {mesh, lod_group, a, b}
LodInfo[lod_count]    48B each:
    u32 vert_size, face_size, vgroup_size   (mirror DESC sizes)
    u32 mesh_count                          consecutive MeshInfo entries
    u16 mesh_tri_count[16]                  per-mesh TRIANGLE counts
MeshInfo[mesh_count]  40B each:
    u32 data_offset     byte off into LOD's vertex block
    u32 verts_offset    first vertex in LOD's logical vertex array
    u32 verts_count
    u32 face_offset     u16-element offset into LOD's face block
    u32 face_count      index count (tris = /3)
    u32 vg_offset       byte offset into LOD's vgroup block
    u32 vg_count        u8 entries
    u32 unk7, unk8      float-looking bit patterns — UNVERIFIED
    u32 layout_index    into VBL section
```

Then three sub-sections, each with its own id:

### LOD section — id `0x8E3E068E`

```text
u32  lod_settings[lod_settings_count]         (structure UNVERIFIED;
      probably holds the ulong_read/unknown_ulong1 arrays)
f32  mesh_lod_thresholds[mesh_lod_count]      (5500,2500,…,0 — screen
      size or distance thresholds)
f32  shadow_lod_thresholds[shadow_lod_count]
u32  mesh_lod_connections[mesh_lod_count]
u32  shadow_lod_connections[shadow_lod_count]
```

### VBL section — id `0x37D749A6`

`vbl_count` vertex-buffer layouts, each:

```text
u32 attr_count
attr_count × 4×u8 { buffer, type, attr_kind, channel }
```

Attributes interleave two buffer streams: **buffer 0** carries position
plus skinning (weights/vgroups), **buffer 1** carries shading
attributes. Byte offset within a buffer = cumulative size of earlier
same-buffer attributes in list order.

Type table (byte sizes verified by stride arithmetic):

| type | encoding        | size |
|------|-----------------|-----:|
| 1    | Vec2<f32>       |    8 |
| 2    | Vec3<f32>       |   12 |   ← the hexpat mislabels as V4F32
| 3    | Vec4<f32>       |   16 |
| 4    | Vec4<u8>        |    4 |
| 9    | Vec2<i16>       |    4 |
| 12   | Vec4<f16>       |    8 |

Attr kinds observed: 0 position, 1 normal, 2 tangent, 3 bitangent,
4 texcoord, 5 color, 6 weights, 7 vertex-group.

Observed strides (sample's 4 layouts):

| vbl | buf0 | buf1 | total | difference |
|-----|-----:|-----:|------:|------------|
| 0   |   24 |   44 |    68 | uv as V2I16 |
| 1   |   24 |   48 |    72 | uv as V2F32 |
| 2   |   24 |   48 |    72 | two V2I16 uv channels |
| 3   |   24 |   56 |    80 | two V2F32 uv channels |

Compute per-file via `VertexLayout::stride()` rather than hardcoding.

Corroborated by the external MMSH field note (Universal Modder KB,
build 3095001 — their "v0x11" = our v17): observed LOD0 attr strides of
44/48 B and a 52 B/vert variant carrying `Vec2F32` uv0 on parts that
need unquantized UVs match the layout table above (buf0 24 B + buf1
variants). Their report that the last 4 bytes of a buf1 record act as a
flags/color word is consistent with our `Vec4<u8>` color attr; whether
it is *the* color channel or separate flags is still unverified.
UV scale conventions differ per template (some assets scale stored
i16 UVs ×2 in-engine vs storing pre-scaled) — treat the quantization
scale as per-asset, not global.

**UV encoding differs by asset class** (sow-rs character census,
VERIFIED): characters use `Vec2S16` = **signed i16 / 32767** (orc:
-0.998..1.0, Talion: -0.953..1.0 — reading the same words as f16
yields ±65504 garbage); `Vec2F32` appears in some layouts; static
prefab meshes use f16. Per-vertex Normal/Tangent/Bitangent are
`Vec3F32` plus a u8x4 colour on every character layout; tangents
follow +u, bitangents +v (v down), and `(N×T)·B` sign is per-vertex
(mirrored UV islands), carried as the tangent's `w`.

Stored UVs range *outside* `[0,1]` by design: `orc_buff_body001.mesh`
carries ~half its verts (everything `pos.x < 0`) at `u ∈ [-1,0]` —
the character's mirrored half samples the mirror of the right half.
Samplers must use **mirrored-repeat** addressing (verified on both
the CPU rasterizer and the wgpu viewer: repeat-wrap smears the entire
left half into streaks). Verified visual: `view_anim3.png` —
BipInf_Sheathe mid-pose, both halves textured correctly.

Normals (kind 1, channel 0) are `Vec3F32` on buffer 1 for character
meshes (orc_buff_body001 layout 0); other encodings seen in the
corpus — `Vec4F32`, `Vec4F16` (first 3 lanes), `Vec4U8` SNORM
(b/127.5 − 1), `Vec2I16` packed (x,y SNORM, z reconstructed) — are
handled defensively in `anduril_render::normals` but only the f32
vec3 form is corpus-verified. The renderers shade per-pixel by
interpolating these normals; skinned meshes rotate them per bone
(`skin_normals` — delta rotation only, no translation).

Asset pairing: a `.mesh` is always accompanied by a same-named `.skel`
sibling when skinned, and animated meshes pair with `.anix` clip
containers. The BONE section name-hash ↔ `.skel` name resolution is
verified (98/98 on the eltariel pair).

### BONE section — id `0x93D9A424`

`bone_count` × 40-byte records:

```text
+0x00  u32   name_hash   — `name_hash()` of the bone's `.skel` name
+0x04  s16   parent index (-1 = root)
+0x06  u16   child_count
+0x08  vec3  position    — GLOBAL bind position (cm), mirrors the skel
                          bone's stored global translation
+0x14  vec4  rotation   — GLOBAL bind rotation (quat-like; non-unit
                          norms observed — normalize)
+0x24  f32   tail       — ≈1.0 (0x3F7FFFFx–0x3F800000); scale term
```

`name_hash`, `parent` and `child_count` were **verified** against
`eltariel_body_base.mesh` ↔ `player_eltariel.skel`: all 98 mesh bones
resolve to skel bone names, parents agree with the skel depth-first
reconstruction 98/98, and index-aligned world positions match 92/98
(the outliers are rig bones like `cam01`/`eyeTracker` whose bind data
differs slightly). The earlier "u32 tail = bone name hash" guess is
**refuted**: `tail` is a float ≈1.0.

**Position/rotation are GLOBAL bind transforms, not parent-local.**
Verified three ways: (1) positions accumulate up chains — the orc
`Torso` records step +12.64 cm per spine segment, impossible for
locals; (2) treating them as locals composes a broken zigzag skeleton
while returning them verbatim yields a clean bind; (3) the
Monstey-Mesh-Publisher reference (`engine/monstey_converter/
native_rig.py`) states records are global in centimetres and asserts
skel↔mesh agreement to 2e-5. `parent`/`child_count` still give the
hierarchy — needed for palette order below — just not for composition.
Locals are derived on demand as `parent_global⁻¹ ∘ global`.

**Skin-palette (vgroup) bytes index Havok sibling-block order, not
file/source order.** Per Monstey (`native_rig.py` + `native_mesh.py`:
"JOINTS = target HK transform-order indices"), each vertex's vgroup
byte resolves through the mesh's vgroup palette to a bone index in
sibling-block order — all roots, then each node's children as a
contiguous block, recursing depth-first per child — which differs
from the source record order whenever bones are reordered.
`Mesh::hk_to_source` builds the permutation; `anduril-render`
`influences` maps palette byte → HK index → source bone index.

**Weights and joint lanes are NOT positionally paired.** Corpus raw
records (orc_buff_body001 layout 0, buf0: pos Vec4F16 @0, Weights
U8x4 @8, VertexGroup U8x4 @12) show joints left-aligned (`J=[1d,00,00,00]`)
while weights sit in nonzero lanes in influence order:

- c1 `J=[1d,0,0,0]` `W=[00,00,ff,00]` → j0 ↔ 0xff
- c2 `J=[1c,13,0,0]` `W=[00,cc,33,00]` → j0↔cc, j1↔33
- c3 `J=[12,11,0d,0]` `W=[48,36,81,00]` → j0↔48, j1↔36, j2↔81
- c4 `J=[07,05,04,01]` `W=[54,8f,0f,0d]` → all lanes

So the k-th nonzero weight lane (in lane order) pairs with joint lane
k; weights sum to 255. Positional zipping silently maps single-bone
verts to palette slot 0 (Pelvis) — that was the "arm fan" artifact:
fingertip verts weighted to Pelvis while hand bones moved.

The record layout now matches the community spec's *field set*
(`{u32 hash, s16 parent, u16 childCount, …}`), just at 40 B total —
the spec's extra `u32 sibling` + trailing `vec3` don't exist in the
real file (142×40 = section size exactly).

## DATA section — id `0x95DBDB69`

Blocked layout (community spec's sequential `[verts][faces][vgroups]`
per LOD is **wrong**):

```text
vertex blocks × ds   (concatenated, sizes from DESC)
face blocks × ds     (u16 mesh-local indices)
vgroup blocks × ds   (u8 bone-palette remaps)
```

Inside LOD *i*'s vertex block, mesh *m*'s blob is at
`mesh_info.data_offset` and contains `[verts_count × buf0_stride]` then
`[verts_count × buf1_stride]`. `data_offset = verts_offset × (s0+s1)`
— verified exactly (330208 = 4856×68).

Face indices are **mesh-local** u16s into the mesh's own vertices
(not LOD-global), offset within the LOD face block by `face_offset`×2.

## Verified sample values (player_baranor_head.mesh)

- 11 LODs = 11 data sections; 14 meshes; 9 LOD groups; 20 connections;
  142 bones; 4 layouts.
- LOD6 holds 4 sub-meshes with running `verts_offset`/`data_offset`
  (mesh7: vOff=4856, dataOff=330208 = 4856×68).
- Positions: `Vec4F32` (x,y,z,1.0) at buf0+0. Normals/tangents/bitangents:
  `Vec3F32` on buf1, unit-length. UVs: `Vec2I16` quantized. Color: `Vec4U8`.

## LOD groups are parts — sow-rs verified

`lod_group_ids[g]` = name hash of the skel's LOD group `g` name; the
connection table lists each group's LOD chain (best first, `lod = -1`
end). A "LOD index" selects **one part at one detail level**, not a
uniform mesh LOD — e.g. an orc head has 11 groups (`Orc_Eye_L` → LOD
0, `Orc_Eye_R` → 1, `Gorecaps_*`, `Decal_2/3/5`, `Beard_2/3/5`,
`Orc_Buff_Head` → LODs 12/13/14). The skel LOD-group table pairs a
distance threshold with a material index per level. A whole character
= each part group in use at its own LOD for the camera distance;
which *variant* (beard/decal) is Nemesis appearance data (viewers
take the first; game's selection logic OPEN).

## Mesh → material binding — sow-rs verified

`Model:<name>` gamedb record binds `ModelFile` to `MaterialSets[0]`.
Material selection is **per LOD group**: match the mesh LOD group to
the skel LOD group by name hash → the selected level's **skeleton
material index** → `SolidMaterial` path → load `.mat` → `tDiffuseMap`.
The material index can differ between groups at the same LOD (Flint
pants: indices 0,1,1,1,0). `Model.scale` multiplies every part.
Alpha handling (partial): skip groups named `decal`/`scar`/`hair`
(non-helmets) when the selected material's shader name contains
`alpha`/`transparent`/`translucent` or its `TranslucentMaterial`
entry is nonempty — a name heuristic, not real alpha-test emulation.
There is **no** per-submesh material map verified beyond this path;
`.mat` paths in `_StructuresMaterialSet` records exist but don't bind
submeshes alone.

## Retail read path (Ghidra decompile, verified)

Loader `FUN_1404b7c08` (located via the `0x48534D4D` immediate):
version check is `version − 16 < 2` → **16 and 17 accepted**. The
`0x95DBDB69` DATA-section marker is a hard check in retail code
(`local_5c == 0x95DBDB69` → load, else `FUN_1404b7800` fail path).
Buffer regions are allocated with **0x80 alignment** for vertex
streams and **0x10** for index/other streams, and each stream is
conditionally read vs skipped by a per-element flag — matching the
sparse-LOD layout.

## Open questions

- `unk7`/`unk8` float fields in MeshInfo (bounds? thresholds?)
- Bone rotation encoding (non-unit quats) and `tail` float's role
  (scale?)
- `lod_settings` u32 array structure; `ulong_read_count`/`unknown_ulong1`
  arrays' location
- Whether vgroup bytes are the bone-palette remap (values < bone_count
  in sample) or a different grouping — **resolved**: palette bytes are
  indices into the vgroup palette, whose entries are Havok
  sibling-block bone indices (see BONE section notes above)
- `Connection` field semantics
