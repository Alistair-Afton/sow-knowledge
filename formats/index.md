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
| `.bndl` / `.embb` | EMBB v5 | bundle manifest + payload | CONFIRMED structure |
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
| `.strdb` | — | localized string table | CONFIRMED (Andúril) |
| `.scri` | — | script/quest data | CONFIRMED (Andúril `docs/formats/scri.md`) |
| `.save` files | — | save data | PARTIAL (Andúril `docs/formats/save.md`) |
| `.fxp` / `.fx` | — | particle FX records | PARTIAL |
| `.cres` | — | shadow tree / cinematic resources | PARTIAL |
| `.subb`, `.vib`, `.vid`, `.uis`, `.uiwd`, `.hkai`, `.pck`, `.bnk`, `.pmo_rcv`, `.veg*, `*.ss_*` | various | Andúril has structural readers + `docs/formats/` notes | PARTIAL |

Loader addresses for the exe-verified formats are in `engine.md`.

## Detailed specs

Full field-level specs per format (provenance: the Andúril
implementation's evidence docs, mirrored here verbatim):

- Archives/containers: [`arch06.md`](arch06.md) (LTAR),
  [`embb.md`](embb.md) / [`bndl.md`](bndl.md), [`slvt.md`](slvt.md),
  [`pck.md`](pck.md)
- Records/DB: [`gamedb.md`](gamedb.md) (digest), [`bvr.md`](bvr.md),
  [`scri.md`](scri.md), [`strdb.md`](strdb.md), [`save.md`](save.md),
  [`subb.md`](subb.md)
- Geometry/skeleton/animation: [`geometry.md`](geometry.md) (digest),
  [`mesh.md`](mesh.md), [`inst.md`](inst.md), [`wldmsh.md`](wldmsh.md),
  [`skel.md`](skel.md), [`anix.md`](anix.md), [`cres.md`](cres.md)
- World/regions: [`world.md`](world.md) (digest),
  [`wldclnt.md`](wldclnt.md), [`objres.md`](objres.md),
  [`terrain.md`](terrain.md), [`veggrid_sndmap.md`](veggrid_sndmap.md),
  [`vegtile.md`](vegtile.md), [`hkai.md`](hkai.md)
- Materials/textures/UI/AV: [`materials.md`](materials.md) (digest),
  [`mat.md`](mat.md), [`tex.md`](tex.md), [`gfx.md`](gfx.md),
  [`uis.md`](uis.md), [`uiwd.md`](uiwd.md), [`vid.md`](vid.md),
  [`vib.md`](vib.md), [`bnk.md`](bnk.md), [`pmo_rcv.md`](pmo_rcv.md),
  [`fx.md`](fx.md)
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
