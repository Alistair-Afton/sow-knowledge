# Acquiring the game's files

> **Status:** **living**
> **Provenance:** install layout and extraction verified on the
> reference install; tool list from `../sources.md`.

Where everything lives and how to get it out.

## Install layout

| Path | Contents |
|---|---|
| `<install>\*.arch06` | Data archives (LTAR v4). Stock files ship here; extra loose `.arch06` files dropped in the root are mods. |
| `<install>\x64\` | `ShadowOfWar.exe`, `oo2core_5_win64.dll` (Oodle), `ShadowOfWarDllLoader.dll` (plugin loader), `default.archcfg` (mount list — **bottom-up** precedence), `plugins\`. |
| `<install>\game\` | Engine config (`SKU.cfg`, `projectconsole.cfg`), loose cinematic/resource files. |
| `<install>\Mods\` | Mod payloads: `.arch06` archives and `<name>\` mod folders. |
| `Steam\userdata\<acct>\356190\remote\` | Save files (encrypted; also mirrored to `%LOCALAPPDATA%\WB Games\Shadow of War\`). |

Key data paths *inside* the VFS:

| VFS path | Contents |
|---|---|
| `database\game\game.gamedb` | The behavior database — most gameplay rules (GADB, ~57 MB) |
| `database\game\*.record06`, `*.strdb` | Records and string tables |
| `worlds\retail\<region>\*` | Per-region bundles: terrain, placements, lighting, scripts |
| `bundles\modelcharacters\*` | Character `.bndl` (mesh+skel+tex+mat) |
| `sound\<lang>\*` | Wwise `.pck`/`.bnk` audio |
| `behaviors\*` | `.bvr` behavior DBs, `.pmo`/`.rcv` anim metadata |

## Getting bytes out

**Rule: never extract by modifying stock files.** Read them in
place or copy bytes out; the install stays pristine so a vanilla
compare is always available.

Archives are **LTAR v4** with **Oodle**-compressed chunks — any
extractor needs an Oodle decode; the install's own
`x64\oo2core_5_win64.dll` provides it (use in place, don't
redistribute).

Working extraction routes:

- **QuickBMS** + `shadow_of_mordor.bms` v0.4 — the community
  extractor; unpacks `.arch06` to loose files.
- **SoW Modding Community ImHex-Patterns / MeshParser** — binary
  templates + mesh-level extraction.
- **WarpZephyr's MordorFormats** — reference parser code.
- **Roll your own** — the LTAR format is fully spec'd in
  `../formats/detail/arch06.md`; a reader is ~150 lines plus an
  Oodle binding.
- **Andúril CLI** (the engine-reimpl project behind this corpus) —
  `ls`/`cata`/`read`/`extract` traverse the mounted VFS read-only
  and decode `game.gamedb`/`.strdb` to JSONL/TSV; the Andúril source
  is a readable reference even if you don't run it.
- **Hex inspection** — ImHex with the community patterns for reading
  structures by hand.

## What needs which tool

| Want | Get it via |
|---|---|
| A texture/mesh/sound file | QuickBMS or any LTAR reader → then the format's own tools (`editing.md`) |
| Gameplay values (stats, loot, nemesis tables) | `game.gamedb` out of an archive, then any GADB reader (spec: `../formats/gamedb.md`) |
| Localized text | `string.strdb` + `../formats/detail/strdb.md` (~60-line parser) |
| Per-region world data | the region's archive — inventory with the LTAR reader, then pull per file |
| A save file | copy from `userdata` / `%LOCALAPPDATA%`; encryption in `../formats/detail/save.md` |
| The original (unpatched) version of a file | read it from the *stock* archive, not the topmost VFS layer |

## Notes

- VFS layering matters: the same path exists in several archives and
  in `Mods\` — the bottom entry in `default.archcfg` wins. Always
  check which copy you extracted.
- Oodle is proprietary: call the install's DLL at runtime, never
  ship it.

Next step: [`authoring.md`](authoring.md) — turning what you
extracted into something editable and back into what the game loads.
