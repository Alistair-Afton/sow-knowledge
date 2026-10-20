# Spec: GameDB delta / overlay merge semantics

> **Status:** verified — 232,319/232,319 records field-for-field vs
> **Provenance:** `anduril-gamedb` delta layer
> (`crates/anduril-gamedb/src/delta.rs`), merged-overlay serialization
> validated against real `database/game/game.gamedb`; merge notes in
> `../../formats/detail/gamedb.md`. Consolidated 2026-10-19.

Mods share `game.gamedb` without shadowing whole files: each mod
contributes **record deltas** applied in load order over the base db,
producing a `MergedDb` view. **VERIFIED.**

## Delta model

- A delta op targets one record and either `set`s a cell, `add`s a
  record, or `delete`s one. Record identity is `"List/Name"`, or a
  bare `"Name"` resolved within the delta's own list. **VERIFIED.**
- `field` may be a named layout attribute or a dotted/raw cell index
  (`setcell` escape hatch). Strings intern into the merged view's
  added-string table, marked `0xEE00_0000 | index` in cells.
  **VERIFIED.**
- Conflicts: two deltas writing the same cell — the later (stronger)
  mod's value stands; the loser is *reported*, not dropped silently.
  **VERIFIED.**

## Ordering

`OverlayDb::push` is called in mod load order (weakest first) so that
merge-time, last-write-wins matches VFS strongest-layer semantics —
the same mod ordering drives both file and record precedence.
**VERIFIED.**

## Serialization

The merged overlay serializes back into a valid `.gamedb`. Known
residue: `namedCount`/`unnamedCount` headers are off by a small fixed
delta vs naive counts (documented in `../../formats/detail/gamedb.md`);
`ADDED_STR` markers become real interned strings. **VERIFIED** for the
fields our writer emits; the counter quirk is `INFERRED` (round-trips
through our own reader; not yet diffed against engine output).

## OPEN

- Whether the retail engine itself performs record-level merges for
  mounted deltas or only file-level shadowing — our overlay is a
  superset convention, not proven engine behavior. **UNVERIFIED.**

## Andúril model

`anduril-gamedb` (`delta.rs`, `MergedDb`) implements the full delta
model above plus a `load`-time view: base records unchanged, deltas
in load order. The engine itself may only do file-level shadowing —
our overlay is a documented superset (see OPEN). Lua `anduril.db.set*`
writes produce deltas, so mods never repack `game.gamedb`. Parity:
232,319/232,319 records field-for-field vs golden merge.
