# GADB — the game database format

> **Status:** **verified****
> **Provenance:** GADB record model probed on real game.gamedb — Andúril + sow-rs.

`database/game/game.gamedb` (56.8 MB), every `.bvr` behavior DB, and
the `GADB` section inside each region `.objres` share this format.
It is the game's answer to "where does the behavior live": nearly all
gameplay data is typed GADB records linked by index.

## Binary layout — CONFIRMED (both projects parse it)

- Header: `GADB`, version, counts and offsets; **string-table offset
  at 0x28** (VERIFIED).
- Strings: `u16` byte length + bytes; a heap follows.
- **Layout records are 15 bytes each**; category and record data
  follow the layout table; file ends with a `u32` zero terminator.
- **Links are `category_index << 20 | record_index`**;
  `0xFFFFFFFF` = null. Links are indices, not hashes — to resolve by
  name you hash the target name and find the record carrying it.
- Categories are paths (`Combat/Attack`, `Faction/Roles/Definitions`,
  `Interface/HUD/Widget`, ...); records are named or unnamed
  (`_Structures...` struct records are typically unnamed).
- Scale (VERIFIED, installed `game.gamedb`): 2,742 categories,
  361,939 layouts, 278,865 named + 415,767 unnamed records,
  ~8.5 M value words.
- Embedded GADBs differ: the `.objres` object DB is GADB version
  0x11 with one category `ObjectDB` of `ObjectRec` records.
- Readers exist in Andúril and sow-rs.

## Attribute names — PARTIAL, structural limitation

Attribute *names are not serialized* — each attribute slot is a hash.
Recover them by hashing candidate strings with the engine name hash
(dictionary + brute matching). sow-rs recovered ~34% of nemesis
attribute occurrences; gameplay-critical names (`DamageFormula`,
`InitialValue`, `MaterialSets`, `ModelFile`, ...) are known because
the same names appear in record categories, heap strings, or were
brute-matched. When documenting a field, cite the candidate name and
whether the hash match is unique.

## Field kind tags (CONFIRMED across our work + sow-rs)

- Ranges are kind `0x07`: a heap offset to two f32 `[min, max]`
  (both zero = unset).
- `Predicate.Command.ActionId`-style fields (kind `0x05`) are heap
  offsets in `.bvr` files to the **command name string**, which names
  a `Profile/Commands` record in `game.gamedb` (VERIFIED — used for
  the input-bindings map).
- String fields hold heap offsets to `str16` values; typed links hold
  the packed `cat<<20|rec` index; struct links nest records inline.

## Content map (record categories that matter)

VERIFIED record paths (sow-rs, read from installed DB):

- **Combat:** `Combat/Attack` (16,976 records), `Combat/Sync`,
  `Combat/Stat/Formula/{Float,Damage,FloatOperation}`,
  `Combat/Stat/Variable/{Float,Int,Bool,FloatIndex}`,
  `Combat/Stat/{Charge,Energy,WraithMeter,Ammo}MeterSetup`,
  `Combat/FightCircle/{Participant,PlayRole}`, `Combat/Predicates/*`,
  `Combat/Buffs/*`, `Combat/CounterType`, `Combat/Hint`.
- **Inventory/skills:** `Combat/InventorySetup:<name>` (character
  loadouts incl. stats), `Inventory/Items`, `Inventory/Skill/Skills`,
  `Inventory/{Weapon,RangedWeapon}Data`, `Difficulty/Settings`.
- **Models/materials:** `Model:<name>` (`ModelFile`, `MaterialSets`,
  `TranslucentMaterial`), `Model/SimpleModel(Pieces)`,
  `Character/Models` (capsule radii, `PlayerMovement`,
  `LocomotionIK`, `ContextActions` = 124 traversal edge tags).
- **Nemesis:** `Faction/{Tribes,Roles,Traits}/Definitions`,
  `Faction/NameTable`, `Faction/ClassMemberSetup` (see nemesis.md).
- **UI:** `Interface/HUD/Widget*`, `Interface/Elements`,
  `Interface/NavMarkerTypes`, `Interface/FloatingText`,
  `Interface/GlyphReplacement` (see ui.md).
- **Weather/world:** `Weather/Climates`, `Weather/Patterns`,
  `Weather/Lighting/Base`, `Weather/MaterialOverride/*`.
- **Input:** `Profile/Commands` (382 records: `ActionId`, `PCKeyboard`,
  `PCGamepad`).
- **Scripts/quests:** structural notes in the Andúril repo's
  `docs/formats/scri.md` (CONFIRMED there).

## Semantics that are proven (not just structural)

- Stat formulas evaluate as a node pipeline: running value starts at
  `InitialValueRange.min`, each node's `Operation` applies when its
  predicates pass, `ReturnImmediateIfPassPredicates` short-circuits,
  `ClampRange`/`ClampOutput` clamps. `FloatOperation` sums each of
  ADD/SUB/MUL/DIV: type 0 `(prev + A − S) × M / D`, type 1
  `prev × M / D + A − S` (VERIFIED — checked against named formulas
  like `Override_Return_10` and `Gem_GreenRing_DefScalar`).
- Side 0 = source character, side 1 = target; a formula reached from
  side 1 swaps them.
- Mesh→material binding: `Model:<n>` record `MaterialSets[0]` indexed
  by the skeleton LOD-group material index (VERIFIED — see
  materials.md).

## Editing implications

A mod that edits `game.gamedb` can retune any formula, stat, meter,
input binding, HUD widget linkage, or perception record without
touching the executable — provided the tool preserves link indices,
layout records, and the string heap. This is the single most
powerful modding surface in the game. Nobody has yet shipped a
robust GADB writer (OPEN).
