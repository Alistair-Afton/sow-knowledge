# Acquiring the game's files

> **Status:** **living**
> **Provenance:** install layout and extraction verified on the
> reference install; tool list from `sources.md`.

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

- **Andúril CLI** (`anduril`): mounts the install read-only through
  its own LTAR/Oodle implementation.
  - `anduril ls <filter>` — find VFS paths by substring
  - `anduril read <vfs-path> > file` — extract one file through the
    mount stack (gets the *winning* version)
  - `anduril cata <archive> <path> > file` — extract from one
    specific archive (pre-override copy)
  - `anduril extract <filter> <dir>` — bulk-extract every VFS path
    containing `<filter>`
  - `anduril db emit <out.jsonl>` / `anduril db rec <name>` /
    `anduril db refs <name>` — decoded `game.gamedb` dumps
  - `anduril str emit <out.tsv>` — decoded string table
- **QuickBMS** + `shadow_of_mordor.bms` v0.4 — community extractor
  for `.arch06` (uses the install's own `oo2core_5_win64.dll`).
- **WarpZephyr's MordorFormats** — reference parser code.
- **SoW Modding Community ImHex-Patterns / MeshParser** — binary
  templates + mesh extraction.
- **Hex/editor**: `ImHex` with the community patterns for inspecting
  structures by hand.

## What needs which tool

| Want | Get it via |
|---|---|
| A texture/mesh/sound file | `anduril read`/`extract`, or QuickBMS |
| Gameplay values (stats, loot, nemesis tables) | `anduril db rec` / `db emit` on `game.gamedb` |
| Localized text | `anduril str emit` on `string.strdb` |
| Per-region world data | `anduril lsa <region archive>` to inventory, then `cata`/`extract` |
| A save file | copy from `userdata` / `%LOCALAPPDATA%` (see `formats/detail/save.md` for the encryption) |
| The original (unpatched) version of a file | `anduril cata` against the *stock* archive, not the VFS |

## Notes

- Oodle decompression uses the install's own
  `x64\oo2core_5_win64.dll` — every working tool either loads that
  DLL or bundles an equivalent. The DLL is proprietary; use it in
  place, don't redistribute it.
- VFS layering matters: the same path exists in several archives
  and in `Mods\` — the bottom entry in `default.archcfg` wins.
  Always check which copy you extracted (`anduril mounts` shows the
  order).
