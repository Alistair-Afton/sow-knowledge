# Modding surface

> **Status:** **living**
> **Provenance:** mod load-order and Lua surface — Andúril + sow-rs evidence.

Every practical way to change the game, ranked by leverage. This
index covers the *mechanisms*; the workflow docs are:

| Doc | Contents |
|---|---|
| [`acquiring.md`](acquiring.md) | Getting files out of the install — layout, archives, extraction tooling |
| [`editing.md`](editing.md)   | Per-format edit matrix — what is writable today, with what, and the traps |
| [`packaging.md`](packaging.md) | Shipping a mod — loose folders, `.arch06`, load order, plugins |
| [`testing.md`](testing.md)   | Verifying a mod — offline services, launch checks, save safety |

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

**In-memory loader patching** (CONFIRMED by a shipped mod — THaych's
Player Skins Overhaul): a plugin hooks the GADB, SKDB and BNDL loaders
(addresses in `../runtime/engine.md`), reads the incoming stream,
patches the bytes, and hands the original loader an in-memory stream
implementing the engine stream interface. No game file is replaced,
so record-level edits coexist with archive mods: same record but
different fields → both survive; same field → the last applier wins;
a reshaped or missing record → a careful patcher refuses and the game
runs unmodded. Plugins load early enough (before the ~8–10 s database
load) for this to work.

## 3. Gamedb overlays — CONFIRMED data path

`game.gamedb` is the behavior database: stats, formulas, skills,
items, meters, input bindings, fight circle, HUD links, weather,
nemesis tables. Editing it changes game rules without code — and it
is *proven* writable: the Universal Modder notes rebuilt it to add a
working tenth tribe (records, roles, strdb keys; see ../formats/gamedb.md for
the sort-by-hash and link-repoint rules that make rebuilds stick).
Readers exist in both projects; a merged-file writer exists and a
rebuilt `game.gamedb` has booted in-game via the loose route.
The engine also natively loads a `database\game\game.gamedb_patch`
sibling (one layer deep, matched against the base header's `+0x20`
identity stamp, records appended to same-named tables; enable flag
on in retail) — the merge semantics are decoded end to end; the
practical how-to (requirements, file naming, tooling, caveats) is in
[`formats/gamedb-patch.md`](../formats/gamedb-patch.md).

## 4. Online services / offline testing

The game's WB/Hydra traffic rides WinHTTP — a search-order
`winhttp.dll` proxy can answer it locally (offline sign-in, local
Garrison, local wallet) with no code patching. This is the
recommended harness for testing any data mod offline: a modded
database must never be exercised while signed in to WB. Protocol
details in `../engine/detail/hydra-online-services.md`.

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
- **Mod archives need their own `archivepatchloosefiles.txt`**
  listing every payload path (layout in `../runtime/vfs.md`) — files not
  listed don't override, except `game.gamedb`, which is always taken
  (THaych; every working mod archive in a reference install carries
  one).
- **Never write the player's save from a buff/data mod, and never give
  a buff an `InventoryItemRemove` on a story item** — THaych lost
  `Talion_Isildur_Ring` from a save this way.
- Nexus flags script installers (`.bat`) — ship DLL + archive +
  README.

## Adding a new weapon (LordMonstey's pipeline)

Full walkthrough in LordMonstey's *sow-weapon-modding-guide*; the
engine facts are in `../gameplay/detail/inventory-loot.md`. The stages
are independent and each needs its own check:

1. **Geometry** — fit to a native template of the same route (rig,
   grip frame, binding), export MMSH (Monstey Mesh Publisher), recompute
   the mesh allocation counters.
2. **Materials/textures** — rebuild native `.mat`/`.tex` for the
   chosen shader; Blender PBR doesn't convert.
3. **Resource closure** — every path the item references resolves
   through the bundle the item actually loads (`DynamicBundle`).
4. **Item graph** — clone a same-slot, same-rarity loot donor's
   records; add to the donor's `Combat.PredicateItem` lists; string
   keys in every shipped locale.
5. **Loot graph** — clone a `NemesisGear`, attach via a gear list;
   check tags, upgrade-tier art overrides and how widely the list is
   shared.
6. **Packaging** — archive + `archivepatchloosefiles.txt`
   registration, archcfg line.

**Composition rules** (LordMonstey):

- A delta must name the exact **parent DB hash** it extends and refuse
  a different parent — rebuilding a modded DB from a clean one drops
  earlier mods' items, strings and loot edits.
- Write each changed list as its complete final value against the
  current parent: keep foreign members in order, replace only your own
  previous entries. Sequential full-list patches from separate
  components silently drop each other's additions.
- Two components changing the same locale table (or the same
  skeleton's animation-library registrations) must be merged into one
  file — two copies at one VFS path don't combine.
- A stale loose overlay can win over a correct new archive; record the
  intended winning bytes and check what is actually mounted.
- Test a fresh natural drop, equip/sheathe/attack, spectral mode, a
  second instance and a full save → restart → reload separately; a
  controlled grant proves none of the loot path.

## Known dead ends (THaych — save the time)

- Plain loose files in the install root are ignored; directory
  sources must be listed in `default.archcfg` (or mounted at runtime).
- Changing a skin's `PlayerID` switches the voice only; changing its
  body type swaps **all** gear (see `../gameplay/detail/player-skins.md`).
- A per-skin hand ring can't be done in data (ring pickers exist only
  for body types 6 and 8; slot-visibility actions hide whole groups).
- Editing a shared vec2 pool constant changes every menu using it —
  give the record its own copy.
- Detouring Wwise `GetIDFromString` breaks callers (they keep live
  values in `r9`–`r11`).
- A global audio mix set at spawn is overwritten by loading hooks and
  menu preview characters — re-assert it on the gameplay event.
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
- **Player Skins Overhaul** (THaych, Nexus) — reference
  implementation of in-memory GADB/SKDB/BNDL patching from a CRT-free
  plugin DLL (symbolic patch keyed by category + record name, refuses
  on unexpected shapes).
- **SoW Community Edition** — another plugin framework
  (`x64\plugins\sowce_bootstrap.dll` + `Mods\*.hotswap` folders);
  coexists with loader-hooking plugins per THaych. Its `.hotswap`
  mechanism is undocumented here (OPEN).
- **sow-rs** — Rust/Bevy reimplementation by **OnSight**; his field
  notes are what this corpus draws on (all-codecs anix decode, CADB
  traversal maps, material censuses, combat/movement records).
