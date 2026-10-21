# `game.gamedb_patch` — the engine's built-in database patch mechanism

> **Status:** verified (loader decode); loose-file patch placement UNVERIFIED
> **Provenance:** Ghidra decompiles of the retail binary
> (`FUN_140503694`, `FUN_1417662b0`, `FUN_1404b9b50`, `FUN_1404ba07c`,
> `FUN_141766594`, `FUN_1404ba1f0`); in-game test of the merged-file
> route (2025-10-11).

The game binary itself can merge a second database onto
`game.gamedb` at load. This page is everything needed to use it.

## What it is

- The engine's whole-file loader checks the path of every file it
  loads. If the file's extension is **not** `gamedb_patch`, it tries
  to open the same-named sibling with the extension swapped to
  `.gamedb_patch` and loads that too, linking it as a second layer.
- A `.gamedb_patch` file never searches for its own patch. The chain
  is exactly **one layer deep**: base `.gamedb` → `.gamedb_patch`.
  There is no patch-of-patch stacking.
- The patch loader is **enabled in retail** (the enable flag is
  statically `1`).

## Requirements a patch file must satisfy

1. **Same GADB format.** A `.gamedb_patch` is a normal GADB v17 file
   — `GADB` magic, version `0x11`, the 0x2C header, string pool,
   15-byte layout table, category/record data. Full byte layout:
   `gamedb.md`.
2. **File name and location.** It must sit where the *winning* base
   `game.gamedb` resolves, with the same base name:
   `database\game\game.gamedb_patch`. The patch lookup is done on the
   resolved self path, so whichever VFS layer wins for
   `game.gamedb` determines the directory that is checked for the
   sibling.
3. **Identity match — the only content check.** The loader compares
   the patch's own header field (verified at patch `+0x2C` in the
   decode) against the base file's **`+0x20` stamp**
   (`0xD186F247` on the installed build). `+0x20` is an identity
   stamp, **not** a checksum recomputed over content — a rewritten
   `game.gamedb` with the same `+0x20` still loads, so a patch only
   needs to carry the stamp of the base it extends. Mismatch → the
   patch is rejected (silently; the game runs on the base alone).

## What merging does

- The body is table-driven. Each table header in the patch is
  resolved by its name hash against the base's tables.
- **Matching tables get the patch's records appended** to the base
  table's record list. Merge key = **table (category) name**.
- Pointer-typed field cells and named-record name offsets are
  rebased against each file's own blob, so the patch file is fully
  self-contained — you do not share string pools with the base.
- Net effect: a patch **adds records to existing categories** (and
  can add categories). It is an additive layer; overriding an
  existing named record is not a documented semantic — a patch record
  sharing a base record's name has UNVERIFIED winner semantics
  (named lookup binary-searches by name hash; which of two same-name
  records wins has not been tested).

## How to actually produce and apply it

You need a tool that can *write* GADB v17, not just read it:

- **Option A — patch file.** Write a GADB containing only your new
  records, laid out in tables whose names match the base categories
  you are extending, with `+0x2C`/`+0x20` lineage set to the base's
  `+0x20` stamp. Ship it at `database\game\game.gamedb_patch` inside
  an `.arch06` (or loose at the resolved base path — loose patch
  pickup is UNVERIFIED).
- **Option B — merged replacement (proven in-game).** Merge your
  changes into a full `game.gamedb` offline and ship the merged file
  at `Mods\database\game\game.gamedb` (loose file, no archcfg line
  needed — `game.gamedb` is always taken from the loose layer).
  Booted title → menu → in-world on the reference install. This
  route needs no lineage stamp and is strictly simpler; the patch
  mechanism is optional, not required.
- **Option C — in-memory.** A plugin can patch the loaded DB
  directly (the record manager supports live `setcell`-style
  writes), bypassing file formats entirely.

Tooling: no public `.gamedb_patch` emitter exists today — the only
writer that has produced a game-accepted GADB is the Andúril
project's `db write` (merged replacement, not a patch file), and
Andúril is not publicly released. Building a writer from the spec
in `gamedb.md` is viable — the format is byte-for-byte decoded
(232,319 records round-trip) and a minimal emitter is a few hundred
lines; `../modding/authoring.md` has the recipe.

## Checklist

1. Read the base's `+0x20` stamp (`0xD186F247` on the retail install
   — verify on your copy).
2. Emit a GADB v17 file containing your added records in correctly
   named tables; set the lineage field to the base's stamp.
3. Name it `game.gamedb_patch` and place it beside the winning
   `game.gamedb` (archive payload, or loose — see caveats).
4. Boot the game; if the patch is rejected the game silently runs
   unpatched — verify a known added record took effect.

## Caveats

- **No patch chaining.** Only one `.gamedb_patch` layer loads. Two
  mods each shipping a patch for the same DB cannot coexist at the
  same path; merge them into one file.
- **Loose vs archive.** The loose-file index
  (`archivepatchloosefiles.txt`) governs which paths the loose layer
  serves; `game.gamedb` is special-cased but `game.gamedb_patch` is
  not confirmed in it — shipping the patch inside a mounted
  `.arch06` is the safer placement until verified.
- **Edits are additive.** To *change* an existing record's fields,
  the reliable route is a merged full replacement (Option B) or
  in-memory patching (Option C); same-name patch records are not a
  confirmed override mechanism.
- **Test offline.** A modded database should never be exercised
  while signed into WB services (see `../modding/testing.md`).
