# Nemesis data layer — faction categories in `game.gamedb`

> **Status:** **verified** (category census + attribute hashing);
> captain-generation semantics PARTIAL
> **Provenance:** sow-rs/OnSight `formats/nemesis-data.md` (record
> walks over installed `game.gamedb`); cross-checked against
> `../../formats/detail/gamedb.md`. Andúril-side sim model:
> `faction-*.md`.

The data layer reads GADB records directly; it does not execute the
game's captain builders. Links are `category_index<<20|record_index`.
**Attribute names are absent from serialized GADB** — an attribute is
identified by `name_hash` of a candidate name.

## Categories

| Category path | Type | Shipped records | Use |
|---|---|---:|---|
| `Faction/Tribes/Definitions` | `Faction.Tribe` | 19 | Tribe display-name strdb key, assets/items/tags/DLC fields |
| `Faction/Roles/Definitions` | `Faction.Role` | 914 | Role definitions; each role contains tribe struct links |
| `Faction/Traits/Definitions` | `Faction.Trait` | 484 | Trait definitions + role/trait pickers |
| `Faction/NameTable` | `Faction.NameTable` | 4 | Name-table definitions |
| `Faction/ClassMemberSetup` | `Faction.ClassMemberSetup` | 5 | Default class-data mapping |

`Faction/ClassMemberSetup:Default` → `FactionClassDataMap` records:
combat + personality tables, name tables, role lists. Tribes expose
the English label via `DisplayName` → `database/string/enus/string.strdb`.

## Attribute dictionary — candidate coverage, not definitive

sow-rs's dictionary starts with field-note names, then brute-matches
hashes from heap strings, record-name tokens and short token joins
against the five category layouts: **16,987 of 49,994 attribute
occurrences (34.0%)** on the installed build. Ambiguous matches remain
candidates; unmatched values keep their hash. Any consumer must
tolerate raw hashes.

## Captain generation — what is and isn't known

- A role's first valid `Tribes` entry decides tribe, with
  `RequiredDLC` checked by game code (field-note evidence). Following
  each role's `Tribes` struct links to tribe definitions works; when
  no pair resolves, independent picks need an explicit fallback flag.
- **Unknown:** DLC eligibility order, name-generation tables, builder
  weights, executable picker logic, trait-combination rules, class
  interpretation. A data-driven captain preview is decodable; it is
  not a save-game or runtime captain generator. Andúril's own sim
  experiments (`faction-*.md`, `anduril-sim`) face the same
  boundary — keep data-read captains and engine-generated captains
  separate in naming.
