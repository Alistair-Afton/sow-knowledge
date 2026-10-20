# Format index

> **Status:** **verified**
> **Provenance:** per-format status table — Andúril + sow-rs.

Every identified file/container type, its magic, and decode status.
"Status" = best available understanding across Andúril + sow-rs;
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
| `.strdb` | — | localized string table | CONFIRMED (ours) |
| `.scri` | — | script/quest data | CONFIRMED (ours, see docs/formats/scri.md) |
| `.save` files | — | save data | PARTIAL (see docs/formats/save.md) |
| `.fxp` / `.fx` | — | particle FX records | PARTIAL |
| `.cres` | — | shadow tree / cinematic resources | PARTIAL |
| `.subb`, `.vib`, `.vid`, `.uis`, `.uiwd`, `.hkai`, `.pck`, `.bnk`, `.pmo_rcv`, `.veg*, `*.ss_*` | various | see `docs/formats/` — Anduril has structural readers | PARTIAL |

Loader addresses for the exe-verified formats are in `engine.md`.

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
