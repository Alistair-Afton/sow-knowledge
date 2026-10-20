# Nemesis system data

> **Status:** **verified**
> **Provenance:** faction categories + attribute hashing — sow-rs.

What the gamedb holds for the Nemesis hierarchy, and what is and
isn't reproducible from data alone.

## Record layer — VERIFIED

| Category | Type | Records | Contents |
|---|---|---:|---|
| `Faction/Tribes/Definitions` | `Faction.Tribe` | 19 | tribe display-name strdb keys, assets/items/tags, `RequiredDLC` |
| `Faction/Roles/Definitions` | `Faction.Role` | 914 | role definitions; each role contains tribe struct links |
| `Faction/Traits/Definitions` | `Faction.Trait` | 484 | trait definitions + role/trait pickers |
| `Faction/NameTable` | `Faction.NameTable` | 4 | name-table definitions |
| `Faction/ClassMemberSetup` | `Faction.ClassMemberSetup` | 5 | default class data → `FactionClassDataMap` records |

- Tribe display names resolve via `DisplayName` →
  `database/string/enus/string.strdb` (VERIFIED).
- Attribute names are hashed-only in the file (see gamedb.md);
  sow-rs's dictionary matched 16,987/49,994 nemesis attribute
  occurrences (34%) — plenty remains nameless.
- Appearance → mesh/materiel: `Combat/InventorySetup` → items →
  `ItemBuilder` → `ArmorData` → `ModelPicker` (`RequiredBodyType`,
  `Required/ExcludedAppearanceTag`) → `SimpleModelPiece` → `Model`
  (see geometry.md for LOD/mesh-index semantics).

## Captain generation — PARTIAL, do not overtrust

A field note reports the game takes a role's **first valid `Tribes`
entry**, with `RequiredDLC` checked in code. sow-rs's preview follows
struct links when resolvable, else picks role and tribe independently
with a warning. **Not reproduced:** DLC eligibility, name-generation
tables, builder weights, and the executable's picker logic. Generated
names/titles, class interpretation, trait combos and choice weights
are guesses — there is no data-only save-accurate captain generator
yet (OPEN).

## Orc gear visuals — CONFIRMED (renders)

`Orc_Infantry`'s `Named_SauronFollowerItems` → `Sauron_{Chest,Pants,
Shoulders,Arms}` item chains render as real armor sets; `MeshIndexPairs`
select numbered group ranges (see geometry.md). LOD-group material
indices bind the `.mat` per group — verified on the Flint set; our
textured animated orc renders confirm the chain end-to-end.

## What the save system holds — PARTIAL

See `docs/formats/save.md` (Andúril's structural notes): the save
captures captain state but the full record schema is unfinished
(OPEN).

## Practical Nemesis modding

- Retuning is gamedb editing: tribe/role/trait lists, item tables,
  appearance tags, name tables — all ordinary records/links.
- New captains/roles need new records + consistent link indices +
  possibly hash-named attributes — feasible with a GADB writer (OPEN
  tooling).
- Appearance variety is asset-bound: new looks need matching
  mesh/material/bundle content.
