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

**Selection** (LordMonstey, from the inspected native logic):

- A draw picks a gear list from the entry's candidates by type weight:
  `P(list) = weight / Σ weights`; a null `LootTypeWeight` counts as 1.
- Inside a list, gear needs a **tag matching the captain's context**
  (an empty `Tags` list is *not* unrestricted); candidates are ranked
  and ties chosen uniformly. Context tags come from `LootTags` /
  `AddLootTags` on `Faction.Tribe` and `Faction.TraitMarker`, via
  `Faction/Loot/TagWeight` → `Tag`.
- A list with no valid/buildable gear is removed and the draw retried.
- **Type weights rotate.** `Faction/Loot/Type` carries
  `InitialWeight`, `OnDropOtherAdjustment`,
  `OnDropOtherAdjustmentScalar`, `OnDropThisReturnToValue` (CONFIRMED;
  `1Sword` = 1.0 / 1.0 / 1.75 / 1.0). The dropped type returns to its
  value; other tracked types update
  `next = clamp((current + adjustment) × scalar, 0.1, 10000)`.
- Already generated drops keep their recorded choices — editing a
  table doesn't reroll them.

**Pitfalls:** an inherited `UpgradeTiers[].ArtOverride` can restore
the donor's art on acquisition or upgrade; `DescriptionOverride` and
preorder flags carry over from donors; entries and lists are shared
across many tables (captain drops, revenge/death-threat routes,
chests), so attaching to a shared list is a broad change.
