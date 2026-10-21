# Materials and textures: `.mat`, `.tex`, `.texa`

> **Status:** **verified**
> **Provenance:** LTMI properties, census, shader models — sow-rs + Andúril.

## LTMI — `.mat` — CONFIRMED structure

Material files bind a shader to typed properties: shader path
(`shaders\shadertrees\...\<name>.fxo`) plus named parameters —
texture references (`tDiffuseMap`, `tNormalMap`, ...), floats
(`k_f*`), vectors/colours (`k_v*`/`k_c*`), ints (`k_n*`), bools
(`k_b*`).

- Full scan (VERIFIED): 1,198 loose files, 0 failures.
- Property types 1/14/15 are **GPU resource-binding tokens**
  (4 B words — source/cloth/skinning/exposure buffers); retained
  uninterpreted, runtime slot semantics OPEN.
- Property order in the file maps to the shader's `cb3` constant
  buffer layout (VERIFIED for `wraith_game_detail.fxo`: `k_*`
  properties pack by HLSL rules — a vector never straddles a 16 B
  row; textures become t0..tn in file order).

## TEXR / TEXA — `.tex`, `.tex0`, `.texuhd`, `.texa` — CONFIRMED

TEXR wraps DDS payloads; `.texa` are DX10 array textures. Full scan
(VERIFIED): 14,147 `.tex` + 9,371 `.tex0` + 1,378 `.texuhd` +
70 `.texa` parsed; `.texa` discovery locates `DDS ` payloads by
magic (experimental per sow-rs; rescan reports 1,137 clean).

- `.tex0`/`.texuhd` are lower/higher-detail copies of the same
  texture — the VFS picks per platform/setting.
- `SLVT` containers pack multiple TEXR records (see archives.md).

## Texture content conventions — CONFIRMED/VERIFIED

- **Normal maps: BC3 (DXT5) with X in green, Y in alpha**; blue ≈
  255, red is a cavity-like map (VERIFIED on city + character maps).
  Convert to BC5 without decode: keep the alpha block as BC4-Y,
  re-encode green→BC4-X (`normal_bc3_to_bc5` trick, error ≤1/14 of
  block range).
- **Weapon recipe** (`talion_weapon` shader, LordMonstey): color,
  packed normal/AO (X in G, Y in A, **AO in B**) and packed
  metalness/roughness textures. Channel meaning is per shader — don't
  infer it from `_n`/`_s` suffixes. Native geometry is Y-up
  centimetres, quaternions XYZW; SKEL record order ≠ Havok palette
  order used by mesh blend indices; vertex color can affect native
  shaders.
- **Diffuse alpha** on 4-layer shaders is a height/cavity map
  (bright faces, dark mortar) — used for height-blend, not opacity.
- **Specular map**: R ≈ specular intensity (~0.66 skin/wood, ~0.92
  polished metal, 0 cloth), B ≈ roughness, G/A detail/unused
  (PLACEHOLDER spec→PBR mapping: spec=R, roughness=B, metallic=0).
- **tMaskMap** (4-layer shaders): per-prefab BC3, RGBA = layer
  weights 0–3.
- **`.texa` slice selection:** `k_vSliceIndices` vec4 picks which
  array slices the four blend layers use (e.g. `[4,5,6,7]` into an
  8-slice 2048 px BC3 array).
- **UV sets:** `spec2uv` prefabs carry two UVs — set 0 = tiling
  layer UV (range ±4+), set 1 = unique mask UV [0,1];
  `k_fMapTileRate` scales tiling (1.0 walls, 9–12 cliffs). `vmask`
  shaders take weights from the vertex colour instead (every static
  vertex has a u8x4 colour).
- **Mirror-repeat:** see geometry.md — mirrored UV halves are real.
- `k_MaskBias`, `k_fBlendSmoothness`, `tHeightMap` drive 4-layer
  blends; the exact formula is OPEN (viewers use a height-lerp
  approximation).

## Shader census — VERIFIED (sow-rs)

Minas Ithil region (728 materials): `spec2uv_4layer_2bdns_tmask`
251, `proxy_specularalphacull` 131 + `proxy_specular` 97 (distance
proxies — not drawn), `specular` 43, `specular_fire`/`_tmask_fire`
(fire maps), `decal_specular`, `translucent_newclouds`,
`zprepass(_nocull)` (depth-only occluders), ~20 more.

Characters (bundle census): orc `gear_lod0/1`, `gear_alphatest_lod0/1`,
`head_lod*`, `body_lod*`, `gpucloth*`, `hair_tint_dither*`; Talion
`talion_armor/cloth/weapon`, `wraith_game/riddick/solid`; detail
arrays `detail_array_01_*` (8-slice texa, `k_fDetailTileRate0..7`);
heads add scar/displacement/wrinkle maps. Wraith (`wraith_game*`):
translucent fresnel shader — full constant-buffer map + pixel-shader
formula documented in OnSight `formats/mat.md` (VERIFIED; the wraith
needs `k_fRevealArm` driven at runtime to be visible).

## Binding chain — VERIFIED

Submesh → `.mat`:
1. **Skinned (.mesh):** `Model` record → `MaterialSets[0]` → skeleton
   LOD-group material index → `SolidMaterial`/`TranslucentMaterial`
   path (details in geometry.md).
2. **Static (.inst/.wldmsh):** descriptor `+0x18` → prefab material
   table / `.wldclnt` `.mat` list.
3. **No universal name-matching fallback** — OnSight checked: no
   gamedb attribute stores `.mesh` paths, and `.mat` paths alone
   don't bind a submesh; an earlier prefix-matching heuristic in a
   viewer (`orc_buff_body001.mesh` → `orc_body001.mat`) happened to
   work on that asset but is NOT the real binding.

The mesh/submesh → `MaterialSet` entry binding field is OPEN — the
LOD-group material index is the proven path.
