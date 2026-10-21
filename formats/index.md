# Format index

> **Status:** **verified**
> **Provenance:** per-format status table — Andúril + sow-rs.

Every identified file/container type, its magic, and decode status.
"Status" = best available understanding across contributing projects
(Andúril, sow-rs);
individual semantics may still be PARTIAL inside a "verified" format.

| Extension | Magic / ver | What it is | Status |
|---|---|---|---|
| `.arch06` | `LTAR` v4 | game archive (VFS container) | CONFIRMED structure |
| `.bndl` / `.embb` | `BNDL` v6 / `EMBB` v5 | bundle manifest + payload | CONFIRMED structure |
| `.gamedb` | `GADB` | master record database (57 MB) | CONFIRMED structure; attribute names hashed |
| `.bvr` | `GADB` | per-character blend/behavior database | CONFIRMED structure; semantics PARTIAL |
| `.mesh` | `MMSH` | character/prop mesh (skinned) | CONFIRMED structure + attributes |
| `.skel` | `SKEL` | skeleton, bind pose, LOD groups, sockets | CONFIRMED (8 known bad files) |
| `.anix` | `ANIX` | animation clips (all codecs decoded) | VERIFIED |
| `.inst` | `PRFB` v8 | prefab: nodes + embedded mesh + child placements | CONFIRMED declared layout; residual flavors |
| `.wldclnt` | `WDSC` v9 | region placement table | VERIFIED |
| `.wldmsh` | `MESH` v5 | region merged geometry (same mesh block as PRFB) | VERIFIED |
| `.objres` | `OBJR` v13 | region object/resource sections (OBJL, CADB, PFDG, GADB, …) | VERIFIED section map |
| `.terrain` | `TERR` v9 | terrain quadtree + height tiles | VERIFIED structure; placement PARTIAL |
| `.veggrid` | `VEGG` v1 | vegetation grid | PARTIAL (header + path only) |
| `.mat` | `LTMI` | material: shader path + typed properties | CONFIRMED structure |
| `.tex`, `.tex0`, `.texuhd` | `TEXR` | texture (DDS payload) | CONFIRMED |
| `.texa` | (TEXR array) | DX10 array texture | VERIFIED |
| (in `.tex`) | `SLVT` v4 | container of embedded TEXR records | VERIFIED (15 files) |
| `.gfx` | `GFX`/`CFX` | Scaleform movie (SWF superset) | VERIFIED tag walk |
| `.strdb` | `SKDB` v9 | localized string table | CONFIRMED |
| `.scripts` | `SCRI` v1 | HavokScript bytecode + prototype text | CONFIRMED structure |
| `.sav` | `SOM3` (encrypted) | save data | PARTIAL (`detail/save.md`) |
| `.cred` | UTF-16LE BOM | credits text | observed (THaych) |
| `.fxp` / `.fx` | — | particle FX records | PARTIAL |
| `.cres` | — | shadow tree / cinematic resources | PARTIAL |
| `.subb`, `.vib`, `.vid`, `.uis`, `.uiwd`, `.hkai`, `.pck`, `.bnk`, `.pmo_rcv`, `.veg*, `*.ss_*` | various | Andúril has structural readers + `docs/formats/` notes | PARTIAL |

Loader addresses for the exe-verified formats are in [`../engine/engine.md`](../engine/engine.md).

## Detailed specs

Full field-level specs per format, in [`detail/`](detail/)
(provenance: the Andúril implementation's evidence docs, mirrored
here verbatim):

- Archives/containers: [`detail/arch06.md`](detail/arch06.md) (LTAR),
  [`detail/embb.md`](detail/embb.md) / [`detail/bndl.md`](detail/bndl.md),
  [`detail/slvt.md`](detail/slvt.md), [`detail/pck.md`](detail/pck.md)
- Records/DB: [`detail/bvr.md`](detail/bvr.md),
  [`detail/scri.md`](detail/scri.md), [`detail/strdb.md`](detail/strdb.md),
  [`detail/save.md`](detail/save.md), [`detail/subb.md`](detail/subb.md)
- Geometry/skeleton/animation: [`detail/mesh.md`](detail/mesh.md),
  [`detail/inst.md`](detail/inst.md), [`detail/wldmsh.md`](detail/wldmsh.md),
  [`detail/skel.md`](detail/skel.md), [`detail/anix.md`](detail/anix.md),
  [`detail/cres.md`](detail/cres.md)
- World/regions: [`detail/wldclnt.md`](detail/wldclnt.md),
  [`detail/objres.md`](detail/objres.md),
  [`detail/terrain.md`](detail/terrain.md),
  [`detail/veggrid_sndmap.md`](detail/veggrid_sndmap.md),
  [`detail/vegtile.md`](detail/vegtile.md), [`detail/hkai.md`](detail/hkai.md)
- Materials/textures/UI/AV: [`detail/mat.md`](detail/mat.md),
  [`detail/tex.md`](detail/tex.md), [`detail/gfx.md`](detail/gfx.md),
  [`detail/uis.md`](detail/uis.md), [`detail/uiwd.md`](detail/uiwd.md),
  [`detail/vid.md`](detail/vid.md), [`detail/vib.md`](detail/vib.md),
  [`detail/bnk.md`](detail/bnk.md), [`detail/pmo_rcv.md`](detail/pmo_rcv.md),
  [`detail/fx.md`](detail/fx.md)
- Script reference (script classes, event surface, event subs):
  [`../reference/script-classes.md`](../reference/script-classes.md),
  [`../reference/script-surface.md`](../reference/script-surface.md),
  [`../reference/scri_event_subs.tsv`](../reference/scri_event_subs.tsv)

## Format families

- **Containers:** LTAR archives → entries; EMBB bundles → entries;
  SLVT → TEXR records. Nested: `.inst` inside `.embb` inside `.arch06`.
- **Records:** GADB is the single typed-record format reused for the
  master DB, `.bvr` behavior DBs, and OBJR-embedded object DBs.
- **Geometry:** one vertex-declaration scheme shared by PRFB static
  meshes and MESH v5 region meshes; a different (attribute-stream)
  scheme for skinned MMSH meshes.
- **Animation:** SKEL (rig) + ANIX (clips) pair; clips reference nodes
  by the shared name hash.
