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

## Appearance pipeline detail — VERIFIED (Universal Modder, in-game)

Field-note findings from working new-tribe mods (Goblins, Falmer),
each point confirmed in the real game:

- **`ArmorData` equipment slots** (attr `0x2FA68885`): 0 helmet/hair,
  1 shoulders, 2 chest, 3 arms, 4 pants, 5 legs, **10 body
  replacement**, **11 head replacement**. Stock `IT_empty_*` items
  blank slots 0–5.
- **Body type is chosen per class, not per tribe**: `BodyType` 2 =
  slim, 4 = stocky, 3 = buff; a body-replacement item must carry a
  `ModelPicker` for *all three* or the member renders invisible.
- **Corpses and the Garrison require stock body-item records.**
  `Faction/Global` keeps a `ModelFixUp` list (slot → pieces → item).
  A *cloned* body item chain makes the body piece vanish on death and
  gets the captain refused by the Garrison; putting new pickers on the
  **stock** `AD_Infested_Body`/`Infested_Body` item + its FixUp entry
  keeps corpses persistent and garrisoning working.
- **Item replace priority** (attrs `0x8A4035A9`, `0xB2EDA81E`):
  ordinary armour uses 2 — the captain gear roll swaps it out;
  `*_DoNotRemove` items use 10, `Infested_Body` 50, `*_Reference`
  100. Role items intended to survive the roll need ≥10.
- **Titles are roles.** A captain's epithet ("the Poet") is the
  role's string key (attr `0x1D4D2940`, text in strdb); the game
  avoids duplicate epithets among living captains — a tribe whose
  roles share too few epithets starves its own generation.
- Grunt/creature body pieces ship with **no bundle** (resident);
  a body piece's own `ArmorDataDynamicBundle` loads only through the
  armour system — ambient-grunt bodies must ride the head's bundle
  in the `DynamicResource.DynamicHeads:NPCHead` loader group (type
  `Cha_Head_G`), shared one-per-look.
- Appending entries to a `GangType_*` template list does **not**
  spawn new ambient looks; appending body entries to a grunt class
  `SimpleModel` body list does (repeating the stock entries sets the
  share). `AI/SpawnAmbient/AITypes` → `AI/Templates` →
  `Character/Models` is the placement chain.
- Prisoner gang types force orc look items (`orc_sauron_brand`,
  `Slave_BodyPaint`, `Slave_HeadPaint`, `IT_Orc_Body_LOD`,
  `IT_Orc_Body_Slave_LOD`) — strip them for non-orc prisoner looks.

## Captain generation — PARTIAL, do not overtrust

The game takes a role's **first valid `Tribes` entry** (`RequiredDLC`
checked in code) — **VERIFIED in-game** by two independent tribe
mods: appending a second tribe to an existing role is never chosen;
tribe assignment needs tribe-specific role clones in the role lists.
sow-rs's preview follows struct links when resolvable, else picks
role and tribe independently with a warning. **Not reproduced:**
name-generation tables, builder weights, and the executable's picker
logic. Generated names, class interpretation, trait combos and choice
weights are guesses — there is no data-only save-accurate captain
generator yet (OPEN).

Other verified generation-adjacent rules:

- Never set `Tribe` on a builder record — it marks members as
  presets (`UI_TargetDialog_PresetBlocked`) and they can't be
  interacted with. Tribe belongs on roles.
- Tribe-specific streaming (gear bundles, UI cards, banners) is
  gated by `Faction/Bundle/Predicate:Pred_TribeRegal*` /
  `Interface/Predicates/FactionMember:IsRegalTribe`-style predicates
  against the `Regal` TribeList — a new tribe must join `Regal` or
  its gear never streams and its cards render blank.
- `Faction/Resurrection/Chance` lists are ordered rules: a
  `{marker, chance}` entry placed first with chance 0 suppresses
  comebacks entirely (verified: `Marker_Falmer` = never returns).
- Market war chests pick a tribe via
  `Marketplace/Builders/FactionMember/Tribe` and validate through
  `Store_*` builders' `IncludedTribes`.

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
