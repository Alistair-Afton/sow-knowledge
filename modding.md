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

**Loose folders work too.** `default.archcfg` lists plain folders as
well as archives (it already has `..\Game`), searched bottom-up — a
folder line appended last (`..\MyMod`) wins over every `.arch06`
with **no archive packing at all** (verified in-game by the
Universal Modder notes; a rebuilt loose `game.gamedb` boots — the
format has no checksum). A Steam verify/repair rewrites
`default.archcfg` and silently drops appended lines — re-add them
afterwards.

**Newest `.bndl` wins.** When several archives hold the same bundle
path, the last in archcfg decides what loads — so "removing" a file
is omitting it from a rewritten `.bndl`, and "replacing" is a loose
file at the stock path plus its new size written into every `.bndl`
that lists it (this reproduces Packet Loader's remove/replace
semantics in pure data — verified in-game, no DLL).

**Overridable by this route:** any VFS file — `.tex`, `.mesh`,
`.inst`, `.mat`, `.anix`, `.gamedb`, `.bvr`, `.gfx`, `.terrain`, …

## 2. Plugin DLLs — CONFIRMED mechanism

`x64/ShadowOfWarDllLoader.dll` is the community plugin loader (a
generic `dll_loader` build from Nexus Mods): at startup it enumerates
`x64/plugins/*.dll` and `LoadLibrary`s each; every plugin may carry
a same-named `.ini` or `<name>/` folder beside it (CONFIRMED — the
loader's own strings show `.\plugins` + `LoadLibraryA/LoadLibraryExW`
only). The installed stack (Mod Menu, patches, RoamingCaptains,
Elvenrun, nemesis.lua mods) proves the model: plugins patch code,
add console commands, and run Lua against engine APIs (see Project
Nemesis for the native API surface — engine functions/offsets are
mapped there and in the Ghidra project).

## 3. Gamedb overlays — CONFIRMED data path

`game.gamedb` is the behavior database: stats, formulas, skills,
items, meters, input bindings, fight circle, HUD links, weather,
nemesis tables. Editing it changes game rules without code — and it
is *proven* writable: the Universal Modder notes rebuilt it to add a
working tenth tribe (records, roles, strdb keys; see gamedb.md for
the sort-by-hash and link-repoint rules that make rebuilds stick).
Readers exist in both projects; Andúril's writer is still OPEN.

## 4. Online services / offline testing

The game's WB/Hydra traffic rides WinHTTP — a search-order
`winhttp.dll` proxy can answer it locally (offline sign-in, local
Garrison, local wallet) with no code patching. This is the
recommended harness for testing any data mod offline: a modded
database must never be exercised while signed in to WB. Protocol
details in `engine/detail/hydra-online-services.md`.

## 5. Asset pipeline notes for would-be creators

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

## Implementations to study

Two independent Rust codebases implement readers for nearly
everything documented here (both works in progress, not yet publicly
released — cited as evidence sources):

- **Andúril** — clean-room Rust engine reimplementation:
  `anduril-vfs/-ltar/-bndl/-oodle` (asset loading),
  `-gamedb/-scri/-lua/-mods` (data + mod glue),
  `-mesh/-skel/-anix/-inst/-tex/-mat/-objr` (format readers),
  `-render/-view` (CPU rasterizer + GPU viewer),
  `-sim/-wcl/-terr` (world/simulation). Its CLI has per-format dump
  and render commands (`inst`, `mesh`, `anix`, `tex`, `mat`,
  `objres`, `wldclnt`, `sim view`).
- **sow-rs** — Rust/Bevy reimplementation by **OnSight**; his field
  notes are what this corpus draws on (all-codecs anix decode, CADB
  traversal maps, material censuses, combat/movement records).
