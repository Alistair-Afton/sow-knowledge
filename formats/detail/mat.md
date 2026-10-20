# `.mat` — LTMI v8 material instances

> **Status:** **verified**** — shader/texture binding decoded; lighting params partial
> **Provenance:** shader-binding + texture refs decoded on the real corpus.

A material binds a compiled shader (`shaders\shadertrees\...fxo`) to a flat
table of named, typed properties — the parameter block a shader tree reads at
draw time. This is the missing link between `.mesh` geometry and `.tex`
textures: `tDiffuseMap`/`tNormalMap`/`tSpecularMap`/`tEmissiveMap` name the
`*_d`/`*_n`/`*_s`/`*_e` texture files.

All integers little-endian.

## Header

| off | field | meaning |
|---|---|---|
| 0x00 | char[4] | `"LTMI"` |
| 0x04 | u32 | version (8 on all shipped files) |
| 0x08 | u32 | shader id (hash) |
| 0x0C | u32 | property data pool size (bytes) |
| 0x10 | u16 | property count, group 1 |
| 0x12 | u16 | property count, group 2 |
| 0x14 | u32 | sub-region size (shader data) — internal, not a trailing section |
| 0x18 | u32 | sub-region size (unknown) — internal |
| 0x1C | u16 | shader path length |
| 0x1E | char[] | shader path, e.g. `shaders\shadertrees\solid\custom\graug_disp.fxo` |

## Layout

```
header
byte[propDataSize] dataPool     // value pool; offsets are relative to here
record[count1 + count2]         // always ends exactly at EOF
```

**`record`**: `u32 type | u16 nameLen | char[nameLen] name | u32 offset`.

## Property types

| type | value | size |
|---|---|---|
| 2 | `f32[3]` — also used for `k_c*` colors | 12 |
| 3 | `f32[4]` | 16 |
| 4 | `u32` number | 4 |
| 5 | `f32` | 4 |
| 6 | `u8` boolean | 1 |
| 8 | cstring | — |
| 11 | `f32[2]` | 8 |
| 1 | `u32` — `tSourceBuffer` resource token | 4 |
| 14 | `u32` — GPU buffer binding token (cloth/particle/skinning buffers; 98 occurrences, all zero) | 4 |
| 15 | `u32` — exposure-buffer token (`ExposureBuffer_SDR/HDR`; 2 occurrences) | 4 |
| 65537 (`0x10001`) | cstring texture path | — |

Conventions: `k_f*` = float, `k_v*` = vec2, `k_c*` = color (vec3), `k_n*` =
number/float, `t*` = texture, `SurfaceRecord` (string, usually `"Default"`) is
always first.

## Verified

- All **1198** `.mat` files in the install parse cleanly.
- Record table ends exactly at EOF in every file — no trailing sections.
- Sample decode: `balrog_body.mat` — 59 props; `tDiffuseMap` →
  `characters\modelcharacters\balrog\materials\balrog_body_d.tex`, a real
  `TEXR` file the `anduril-tex` crate parses.

## Unknowns

- The `count1`/`count2` split (records are stored concatenated).
- The two u32 sub-region sizes at 0x14/0x18 — pools internal to the data
  region; not needed for decoding.
- `shader_id` hash algorithm.
- Runtime slot semantics of the type-1/14/15 resource tokens.

## Shader binding — sow-rs verified

Property order in the file maps to the shader's `cb3` constant buffer
by HLSL packing rules (verified on `wraith_game_detail.fxo`: texture
properties become `t0..t6` in file order; `k_*` properties pack into
`c0..c14` — a vector never straddles a 16-byte row). The compiled
`.fxo` is an `SRHR` container of `DX11` DXBC blobs (vs/ps pairs).

## Texture-map semantics — sow-rs verified

- `tNormalMap`: **BC3 with X in green, Y in alpha**; blue ≈ 255, red
  is a cavity-like map. Re-encode to BC5 losslessly by keeping the
  alpha block (BC4-Y) and re-encoding green→BC4-X.
- `tSpecularMap`: R ≈ specular intensity, B ≈ roughness, metallic 0
  (approximation — the game's BRDF differs).
- `tDiffuseMap` alpha on 4-layer shaders is a **height/cavity map**
  used by the layer blend.
- `tMaskMap` (tmask shaders): RGBA = blend weights for layers 0-3;
  `k_vSliceIndices` (vec4) picks the `.texa` array slices; `vmask`
  variants take weights from vertex colour instead.
- `spec2uv` meshes: uv0 = tiling layer UV, uv1 = unique mask UV;
  `k_fMapTileRate` scales the tiling rate.
- `gear_lod0`/`head_lod0` add `tDetail*` arrays
  (`detail_array_01_*.texa`, `k_fDetailTileRate0..7`); heads add
  scar/displacement/wrinkle sets; `talion_*` add `tTintMaskMap`,
  `tBloodDecalMap`, `tRevealMapArray`, `tBayerDitherMap`.
- `proxy_*` shaders (`tProxyDiffuseMap`, `*_proxd.tex`) = distance
  proxies — not drawn up close; `zonlyprepass` = depth occluders.

## Shader census (sow-rs, Minas Ithil + character bundles)

City census (6,556 placements, 277 prefabs → 728 unique materials;
all parsed). Dominant shaders:

| Shader | Mats | Slots |
|---|---:|---|
| `spec2uv_4layer_2bdns_tmask` | 251 | tDiffuse/tHeight/tMask/tNormal/tSpecular |
| `proxy_specularalphacull` + `proxy_specular` | 228 | tProxy* (distance proxies) |
| `spec2uv_4layer_2bd_tmask` | 92 | same 5 |
| `specular` | 43 | tDiffuse/tNormal/tSpecular |
| `specular_fire` (+ `_tmask_fire`) | 24 | + tFire/tFlame/tNoise |
| `spec_4layer_tmask_macron` (+ `2bd`) | 18 | + tMacroNormalMap |
| `decal_specular` | 8 | + `k_fDiffuseAlpha` |
| alphatest/backlit/lightbleed/banner/translucent/disp/vmask | ~30 | variants |
| `zprepass`/`zprepass_nocull` | 6 | none (depth occluders) |

4-layer `tmask` mechanics (52 % of region materials — most drawn
walls/floors): `tDiffuseMap`/`tNormalMap`/`tSpecularMap` are `.texa`
arrays; `k_vSliceIndices` picks 4 slices (e.g. [0,1,2,3]). `tMaskMap`
is a per-prefab BC3 `.tex` whose RGBA weight layers 0-3. `spec2uv`
verts carry two UV sets — uv0 tiling layer UV (can exceed [0,1]), uv1
unique mask UV in [0,1]; `k_fMapTileRate` scales uv0. `k_MaskBias`
(vec4), `k_fBlendSmoothness` (0.025–1.0) and `tHeightMap` (one height
per layer) drive the blend — exact formula OPEN (sow-rs uses a
height-lerp placeholder, as should we).

Character census (dcbl_darktalion + record-driven): dominant families —
`gear_lod0/1` (+`alphatest` variants: tint/blood/frost sets, lod0 adds
`tDetail*` 8-slice arrays + `k_fDetailTileRate0..7`), `head_*`/`body_*`
(+ scar maps, displacement, wrinkle sets), `talion_*` (+ `tTintMaskMap`,
`tBloodDecalMap`, `tRevealMapArray`, `tBayerDitherMap`),
`wraith_*` (`tDiffuse/tNormal/tSpecular/tRevealMapArray`), hair/fur
(`tHairAlphaMap`/`tTangentMap`/baked AO — **no normal or specular map**).

## Wraith shader (`translucent/fxville/wraith_game*.fxo`) — sow-rs verified

Per-slot `TranslucentDepthProxyMaterial` (`wraith_proxy.fxo`:
`k_fAlphaTest` 0.05, white diffuse, reveal params) is drawn first so
only the nearest wraith surface shows.

Constant buffer `cb3` = `.mat` properties in file order, HLSL-packed
(vector never straddles a 16-byte row); textures t0..t6 are texture
properties in file order. Registers (wraith_game_detail): c0
`k_bRelative`/`k_fDepthAdjust`/`k_vRevealMaskUVScale`; c1
`k_vRevealMaskUVOffset`/`k_fDetailAspect`/`k_fDetailTileRate0`; c2-3
`k_fDetailTileRate1..7`; c4/c5 fresnel near/far data; c6
`k_vFresnelGlowDistance`/`k_fFresnelGlow`; c7 `k_cFresnelGlowColor`/
`k_fAmbient`; c8/c9 `k_cAmbientLightColor`/`k_vLightVec`; c10
`k_cLightColor`/`k_fLightIntensity`; c11 reveal width/arm controls;
c12 `k_fFixedReveal_Arm`/`k_nRevealMapIndex`/`k_fDissolve`/
`k_fFresnelMin`; c13 `k_fFresnelMax`/`k_fAlphaFloor`/`k_fOpacity`/
`k_fOpacityMin`; c14.x `k_fBloomScale`.

Shader in words: detail slice = round(uv1.x·10 − 1) (below −0.5: none),
tiled by uv0 × (rate×aspect, rate). Diffuse overlay-blended with
detail diffuse. Normals = base XY (green+alpha) + detail XY, Z from
base. One fixed light (`k_vLightVec` rotated per frame from frame
constants — camera-relative), GGX + Schlick spec × light colour /
π + Lambert; ambient × `k_fAmbient` × albedo × occlusion. Fresnel:
near/far data blended by view depth over `k_vFresnelGlowDistance`;
glow = saturate((N·V − x)/(y − x))^z × `k_fFresnelGlow` — with stock
near data that's (1−N·V)^2.2, zero facing camera, full at silhouette.
Alpha = (½ luminance + sqrt(max(saturate((|N·V|−min)/(max−min)),floor)))
× reveal × opacity, ^ `k_fOpacityMin`, × diffuse alpha, −0.06.
Reveal = saturate((reveal texel + (1+width)×arm − 1)/width) — stored
arm 0 means hidden, so the game drives `k_fRevealArm` at runtime.
Second RT gets alpha×`k_fBloomScale` (bloom mask), third motion.

Shared slot values: light (0.635,0.816,0.969)×2, vector
(0.692,0.692,−0.208); ambient (0.541,0.729,0.918)×0.1; fresnel glow
0.5 in (0.710,0.855,0.976); near (1,0,2.2)/far (1,0.4,1) over 0..1000;
fresnel min 1/max 0.1; alpha floor 0.6; depth adjust 0.25; reveal
width 0.1 arm 0.

