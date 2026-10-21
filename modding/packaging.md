# Packaging a mod

> **Status:** **living**
> **Provenance:** every route below is observed on the reference
> install or confirmed by a shipped mod.

Three delivery routes, weakest to strongest. All share one rule:
**never modify stock archives or the exe** — always override.

## Route 1 — loose folder (no packing)

`default.archcfg` lists plain folders as well as archives, searched
bottom-up — a folder line appended last wins over every `.arch06`.

```
x64\default.archcfg  →  append:  ..\MyMod
MyMod\database\game\game.gamedb   (or any VFS-path mirror)
```

- Verified in-game (Universal Modder notes; a rebuilt loose
  `game.gamedb` boots).
- **A Steam verify/repair rewrites `default.archcfg` and silently
  drops appended lines — re-add them afterwards.**
- Best for iteration: edit files directly, no packing step.

## Route 2 — `.arch06` archive

Drop the archive in the install root (mounts after every archcfg
entry) or under `Mods\`.

- Payload must be LTAR v4 — needs an LTAR writer (the game's chunked
  layout; Oodle via the install's own `oo2core_5_win64.dll`).
- **`archivepatchloosefiles.txt`**: every payload path must be
  listed, or the loose-file index won't serve it. Exception:
  `game.gamedb` is always taken.
- Newest `.bndl` wins for shared paths — see `index.md` §1 for the
  remove/replace semantics.

## Route 3 — plugin DLL

`x64\plugins\<name>.dll` (+ optional `<name>.ini` / `<name>\`
folder) — `ShadowOfWarDllLoader.dll` `LoadLibrary`s each at startup.

- Plugins patch code, add console commands, run Lua, or **hook the
  file loaders and patch bytes in memory** (THaych's Player Skins
  Overhaul is the reference: symbolic GADB/SKDB/BNDL patch, refuses
  on unexpected shapes — coexists with archive mods because no file
  is replaced).
- Plugins load early enough to intercept the ~8–10 s database load.

## `game.gamedb` specifics

- Loose replacement: `Mods\database\game\game.gamedb` — proven
  in-game (title → menu → world, 2025-10-11).
- Native patch layer: `database\game\game.gamedb_patch` sibling —
  decoded, see `../formats/gamedb-patch.md`.
- In-memory: hook the loaders (route 3).

## Composition rules (LordMonstey)

- A delta names the exact **parent DB hash** it extends and refuses
  a different parent — rebuilding a modded DB from clean drops
  earlier mods' items/strings/loot edits.
- Write each changed list as its complete final value; keep foreign
  members in order; replace only your own previous entries.
- Two components editing the same locale table must be merged into
  one file — two copies at one VFS path don't combine.
- A stale loose overlay can win over a correct new archive — record
  the intended winning bytes and check what actually mounted.

## Naming

VFS paths hash **case-insensitively with backslashes** — payload
files must land under the exact VFS path they replace.
