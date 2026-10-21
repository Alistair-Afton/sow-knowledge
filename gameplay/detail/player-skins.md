# Player skins — menu, model, profile chain

> **Status:** **verified** — record chain and field names CONFIRMED on the installed `game.gamedb`; skin-behavior semantics from THaych's Player Skins Overhaul (shipped mod, tested in-game).
> **Provenance:** Andúril gamedb dumps (field names via attr-name hashes) + THaych's Player Skins Overhaul notes (external, in-game).

A playable skin is three linked GADB records: a menu node, a
character model, and a combat profile. Everything about how a skin
looks, sounds and fights hangs off that chain — no code involved.

## Record chain — CONFIRMED

```
Interface/Menu/MenuTreeNode/Skin:<node>     (9 retail nodes)
  └─ [9] CharacterModel ─▶ Character/Models:<model>
                              ├─ [49] CombatTree ─▶ Combat/Tree:<profile>
                              └─ [65] CharacterModelPicker ─▶ Character/Models/ModelPicker
```

**Skin node fields** (`Interface/Menu/MenuTreeNode/Skin`, e.g.
`Skin_DarkTalion`): `0 SkinName` (string key), `1 IconTexture`
(`.tex` path), `2 SkinLockedDescription`, `3 LockedTexture`,
`4 SkinDescription`, `5 NotificationStorage`, `6 RequiredDLC`,
`7 SkinUnlockedStorage`, `8 HasBeenViewedStorage` (the three
storage refs are `Profile/GlobalOptions` flags), `9 CharacterModel`,
`10 PrimaryState`, `11 CreationPredicates`, `12 LockCommonPredicates`.

The `GlobalOptions` flags at fields 5/8 are required — a node without
them is hidden. Retail has no spare skin node, but the tree record
already ships 8/9/10-slot layouts, so growing `BaseNodes` reuses an
existing shape (THaych).

**Model fields that matter** (`Character/Models`, CONFIRMED names):
`41 SimpleModel` → `Model/SimpleModel` (entry struct
`[?, bodyType, ?, pieces, head]`), `48 PlayerID`
(`Character/Models/PlayerID`: Talion, Celebrimbor, …), `49 CombatTree`,
`65 CharacterModelPicker`.

- **Body type** selects every gear `Inventory/ArmorData` ModelPicker
  entry — armour, weapons and the hand ring: 6 Talion, 8 Baranor/Serka,
  9/11 Eltariel, 10 Celebrimbor. Changing it changes all gear (a
  body-10 Talion wore Celebrimbor's armour) — THaych.
- **`PlayerID` drives the voice set**, not gear (setting a Talion skin's
  PlayerID to Celebrimbor switched the voice and left the ring alone).
- **Combat profile** (`Combat/Tree`) carries `FXLookupKey`,
  `CompanionModel` (the wraith), `MaterialController`, `InventorySetup`,
  `PermanentItems` (skills incl. the ultimate), `PermanentBuffs`.
- Inventory markers persist across skin changes; buffs don't — use
  buffs for per-skin tests.

**Endgame variants.** After the Shadow Wars ending, skins switch
model through their `ModelPicker` (`ShadowWarUnlocked ⇒ …_ShadowWar`
variant; e.g. `PlayerSkin_DarkTalion_Main` → picker `TalionSkin_Dark`).
The `_ShadowWar` profiles grant the `PC_IsDarkTalion` buff that
drives the green Dark-Talion presentation (THaych).

- `PlayerSkin_Celebrimbor_ShadowWar` (model **and** its
  `Combat/Tree` profile) is reachable **only** through the Celebrimbor
  skin — CONFIRMED by full-corpus reverse-ref scan: the model's sole
  incoming ref is `CelebrimborSkin_Basic.Entries[0]` (the skin's own
  `ModelPicker`); the profile's only incoming refs are the model's
  `CombatTree` link and two `Faction/World/Predicates/Character`
  records (`Talion`, `Talion_NoEltGraft`) that merely *classify* it
  as a Talion-type character, not grant it. Patching it changes that
  skin and nothing else. Its retail profile is a "Dark Celebrimbor":
  `FXLookupKey DarkTalion` (green), `CompanionModel
  DarkCelebrimborWraith`, `InventorySetup Player_DarkTalion` (dark
  brand + Isildur's ring), the Ringwraith ultimate items, and
  `PC_IsDarkTalion`.
- The story item `Inventory/Items:GP_PlayerIsDarkTalion` (owned by
  every post-ending save) **also applies `PC_IsDarkTalion`** —
  CONFIRMED: its `PassiveBuffData` links `Combat/Buffs:PC_IsDarkTalion`.
  Restoring a pre-ring skin must re-point the item's buff or the dark
  behavior returns. THaych points it at `DoesNothingBuff`; the item
  stays so story checks still work, and it's safe because every
  *other* endgame profile grants the buff itself.
- The **Statues of the Past records are unrelated**:
  `PlayerSkin03_StatuesOfThePast`/`Player_Character_StatuesOfThePast`
  surface in audio tests and profile lists, but the real pre-ring
  model is `PlayerSkin_Celebrimbor_Main` — nothing about SotP feeds
  the Celebrimbor skin.
- **Raise Dead needs more than the attacks:** the `NecroAoE_Attack_*`
  records gate on `PC_IsDarkTalion` (CONFIRMED:
  `PredicateBuffAttackerIncluded:PC_IsDarkTalion` in `Predicates`),
  *and* the animation needs the `PDARKTALION` tag that the
  dark-talion profile carries (CONFIRMED: `NativePredicates+0` =
  `"PDARKTALION"`). THaych's fix repurposes the unused buff
  `zOld_Krak_Graug_StaminaMeter_Default_Clear` (CONFIRMED present in
  `Combat/Buffs`) with `PC_IsDarkTalion`'s record layout but **only**
  the anim tag (no green FX, dark audio, or Ringwraith), then swaps
  each attack's buff requirement for a second copy of its own
  `CSP_RaiseWights` skill requirement — the Ringwraith ultimate keeps
  its buff gate, so it stays locked.

## Skin menu layout — CONFIRMED

`Interface/Menu/MenuTreeBase:SkinSelect_OverviewTree` lists the
selectable nodes in `BaseNodes` (8 slots retail). Node spacing
(`HighlightedNodeGap`, `IdleNodeGap`) are **vec2 pool constants**
(kind `0x07`) — and retail **shares one constant between both
fields** (same pool offset). Constants are often shared across
records, so give a record its own copy before changing one. Menu
stage is 1280×720; the selection frame is fixed, so adding slots
means shrinking the step (THaych fit 10 icons by 70 → 55).

Skin icons are 128×128 DXT5 `TEXR` textures, one mip, under
`interface\textures\hud\questicons\new\…`. The gap attributes hash
to `HighlightedNodeGap` `0x12DCE869` and `IdleNodeGap` `0xF4976A30`
(names CONFIRMED by hash); the first-node offset is attr `0xC00F4041`
(name unrecovered). Steps look like `(0,57)` for vertical lists and
`(60–80,0)` for rows; screen px = units × width/1280. The tree's
`MenuWidget` names the Scaleform symbol (`LIB-QuestMenuTreeBaseSkin`
in `interface\flash\widgets\componentlibrary.gfx`), whose blue frame
is fixed art sized for 8 nodes at step 70. A UI texture that is not
listed in a loaded bundle (`ingame_tex.bndl`, type `…UI_Textures`)
renders **magenta** (THaych).

## Profile semantics (THaych, in-game)

- **VFX lookup keys** (`Client/FXLookup/Table` rows `{fxPath, Key}`):
  0 Fellbeast, 1 Talion (blue), 2 Eltariel, 6 Nazgûl, 12 DarkTalion
  (green), 14 Celebrimbor.
- **Ultimates:** Elven Rage (`PC_GC_Initiate`) requires
  `PC_IsDarkTalion` **absent** plus `PS_CBGameChanger_SKILL`;
  Ringwraith (`PC_GC_2_Initiate`) requires it **present**.
- **Raise Dead:** the 13 `NecroAoE_Attack_*` records gate on
  `PC_IsDarkTalion`.
- **Buff fields:** 5 visual tag, 8 animation tag, 19/57 audio keys
  on remove/apply.
- The hand ring (`Talion_Isildur_Ring` → appearance
  `AD_Talion_Isildur_Ring`) picks its model by **body type**
  (6 = all Talion skins, 8 = Baranor/Serka, 10 = Celebrimbor), so it
  cannot vary per Talion skin through data alone — body-type changes
  also swap armour, ring pickers exist only for body types 6 and 8,
  and `InventoryItemSlotVisible` actions hide whole slot groups
  (2 = weapons, 9 = melee + offhand), not one item. Item chain:
  `Talion_Isildur_Ring` / `Talion_Celebrimbor_Ring` → `IB_…` → `AD_…`;
  when both are owned, Isildur's wins.

## Runtime notes (THaych)

- At runtime each record is an object of 8-byte fields; links become
  pointers to **name-table entries**
  `{ptr, object, category, name ptr, u32 category id, u32 hash}`.
  Bundle-definition objects start with a pointer to their bundle path.
- The ring model is read at `exe+0x27877f` (inside the generic
  record-field reader `0x140278758`) while a character is built, and
  earlier at spawn at `exe+0x4d6aad` — before the skin is known — so
  per-skin swaps need persisted state.
- **Lessons from the shipped overhaul** (THaych): copying a model's
  `CharacterModelPicker` link makes the model pick itself — copy only
  look fields (bundle, path query, preloads, client FX). Model-based
  audio tests run at spawn **before** the skin model is applied, so a
  model test alone can't distinguish skins — drive mixes from buffs
  instead. Never remove an item through a buff action on the ring —
  an early experiment deleted Isildur's ring from the save
  permanently, on every skin; the New Ring should come from the
  normal `Player` `InventorySetup`, not by removal.

## Making a new skin (proven recipe)

Spare/unused `Character/Models` + `Combat/Tree` records (layout
checked), a new `MenuTreeNode/Skin` record **whose name hash sorts
last** (appendable without renumbering — see `formats/gamedb.md`),
an icon texture + its bundle entry, string keys for title and
description, and a `BaseNodes` slot. THaych shipped two skins this
way.
