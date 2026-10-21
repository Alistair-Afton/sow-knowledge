# Authoring & round-tripping files

> **Status:** **living**
> **Provenance:** commands verified on the reference install;
> community tools per `../sources.md`.

Creating a file is always three steps: **out of the binary** →
**edit in a sane form** → **back into the binary layout the game
loads**. This page is the per-format recipe. Nothing here requires a
specific toolkit — wherever a tool is named, any equivalent that
speaks the format works; the byte-level specs in `../formats/` are
complete enough to build your own.

## The golden rule

You almost never hand-edit the game's binary formats. Each has (or
needs) a **decoded intermediate form** — a text dump, TOML deltas,
DDS, OBJ — that you edit, then a tool re-emits the binary. Formats
with no working writer are marked **NO WRITE PATH**: for those you
build the emitter yourself from the spec doc — all of them are small
enough that a writer is a script, not a research project.

## `game.gamedb` — doing it without going insane

The database is the worst format to hand-edit and the most valuable.
Two practical routes:

### Route A — in-memory patching (no file format at all)

A plugin DLL hooks the GADB loader, patches records as they stream
in, and hands the engine an in-memory stream. **This is the route a
shipped mod already uses** (THaych's Player Skins Overhaul): the
patch is keyed symbolically (category + record name + expected
field), and a careful patcher *refuses* when a record shape is
unexpected — so a bad patch degrades to "game runs unmodded", never
a corrupt DB. See `../formats/gamedb.md` for the loader hook
addresses and `packaging.md` route 3 for how plugins load.

To write one you need: a minimal DLL, the loader address for your
build, and the record accessor the binary exposes for "find record
by name hash, write cell N". That's ~200 lines of C++ once the
addresses are known — they're mapped in `../runtime/engine.md`.

### Route B — a rewritten `.gamedb` file

Editing a record = rewriting the file. A GADB writer must:

1. Keep **named records sorted by `name_hash`** — the engine
   binary-searches them.
2. **Repoint every link cell** (`class_index << 20 | ord`) into the
   new index space — links are positional, not names.
3. Rebuild the string pool (suffix-deduplicated, offsets relative
   to pool start, pointer 0 lands `0x1B` in) and patch the header
   counts.

None of that is guesswork — the full byte layout is in
`../formats/gamedb.md`, verified field-for-field on 232,319 records.
A minimal writer in Python is a few hundred lines; the known
pitfalls are all documented there (sort order, link repointing, the
`0x47` pool base, the self-path at `+0x2C`).

Then ship it as `Mods\database\game\game.gamedb` (loose override —
proven in-game) or inside an `.arch06`. The native `.gamedb_patch`
layer works too but needs the same writer plus the lineage stamp —
see `../formats/gamedb-patch.md`.

## Strings (`.strdb`)

- Extract/decode with any reader built to `../formats/detail/strdb.md`
  (hash → UTF-8 text, ~60-line parser).
- Edits = add/replace/delete keyed by `name_hash` of the string key.
- Rebuild the table and ship loose from `Mods\`.
- A merged strdb has been rendered in-game (verified 2026-09-24).

## Textures (`.tex`)

- The TEXR container is a small header around a standard **DDS**
  payload — extract the payload with a hex editor or 30-line script
  (offsets in `../formats/detail/tex.md`), edit in any DDS-capable
  tool (Paint.net, GIMP+DDS, texconv), re-wrap into TEXR.
- Formats: BC1/BC3/BC5/BC6H; normal maps are **BC3 X-green/Y-alpha**.
- `.texa` = DX10 texture arrays; slice choice is a material property.

## Meshes & skeletons (`.mesh`, `.skel`)

- Out: the SoW Modding Community **MeshParser** /
  ImHex patterns decode MMSH to editable geometry; spec in
  `../formats/detail/mesh.md`.
- Edit: Blender/Max **against a native template** — same rig, same
  grip frame, same binding; custom skeletons are not yet writable.
- Back: **Monstey Mesh Publisher** exports MMSH; the mesh allocation
  counters must be recomputed (documented in the spec).

## Animations (`.anix`)

- Decode: all codecs are documented (`../formats/detail/anix.md` —
  one rare codec is undecoded community-wide).
- Write: emit a clip in one of the **`U*`** uncompressed codecs —
  simplest possible track format. No turnkey exporter exists; this
  is a build-your-own-writer format.

## Materials (`.mat`)

- Simple typed-property file (`../formats/detail/mat.md`) — editable
  directly. Use it to point meshes at your rebuilt `.tex` or tune
  shader params. New *shaders* are out (`.fxo` is compiled DXBC —
  no compile path).

## Audio (`.bnk`, `.pck`)

- Wwise containers. Media payloads extract/replace with standard
  Wwise tooling (wwiser, bnk_extr, etc.); rebuilds need a matching
  Wwise version (PARTIAL — verify against the game's banks).

## Placements & world (`.wldclnt`, `.objres`, `.bndl`)

- `.wldclnt` records are 76-byte typed structs — directly editable
  per `../formats/detail/wldclnt.md`.
- `.bndl` manifests rebuild cleanly; "remove" = omit from a rebuilt
  manifest, "replace" = loose file + size fix in every listing
  (semantics in `index.md` §1).

## Saves (`.sav`)

- SOM3-encrypted; decrypt/encrypt round-trips byte-for-byte
  (key schedule + chunk layout in `../formats/detail/save.md`).
- Useful for *inspection* and save surgery. A gameplay mod should
  **never** write saves — see `testing.md`.

## No write path (yet)

`.fxo`/`.fx` shaders, `.scri` script bytecode, `.hkai` navdata:
readers exist, writers don't. The practical "edit" is a plugin hook
or a neighboring data record that redirects the reference.

## The general recipe (any format)

1. Read `../formats/detail/<fmt>.md` — layouts are byte-verified.
2. Extract samples (QuickBMS `shadow_of_mordor.bms` for `.arch06`,
   or any LTAR reader built from `detail/arch06.md`; Oodle via the
   install's own `oo2core_5_win64.dll`).
3. Make the smallest possible change — one field, one record — with
   a hex editor or a short script.
4. Boot offline (`testing.md`) and verify it took. A malformed file
   usually loads *unpatched* rather than crashing — confirm the
   change is actually live before iterating.
5. Once one edit round-trips, automate — that's how every working
   tool in `../sources.md` started.
