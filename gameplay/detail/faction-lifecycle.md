# Spec: Faction member lifecycle — rarity, promotion, resurrection, seating

> **Status:** **verified** — rarity/promotion/resurrection vs live data
> **Provenance:** live roster census + picker/trait surface diffs
> (`Helper/roster_live.tsv`, `live_traits*.txt`, 562-member endgame
> board, 223-member lv20-43 board), gamedb record reads (`anduril db`),
> `Faction/Resurrection/*`, `Faction/Rewards/Definitions` kill/promote
> defs.

## Rarity (VERIFIED)

- Rarity (`Marker_OrcRarity_Common|Epic|Legendary`) is decided **at
  creation** by the `Default_OrcRarity`/`Default_OrcRarity_Act1`
  reward bundle — before the picker pass, since `PickerList_OrcRarity`
  member rows gate on it.
- Creation rarity is the *floor*: live boards skew higher with age
  via promotion/epicification (below).
- `Undead` status == **player Raise Dead only** — verified: all 54
  `Marker_Resurrected_PlayerAbility_RaiseDead` holders carry
  `Quirk_Undead`; no other undead path exists in records.

## Player-kill epicification (VERIFIED — live census + defs)

- `KillPlayer_BecomeEpicChance` pool = {None w=75, MarkerSwap w=25}:
  a captain that kills the player has a **25% Common→Epic promotion**
  (`Marker_OrcRarity_Common` removed, `_Epic` granted; excluded when
  Legendary).
- `KillPlayer_PickEpicTrait` = 33-def uniform pool (16 Champion + 17
  shared `Default_EpicOrc` defs), gated on `MarkerList_OrcRarity_Epic`
  — epic killers stack an epic trait *per player-kill*. Live: every
  epic member carries ≥1 `ET_` trait (129/129).

## Promotion rarity upgrades (VERIFIED)

- Hierarchy promotion runs `Orc_Upgrade_ToEpic` (Common→Epic) at
  Warchief, and `OverlordSetup_LegendaryUpgrade_{Common,Epic}Orc`
  trait pools at Overlord — live inner-warchief rows skew 50-67%
  Legendary.
- Epic-trait draws belong to **OverlordSetup only**: restricting the
  promotion epic-trait pick to `next == Overlord` gives sim 20% vs
  live 19% at lv28 (was 12-14% / 32% under other models). At lv80 the
  residual gap (sim ~14-17% vs live 37%) is member-aging accumulation
  — OPEN, tied to the unmodeled promotion cadence.

## `Follower_GrabAndThrow` seat-strip (VERIFIED — live resolution)

- Fresh-spawn presence ≈ **42%** flat (live n=562 census, members
  with no return markers: unseated 100/240 ≈ 42%). Earlier
  level-scaling readings (21% ≤20 → 51% @51+) were *returner
  contamination* — mature boards are dominated by returns.
- Returns re-grant: unseated returners 68% (79/116), seated
  returners 38% (5/13) — raise-dead/cheat-death/escape returns
  restore it. RaiseDead holders among seated keepers: 3/39 vs
  0/166 stripped — strongly enriched.
- Seating strips it: fresh seated 18% vs fresh unseated 42% →
  ~43% of carriers keep it ⇒ **chance strip, not outright remove**.
  Flat across seat tiers (Captain_0 19%, _1 19%, _2 14%, Outer WC
  30%, Inner WC 15%, Overlord 14%) and classes — seat, not rank
  or class, is the trigger.
- **The gate is decoded**: the `Picker_Follower_GrabAndThrow` entry
  carries `RequirePlayerFollower=True` (VERIFIED record field) —
  the trait only lands on follower-eligible members (ambient/event
  follower-candidate spawns, returns). It is NOT a random keep or
  an outright seat strip: board-fill seats are just rarely
  follower-flagged.
- Corollaries that now resolve: region spread on fresh unseated
  (18% MinasIthil → 55% desert) = follower-flag share per board
  (fill path mix), not a level mechanic; `Default_Act1` members
  carry it less (30% vs 46%) because Act1 board fill is
  non-follower; `Quirk_IronWill` anti-correlates (3/140 vs 53/100)
  — INFERRED same-path exclusivity.
- Sim models: flat eligibility roll p≈0.42 at spawn, seat keep
  0.43 (≈ seated follower-flag share), re-grant on return.
  Residual region spread UNVERIFIED at mechanism level.

## Row seats (VERIFIED — live roster occupancy)

Each region board holds a fixed seat grid; on the mature live
roster every row reads at the same cap, and every member beyond
the caps carries `row='-'` (a roamer — ~63% of the board):

| row | seats |
|---|---|
| `Row_Captain_0` | 4 |
| `Row_Captain_1` | 8 |
| `Row_Captain_2` | 7 |
| `Row_Inner_Warchief` | 2 |
| `Row_Outer_Warchief` | 4 |
| `Row_Overlord` | 1 |

≈26 seated per region. Seating is therefore a *contest for a
finite slot*: a spawn seats only while its tier's rows have a
hole, and a dead/shamed occupant holds the seat until the
hole-timer path refills it. (`Faction/Rows` holds 83 unnamed row
records — the per-row capacity field is UNVERIFIED in the schema;
the caps above are measured occupancy, not decoded config.)

## Resurrection return package (VERIFIED — 58-member diff)

`Quirk_Unbreakable` is the player-raise-dead return state: 54/54
`Marker_Resurrected_PlayerAbility_RaiseDead` holders are Unbreakable.
The package (UB holders vs others): `Quirk_Undead` 93/0,
`Marker_Deranged` 93/6, `Vulnerable_Fire`(+perm picker) 98/6,
`Immune_Poison` 91/39, rarity→Epic 66/18, `ET_GreatToughness` 52/9,
`Poison_Weapon`+StatusEffect marker 62/22, `Marker_EscapedPlayer_2`
36/2, `Picker_Undead`/`Picker_Unbreakable_` 33/0, undead personality
~30/0. DB: `Faction/Resurrection/Chance` →
`MarkerList_Resurrected_PlayerRaiseDead` → the marker; region share
tracks conquest order (19% MtnPass vs 2% snowMtn).

## Shame reward pools (VERIFIED — GameDB `Faction/Rewards/Lists`)

A shame transition draws this suite (def gates pick normal-vs-super
through the shame-tier markers; verified record content, apply order
INFERRED from gate dependencies — `Shame_MarkShamed` must run first
because its granted `Marker_InActOfBeing_{Shamed,SuperShamed}` gates
the XPLevelAdjust defs):

- `Shame_MarkShamed` (pick 1): `Shamed_MarkShamedOnce` /
  `Shamed_MarkShamedTwice` — the shame-tier markers.
- `Shamed_XPLevelAdjust` (pick 1): `ReduceLevel_Shame` /
  `ReduceLevel_SuperShame` / `IncreaseLevel_ShameManiac`, gated on
  `MarkerList_InActOfBeing_{Shamed,SuperShamed}` /
  `Marker_BecomingManiac`; each carries a `PlayerShame*_{Lower,
  Increase}XP` record ref (level delta lives there — OPEN).
- `ShameChance_to_become_Maniac` (pick 1):
  `ShameBecomes_DerangedNotManiac` vs `ShameBecomes_Maniac`.
- `MaybeNewRole_Shamed` (pick 1):
  `Reward_TransformRole_UnashamedAmbush`,
  `Reward_PinnicleScar_Shamed{,Bitter}`, `Reward_Shamed_DoNothing`.
- `SuperShameChance_to_become_Maniac`,
  `SuperShame_Deranged` (~80-entry deranged-personality pool),
  `SuperShameNonManiac_ReduceResistances` (pick 2):
  `Reduce_{Beast,Combat,Fire,Poison,Ranged,Stealth}Resistance` +
  `Remove_FrostImmune`. The `Reduce_*` defs gate on
  `PickerListRaw_{channel}ImmuneOrResistant` +
  `MarkerList_ShamedTwiceOrMore` and exclude
  `MarkerList_IsA_Maniac` — VERIFIED super-shame/non-maniac gating.
- `ResReward_FireResistant_Reduce`: gate(holds Resistant_Fire, not
  MortalVuln) → `Picker_Resist_Fire_Vulnerable` +
  `Picker_Resist_Fire_Resistant` refs — one-step channel downgrade;
  the lv80 vuln-regeneration source the XP-row system never sees.

### Andúril model

`Director::shame_rewards` draws the suite on every `Status::Shamed`
transition (duel loss, `Eject` mission resolution) through the shared
`roll_list` pool roller (extracted from `reward_bundle`'s row
closure); `Marker_Becoming{Maniac,Deranged}` sets `Captain.deranged`.
The shame-tier def gates resolve super-vs-normal internally.
`PlayerShame*` level deltas ARE applied: the winning adjust def's
marker selects an `EventAwards` record whose
`_StructuresFactionPlayerBasedXPLevelAward` rows `[hi, lo, _, Delta
ref]` band on player level; the `Delta` record's tail cells are an
equiprobable draw (`negative_05`→-5, `about_31_levels`→{29..33}).
Sim applies `power += delta` clamped [1, 80] — INFERRED application
(retail may set relative to player level rather than adjust).
Scar-visual defs (`PinnicleScar_*`) still OPEN (appearance only).

## Return reward pools (VERIFIED — GameDB `Faction/Rewards/Lists`)

- `MaybeNewRole_CheatDeath_<manner>` (pick 1) — 23 manner pools
  (ArmSever, Balefire, Fall, FireNormal, Poisoned, OrcSlash…), each
  carrying `ResReward_{TransformRole,PinnicleScar,ChangeRole}_*_Delay`
  scar/role defs: the stitched, bag-head, machine, claw, hook,
  survivor and flame-of-war transformations. Decapitation routes to
  `NewRole_CheatDeath_HeadRemoved_Olog` (`TransformRole_Stitch`).
- `ResurrectionOrc` (pick 1) — the killer-memory draw:
  `ResReward_{Hate,Terror}_{PowerfulEnemies,Rival}` at ~30% vs a
  weight-70 `Nothing` — returning captains remember their killers.
- `MaybeNewRole_BecameDeranged` (pick 1) —
  `Reward_StoryRole_Deranged{,Mindless,Ruined,Painless}`; `Deranged`
  is the ~80-entry deranged-personality pool (same shape as
  `SuperShame_Deranged`).

### Andúril model

`Director::return_rewards` draws on Transform resolution:
`cheat_death` → a manner pool + `ResurrectionOrc`; `deranged` →
`MaybeNewRole_BecameDeranged` + `Deranged`.

Manner routing (VERIFIED): `Captain::death_manner` carries the
`MemberDeath_*` tag set at each kill site — duel/raid kills take
the winner's weapon element (`Flaming_Weapon`→`FireNormal`,
`Poison_Weapon`→`Poisoned`, `Curse_Weapon`→`Curse`, else
`OrcSlash`), executions `Executed` (named executioner) or `Generic`
("the wilds"), player kills `PlayerSlash`, decapitation keeps
`NewRole_CheatDeath_HeadRemoved_Olog`. `Faction/Resurrection` holds
all 31 `MemberDeath_*` tags incl. ones with no role pool
(`Executed`, `Curse`, `OrcClub`, `PlayerStealthSlash`…) — pool
lookup falls back to `_Generic`. Scar visuals now land:
`Resurrection_<manner>`'s `Combat/Scar/Type` refs (Burns,
MetalJaw_Helmet, ExposedBrain, Sack_Head, Poison…) record onto
`Captain::scars` on return — appearance-only, accumulating across
deaths like retail stitches+burns. Still OPEN: beast/fall/arrow
manners need a finer kill-cause model; scars are census-visible
but not rendered.
Observed on runs: killer-memory `Hate/Terror_PowerfulEnemies` and
scar-role traits on returnees.

## Training Orders / Contracts (VERIFIED — GameDB + binary strings)

`Faction/Contracts/*` is the **Training Orders** system: player-owned
inventory items applied to captains (followers) from the hierarchy
menu — not autonomous nemesis missions and not board jobs. Nothing
in the event/generation tables references contract records because
nothing schedules them; only the two `Faction/Contracts/List`
records (`Training`, `Equipment`) enumerate them for the store/
inventory UI.

- `Contract` record (34 defs — `MountTraining`, `Epic_DireMount`,
  `BattleTraining`, `Element_*`, `Gang_*`, `Epic_*`,
  `ReassignmentOrders`…) carries: mirian cost (cell1 — 750 for
  `MountTraining`, 3000 for `Epic_DireMount`), title/description
  strings, `Contracts/Failure` condition list (`MountTraining` is
  blocked by `HasMount`, `IsOlog`, `HasGang_Ologs`,
  `UnsupportedPersonality`, `HasDecoys`, `CommandDisabled`),
  world-item/icon/movie refs, `Contracts/UIGroup`, and a
  `Faction/Events/Events` ref at cell 16 (`FactionEvent`) naming
  the event that resolves the order (`MountTraining` →
  `Contract_Mount`).
- The `Contract_*` events are a uniform template: three
  `Faction/Rewards/Lists` refs — cell15 `Contract_RemoveAnEpicTrait`,
  cell16 the contract's own list (`Contract_Mount` → `GrantMount`),
  cell17 `Upgrade_CommonOrcToEpic`. Applying an order strips an
  epic trait, grants the contract reward, and upgrades the captain
  common→epic.
- Binary evidence that this is the player-inventory lane:
  `ActionApplyContract`, `ContractApplyFromInventoryHeldAction`,
  `OrdersContract_ApplyWithMirian`, `PurchaseContractPrompt`,
  `ContractCostPrompt`, `Stash_TrainingOrderChest`,
  `MenuLayerContractAssignment` ("Contract assignment for faction
  members").
- Corollary for AI-side trait drift: mounts/pets/elements/gangs on
  enemy captains do NOT come from contracts — only spawn/level
  reward rolls (e.g. `Default_HasMount` checkpoints) and event
  reward pools apply. Orders are a player-only lane.

## OPEN

- Member-aging promotion cadence (how often captains promote/upgrade
  post-spawn) — needed to close `epictrait`/`bodyguards` lv80 gaps.
- `humiliator` RESOLVED (INFERRED): not a trait grant — a
  **role transform**. `PS_Humiliate` is the post-humiliation reward
  pool; its sole def `PS_HumiliateExit_Humiliator` carries a
  `Faction/Predicates/RoleLists` ref `RoleCategory_Humiliator` →
  `Roles/Definitions` `Role_Humiliator`, plus the
  `UI_Promoted_NewRole_Humiliator` string. The role record itself
  carries `Traits/Picker` `Picker_Humiliator` → `Quirk_Humiliator`,
  so the transform produces both the epithet and the quirk. Same
  shape as `PS_DuelWin_BrawlMaster`/`ResReward_{Change,Transform}-
  Role_*`/`Reward_StoryRole_Deranged*` (`RoleCategory_*` →
  `Role_*`). `PS_Humiliate_CanChangeRole` chains the exit def.
  TRIGGER (VERIFIED): 31 `*_{Sub,Oth}{_,}Humiliator` outcome events
  (Duel/Execution/Chase/Raid*/Feast/Invasion/Camp/Rescue/Hunt/Spy
  families — all `Replay/Type` `PS_*` = player-story missions, NOT
  AI-vs-AI) carry `PS_Humiliate_CanChangeRole` at cell 16 (the
  resolution reward-pool slot). The pre-death dice are
  `Combat/Stat/Variable/Modifications`
  `Chance_of_Humiliation_Exit_version`:
  `Humiliation_Exit_vs_Stay` = 0.26 (killer humiliates and exits →
  transform), `Humiliation_Minigame_vs_Stay` = 0.51 (stands and
  fights the minigame). `Role_Humiliator` is sticky — live
  humiliators keep the epithet (15/61 also carry shame/deranged
  markers, so prior transforms don't block either).
  Sim: `apply_player_story_reward` rolls 0.26 per player-kill then
  draws the pool; `apply_event_role_reward` covers the mission-
  outcome path (named side wins). Live 61/562 = 10.9% quirk; sim
  ~47% at 400 ticks — residual is the player-kill encounter
  economy (every sim vendetta is player-facing), not the
  mechanism.
