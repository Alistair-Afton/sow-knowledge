# Inventory items and nemesis loot

> **Status:** **verified** — category paths and field names CONFIRMED on the installed `game.gamedb`; selection/weight behavior and authoring rules from LordMonstey's weapon modding guide (inspected implementation + targeted in-game tests).
> **Provenance:** Andúril gamedb dumps + LordMonstey, *sow-weapon-modding-guide* (MIT).

A weapon is two separate graphs: an **item graph** (what the item is
and how it looks) and a **loot graph** (how a captain drop reaches
it). A model can render perfectly yet be unreachable as loot, and an
item can be reachable yet load no resources — both must connect.

## Item graph

```text
Inventory/Items (Inventory.Item)
  ItemBuilder ─▶ Inventory/ItemBuilder
    WeaponData[] ─▶ Inventory.EquippedItemData
      DynamicBundle[] ─▶ DynamicResource.BundleDefinition ─▶ .bndl path
      Model[] ─▶ InventoryItemModel
        Models[] ─▶ InventoryModel
          ModelRecord ─▶ Model ─▶ ModelFile (.skel, matching .mesh beside it)
                                  MaterialSets[]
          CompanionModelRecord ─▶ Model   (spectral look, optional)
      PreviewModel ─▶ Model
  BaseWeaponData ─▶ combat data          (≠ ItemBuilder.WeaponData = appearance)
  UIItemData, WorldItemData, ItemSet, PairedItems
```

CONFIRMED on `Primary_1Sword_4Epic`: `ItemBuilder` (→ `IB_1Sword_4Epic`),
`WorldItemData` (→ `Gear_1Sword_4Epic`), `UIItemData`, `ItemSet`,
`MaxQuantity`, `CanSaveThisItem`, `TriggerPlayerSave`,
`DynamicMaxQuantity`. Scale: 5,618 `Inventory/Items`, 2,091
`Inventory/ItemBuilder`.

Fields worth knowing (LordMonstey):

| Record | Fields |
|---|---|
| `Inventory.Item` | `ItemBuilder`, `BaseWeaponData`, `UIItemData`, `WorldItemData`, `ItemSet`, `PairedItems`, quantity/save flags |
| `Inventory.ItemBuilder` | `WeaponData`, `AffixPicker`, `NemesisGearItem`, `OverrideILvl`, `OverrideILvlMatchPlayerLevel` |
| `Inventory.EquippedItemData` | `ItemName`, `ItemDesc` (string keys), `Model`, `DynamicBundle`, `AddItemOnEquip` (e.g. scabbard), `PreviewModel` |
| `InventoryItemModel` | `Models`, `OverrideModels`, `MaterialOverrides`, attachment nodes/animations |
| `InventoryModel` | `ModelRecord`, `CompanionModelRecord`, `MinMeshIndex`, `MaxMeshIndex` |

- **Repeatable drops** use `MaxQuantity 0`, `CanSaveThisItem` +
  `TriggerPlayerSave` true, `NemesisGearItem` null and both ILvl
  overrides off — the item then keeps the level the drop constructor
  supplies. Story rewards use grant settings (player-level override,
  full affix picker) and make poor drop donors.
- **Combat predicates don't learn new items.** Native
  `Combat.PredicateItem` lists (`PC_Sword`, `PC_SwordOrDagger`,
  equipment-activation lists) gate behavior on item identity; a new
  item must be added to the lists its donor belongs to.
- **Slots are fixed.** A new item lives inside an existing equipment
  slot — an axe or hammer uses the Sword slot and its stance; a model
  name creates no category. Dagger stays Dagger to keep stealth/
  execution syncs; crossbows use the Bow route. Native weapon rigs:
  sword `root`/`sword_handle`/`sword_tip` (CONFIRMED on
  `player_blacknumswordt1.skel`; handle at Y −23.57, tip at Y +100.53
  cm), dagger
  `root1`/`dagger_handle`/`dagger_tip`, bow `bow_handle`/`bow_draw` +
  limb bones, crossbow `root`/`base`/`Bow_R`/`Bow_L`/`String`/`Bolt`
  (native fire clip `BRXbow_Fire`).
- **Behavior-state identity matters.** Cloned behavior nodes are
  invisible to native `Predicate.ActiveNode` checks, so transitions
  treat the weapon as not in that state (a cloned bow draw kept
  dropping out of draw). Reusing the native draw/hold/fire nodes fixed
  it in game.

## Loot graph

```text
Faction/Loot/LootTable                       (188)   Items[] ─▶
  FactionLootEntry (struct)                          NemesisGearList[] ─▶
    Faction/Loot/NemesisGearList              (54)   LootTypeWeight ─▶ Faction/Loot/Type (95)
                                                     Entries[] ─▶
      Faction/Loot/NemesisGear               (366)   Item ─▶ Inventory/Items
                                                     NemesisAffix ─▶ Inventory/Affix/Definition
                                                     Tags[] ─▶ Faction/Loot/Tag
                                                     UpgradeTiers[] ─▶ Faction/Loot/NemesisGearUpgradeTier
```

CONFIRMED on `NemGearList_4Epic_1Sword` (type `1Sword`, 22 entries)
and `NG_1Sword_E_AllHealthMightLC_4E`, whose fields are
`IsPreorderItem`, `RoleFormatString`, `RarityFormatString`, `name`,
`ItemDroppedByFormatString`, `Desc`, `DescriptionOverride`,
`AdditionalNemesisAffixList`, `Tags`, `Item`, `NemesisAffix`,
`UpgradeTiers`. Native list naming: `NemGearList_<n><Rarity>_<slot>`
(`3Rare`, `4Epic`, `5Lgnd`; `_All`, `_NoRing` variants).

**Selection** (LordMonstey's reading, now VERIFIED-STATIC against
`FUN_140648214`/`FUN_140579f90`/`FUN_14057b024`/`FUN_140648db8`):

- A draw picks a gear list from the entry's candidates by type weight:
  `P(list) = weight / Σ weights`; a null `LootTypeWeight` counts as 1.
  The weight is the tracker's *current* value, not `InitialWeight`.
- Inside a list, gear needs a **tag matching the captain's context**
  (an empty `Tags` list is *not* unrestricted — a gear with no
  matched tag is rejected `"Is NOT Valid"`). The captain's tag set
  is built by `FUN_140646b40` from member record lists (traits,
  tribe, role, personality…); each `GearTagWeight_<kind>_<name>`
  record (`{weight:f32, index:u32, GearTag ref}`) contributes one
  `{tag, weight, index}` triple.
- Scoring: a gear's score = Σ matched `GearTagWeight` weights and
  its "BestIndex" = the first matched tag's index. The winner is the
  **lowest** BestIndex, ties by **higher** score, then a random
  pick among remaining ties. A gear sharing an affix with an
  already-generated item is rejected (`"duplicate Affix"`).
- A list with no valid/buildable gear is removed and the draw retried.
- **Type weights rotate.** `Faction/Loot/Type` carries
  `InitialWeight`, `OnDropOtherAdjustment`,
  `OnDropOtherAdjustmentScalar`, `OnDropThisReturnToValue` (CONFIRMED;
  `1Sword` = 1.0 / 1.0 / 1.75 / 1.0). The dropped type returns to its
  value; other tracked types update
  `next = clamp((current + adjustment) × scalar, 0.1, 10000)`.
- Already generated drops keep their recorded choices — editing a
  table doesn't reroll them.

## Retail drop algorithm — VERIFIED-STATIC (Ghidra)

Decoded from the retail exe; the engine's own `[LOOT]:` debug strings
name each step, which anchors the reading. Field names CONFIRMED by
attr hash on the installed DB.

**Which tables are considered.** `Faction/Loot/Setup` (`Default`)
holds `Groups[]` → `FactionLootSetupEntry` rows, each an **ordered
list of loot tables** plus a predicate ref; e.g. one group is
`Equip_AllEquipment_Drops, Equip_EpicEquipment_Drops,
Equip_LegendaryEquipment_Drops, …_RevengeMission, …_DeathThreat,
…_DeathThreat_Ambush, Equip_Zog_CarnanSide6`, a sibling group the
`_NoRing` variants, others DLC3/DLC4/treasure-orc/Mirian tables. The
active setup is fetched by `FUN_140f84870`: mode 0/1/2 dispatch to
three hash-keyed field accessors (`FUN_140f86ee0`/`FUN_140f87200`/
`FUN_140f87520`, field hashes `0x338a9b`/`0x338b4a`/`0x23497d4`) on a
runtime `Global` object resolved through the `DAT_142701a78`
name→instance map — the slot→setup binding is runtime-resolved, not
static in the db (no db record links 2+ `Faction/Loot/Setup`
records). The mode comes from `FUN_140f87840` on member `+0x508`
(VERIFIED decode): bit 29 must be set or nothing drops; bits
48/49/50 select mode 0/1/2 with first-set precedence; none set → no
setup. Members spawn with their setup preloaded (`FUN_140356370`).
That `Default` is the setup normally active is INFERRED.

**Drop entry** `FUN_1404f2748` (per dying member):

1. Member `+0x508` bit 40 → `"FactionMember %s has Loot disabled"`;
   nothing drops.
2. Member `+0x260` set → `"has Stolen Loot"`: the stolen item drops
   instead, at its level **+5**, clamped `[1, 90]`.
3. **Rare roll** (`FUN_1404f3378`, record
   `Faction/Loot/LootRareChanceSetup:Player`): draw
   `r ∈ [0, RandomRollMaxValue]`; sum four terms, each
   `clamp(Flat + n × EnemyLevelScalar, 0, Max)` and present only if
   the member qualifies — Captain-or-Warchief (`Warchief*` fields when
   a warchief, else `Captain*`), DeathThreat, Revenge, Vendetta —
   where `n = member+0x470 + 1 + member+0x4b8`. Roll passes when
   `r ≤ sum`. Retail values: Warchief 0.4 / DeathThreat 0.4 / Revenge
   0.1 / Vendetta 0.1 / Captain 0 flat, all level scalars 0, max 1.
4. Collect the tables passing the filter below, in list order; for
   each: `"Selected: LootTable: %s"`, compute item level and rarity,
   try to award — **the first table that awards anything wins**.

**Table filter** `FUN_1404f2c04` — rejection order (each logs
`"Rejecting Loot Table: %s - <reason>"`):

| # | Reason | Rule |
|---|---|---|
| 1 | `NamedOnly` | flag set and the victim isn't a named member |
| 2 | `RandomRollPredicate` | flag set and the rare roll failed |
| 3 | `GameMode` | table lists game modes and the current one isn't among them |
| 4–5 | `CharacterTypeList-Included/-Excluded` | victim character type |
| 6–7 | `KillerCharacterTypeList-Included/-Excluded` | killer's character type |
| 8 | `KillerMustBePlayerAlly` | int tri-state: 0 any, 1 must be ally, 2 must not |
| 9 | `MustHaveDeathThreat` | bool predicate must equal the victim's death-threat state |
| 10–11 | `Included/ExcludedAttackerCategory` | killing blow's `Combat/Category` (stealth, melee, counter, …) |
| 12–14 | `IncludedItemPlayer`, `IncludedItem`, `ExcludedItem` | `Combat/Predicates/Item` against player / victim inventories |
| 15–16 | `Min/MaxFactionMemberKills` | victim's kill count (0 = no bound) |

**Item level** `FUN_14075ad7c` (`Faction/Loot/ILvl`): with
`OverrideMinLevel/OverrideMaxLevel` set, a uniform level in that range;
otherwise `player_power + level-diff bonus (LevelDiffBonus table,
enemy vs player) + PlayerItemBonus + MeanOffset`, perturbed by a
`StdDev` draw, clamped to `[mean − MaxNegativeDifference,
mean + MaxPositiveDifference]`, rounded, min 1 (`"CalcILvl Start"`).

**Rarity** `FUN_14075a6f8` (`Faction/Loot/RaritySetup`; log
`"CalcLootRarity Start - %s - RowRank = %s, Vow = %s, RevengeKill =
%s"`): score = `WeightTableOffset + (LevelDiffBonus(enemy, player) +
RankBonus[row rank] + weight-table term + (vow ? bonus : 0) +
(revenge kill ? bonus : 0)) × WeightTableOffsetScalar`, floored at
`MinRarityScore`, rounded. "Vow" is member `+0x508` bit 12 — the same
bit the `MustHaveDeathThreat` filter tests, so a death-threat victim;
"RevengeKill" is bit 24. The two bonuses are the setup's
`DeathThreatBonus` / `RevengeBonus` (binding INFERRED from names;
`Rarity_EpicOrc`: `DeathThreatBonus 2.0`, `RevengeBonus 0`,
`WeightTableOffsetScalar 4.0`). The item-level terms above are bound
to fields the same way (structure VERIFIED-STATIC, field roles
INFERRED). Term details: `RankBonus` rows are `{Bonus, RowRank}`
matched to the victim's rank record; `DeathThreatBonus` is a row
list — `FUN_14075b24c` sums each row's `Bonus` whose threat link is
in the active-threat set (VERIFIED-STATIC); `LevelDiffBonus` rows
(`FUN_14075b52c`) map (enemy level − player level) through
`{in_lo,in_hi}→{out_lo,out_hi}` then a `Combat/Stat/Formula/Float`
(returns 0 without one — VERIFIED-STATIC; row-field bindings
INFERRED from values: `Rarity_EpicOrc`'s row reads
`out −5…+5 over in −30…+30`). Each `Entries[]` rarity
(`Faction/Loot/Rarity`) then gets the value
`WeightTable[score] × PlayerLevelWeightTable[player level]` — both
`Combat/Stat/Constant/FloatArray` curves read by
`FUN_1402ca0e0` = `array[clamp(index, 0, len−1)]` (a missing table
falls back to the ≈1.0 default; the arrays are ~81 raw f32s:
`Loot_Common` declines 0.74→0.1, `Loot_Rare` rises 0.2→1.5,
`Loot_Epic` 0.06→0.9, `Loot_Legendary` is all-zero, and the
`Loot_LevelComparison_*` tables stay 0 for the lowest player levels
then rise toward ~2.75–3.5). One rarity is picked by **weighted
reservoir sampling** over those values — `FUN_1401b3b90` keeps an
accumulator `acc += w` and keeps the entry iff `rand×acc < w`, i.e.
an exact weighted draw (values ≤ ~0 never win) (`"CalcLootRarity
Rarity Type = %s, Value = %0.3f"` / `"Final"`).

**Award** `FUN_140648214` — walks the table's `Items[]`
(`FactionLootEntry`), keeping entries whose rarity list is empty or
contains the chosen rarity:

- Gear lists of the entry are drawn by **weighted reservoir sampling**
  on their loot type's *current tracked* weight (untracked/null type =
  1.0); the drawn list builds gear (tag-ranked pick inside the list);
  a list that builds nothing is removed and the draw repeats.
- On success the type tracker updates (`FUN_140648db8`): the drawn
  type resets to its return value, every other tracked type becomes
  `(w + OnDropOtherAdjustment) × OnDropOtherAdjustmentScalar`, all
  clamped to `[0.1, 10000]`.
- Entries can instead carry a plain item or one of 12 resource types,
  each competing at weight 1.0; nemesis gear found wins
  (`"Awarding NemesisGear %s"` / `"Awarding Item %s"` / `"Awarding
  Resource %s"`).

**Weighted reservoir sampling** (`FUN_1401b3b90`, used throughout):
keep a running total `T`; a candidate of weight `w > 0` replaces the
current pick when `u·(T + w) < w`, `u ∈ [0,1)`; then `T += w`.
Weights ≤ 0 never win. RNG step: `x = x·0x0BB38435 + 0x3619636B`,
high 32 bits / 2³².

**Pitfalls:** an inherited `UpgradeTiers[].ArtOverride` can restore
the donor's art on acquisition or upgrade; `DescriptionOverride` and
preorder flags carry over from donors; entries and lists are shared
across many tables (captain drops, revenge/death-threat routes,
chests), so attaching to a shared list is a broad change.
