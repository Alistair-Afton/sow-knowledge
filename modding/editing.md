# Editing by format

> **Status:** **living**
> **Provenance:** writability column = shipped-mod or round-trip
> evidence; tool column = known working path.

What can be changed, with what, and what breaks. Byte-level specs
are in `../formats/`; this page is the practical matrix.

| Format | Writable? | How | Traps |
|---|---|---|---|
| `game.gamedb` (GADB) | **Yes** — proven in-game | Full rebuild (merged file via `Mods\database\game\game.gamedb`) or `.gamedb_patch` sibling or in-memory plugin patching | Named records must stay `name_hash`-sorted; link cells are `class<<20\|ord` indices that must be repointed on rebuild; see `../formats/gamedb.md` + `../formats/gamedb-patch.md` |
| `.strdb` strings | **Yes** — proven in-game | Rebuild merged table, serve loose from `Mods\` | String keys are name hashes — new keys need matching `name_hash` |
| `.tex` textures | **Yes** | DDS payload in TEXR container; author DDS (BC1/BC3/BC5/BC6H) and repack | Normal maps are BC3 **X-green/Y-alpha**; `.texa` arrays selected by material |
| `.mat` materials | **Yes** — simple typed properties | Text/struct edit; point at your `.tex` | Shader selection is fixed — parameters only |
| `.mesh` skinned meshes | **Yes** | Fit to a native template (rig, grip, binding); Monstey Mesh Publisher exports MMSH | Mesh allocation counters must be recomputed |
| `.skel` skeletons | Partly | Readable; retargeting edits reported | Writing new skeletons is OPEN |
| `.anix` animations | Partly | All codecs decoded; **U\*** uncompressed codecs are the simplest emitters | ~6 rare codecs undecoded; authoring toolchain is immature |
| `.bndl` bundles | **Yes** | Rebuild manifest+payload; "remove" = omit from a rewritten `.bndl`; "replace" = loose file + size update in every `.bndl` listing it | Newest `.bndl` wins by archcfg order; region bundles disagree with some DLC manifests — index by walking `u32 size, data` records, not just the manifest |
| `.wldclnt` placements | **Yes** — 76-byte records | Binary edit per `../formats/detail/wldclnt.md` | Region coherence needs `.objres` + lighting sections too |
| `.gfx` HUD (Scaleform AS2) | Partly | SWF tooling + external-image convention | Widget placement is engine-side (OPEN) |
| `.bvr` behavior DBs | **Yes** — same GADB format | Same as `game.gamedb` | Per-character scope |
| `.fx` shaders | No | DXBC blobs | No compile path known (OPEN) |
| `.terrain`, `.inst`, `.objres` | Readable | Full specs exist | Writing not yet exercised end-to-end |
| `.sav` saves | Round-trips byte-for-byte | Decrypt → edit chunks → re-encrypt (`formats/detail/save.md`) | Every size field must stay consistent; **never have a mod write the save** — see `testing.md` |

## Rules of thumb

- **Data before assets**: most rule changes are `game.gamedb`
  edits, not file edits — check the record graph first
  (`anduril db rec`, `db refs`, `db emit`).
- **Clone a donor**: new content is made by cloning a working
  record/item of the same class, not by building from scratch —
  see the weapon pipeline in `index.md`.
- **Determinism traps**: declared sizes/counts are checked by
  loaders; a file that "looks right" but lies about a count crashes
  or is ignored.
