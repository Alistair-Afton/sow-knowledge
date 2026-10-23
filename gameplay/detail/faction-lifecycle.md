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

## `Follower_GrabAndThrow` — follower-eligibility (VERIFIED)

- The picker gate is `RequirePlayerFollower=True` (VERIFIED record
  field on `Picker_Follower_GrabAndThrow`) — the trait only lands
  on follower-eligible members. It is not a random keep and not a
  seat strip: board-fill spawns are simply rarely follower-flagged.
- Rates on the live roster (n=562): fresh unseated **42%**
  (100/240), fresh seated **18%**, unseated returners **68%**
  (79/116), seated returners 38% (5/13). Returns re-grant it;
  RaiseDead holders among seated keepers 3/39 vs 0/166 stripped.
- The 18%↔42% seated gap is the follower-flag share of each fill
  path, flat across seat tiers and classes.
- `Default_Act1` members carry it less (30% vs 46%) — Act 1 fill
  is non-follower. `Quirk_IronWill` anti-correlates (3/140 vs
  53/100) — INFERRED same-path exclusivity. The fresh-unseated
  region spread (18% MinasIthil → 55% desert) tracks each board's
  fill-path mix; the mechanism behind which spawn is flagged
  follower-eligible is UNVERIFIED.
- Sim models: flat eligibility roll p≈0.42 at spawn, seat keep
  0.43 (≈ seated follower-flag share), re-grant on return.
  Residual region spread UNVERIFIED at mechanism level.

## `MustBePlayerFollower` — reward allegiance gate (VERIFIED)

Every `Rewards/Definitions` reward carries a `MustBePlayerFollower`
gate: census over 1,239 defs — 1,099 `=0` (don't care), 132 `=2`,
8 `=1`. The split is semantic (VERIFIED by member names): `=1` marks
follower-only rewards (`Betrayal_CheatDeath_Ambush`,
`FollowerLoyal_CheatDeath_Ambush`, `Reward_StoryRole_Reject`) — things
only a dominated captain can receive; `=2` marks enemy-only rewards
(`Champion*`/`Mighty*`/`PS_*` titles, `AI_BreakPlayerSword_Breaker`,
`BloodBrother`) — a dominated captain stops drawing them. So
domination doesn't just change side: it swaps the member into a
different reward space.

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
hole-timer path refills it.

**Seat assignment serializes per member — VERIFIED (save
WHDC):** each member's post-tail section carries a
`<region>_Row_<name>` ref alongside `Faction_<region>` and
`AI_<role>`. Retail endgame census across the six region
boards (MtnPass, volcano, snowMtn, island, desert,
MinasIthil): Captain_1/Captain_2 carry the most refs
(9–19/board), Outer_Warchief 6–11, Captain_0 6–11,
Inner_Warchief 4–7, Overlord 3–4. Counts exceed seat caps
roughly proportionally — INFERRED the row name marks the
member's *assigned* row and persists on roamers/unseated
members rather than denoting current occupancy.

**The caps are schema, not just occupancy — VERIFIED
(`Faction/Rows` `ColumnMax` attr, all 83 records):** Row records
are variable-layout attribute lists; `ColumnMax` is the seat cap.
Every standard region row set reads exactly the table above, with
two real exceptions — `MinasIthil_Row_Outer_Warchief` (and its
`DLC3_` twin) = **5**, and Desolation of Mordor's desert board
adds `Row_Mercs_0/1/2` = 5 each (mercenary seats, DLC4-only).
`GoldenPath_Zog_Carnan_{1,5,6}` rows cap at 1 (story boards),
`player_created` caps at 32 (player-appointed bodyguards),
Prototype/demo rows at 1. Other decoded Row attrs:
`ResurrectionEnabled` (bool — VERIFIED `false` only on `Prototype_*`, DLC4 `Mercs_*`, and `player_created` rows: mercenary and player-appointed seats never resurrect; every standard-board row is `true`),
`HenchmenRows` (VERIFIED ref list: every Warchief/Overlord
row links to its region's three `Row_Captain_*` rows — the seats
a boss's henchmen occupy; `DLC3_MtnPass_Row_Overlord` and
`DLC_MinasIthil_Row_Overlord` carry an empty list, no henchmen), `RewardListOnEnterRow` (VERIFIED packages rolled on seat
entry: captain rows → `Default_BloodBrother` — a
spawn-duplicate def (`CopyAndHideFactionMemberData`,
`SetRoleFromFilter`), i.e. sitting in a captain seat can
spawn a paired blood brother (gated on
`Marker_Generate_BloodBrother`, consumed on fire — the marker is
granted at member generation: `Setup::Default`'s
`PostRoleMarkerTraitLists` roll `MarkerList_HasBloodBrother`,
~7% weighted `[Marker_Generate_BloodBrother w0.075, nothing w1.0]`,
forbidden to deranged/butterfly members; ~2% of the live roster
carries it pending, i.e. granted but never captain-seated. A
second channel exists — `RewSub_SubSucOth{Fled,Died}_Backup`
Events fired by `SimEventResultJobEventPairs` on 'subject
succeeded, other died/fled' results roll the `BloodBrother`
spawn-def directly, ~15% vs `Nothing_Weight_85`. The pair rows
bind each sub-event to a job slot — `PS_Subject`/`PS_Other`
name existing participants while `PS_Backup2_05`/`PS_Ambusher_05`
are spawn slots: the sub-event spawns its own member, the same
mechanism behind the `RewSub_SubSucOthDied`/`RewOth_*` backup
and ambusher reinforcements);
warchief rows →
`Default_TerrorBodyguardDefeat` — a pick-1 weighted draw
over `SW_Terror_/Hate_/WeakBy_BodyguardDefeat`+
`Nothing_Weight_40` (Shadow-Wars-gated); overlord rows →
`OverlordSetup_LegendaryUpgrade_{Common,Epic}Orc` +
`Default_EpicTraits_PickRemovable` +
`Contract_LegendaryTraining_MarkerSwap` +
`Overlord_ElementalVent_Resistances`), `InitialLevels` (VERIFIED struct: a list of
`_StructuresFactionInitialXPLevel` candidate rules, each =
`Difficulty_Captain_BonusLevels` base + `GlobalVarPredicate`
gate + optional `PlayerBasedXPLevel/Delta` ref — spawn level
is difficulty-scaled and can go player-relative; gates seen:
`Act1`, `DLC3_Finale_Completed`, `ShadowWarUnlocked`; deltas
incl. `PlayerLevel_Plus5`, `Plus5_To_Minus15`,
`Plus15_To_Plus0`, `Minus5ToPlus5`, `1_to_2_levels`),
`HireCost` (merc rows), `HenchmenAddChance` (VERIFIED:
0.35 on every Warchief/Overlord row, 0 elsewhere — the
boss-row rate for gaining henchmen), `CanBeRevengeTarget`
(VERIFIED `false` on every Warchief/Overlord row — boss-tier
members are never drawn as revenge targets),
`RequiredDataLevelToSpawn` (VERIFIED: 1 on every
`Row_Overlord` + prototype/demo rows, 0 on captain/warchief
rows — overlord seats additionally require the later data
level, independent of `OverlordsUnlocked`),
`GlobalVarPredicate` (per-row story gate —
VERIFIED map: `MtnPass` Inner/Outer Warchief +
Overlord carry `WarchiefsUnlocked`/`OverlordsUnlocked`,
`island` Overlord `OverlordsUnlocked`, `MinasIthil` Outer
Warchief + Overlord `IntroComplete`; `snowMtn`, `volcano`,
`desert` rows are ungated; the `DLC3_*` mirror rows repeat the
pattern). Locked rows seat nobody until their flag flips.

**Blood-brother avenger (VERIFIED catalyst chain):** the
`BloodBrother_SpawnIn` relationship is driven by trait
`Quirk_BloodBrother_Killed` → catalyst
`Quirk_BloodBrother_KilledCatalyst` → procs
`Proc_{Player,AI}KillBloodBrother`, which fire when the killer of a
member's linked brother strikes — the proc's
`TargetMustBeInRelationship=BloodBrother_SpawnIn` condition is what
makes it brother-specific. The catalyst result
`BloodBrother_BetrayPlayer_KilledBrother` grants items
`CounterBlock_1`, `Item_Immune_GrabKnockdown`,
`EventItem_I_Am_Enraged`, buff `AI_Rage`, and
`FleeSetup=BetrayPlayer_BloodBrotherKilled`: **a dominated follower
whose blood brother the player kills betrays and enrages** (the
classic "you killed my brother" moment); an AI-kill enrages without
the allegiance flip. Brothers also surface as ambushers —
`ActiveBecame_Maniac`/`ActiveBecame_DerangedNonManiac` Events draw
their `AmbusherFromRelationship` from `BloodBrother_SpawnIn`, and a
`Quest/Jobs/Relationship/BloodBrother` quest-job exists
(`MustBeSource=0`).

The full `Faction/Traits/Relationships` census is five records
(VERIFIED): `BloodBrother_SpawnIn`, `Script_BloodBrother_SpawnIn`,
`BloodBrother_BazGaz` (Baz/Gaz DLC twins), `Hate_Nemesis` and
`Terror_Nemesis` — the nemesis-rival bonds. Reward defs carry
`RelationshipToAdd` for all five: `HateNemesis`,
`Default_Nemisis_{Hate,Terror}`, `Sub{Suc,Fled}_Ambu_{Hate,Terror}Nem`
(ambusher sub-events bind the spawned avenger's nemesis), and
`ResReward_{Hate,Terror}_Rival` in the resurrection package — a
captain who cheats death comes back hating his killer.

`Default_Nemesis` is also a `Faction/Rewards/XPLevel` bundle: a
flag=0 creation-time row at Chance≈0.05 (VERIFIED), so roughly
one in twenty captains spawns already carrying a
`Default_Nemisis_{Hate,Terror}` nemesis bond. The nemesis axis is
player-facing — `Rival_*` quest-jobs, `PS_Optional_Terror_Nemesis`
objectives, and the `BrotherKilledBy*` event forks all face the
player — so these defs bind the member's nemesis to the player,
distinct from the member-vs-member binds of the sub-event defs
above (INFERRED direction there). Matching
`Quest/Jobs/Relationship/Rival_{Hate,Terror}` quest-jobs exist, as
do `CombatCritElementState` and `ObjectivePicker` presentation
records for both rival types.

`Quest/Jobs/Relationship` holds five records (VERIFIED), each
carrying only a `RelationshipType` ref: `BloodBrother`,
`BazGazBloodBrother`, `Script_BloodBrother_OutOfHierarchy`,
`Rival_Terror`, `Rival_Hate`. They are the endpoint half of
`_StructuresQuestJobRelationshipPair` (120 rows) — pairs that bind
a mission job slot (`PS_Subject`, `PS_Other`, and scripted story
captain jobs like `BladeSide5_Captain2`, `She2_Captain3`) to a
relationship job. That is the authored channel that casts the
subject's blood brother or nemesis rival into a mission slot.

The avenger delivery channel is also authored:
`Faction/Traits/RelationshipEventPicker` holds one record,
`BloodBrother_SubjectDeath`, with two entries forked on killer
type (VERIFIED) — `OutsiderCharTypeIncluded=All_Orcs` +
`OutsiderMustBeFactionMember` fires `BrotherKilledByOrc`, while
`Player_Character` fires `BrotherKilledByPlayer`.
`BrotherKilledByOrc` sets `UseThisFactionMemberForAmbusher` (the
avenging brother physically arrives as an ambusher) and carries
`GearTagWeight_bloodBrother` loot tags. `Faction/Traits/
RelationshipTypes` groups the five bonds into two categories,
`BloodBrother` and `Nemesis`.

A member's rank and seat class agree on the live roster — no living
captain-tier member holds a Warchief/Overlord seat and vice versa
— so a demoted/death-returned member re-seats into its new tier's
rows (or roams) rather than keeping the old seat (INFERRED from
the absence of rank/seat mismatches; the demotion-time seat swap
itself is not directly observed).

## Promotion cadence (VERIFIED record graph — timing INFERRED)

Seat filling is not continuous: `Faction/Events/TimedEvents` holds
four timer records — `HalfEmpty`, `CompletelyEmpty`,
`Resurrection`, `FillUpCompletely_LongDelay` — each carrying a
`PromotionEventSetup` ref. `HalfEmpty` and
`FillUpCompletely_LongDelay` point at `PromotionEventSetup/Default`
(`MinasIthil` and `HardMode` variants exist), whose
`Faction/Events/Events` payload is `CreatePromotions` — the
record that spawns the `Promote{Empty,Dead}_Builder_Transform`
SimEvents that actually seat a promoted member. The TimedEvent
records' decoded cells (raw values, VERIFIED):

| record | TimeToAddDuringQuest (f32) | MinNumHoles | MaxNumHoles | Cooldown | InitialDelay |
|---|---|---|---|---|---|
| HalfEmpty | 0.2 | 2 | 99 | 3,223,880 | 6,709,980 |
| CompletelyEmpty | 0.2 | 19 | 99 | 16,748 | 16,748 |
| FillUpCompletely_LongDelay | 0.2 | 0 | 99 | 6,708,720 | 6,708,720 |
| Resurrection | 3.0 | 0 | 0 | 1,848,440 | 1,848,440 |

`MinNumHoles`/`MaxNumHoles` are u32 thresholds — `HalfEmpty`
fires once ≥2 seats stand empty; `CompletelyEmpty` needs ≥19
(roughly the whole captain layer). `TimeToAddDuringQuest` is
f32 — 0.2 on every promotion timer vs 3.0 on Resurrection —
consistent with a quest-time acceleration factor, not a
duration. `Cooldown`/`InitialDelay` are u32s. The units stay
UNVERIFIED but **milliseconds gives a coherent cadence
family** (INFERRED): CompletelyEmpty ≈ 16.7 s (urgent
refill when the board is gutted), HalfEmpty ≈ 54 min,
FillUpCompletely_LongDelay ≈ 112 min, Resurrection ≈
31 min — sensible retail pacing where urgent states fill
fast and routine promotion runs on the hour. Frames@30
(20–62 h) is implausibly slow; engine-day fractions
don't fit the ratios. `Resurrection` is the
timed-events entry for return-from-dead scheduling
rather than promotion.

Consistent with cadence-limited filling, endgame boards sit well
under the seat caps: the live census shows ~45% warchief-seat
occupancy (19 holders over ~44 standard seats) and overlords on
only 2/7 boards. Overlord seats in particular appear to fill
through events (`PromoteDeadPlayerKiller_to_Overlord`,
`Promote_DarkTalion_Overlord`, fortress-assault outcomes) rather
than the ambient ladder — INFERRED; whether a slow ambient
overlord fill exists at all is unresolved.

## Campaign-long roster churn (VERIFIED — save-pair GUID join)

Joining two saves ~10 weeks apart on each member record's
bound-entity GUID (`05`-blob `b`; see `formats/detail/save.md`)
links 32 of ~150 live members across the pair — meaning roughly
**4 in 5 live board members are replaced over a campaign**
(early-campaign file: 158 members, median level 10; endgame file:
165 members, median level 79; same account/campaign). The churn is
both directions: ~118 GUIDs disappear and ~105 new GUIDs appear.
Caveat: `b` is a bound-entity reference and is shared by
sentinel/resurrection bindings, so the persistence figure is an
upper bound — direction and scale are nonetheless clear.

A second result from the same join sharpens the churn model:
**none** of the ~118 GUIDs that left the live roster appear in the
later save's ~360-record dead archive — dead records are purged
from WHDC over time (mechanism undecoded), so the archive is a
*rolling* graveyard holding
only recently dead members. Live-roster diffing therefore
undercounts total deaths, and killer attribution across long gaps
is impossible — the dead records and the edges into them are
garbage-collected.

Two corollaries, both verified by the same pair:

- **Board level tracks player level.** The early save's board
  median is ~10 — matching a ~level-10 player — while the endgame
  board sits at ~79. The board does not carry a fixed level
  structure; it rides the player's level via
  `PlayerBasedXPLevel` spawn deltas, and surviving members climb
  with the campaign.
- **The endgame low tail is spawn recency, not a stable class.**
  Persisted members mostly climbed (median delta strongly
  positive; 29/32 gained levels, many to the 75–84 band). Members
  found at low level in the endgame save are recent spawns, not a
  permanently weak tier — consistent with level tracking being
  per-member accumulation, not a seat property.

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

**Undead is a raise, not a cheat — VERIFIED (roster split):**
cheat-death and escape returners are 0% undead (`Marker_MoreLucky_
to_CheatDeath` 0/37, `Marker_EscapedPlayer_*` 0/~34); undead
members are exactly the raised set — `RaiseDead` holders 98%
undead. Retail sources of the undead package are therefore the
player's Raise Dead ability and necromancer-captain resurrection
(`DLC5_Necromancer_SelfRes*`, `Picker_Undead`); a natural
cheat-death returns scarred (`Marker_Deranged` etc.) but living.

**Necromancer channel (VERIFIED records):** the specialization is
`Marker_IsNecromancer` (12/562 live members ≈ 2%; excluded by
`Marker_DoNotBecomeNecromancer`, 69 holders). Record surface:
`Item_Necromancer_SelfRes`/`_Activated`/`_HasRisenThisSpawn` — the
once-per-spawn self-resurrection gate; `BIP_Necro_RaiseDead_AOE`
+ `Combat/ResurrectionData` — the AoE raise; `Faction/Events:
DLC5_Necromancer_SelfResurrect_Activated` — the board event fired
when one resolves; `Picker_Necromancer_SelfRes` +
`Necromancer_SelfRes_RemovePicker` for grant/removal; the audio
system conditions on `SourceHasMarkerNecromancer`. 2/12 live
necromancers are undead (self-res resolved at least once).
`Faction/Resurrection/Chance` also encodes the serial-returnee
decay — separate tables per return count (`Resurrection_1st`,
`_2ndFew`, `_3rdFew`, `ResurrectSameGuy_Again{,2,3}`) keyed by
death manner (`_OlogHeadRemoved`, `_FallOnly`, `BodyCleave`).

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
Sim applies `power += delta` clamped [1, `max_level`] (default 85 —
the roster reaches 84 and delta records clamp at 85) — INFERRED
application (retail may set relative to player level rather than
adjust).
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

## Recurrence lockouts & scar pools (VERIFIED fields + mod evidence)

The scar/cheat-death return system is gated by real `Faction.Role`
fields — verified on the installed `game.gamedb` (`Faction/Roles/
Definitions` schema resolves cleanly):

- `BaseWeight`, `MinWeight`, `WeightReductionPerUse` — role pick
  weight decays each time the role is used, down to `MinWeight`.
- `RoleGroup` — mutual-exclusion group (shared-group titles like
  `FlameOfWar`/`Inflamed`/`Burned`, `Blight`/`Poisoned` cannot
  coexist on the board).
- `RoleReplaceValue` — replacement priority ordering.
- `ScarType` — links a role to its `Combat/Scar/Type` row.
- `Prediate_GlobalVar` (sic — retail typo), `Predicate_Class`,
  `Predicate_Personality`, `Predicate_AppearanceTags`,
  `Predicate_Scale` — gating predicates per role.

`Combat/Scar/Type` holds 24 records whose `WeightPerUse` attr is a
decay ladder applied per use — standard scars carry
`1000, 10, 1, 0.01, 0.0001`; `ScarBookmark_Role_{Claw,Machine,
Stormbringer}` carry `10000, 1, 0.01, 0.0001` (10× heavier first
draw = the cheat-death/scar pool disparity); the player's
`Resurrected_PlayerAbility_RaiseDead` carries a flat `1`. Scar rows
can also exclude items (`ExcludedItems` → `Item_Necromancer_
SelfRes*` on `Burns`).

`Faction/Rewards/Types` is the **`Faction.DynamicWeight`** table
(40 records) — the self-mutating weight system behind the
one-use lockouts. Each row carries `InitialValue`,
`OnUseThisReturnToValue`, `OnUseOtherValueDeltaAdd` and
`OnUseOtherValueDeltaScale`: when the weight is consumed it snaps
to `OnUseThisReturnToValue` (0 = locked out) while every other
weight in the pool is add/scaled — a recover-over-use economy.
The rows cover `OriginDeathTaunt_*` (21 title taunts),
`Transformed_{Blight,Forsaken,Infested}`, `AdvancedClass_*`
(11 classes) and `Weight_*_BlownOutInitially` starters. This is
the "DynamicWeight 1-use lockout" the mod bypasses — the field
names are verified retail, and `OnUseThisReturnToValue=0` on the
`Transformed_*` rows is the serialized lockout itself.

**Save persistence (VERIFIED):** the live weight values are
serialized into the campaign save — slot-1 WHDC `_FORT_` records
carry a name-hash + f32 list covering exactly the DynamicWeight
vocabulary: `OriginDeathTaunt_*` (~21), `Transformed_{Blight,
Forsaken,Infested}` (each at `0` = locked out), `AdvancedClass_*`
classes and `Weight_*_BlownOutInitially` starters. So lockouts and
decay state persist per save — the GameDB `InitialValue` is only
the new-game seed. (Pairing order of hash↔f32 entries and whether
values are raw weights or fixed-point flags is INFERRED; some
values are powers of two.) The same FORT record also serializes
the region board: `Faction_<region>`, `FortAssault` and
`<region>_Row_Captain_*` references with a u32 captain-index list.

**Mod evidence (Nemesis Recurrence v1.0.0, YoRHa9S — Nexus 278):**
the mod's in-memory patcher names nine retail code sites it
neutralizes, which independently confirm the mechanic set:

- RoleGroup regional lockout (one shared-group role per region)
- Active `Faction.Role` duplicate prevention
- `RoleReplaceValue` priority gate
- Role usage weight decay (`WeightReductionPerUse`)
- `DynamicWeight` one-use lockout (Legendary/Pinnacle/Cheat-Death
  scars/titles are intended to recur only once per save)
- `ScarType` decay-index init + update paths
- `TotalResurrected` global counter — decays cheat-death odds on
  long-running saves

The mod also ships the retail **story-role isolation list** — roles
the recurrence system must never touch: `Ocker_Baz`, `Ocker_Gaz`,
`Ripper_Daz` (the Bruz brothers' internal role names), `PlanMaker`,
`GoldPath`, `WitchKing`, `Castimir`, `RangerRoleDef`,
`FakeDarkTalion`, `Tutorial`, `CommanderOlog`,
`MachineDestroyerOlog`, `FeralTrackerOrc`, `Role_IronSkull`,
`Role_Stitch`, `Role_FlameOfWar`, `Role_RazorJaw`, `Role_Jaws`,
`Role_LockJaw`, `NemesisForge_Role_{Jaws,LockJaw}`, plus the scar
catalog `Role_{BagHead,Faceless,Hook,Blight,Inflamed,Poisoned,
BrokenShield,PlagueBringer,Claw,Defaced,WraithTouched,Burned,
StormBringer,Machine,Skinless,RawHead,EverWounded,ForgeClaw,
Forsaken,NamelessOne,SlashFace,Immortal,Unashamed,Abandoned,
Shamed,Reject,OfTheFlies,Specter,Survivor,Bitter,Infested,
Unkillable,Beast,Deranged,Mindless,Ruined,Maniac}` — the
near-complete cheat-death role vocabulary, matching the
`ResReward_{TransformRole,ChangeRole,PinnicleScar}_*[_Delay]` and
`PS_*_PinnacleRole_*` reward-row naming in gamedb.

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
