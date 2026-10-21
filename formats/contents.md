# File contents — what each type holds, and how much of it ships

> **Status:** **verified** — counts from a full VFS listing of the
> retail install (432,067 paths, 2026-11); content summaries point
> at the layout specs in `detail/`.

The [format index](index.md) documents *layouts*. This page is the
inventory side: how many of each type the game ships and what is
actually inside them — useful before deciding which format is worth
tooling.

## Census

Paths per extension in the mounted retail VFS:

| ext | count | what's inside |
|---|---:|---|
| `.bndl` | 192,277 | bundle manifests — a typed resource table naming the sibling `.embb` payload's members (meshes, skels, anims, soundbanks, presentation scripts). Almost every asset in the game is bndl+embb pair packaging |
| `.embb` | 190,367 | bundle payloads — the actual embedded resources listed by the `.bndl` (OBJR/EMBB objects: mesh, skel, anix, bnk, scripts, fx, …) |
| `.tex` / `.tex0` / `.texuhd` | 30,892 | TEXR textures — DDS payloads at PC (`tex`), console-legacy (`tex0`) and UHD (`texuhd`) variants; diffuse/normal/spec/mask maps |
| `.bnk` | 5,547 | Wwise soundbanks — events + embedded/loose audio media (dialogue `dx_*` lines live here, not as loose files) |
| `.anix` | 1,666 | animation clips — all codecs decoded (`detail/anix.md`) |
| `.fxp` / `.fxo` | 2,146 | particle/FX records (`fxp` = effect params, `fxo` = compiled FX objects incl. `debug*` dev set) |
| `.mat` | 1,198 | materials — shader path + typed property values, pointing at `.tex` |
| `.vegtile` / `.veggrid` / `.sndmap` | 932 | vegetation grid + scattering tiles, one soundscape map |
| `.inst` | 838 | prefabs — node tree + embedded mesh + child placements |
| `.scripts` | 835 | SCRI bytecode + prototype text — all gameplay/UI scripts (readable, no writer) |
| `.texa*` / `.texcpu` | 1,543 | texture arrays (DX10) and CPU-side texture data |
| `.vib` | 741 | vibration/rumble curves for presentations |
| `.objres` | 680 | region object/resource sections (OBJL/CADB/PFDG/GADB…) |
| `.hkai` | 444 | Havok AI navigation — navmesh data (no decode) |
| `.skel` | 381 | skeletons, bind poses, LOD groups, sockets |
| `.mesh` | 348 | skinned character/prop meshes |
| `.uiwd` / `.uis` / `.gfx` | 385 | Scaleform UI — widget defs, images, movie files |
| `.vid` | 152 | videos (`.subb` = subtitle tracks, 121 files) |
| `.cres` | 122 | cinematic resources / shadow trees |
| `.wldclnt` | 109 | region placement tables — one per world/quest state variant |
| `.pck` | 77 | streamed Wwise media packs |
| `.wldmsh` | 47 | merged region geometry |
| `.rcv` | 45 | character controller/movement params (`player_*.rcv` traversal moves) |
| `.bvr` | 29 | per-character blend/behavior databases (GADB) |
| `.strdb` | 28 | localized string tables — one per language |
| `.pmo` | 24 | photomode data |
| `.lua` | 6 | loose Lua (debug/console helpers) |
| `.gamedb` | 1 | `game.gamedb` — the 57 MB master record DB |
| `.terrain` | 1 | terrain quadtree + height tiles |
| `.arch06` | 1 | (the VFS' own container, seen via a nested mount) |
| misc | ~20 | `.cred` credits text, `.cfg`, `.txt`, one `.py`, `.off`, `disabled` stubs |

## Reading the census

- **Bundling dominates.** `bndl`+`embb` ≈ 88% of all paths — a
  bndl is never the asset itself, it's the manifest; the payload
  sits in the sibling `embb`. To touch a mesh/anim/sound you
  enumerate the bndl's resource table, then read that member out
  of the embb.
- **Voice lines are not loose files.** `dx_*` names resolve only
  inside `.bnk` event tables (see
  `gameplay/detail/audio-hooks.md`); the strdb key gives the
  subtitle, the bnk gives the media id.
- **Localization = strdb.** All UI/subtitle text is in the 28
  `string.strdb` tables — every other "string" in a path is a name,
  not text.
- **One DB to rule the spawn tables.** A single `game.gamedb`
  holds all record data; `.bvr` are the same GADB format at
  per-character scope.
- **No-write formats:** `.scripts` (no HavokScript compiler),
  `.hkai`, `.fxo` internals — read-only until a writer exists.
