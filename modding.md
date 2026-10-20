# Modding surface

> **Status:** **living**
> **Provenance:** mod load-order and Lua surface — Andúril + sow-rs evidence.

Every practical way to change the game, ranked by leverage.

## 1. Loose archive mods — CONFIRMED mechanism

Drop a `.arch06` in the install root → it mounts after all entries
of `x64/default.archcfg` (bottom-up precedence) → its files override
stock ones by path. This is how every shipped asset mod works
(texture swaps, region overrides). To author one you need an LTAR
writer producing the game's chunked layout (tools exist
community-side; Andúril has a reader, not yet a writer).

**Overridable by this route:** any VFS file — `.tex`, `.mesh`,
`.inst`, `.mat`, `.anix`, `.gamedb`, `.bvr`, `.gfx`, `.terrain`, …

## 2. Plugin DLLs — CONFIRMED mechanism

`ShadowOfWarDllLoader.dll` loads `x64/plugins/*.dll` at startup
(each may carry `<name>.ini` / `<name>/`). The installed stack
(Mod Menu, patches, RoamingCaptains, Elvenrun, nemesis.lua mods)
proves the model: plugins patch code, add console commands, and run
Lua against engine APIs (see Project Nemesis for the native API
surface — engine functions/offsets are mapped there and in the
Ghidra project).

## 3. Gamedb overlays — CONFIRMED data path

`game.gamedb` is the behavior database: stats, formulas, skills,
items, meters, input bindings, fight circle, HUD links, weather,
nemesis tables. Editing it changes game rules without code. Needs a
GADB writer preserving layouts/links/string heap (OPEN tooling;
readers exist in both projects).

## 4. Asset pipeline notes for would-be creators

- **Textures:** DDS payloads inside TEXR; BC1/BC3/BC5/BC6H used;
  normal maps are BC3 **X-green/Y-alpha** (see materials.md).
  `.texa` = DX10 arrays; slice selection is a material property.
- **Meshes:** skinned content = MMSH + SKEL pair; statics = the
  PRFB/MESH declared scheme; both fully documented in geometry.md.
  Custom geometry is feasible — formats are writable.
- **Animations:** all ANIX codecs documented; writing new clips is
  feasible (U* uncompressed codecs are the simplest emitters).
- **Materials:** `.mat` is a simple typed-property file — texture
  swaps and shader-parameter tweaks are easy; new shaders need
  `.fxo` compile (engine-native format, OPEN).
- **HUD/UI:** Scaleform AS2 movies — authorable with SWF tooling +
  the game's external-image convention; widget placement is
  engine-side (OPEN).
- **World edits:** placements = `.wldclnt` records (76 B, documented);
  regions also need `.objres` (objects, nav `PFDG`, context edges
  `CADB`) and lighting sections for a coherent result.

## Caveats

- **Never modify stock archives or the exe** in place — write
  override archives / rebuild copies.
- **Oodle** is proprietary: use the install's `oo2core_5_win64.dll`
  at runtime; do not redistribute it.
- **Naming:** VFS paths hash case-insensitively with backslashes;
  content must land under the exact VFS path it replaces.
- **Determinism traps:** formats carry counts/sizes that must stay
  consistent (declared byte sizes are checked by loaders);
  GADB link indices must stay valid.
- Region bundles disagree with some DLC-era manifests — index by
  walking `u32 size, data` records, not just the manifest
  (VERIFIED pitfall).
- Existing health check: `x64/modmenulog.txt` after a launch should
  show `Injected successfully!` + all offsets resolved.

## Where to look in this repo

- `crates/anduril-vfs`, `-ltar`, `-bndl`, `-oodle` — asset loading
- `crates/anduril-gamedb`, `-scri`, `-lua`, `-mods` — data + mod glue
- `crates/anduril-mesh/-skel/-anix/-inst/-tex/-mat` — format readers
- `crates/anduril-render`, `-view` — CPU rasterizer + GPU viewer
- `crates/anduril-sim`, `-wcl`, `-terr`, `-ui` — world/simulation
- `anduril <cmd>` CLI: `info`, `mounts`, `gamedb`, `inst`, `mesh`,
  `anix`, `tex`, `mat`, `wldclnt`, `sim view`, `mesh view`
