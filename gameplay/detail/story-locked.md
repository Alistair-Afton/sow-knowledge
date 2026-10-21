# Story-locked slices of systemic mechanics

> **Status:** **verified structure** — records below are confirmed in
> `game.gamedb`; which quest calls which record is named-evidence
> (INFERRED by name unless marked VERIFIED).
> **Provenance:** GameDB record census, live roster, faction specs.

The nemesis/faction systems are data-driven end to end — story
missions don't bypass them, they drive the same machinery through
quest-scoped records, tags, and pickers. This doc maps the pockets
of normally-freeform systems that only exist under story/quest
control.

## Story nemeses as system participants

Named characters participate in the faction sim through dedicated
records rather than bespoke logic:

- **Bruz** — the largest story-controlled lifecycle in the DB:
  tutorial ambushes (`GP_BruzTutorial_Ambush_1..3` across
  `Ambush/Chance`, `/Definition`, `/TimedTrigger`, `/Result`),
  hierarchy events (`DuelIntoHierarchy_Bruz`, `Bruz_EnterHierarchy`
  picker, `Promote_Bruz_WC`, `Promote_Bruz_Overlord`,
  `PromoteDead_Bruz`, `DuelFrom_Bruz_{Simulate,Clear,End}`),
  `GP_Bruz_BecomeUnbreakable`, `GP_Bruz_AddLevels_Ratbag*`,
  `Domination_CalculateBruzShameRewards`, `BR_FD_EnterBruz`
  playback, `Job_Bruz` member-state predicate, and debug pickers
  (`Debug_BruzToOverlord`, `Debug_BruzToWarchief`). His "betrayal"
  is scripted through the standard event/ambush/dominion verbs.
- **Baz / Daz / Gaz (BruzBrothers)** — ambush definitions and
  `*_Released_To_Sandbox` events plus `BruzBrothers_ReleasedToSandbox`
  in `Events/Lists`: story members are sandbox-released when their
  chapter ends.
- **Ratbag** — `GP_Ratbag0..3` XP/quest events,
  `GP_Ratbag{1,2,3}_SubjectWinsAndPromoted` playback events (story
  missions that hand Ratbag a win), `Ratbag1` SimEvent +
  `GP_Ratbag` generation type.
- **Zog** — `General_Ambush_Zog` + `SM_Zog_Ambush` definitions and
  `Zog_ReleasedToSandbox`: the necromancer is kept out of the
  sandbox board until the story releases him; also
  `Loot_IgnoreList_Zog_CarnanSide6` (no loot off Zog in that quest).

## Quest-tagged sim events

`Faction/Events/SimEventTag` carries a `Quest_*` family
(VERIFIED records) — the nemesis event kinds a story mission can
raise on the board: `Quest_Arena`, `Quest_BeastHunt`,
`Quest_ChaseAmbusher`, `Quest_ChasePursuer`, `Quest_DarkHunt`,
`Quest_Drills`, `Quest_Duel`, `Quest_EscortCaravan`,
`Quest_Execution`, `Quest_Feast`, `Quest_GraugHeist`,
`Quest_GuardCamp`, `Quest_GuardCaragors`, `Quest_GuardCheckpoint`,
`Quest_GuardSupplies`, `Quest_GuardWarCouncil`, `Quest_Invasion`,
`Quest_RaidCamp`, `Quest_RaidCaragors`, `Quest_RaidCheckpoint`,
`Quest_RaidSupplies`, `Quest_Rescue`, `Quest_SlaveLabor`,
`Quest_TrialCaragors`.

The inverse gate exists too: `Events/Lists/EventsThatBlockQuests`
names the board events that may not fire while a quest runs, and
`SimEventTagList/QuestArena` + `AddGauntletEnemy`/
`RemoveGauntletEnemy` tag mods let quests rewrite a mission's tag
set. Menu predicates `QuestStart_CreateSimEvent` /
`QuestStart_AugmentSimEvent` confirm quests instantiate real sim
events rather than scripted stand-ins.

## The Gauntlet — a quest-ridden nemesis subsystem

The Blade-of-Galadriel gauntlet (Eltariel's follower trials) is a
self-contained nemesis channel: full per-region builder sets
(`Gauntlet_{Desert,Island,MinasMorgul,MtnPass,SnowMtn,Volcano}_Lvl{1,2,3}`),
`Gauntlet_Subject`/`GauntletEnemy` builders and participants,
`Gauntlet{,2,3}` SimEvents with `DuelIntoHierarchy_GauntletEnemy`,
`PromoteDead_GauntletEnemy`, `PromoteEmpty_GauntletEnemy`, its own
quest screens/cameras (`QS_Gauntlet*`), and
`PS_Gauntlet_Enemy_Unavailable` presentation state. The gauntlet
foe is spawned, staged, dueled and resolved through the same
hierarchy verbs the freeform board uses — pointed at a private
participant list.

## Act and campaign-state gating

- `SimEventGenerationType` splits by act: `Hunting_MinasIthil_Act1`
  vs `_Act2`, `MinasIthil_PowerStruggle_Act2`, `Hunting_Act2`,
  `PowerStruggle_Act2`, `NeverEndingShadowWar` (the Shadow War
  endgame channel), and `OutpostGeneric_OP1..5_NoAct1` (outpost
  events suppressed during Act 1).
- `Act2CaptainLevelUp_{MinasIthil,MtnPass}`,
  `Promote_Act2_Overlord_MtnPass`,
  `Debug_Act2MtnPassOverlord` — scripted board promotions at act
  boundaries (the conquest hand-off).
- `Faction_*_QuestReplay_*` definitions (MinasIthil CS06, MtnPass
  BS2, island RB2/RB3, volcano BLS5, snowMtn) — replaying a story
  quest swaps in a dedicated board variant rather than reusing the
  sandbox state.
- `ShadowWarUnlocked` GlobalVar gate on `InitialLevels` entries —
  the board's spawn levels switch from fixed bands to
  player-relative deltas once the endgame unlocks (see
  `faction-lifecycle.md` row-level table).
- `Default_Act1` spawn-path members — Act 1 board fill is a
  distinct fill profile (measurably different trait mix, e.g.
  lower `Follower_GrabAndThrow` share).

## Campaign mode variants

`Invasion/Modes` splits `ConquestCommunity` / `ConquestRanked` /
`ConquestFriend` — online conquest is a mode parameter on the same
invasion system. `Faction/Ambush/*` and `Combat/XP/EventQuest`
(`Bruz_Side01..04`) show side quests reusing ambush and XP systems
with quest keys.

## Open ends

- Which quest records *call* the Zog/Bruz/guuntlet events is
  name-evidence only — tracing quest scripts (`Quest/` lists, 96
  `QuestLists` records) to the sim event invocations is open.
- `DLC3_StoryFollowers` event list and `Domination_` quest verbs
  suggest story-mission domination has its own reward path
  (`Domination_CalculateBruzShameRewards`) — unmapped.
