# Spec: Faction member lifecycle — rarity, promotion, resurrection, seating

> **Status:** verified — rarity/promotion/resurrection vs live data
> **Provenance:** live roster census + picker/trait surface diffs
> (`Helper/roster_live.tsv`, `live_traits*.txt`, 562-member endgame
> board, 223-member lv20-43 board), gamedb record reads (`anduril db`),
> `Faction/Resurrection/*`, `Faction/Rewards/Definitions` kill/promote
> defs. Journal: `../MODLOG.md` 2026-10-14..18. Date: 2026-10-19.

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

- NOT creation-fixed: unseated rates climb with level
  (21% ≤20 → 29% ≤50 → 51% @51+), resurrection re-grants it
  (88% undead vs 37% living unseated), and seating strips it
  near-totally (**4% seated vs 44% unseated** — the only ability with
  that signature; BodySlam/ThroatGrab are seat-invariant).
- No record removes it at data level → engine-side seat/lifecycle
  strip. Sim models: level-scaled keep roll (p ≈ 0.13 + 0.0055×level,
  cap 0.9), permanent strip on Warchief/Overlord seat (survives
  demotion), re-grant on undead return.

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

## Andúril model

`anduril-sim` implements rarity epicification, promotion upgrades,
resurrection packages (incl. Unbreakable), and follower grab/throw —
verified against live board diffs. Grab/throw carries a ~10pt residual
(tracked in faction-parity). No toggleable quirks currently.

## SimEvent scheduler (DECODED 2026-11 — schema verified, semantics partial)

The retail tick is **event-scheduled**, not per-captain band-rolled.
`NemesisRules`' flat d100 bands (death 3 / promotion 4 / shame 5 /
vendetta 7 per Alive-roll per tick) are Andúril placeholders; the
decoded retail surfaces:

### `Faction/Events/TimedEvents` (4 recs) — board-refill timers

Named attrs (`db attrs`):

```
cell0  TimeToAddDuringQuest  f32   — sim-time accrued during quests
cell1  MinNumHoles           int   — trigger threshold (empty seats)
cell2  MaxNumHoles           int   — fill ceiling
cell3  Cooldown              (difficulty-keyed entry)
cell4  InitialDelay          (difficulty-keyed entry)
cell5  PromotionEventSetup   ref   — the fill action
cell6  GlobalVarPredicate    ref/unset
```

Records: `CompletelyEmpty` (min 19, max 99,
`PromotionEventSetup:Default`), `HalfEmpty` (2–99),
`FillUpCompletely_LongDelay` (0–99, delay 0.2),
`Resurrection` (delay 3.0, no PromotionEventSetup — the dead-return
timer). Open: whether Min/MaxNumHoles are literal hole counts or
`_StructuresFactionDifficultyEntry` ordinals (cells 1-2 read as
difficulty-entry refs; 19/99 are plausible either way — verify vs
native hole-count compare).

### `Faction/Events/PromotionEventSetup` (4 recs)

`Default` → `Faction/Events/Events` `CreatePromotions` + six
`_StructuresFactionPromotionEventData` rows (ords 12-17). Each row:
`cell0` = rank tier index 0-5, `cells 1..` = int ladder (row16:
`0,0,1,1,1,2,2,3,4`), terminal `Unset` + two `SimEvent` refs:
`PromoteEmpty_Builder_Transform` (ord 137) and
`PromoteDead_Builder_Transform` (ord 261). The int ladder is the
per-tier seat quota/fill order — field names unavailable (struct
arrays carry no attr headers). HardMode/MinasIthil variants exist.

### `Faction/Events/SimEvent` (431 recs) — the job unit

`PromoteEmpty_Builder_Transform` shape (attr-keyed):

```
cell0   difficulty entry ref     cell5  0x4c const
cells12-15 event refs: Simulate=`PromoteEmpty_BuilderTransform_Simulate`
cells13,15 job defs `PS_Other`/`PS_Subject`
cells16-17 `SimEventParticipant` refs (_01/_02)
cells18,20 outcome events `PromoteEmpty_Subject`/`PromoteEmpty_Other`
cell25  `SimEventChance` ref (here: `SubjectAlwaysWins`)
cells28-29 `..._Clear` / `..._End` events
cells30-35 struct rows: AssignJobGroupPicker, WUIPresentationPicker
  ×2, SimEventResult, SimEventPortrait, SimEventSimulateOption
```

`Promote{Empty,Dead}_Builder_Transform` = seat refill: subject is the
empty seat, a `Builder`-made captain transforms in — i.e. **the board
refills by spawning new captains into holes**, matching the TimedEvents
hole thresholds.

### `Faction/Events/SimEventGenerationSetup` (18 recs)

`Default_Warchief`/`Default_Overlord`/`{Region}_Captain` hold
`_StructuresFactionSimEventGenerationList` rows →
`SimEventGenerationType` refs (Betrayal, Feast_UnfinishedBusiness,
SpawnConditions{,_KillMissions}, Location, FortAssault{,_Unfinished-
Business}, NeverEndingShadowWar) — the **event categories allowed to
produce each rank**, weighted via
`SimEventGenerationTypeWeight` (PowerStruggle, Hunting + _Act2).

### `Faction/Events/SimEventChance` (34 recs) — outcome tables

Per-event outcome weights, f32 cells: e.g. `Default` =
`[0.15, 0.05, ?, -1, 0.5, ?, -1, 0.15]` with trailing
`SimEventResultType`-ref cells; `SubjectAlwaysWins`/`SubjectAlwaysLoses`
are degenerate 0/1 rows; `Default_2orc_captive` = `[0.25, ?, ?, 1.0,
0.9, ?, 0.2, 0]`. Field names undecoded — likely Subject-win /
Subject-flee / draw / Other-win band cells. `FortAssault`/
`FortDefense`/`SubjectAlways*` use sentinel last-cell refs
(`0x6d6018`/`0x38f8` = record refs, not ords — decode pending).

### Cooldowns/lifespan

`SimEventCooldownActivate` = per-activation tier rows (`[0,1,2]`
+ ref); `SimEventCooldown` records are empty-cell stubs;
`Difficulty/SimEventLifeSpan` has 2 diff-adjust rows (empty cells —
values live in the difficulty entry structs, undecoded).

### Implication for `anduril-sim` (next implementation axis)

- Death/shame/vendetta churn should flow through SimEvent **jobs**
  with cooldowns, not per-captain-per-tick bands — explains the
  t80 91% churn vs retail's stable board.
- Board refill: per-region hole counting → `PromoteEmpty/Dead` →
  spawn new captain via builder (`generate_captain`) seated at the
  vacant rank; TimedEvents cadence gates the checks.
- `SimEventChance` outcome tables replace flat duel/contest odds.
- Resurrection timer 3.0 gates cheat-death returns.

### Andúril scheduler slice (implemented 2026-11)

`NemesisRules.region_jobs > 0` switches `nemesis_tick` to the
event-scheduled model (`schedule_board_jobs` + `board_promotions`,
`crates/anduril-sim/src/nemesis.rs`):

- Alive-captain `death`/`promotion`/`shame`/`vendetta`/`rivalry`
  bands are skipped; ambient `hunting` stays per-captain, and
  Dead/Shamed return rolls (`cheat_death`/`deranged`) stay — they
  model the `TimedEvents` resurrection timers.
- Per canonical board, while concurrent missions <
  `region_jobs` (CLI runs `region_jobs=2`), a `region_job_chance`
  roll (default 40 — INFERRED cadence; the retail
  `SimEventGenerationType` cells-0-2 units are undecoded, so the
  knob is the mod-reachable stand-in) spawns one job.
- Context pick follows board+story state: `Revenge` when the
  subject has a live `nemesis_foe` on board (its 999 params ≈
  never ambient — feud-triggered), `Vendettas` when a `Player` is
  present (fixed share = `vendetta` knob), `Prisoner` when the
  board holds a shamed captive, `PowerStruggle` otherwise.
- The record resolves via `draw_generation_board`, the cast via
  `cast_mission_slots`, the kind via `classify_simevent`
  (`*_Vendetta` records in a board context stay Duels); outcomes
  land in `mission_tick` as before.
- `board_promotions` fills capped-tier vacancies from live seat
  counts — highest-power living Captain on the board promotes into
  open Overlord→Warchief seats with the same rarity-upgrade
  side-effects (`Orc_Upgrade_ToEpic`,
  `OverlordSetup_LegendaryUpgrade_*`, follower strip). This is the
  `PromotionEventSetup` ladder modeled at tier granularity.
- Legacy bands remain the default (`region_jobs = 0`) for
  synthetic worlds without `SimDb`; `sim nemesis`/`sim resume`
  enable the scheduler. Rules persist in snapshots.

Not yet modeled: `SimEventChance` outcome tables on resolution
(missions still use power-weighted contests), per-event cooldowns,
`SimEventLifeSpan` durations, `TimeToAddDuringQuest`, and the
`SimEventGenerationSetup` per-rank allow-lists (context choice is
state-driven).

### `SimEventGenerationTypeWeight` (4 recs) — DECODED 2026-11

Each record's cells are class-`0x8c0` refs to unnamed
`_StructuresFactionSimEventGenerationTypeWeight` rows shaped
`{f32 weight, Difficulty/VariableFloat name}`. Verified contents:

| record | rows | VariableFloat names |
|---|---|---|
| `PowerStruggle` | `[1,2,3,4]` | `SimEvent_PowerStruggle_Weight{1,2,3,4+}` |
| `PowerStruggle_Act2` | `[1,2,3]` | same |
| `Hunting` | `[1,2,3,4]` | `SimEvent_Hunting_Weight{1,2,3,4+}` |
| `Hunting_Act2` | `[1,4]` | `Weight1`, `Weight4+` |

INFERRED semantics: `WeightN` is the spawn weight of the Nth
simultaneous job of that type — boards get *busier* as jobs
accumulate (`4+` flat past the third). Andúril consumes it as
`generation_weights`: job `#active+1` rolls
`region_job_chance * w[active]/w[0]`; contexts with no weight
record (Revenge/Vendettas/Prisoner) roll flat.

### XPLevel row draw semantics (VERIFIED — `FUN_14034f9f0` + `FUN_140352dc4` decompiled 2026-11)

The menace-level setter calls `FUN_14034f9f0`, which iterates levels
`stored_level ..= menace_level` (`member+0x4c0` vs `+0x4b8`) and then
writes `+0x4c0 = +0x4b8`. Consequences:

- Generation zeroes `+0x4c0` → the whole `0..=spawn` range backfills.
- Every subsequent level-up re-processes `[old, new]` **inclusive of
  old** — the checkpoint at the level being *left* fires once more.
  (Andúril: `ck` in `[old_power, new_power]`; the re-fire is mostly
  idempotent through def gates but matters for `Nothing` weights and
  quantity rows.)
- `FUN_140f847a0` supplies an early-exit bound (`if bound < level:
  break`) — consistent with the observed menace cap of 80.

Per level, `FUN_1403513dc` walks the bound-bundle row list and calls
`FUN_140352dc4(row, level, ctx_record, …)`, which for each candidate
in the row's `Rewards/Lists` pool:

1. dedups against already-picked (`param_5`) and already-owned
   (`param_4`) reward refs — **one pick per pool per call**;
2. reads `flag` (`FUN_1403530f0`): when set, the row's `lo`/`hi`
   bounds are remapped through `FUN_140352f8c` — a `{lo,hi,delta}`
   band table resolved from `member+0x280` via `FUN_141156510`
   (`bound += delta` when `lo <= bound <= hi`, clamped ≥0). So
   **flag=1 is a level-bound remap marker, not a probability gate**;
   stock correlation with level-up rows holds because act/difficulty
   remapping applies to them. The `+0x280` context record is not yet
   decoded — the remap itself is UNMODELED;
3. effective band is `level >= lo-1 && level <= hi-1` (stored fields
   minus one, remapped when flagged);
4. probability `p` (`FUN_140353d54`): auto-pick when
   `p >= 1.0` (global `DAT_14202e5a0`), else `U(0,1) <= p` —
   a single Bernoulli draw (`FUN_1401b3c6c`, LCG top-24 × 2^-24);
5. first passing candidate is appended and the row's scan `break`s.

Each picked reward then goes through the picker roulette
`FUN_140350500` (weight = `FUN_1403505f4`, eligibility =
`FUN_140105aec` — the IncludedTraitMarkerList/Excluded* filter with
`[FACTION|REWARD]`/`[FACTION|REWARD|REJECT]` debug logs) and applies
via `FUN_140351af8`.

### `PickerLevelRemap` — VERIFIED + MODELED 2026-11

`Faction/Traits/PickerLevelRemap` (5 recs) is the `member+0x280`
context record the `flag=1` branch of `FUN_140352dc4` resolves via
`FUN_141156510`. Each record is a `{hi, lo, delta}` band table
applied by `FUN_140352f8c`: first row with `lo <= bound <= hi`
returns `bound + delta` (clamped ≥0). Decoded rows (real merged db):

| record | bands |
|---|---|
| `TraitLevelOffset_0` | `[99,0,0]` — identity |
| `TraitLevelOffset_-1` | `[10,5,-1] [20,10,-2] [30,20,-3] [45,30,-5] [60,45,-7]` |
| `TraitLevelOffset_-2` | `[10,3,-1] [20,10,-3] [30,20,-5] [45,30,-10] [60,45,-15]` |
| `TraitLevelOffset_+1` | `[5,0,+1] [10,5,+2] [20,10,+4] [30,20,+6] [60,30,+9]` |
| `TraitLevelOffset_+2` | `[5,0,+2] [10,5,+4] [20,10,+9] [30,20,+15] [60,30,+21]` |

INFERRED tier selection: the captain's drawn `XPLevel/Delta` spawn
offset clamps into `[-2,+2]` → record name (a captain spawned below
baseline gets its trait checkpoints pulled earlier — a catch-up
mechanic). Andúril stores `Captain::xp_offset` (snapshot-persisted)
and `Nemesis::level_remap` applies the table to every `flag=1`
row's `lo-1`/`hi-1` bounds, generation and level-up alike. The
exact mapping from member state → offset record (delta vs.
difficulty vs. act) is still `blocked` — the `+0x280` writer was
not traced.

### Retail job pipeline (VERIFIED — decompiled 2026-11)

The real SimEvent job path, traced from the Faction script-API
table (`FUN_141b96e54` — ~90 name→impl pairs, the authoritative
mod-callable event surface):

1. **Request** — `FUN_141b5a47c` (`AddFactionEventJobSourceAndOther`)
   / `FUN_141b5b610` (`CreateAndProcessFactionEvent_Job`): resolves
   subject + other via `FUN_141bd2a54` against the region's member
   store (`faction_mgr+0x98` → region → `+0x20`), builds a request
   `{subject id @member+0x174/+0x178, event record, other member
   (nullable, flags @+0x51b), req fields}`, pushes to the board's
   queue at `mgr+0x1f0`.
2. **Pump** — `FUN_14040c0ac` drains `+0x1f0` per tick (reentrancy
   flag `+0x208`; queue empty → `FUN_1411596d0`), calling
   `FUN_14040c13c` per request.
3. **Instantiate** — `FUN_14040c13c`: dedup check
   (`FUN_14040ca40`, "same event already active?"), enqueue at
   `mgr+0x110`, notify the area record (`FUN_14018de04` +
   `FUN_14062e670`), then iterate the event's job defs — job kind
   `0xd` casts the OTHER member onto the board via
   `FUN_141b5f33c`; a second path builds a `0x1b`-type deferred
   struct when `FUN_14040c298` says the event defers.

Member identity is `{u32 @+0x174, u64 @+0x178}` (consistent with the
ScanMember +0x1f8 bundle-list shape). `mgr+0x1b8` is the board/faction
context shared across the pump, the API bindings, and job casting.

Our scheduler's `schedule_board_jobs` models steps 1-3 at the
generation-context level (request ≈ `cast_mission_slots` spawn);
the request-queue/dedup/instantiate split is not separately
modeled — dedup exists implicitly via `drawn`/active-mission scans.

`SimEventGenerationType` leading cells are a **variable-length,
heterogeneous param vector** — everything before the first ref cell
(weight record / GlobalVarPredicate / SimEvent refs). Verified
shapes on the merged db: `PowerStruggle`/`_Act2` = three f32
`[10,7,4]`; `Vendettas` = `[1.0, 0, 0, -1, -1]`; `DeathThreats` =
`[1.0, 2, -1, -1]` (int param!); `SpawnConditions` =
`[4,3,2,1,1,1,0,0]`; `Prisoner` = six `1.0`s; outposts/Gauntlet =
`1000.0`s (≈never); `0xFFFFFFFF` = unused slot. Andúril stores the
full vector as `generation_params_raw` (the `[f32;3]`
`generation_params` view stays for the Lua API). Units/consumer
still `blocked` — the periodic generation driver that reads them
is above the request path and untraced.

## JobList / JobSetup pipeline (retail, decompiled)

`SimEventGenerationType` records are the engine's **JobLists**;
`GenerationSetup` records are **JobSetups**. The retail chain:

1. **Trigger** — `FUN_1405769a8` ("FadeOut_OnEnter" region-enter and
   queued timed events on the board object `+0x1c8`, popped by
   `FUN_1405756bc`) resolves the active setup via
   `FUN_14016e1d0`/`FUN_1405768e4` and calls `FUN_141b5e19c`
   ("Processing Job Setup"). There is no naive per-frame generation
   tick: setups fire on region-enter and on queued timed events.
2. **JobSetup** — `FUN_141bd4790` collects supporter candidates
   from the board member list (role checks `FUN_14114dfe8`/
   `FUN_14114e020`, allegiance `+0x178`, alive `FUN_141159830`),
   then evaluates each job list and adds supporters
   (`FUN_141bd5348`), then processes each job list via
   `FUN_141bd4674`.
3. **JobList eval** — `FUN_1405767a8` = `[QUEST|JOBROLL]`:
   resolves the record's chance cell (`FUN_1416cee38`/
   `FUN_14030e7a4` name lookup + `FUN_14030ecd8` read) then
   `U(0,1) <= chance` — one Bernoulli per JobList per attempt.
4. **Job pick** — `FUN_141bd4428` / `FUN_1405763c4`
   ("AddJobForMemberFromList"): iterate the list's job records,
   keep those with weight `>= DAT_14202e9b8`, weighted-random
   (or deterministic-best) pick via `FUN_1401b3b90`, default-job
   fallback `FUN_141ace168`, assign via `FUN_1404fc51c`.
5. **Event fill** — `FUN_14061ef8c` ("[SIMEVENT] Evaluating
   Event"): draws a random delay `lo + U·(hi-lo)` from a range
   cell pair (`FUN_14061fe4c`), stores `delay × DAT_14202e464` at
   job-state `+0x90`; per job-def runs the fill-roll constructor
   `FUN_14061f604` ("will be filled"/"NOT be filled", flag `0x10`),
   assigns participants via `FUN_141bd2a54`, rejects on missing
   required participant, then picks rewards per participant via
   `FUN_140350500` at fill time.

The GenerationType leading param vector remains `blocked`: the
chance cell is resolved *by name* (`FUN_1405768c8`/`FUN_1405768d8`
provide the lookup key), so the `[10,7,4]`-style cells are most
plausibly named scalars consumed by the JOBROLL/setup predicates,
not positional fields. The `[SIMEVENT]` delay range is a separate
named cell pair on the SimEvent record, not the param vector.

The GenerationType "param vector" is fully resolved by the record's
**layout attributes** (named fields, not positional cells):

| attr | kind | role |
|---|---|---|
| `Weights` | f32 vec | per-Nth-concurrent-job weight curve; `0` = concurrency cap. `PowerStruggle=[10,7,4,0]`, `SpawnConditions=[4,3,2,1,1,1,0]`, `Vendettas=[1,0]`, `DeathThreats=[1]`, `Prisoner=[1×6]`, `Gauntlet=[1000,1000,1000,0]` |
| `ZoneMustBePlayerOwned` | int | zone-ownership gate (all retail = 0) |
| `WeightOverride` | ref | GenerationTypeWeight record that **overrides** the inline `Weights` curve (`PowerStruggle`/`Hunting` → `[1,2,3,4]`, Act2 variants) |
| `GlobalVarPredicate` | ref | act/story gate: `PowerStruggle`/`Hunting`→`Act1`, `Prisoner`→`Act2`, `SpawnConditions`→`WarchiefsUnlocked` |
| `SimEvents` / `SimEventsHighPriority` | refs | event candidate lists (`SpawnConditions` high-priority = `MatchOverlord`) |

Andúril decodes by attr name via `record_attrs` + `attr_name`;
delta-added records (no layout) fall back to a semantic-ref-prefix
heuristic — f32 params bit-pattern like tagged refs, so the prefix
ends at refs into the JobList semantic lists only. `generation_weights`
= `WeightOverride` curve when present, else the inline `Weights`
vector. New maps: `generation_predicates`, `generation_zone_owned`,
`generation_priority` (`generation_params` keeps the first-three-
float projection for the Lua API). `db rec` now prints the attr
name per cell — the names were in the schema dump all along.

`SimEventChance` records decode by attr name into
`SimEventChanceInfo` (`NemesisEvents::chance_info`):
`StartChance` (the `chances[..4]`/`win_chance` value — verified
same cell), `SupporterDeltaScalar`, `CharacterLevelDeltaScalar`,
`HenchmanDeltaScalar`, `TotalInteractionScoreDeltaScalar`,
`PlayerFollowerTotalISDeltaScalar` (each + `Max` bound),
`ClampFinalChance` (kind-0x07), and `TraitPairs` —
`_StructuresFactionSimEventChanceTraitCombination` rows
`{f32 delta, TraitList A, TraitList B}` (e.g. `Ranged_Has` vs
`Ranged_Terror` → 1.0). Formula INFERRED: `StartChance +
Σ(participant deltas × scalar) + Σ(matched trait-pair deltas)`,
clamped — consumer not yet traced.

**Consumed (2026-12)** — `NemesisEvents::outcome_bias` now scores the
Duel resolution roll through `SimEventChanceInfo::subject_chance`:
`level_delta` = subject−other power, `supporter_delta` = living
bodyguard-count difference, `trait_pair_delta` = Σ deltas of rows
where the *subject* holds a trait in list A *and* the other holds one
in list B. Trait-pair direction is inferred (`Ranged_Has` subject vs
`Ranged_Terror` other); `HenchmanDelta`/`TotalIS`/`PlayerFollowerIS`
are undecoded stats and contribute 0. `Faction/Predicates/TraitLists`
decodes to `trait_list_members` for membership evaluation
(`Ranged_Has` = `{Power_Shot, Marker_Is_Archer}`, verified on the
real db). IS terms clamp to their `*Max`; final chance clamps to
[0,1]. The retail outcome consumer itself remains untraced —
`[SIMEVENT|OUTCOME]` (`FUN_14040c304`) turned out to be the result
*action* dispatcher (MOVE/Henchmen/ClearBoss command codes), not
winner selection.

`SimEvent` record attrs also decoded by name into `SimEventInfo`:
`Type` int, `LifeSpanGenerated`/`LifeSpanPlayerCreated` int enums
(`Duel`=3/3 → `LifeSpanDifficulty`=`SimEvent_LifeSpan3_DiffAdjust`),
`RequiredCooldown` (`SimEventCooldown` ref),
`WinChance`/`WinChanceJobSubject`/`WinChanceJobOpposition`
(`Quest/Jobs/Definition` refs — `Duel`=`PS_Subject`/`PS_Other`),
`UniqueSimEventGroup`/`UniqueSimEventGroupPerFaction`. Other
verified attrs (unstored): `EventOnSimulate`, `EventOnClear`,
`EventOnQuestEnd`, `EventOnAssignSubject/Other`, `EventOnClear`,
`CanStealQuestFrom`, `ClearExistingSimEventGroup`,
`ClearExistingSimEventResultType`, `MaxParticipantsNewSimEvent`,
`OtherSpecificPurposes`, `SubjectSpecificPurposes`,
`PreferredSimEventTags`, `ResultPickerJobSubject/Other`,
`PlayerCreateHeader`, `PlayerCreateDescPicker`,
`NoQuestTimeout`, `SimulateOnQuestTimeout`,
`CanSimulateWithAllUnknown`, `ConfirmWUIPresentation`,
`CreateWUIPresentation`,
`PlayerAssignmentJobGroupPicker`, `PortraitSettings`.

`SubjectWinsSimulate`/`OtherWinsSimulate`/`Results` are decoded into
`SimEventInfo` and **consumed**: the simulate-option rows are
`{f32 weight, SimEventResultType ref, optional VariableFloat}` —
VERIFIED `DuelFromPlayerKiller` subject-wins → `OtherKilledBySubject`,
other-wins → `SubjectKilledByOther`; plain `Duel` other-wins →
`SubjectCaptured` (+`FailedPowerStruggle_Captured` var). `Results`
rows are `_StructuresFactionSimEventResult`:
`{Str recap_presentation, SimEventResultType ref, Unset,
JobEventPair…}` — the event's legal outcome table (e.g. FeastSingle:
`SubjectSucceeded` + the `SubjectKilledByPlayer/Orc/Goblin` death
variants). When both simulate names decode, `resolve_duel` emits the
retail result name on `mission_resolved` instead of the
kind-mapped default — so a simulated Duel loss reports
`SubjectCaptured` as retail does. Status effects (dead/shamed) still
follow our fatal/shame rules; the weight/aux cells and
`JobEventPairs` rows are not yet stored.

`SimEventParticipant` records decode by attr name (`EventRole`
named fields): `EventOnAssign` = the role record (`Duel_Subject` →
`Subject`, same value the class-`0x259` decode produces — verified
equivalent), `JobRecord` = the job def (`PS_Subject_Cap_Duel`),
`Required`/`Exclusive`/`CanBeBodyguard` kind-0x01 slots
(`0xFFFFFFFF` = unset), `ExclusiveMustBePlayerFollower` int,
`JobGroup`/`RewardList`/`AbortCondition`/`PositionOnly` refs,
`Purposes` ref list, and `ActivateCooldownOnFinish` — the
`SimEventCooldownActivate` record started when the participant's
event finishes (`Duel_01` → `PowerStruggle`, which chains to
`PowerStruggle_CD`; the cooldown *activate* record maps event
names to cooldown records).

`Difficulty/SimEventLifeSpan` named records (`SimEvent_LifeSpanN_
DiffAdjust`) are **cell-less selector stubs** — the unnamed
`_StructuresDifficultySimEventLifeSpan` table maps a small int
(difficulty enum) → which `_DiffAdjust` record applies; the actual
duration values live behind the Difficulty system (`blocked` —
durations not yet located).

### Cooldown lifecycle — consumed in the sim (2026-12)

The two edges now form a closed loop in `anduril-sim`:

1. **Activation** — when a mission *resolves* (not aborts; "OnFinish"
   is the completion edge), `mission_tick` starts a
   `NemesisCooldowns` group per role `activate_cooldown`
   (`ActivateCooldownOnFinish`). Verified chain: `Duel_01` →
   `PowerStruggle` (activate record) → `PowerStruggle_CD` (cooldown
   record). `NemesisEvents::cooldown_activate` decodes the join
   table — each `SimEventCooldownActivate` record's
   `Faction/Events/SimEventCooldown` ref.
2. **Suppression** — `schedule_board_jobs` refuses a JobList context
   while a group of the same name is active (activate record names
   match the JobList they suppress: `PowerStruggle` ↔
   `PowerStruggle`), and refuses any drawn SimEvent whose
   `RequiredCooldown` names a cooldown record an active group maps
   to (joined via `cooldown_activate`).
3. **Expiry** — groups tick down in `nemesis_tick`; retail durations
   live on the cell-less `SimEventCooldown` stubs (`blocked`), so
   `NemesisRules::cooldown_ticks` (default 8) supplies the length.
   `0` disables the whole cooldown system.

Synthetic coverage: `simevent_cooldown_lifecycle` exercises
activation-on-resolve, context suppression, the `RequiredCooldown`
gate via a differently-named group, and expiry. The activate→cooldown
join is asserted on the real db in
`simevent_participant_attrs_decode`. Unmodeled: the `CooldownActivate`
int cells (`[0,1,2]` — semantics undecoded) and whether retail
cooldowns are per-faction, per-region, or global.
