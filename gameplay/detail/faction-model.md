# Nemesis model

> **Status:** working — model overview; parity gaps tracked in faction-parity.md
> **Provenance:** real-install `game.gamedb` (builder/trait/picker records via `anduril db`),
> endgame-save captain census (`anduril save`), live-roster walks (`tools/roster_live.py`),
> and Project Nemesis research notes (evidence, not authority). Compiled over 2025 sessions;
> consolidated 2026-10-19.

How `anduril-sim::nemesis` models Shadow of War's nemesis system, what
is verified against the real gamedb, and what remains heuristic.

## Evidence basis

The real system is **event-driven, not rate-driven** (VERIFIED-STATIC,
`Faction/Events/*` lists): `SimEventPicker` records (~80) map
triggers — `Quest/Jobs/Definition` refs like `PlayerKiller`,
`EjectedWarchief`, `DLC3_Overlord` — onto `Faction/Events/SimEvent`
definitions (~431). The engine picks *which* event fires when a
context arises; per-roll probabilities as we use them do not exist in
the data (they live in engine code/jobs, undecoded).

Real event families vs. our bands:

| Real SimEvent family (count)              | Andúril band   |
|-------------------------------------------|----------------|
| `PromoteEmpty*` (29), `Promote*` (11) — board ops | `promotion` |
| `PromoteDead*` (28) — return events | `cheat_death`/`deranged` (Transform missions) |
| `Eject*` (7)                              | `shame`        |
| `DuelIntoHierarchy*` (18), `Feast*` (4)   | `rivalry`      |
| `Guard*` (12)                             | bodyguard graph|
| `Execution*`/`Trial*`/`Kill*` families    | `death`        |
| `Transform*` (8)                          | `cheat_death` / `deranged` |
| vendetta chains (player-killer arcs)      | `vendetta`     |

So: the **event types** we model are real, the **rates are placeholder
heuristics** — tune `NemesisRules` freely; matching real cadence needs
the engine's job-trigger logic decoded (open work).

## Model

- `Captain` — entity component: builder provenance (`refs`), resolved
  semantics (`role`, `traits`, `trait_markers`, `models`, `buddies`,
  `instance`), plus live state: `power`, `rank`, `rarity`, `status`,
  `vendettas`, `deranged`, `nemesis_foe`, `bodyguards`/`guarding`,
  `history`. Trait draws are picker-weighted: builder
  `Faction/Traits/Picker` refs and `PickerListRaw` lists resolve to
  `{weight, trait}` pairs (Picker cell 1 = selection weight; a direct
  builder picker ref grants its trait outright). `rarity` projects
  `Marker_OrcRarity_*` refs to Legendary/Epic/Normal (INFERRED).
- `Rank`: Soldier→Captain→Warchief→Overlord; the top two tiers are
  slot-capped (`overlord_slots`, `warchief_slots`).
- `Status`: Alive | Shamed | Dead; Dead rolls only `cheat_death`,
  Shamed rolls only `deranged`.
- `NemesisRules` — world resource; every band width, power bump, and
  slot cap is data. `nemesis_tick` copies it once per tick.
- Determinism — draws come from the world PRNG in entity order; the
  in-tick roster snapshot applies status/rank changes immediately (a
  same-tick death frees a seat for a later promotion).

## Tick semantics

Each tick: capped-tier bosses first refill their bodyguard detail
(lazy top-up, roster order, no draws), then every captain rolls d100:

```
Alive:  <death> spawn Execution mission |
        <promotion> promote (slot-gated) |
        <shame> spawn Eject mission | <vendetta> spawn Vendetta
        mission vs the Player (or passive +vendetta with no player) |
        <rivalry> draw from the real `PowerStruggle` generation list
        (foe → `Revenge` list) — the drawn event drives the mission
        kind: Duel/Trial/Raid/Feast/Execution or a mod-owned Custom
        (BeastHunt, Chase, DarkHunt…) |
        <hunting> draw from the real `Hunting` generation list —
        an ambient `Custom` mission (Caravan, Drills, SlaveLabor…)
        carrying what the captain is *doing*; solo participant,
        mod-owned resolution, skipped when the captain is already
        in a mission | quiet
Dead:   <cheat_death> spawn Transform mission (return_kind=cheat_death)
Shamed: <deranged> spawn Transform mission (return_kind=deranged)
```

Duel: rival pick prefers a living `nemesis_foe` (blood feuds), then
guarded bosses redirect to a living bodyguard. The rivalry roll does
**not** resolve instantly — it spawns a `Mission` entity
(`kind=Duel`, `participants=[challenger, rival]`, `region`,
`ticks_left=mission_duration`, freeform `custom` fields) and
`mission_tick` counts it down. At 0 the duel resolves: power-weighted;
winner +`rivalry_power`, loser shamed — or slain at >=2x dominance.
Loser records the winner as `nemesis_foe`. A mission whose required
participants are dead/gone emits `nemesis:mission_aborted` instead.
Missions despawn on resolution — outcomes arrive via events.

While a mission runs, each tick a `Player` on the board may crash it
(`intervention` rule, % per tick) — the mission converts to a Vendetta
against the player and emits `nemesis:interrupted`. The SoW
mission-interruption mechanic, now first-class.

Missions are the mod seam for every SimEvent family: `MissionKind`
covers Duel/Execution/Feast/Raid/Trial/Eject/Transform plus
`Custom(name)`. Native resolutions (INFERRED — provisional shapes,
all deterministic and rules-tunable):

| kind | participants | resolution |
|------|--------------|------------|
| `Duel` | challenger, rival | power contest; loser shamed (slain at 2x) |
| `Execution` | [executioner,] target | target dies (cheat-death can return) — the `death` roll spawns these; lone-participant missions pick a random living rival on the board as executioner at resolution |
| `Feast` | host, guests… | each living participant +`feast_power` |
| `Raid` | raider, defender | fatal power contest — loser dies |
| `Trial` | accuser, accused | power contest — loser shamed |
| `Eject` | target | stripped to Soldier + shamed, detail released — the `shame` roll spawns these |
| `Vendetta` | captain, player | power contest — captain wins: vendetta +`vendetta_power` (doubled when `custom.death_threat` is set), `killed_player`; player wins: captain slain (`slain_by_player`) |
| `Transform` | target | return arc selected by `custom.return_kind`: `cheat_death` → dead→alive +`cheat_death_power` (`nemesis:cheated_death`); `deranged` → shamed→alive + `deranged` +`deranged_power` (`nemesis:returned_deranged`); plain → +`transform_power`, `deranged` cleared (`nemesis:transformed`). Re-entry re-seats at the highest open tier ≤ old rank. The dead/shamed bands spawn these — a mod can *deny a resurrection* by despawning the mission |
| `Custom` | any | emits `mission_resolved` for mod listeners |

Kind events: `nemesis:executed`/`nemesis:raid`/`nemesis:trial` carry
`{a,b,a_name,b_name}` or `{winner,loser,winner_name,loser_name}`;
`nemesis:feast`/`nemesis:ejected` carry the mission context. Bosses falling (death/shame/duel loss)
free their guards; a returning captain re-enters at the highest open
tier at or below its old rank.

Vendetta: a `Player` component (`power`, mod-adjustable) turns the
vendetta band into a Vendetta mission — the captain hunts the player
for `mission_duration` ticks, then the contest resolves (captain win
→ `vendettas += 1` + `vendetta_power`, `killed_player`; player win →
captain dies, `slain_by_player` + `died`). Powers are read at
resolution — mods can level the player mid-mission or cancel the
hunt. No player entity → passive vendetta counter only.
`sim nemesis` spawns a power-12 player.

Death threats: a `death_threat`-rule share of spawned Vendetta
missions arrives threatened — the captain issues a challenge
objective drawn from the real `DeathThreats` context
(`<region>_DeathThreats` first: `Challenge_DT`, `KillArchers_DT`,
`SustainedAlarm_DT`…). The objective is stored on
`mission.custom.death_threat` and announced via
`nemesis:death_threat` (`{entity, name, objective, …}`); a
threatened captain's vendetta win pays double `vendetta_power`
(INFERRED boost — the real records don't state it).

Outpost raids: with a `Player` on the field, an `outpost`-rule share
of ticks (default 3%, world-level — not per-captain) spawns a
kill-squad raid: a board is picked, one of its real
`{region}_Outpost_OPn` lists is probed (`OP1..OP9` indices that exist
in the loaded data), and the objective
(`KillArchers_Outpost_01`, `ReleasePrisoners_UB_OP1`,
`SustainedAlarm_*`) becomes a `Custom` mission tagged
`custom.outpost` (`"SM_OP3"`-style) with participants
`[defender, player]`. Resolution is mod-owned; at most two outpost
raids pend at once.

## Events (Lua)

Every transition emits `nemesis:*` with `{entity, name, rank, power,
vendettas, tick}`; `nemesis:duel` carries `{winner, loser,
winner_name, loser_name, fatal, tick}`; `nemesis:appointed` /
`nemesis:released` carry `{boss, guard, ...}`; mission lifecycle events
`nemesis:mission_started` / `nemesis:mission_resolved` /
`nemesis:mission_aborted` carry `{mission, kind, region, participants,
tick}`; `mission_started` adds `simulate` (the `<E>_Simulate`
sub-event) and resolved/aborted add `sub` (`<E>_End` on resolve,
`<E>_Clear` on abort — MODELED mapping, names are real data) when
the SimEvent decodes. Board entry emits `nemesis:enter_hierarchy` with
`{entity, picker, event, mode, region, rank, foe?, tick}` —
`mode` is `promote` (open or dead-held seat) or `duel` (seat full).
The Lua host drains
them before `tick` listeners. See `../docs/MODDING.md` for the full
`anduril.sim` surface.

## Board entry (SimEventPicker)

A builder carrying a `Faction/Events/SimEventPicker` ref (real data:
`Builder_EnterHierarchy` → `EjectedWarchief` job +
`PromoteEmpty_Builder` / `PromoteDead_Builder` /
`DuelFromBuilderCreate`) routes spawn through the hierarchy
grammar (MODELED — the routing is real, the seat arbitration is
ours): the builder name picks the seat tier (`*Overlord*` /
`*Warchief*` → that rank, else `Captain`), the region board is
checked against `NemesisRules` slots, and the picker names the
SimEvent: open seat → `PromoteEmpty`, dead-held seat →
`PromoteDead`, full seat → `DuelFromBuilderCreate` spawns a `Duel`
mission against a seated holder. Builders without a picker enter as
before (spawn-time rank is the mod's own choice).

## Open fidelity work

- Rates vs real data — decoded so far (VERIFIED-STATIC, cells raw):
  `Faction/Events/SimEventChance` (34 records: Duel, Feast,
  Execution1/2, Trial, RaidGondor, CampRaid, Convoy, Rescue, Arena,
  Invasion…) each carry an 8-float vector — e.g. Duel =
  `[0.15, 0.02, 0.0025, 1.0, 0.5, 0.003, 1.1, 0.15]`,
  Feast/Rescue/`Default_2orc_*` = `[0.25, 0.008, 0.0025, 1.0, 0.9,
  0.003, 0.2, 0.0]`, `SubjectAlwaysWins`/`FortAssault` use -1.0
  sentinels (scripted outcomes). Field meaning not yet mapped —
  likely base chance + modifiers (difficulty, warning time…).
  `Difficulty/SimEventLifeSpan` exists (cells empty in both records —
  table-driven elsewhere) — corroborates missions having lifespans.
  `SimEventParticipant` (572) carries per-event actor setup;
  `SimEventAbortCondition` (3) names real abort cases;
  `SimEventGenerationSetup/Type/TypeWeight` (18/68/4) hold
  spawn-selection weights; `SimEventPicker` (80) are 3-cell tables
  incl. interned refs. Decode target: map the 8 floats + generation
  weights to retune `NemesisRules` from real data.
- SimEvent record structure (VERIFIED-STATIC, `db analyze` +
  `db emit`): cell 13 → `Quest/Jobs/Definition` (participant-slot
  job: `PS_Other` on most records, `PS_Subject` appears too — not a
  single "driving job"); cells 16+ → `SimEventParticipant` cast
  records; cell 12 → `<E>_Simulate` lifecycle sub-event (name-verified
  across Duel/Feast/Promote*/BeastHunt); `<E>_Clear`/`<E>_End` appear
  at varying later cells (28/29, 30/31, 32/33 — position is NOT
  fixed); refs to other SimEvent records exist (Duel →
  `DuelIntoHierarchy_TribeBattler_03`) but their slot varies and
  semantics are undecoded — do NOT read as success/fail chains.
  Cell 25 → `Combat/AOE/Picker`; cell 31 → `Combat/Stat/Variable/Bool`;
  cell 19 → `Character/RenderIDs/Setup`; cell 14/21 →
  `AI/CombatSpacing`; cell 23 → `AI/Simulation/OffMesh`.
  Class-`0xA8F` cells → `AudioHooks/Actions` (the event's audio/shout
  pool — `FeastSingle`'s 14 cells land on a contiguous
  `HypeManPairWchf131B-138A` A/B-pair run; VERIFIED-STATIC).
  **Cell 0 is a type enum** (INFERRED): 24 = encounter
  (Duel/Feast/BeastHunt/Rescue/Trial/RaidGondor), 25 = board op
  (PromoteEmpty/Eject/SendToGarrison), 16 = kill (Execution1),
  27 = resurrect (PromoteDead_*). `SimEventChance` and
  `SimEventParticipant` bind to events **by name** — 19 of 34 chance
  profiles and 38 of 572 participant records share a SimEvent name
  (rest are generic profiles like `Default_1orc_general`).
  The `_Simulate` sub-event's cell-0 name is the choreography attack
  (`PL_St_L1_D1_L_LL_LL_R2L_2` on Duel/Feast/Promote*/Trial —
  `attack` in `event_info`). The `PL_*` grammar is INFERRED:
  `actor_context_size_direction_limbs..._swing_variant_move` —
  actor prefix (PL player, AI, Def, PC, PLBar), context (St
  standing, KD, Sync…), size class (L1/L2/Sm/Med/Lrg), direction
  (D0-D3), limb sequence (L/R/B/LL/RL/LH/RH…), swing (R2L/L2R/U2D/
  D2U — anim-variant selector, verified by paired-name diffs),
  numeric variant, move name (`SpinSlash`). `sim.attack_parts()`
  exposes the parse.
  `SimEventGenerationType` cell 7 (class 0x7ab) lists the candidate
  events per generation type — 478 refs across 68 records.
  `SimEventGenerationSetup` adds the regional half: 18
  region_tier tables (`Desert_Captain`, `Default_Warchief`…) whose
  cells ord-ref `Quest/SimEventPredicates` — the per-board event
  whitelist (`sim.generation_setup`). NB: class `0x922` auto-binds
  `CinematicSection` at delta-0 — a perfect-looking misbind;
  resolved by direct ord lookup instead.
  The whitelist is now live: `draw_generation_board` (used by
  `nemesis_tick` for Vendettas/PowerStruggle/Hunting draws) picks the
  generation list by the usual `{region}_{base}_Act{act}` →
  `{region}_{base}` → `{base}_Act{act}` → `{base}` chain, then
  intersects candidates with the `{region}_{tier}` table (falling
  back to `Default_{tier}`). Whitelist entries are predicate names —
  a `SimEventPredicates` record's class-`0x7ab` refs expand it into
  the SimEvent names it admits (`Pred_Duel` → Duel, Duel_Origin…).
  Policy: **narrow, never starve** — no table, no expansion, or an
  empty intersection all fall back to the unfiltered seeded draw, so
  mod-added events stay spawnable on unmapped boards. The special
  `Revenge`/`Prisoner` lists are drawn unfiltered (they encode
  captive/blood-feud state, not board texture).
- SimEvent records also carry ~18 cells of class-`0xA8F` refs —
  DECODED: they index `AudioHooks/Actions` directly (2,336 refs).
  VERIFIED-STATIC by ord-fit + name coherence: `FeastSingle`'s 14
  cells resolve to a contiguous `HypeManPairWchf131B–138A` A/B-pair
  run (a feast's hype-man banter); `Duel`'s resolve to mixed
  `dx_seq_`/`dx_cbt_`/`mus_`/`char_` action records. Exposed as
  `event_info(name).actions` — the event's audio/shout pool —
  with out-of-range ords staying `raw` `{cls, ord}` pairs.
  Our `Mission.event` field carries this record name so mods can
  match missions to real event data. The `NemesisEvents` world
  resource (`from_db`) classifies every `Faction/Events/SimEvent`
  record by name into `MissionKind` buckets; mission spawns draw a
  real record name from the pool via the world PRNG (seeded →
  replay-safe), so an Eject mission runs e.g. `Eject_Baranor`, not a
  generic label. Spawns are **context-sensitive** (`draw_tagged`):
  player-killers (`vendettas>0`) draw `*PlayerKiller`/`*_Vendetta`
  variants (`PromoteDeadPlayerKiller`, `Execution1_Vendetta`,
  `DuelFromPlayerKiller`), blood-feud victims `*_Revenge` — matching
  the real table's name-scheme (INFERRED). Vendetta missions search
  `*_Vendetta` variants across all families, since no `Vendetta*`
  family exists. A populated bucket never leaks across families;
  variant lists are sorted so seeded draws replay across processes.
  The CLI inserts it into `sim nemesis`/`sim resume`
  worlds; absent/empty pool → the kind name is used. Mods can
  overwrite the resource or the field outright.
- `SimEventChance` vectors are now loaded too
  (`NemesisEvents.chances`, keyed by record name). **Cell 4 is the
  subject's win chance** (INFERRED, high confidence:
  `SubjectAlwaysWins`=1.0, `SubjectAlwaysLoses`=0.0,
  Feast/Execution2/Rescue=0.9, Duel/general=0.5–0.6). Mission
  contests (Duel/Raid/Trial/Vendetta) blend the recorded chance 50/50
  with the power share — real event balance pulls the odds without
  power becoming irrelevant. Lookup chain: exact event name →
  `_seg`-trimming (`Execution1_Vendetta` → `Execution1`) → kind name
  → `Default`. Lua: `sim.event_chance(name)` → 9-float table.
- `SimEventParticipant` decodes into `NemesisEvents.roles`: event →
  `{record, role, job}` slots. Binding is by record name —
  `<Event>_<NN>` or `<Base>_<Role>_<Variant>` infix
  (`DuelFromInvasionImport_Other_WC3BG` →
  `DuelFromInvasionImport_WC3BG`, role `Other`). Role from the
  `Faction/Events/Events` ref suffix (`Duel_Other`); job from the
  `Quest/Jobs/Definition` ref — `PS_Subject_Cap_Executioner1` shows
  Execution1's subject is the *executioner*; `PromoteDead_Builder`'s
  slots are target-selection jobs (`EjectedWarchief`,
  `DeadOrEmptyPosition`). cell0 `0x0b` = captain participant, `0x03`
  = ambient AI participant (Investigation refs, no role/job — e.g.
  Duel_03..05 = LostTarget/WarlordSearch/Attract). Lua:
  `sim.event_participants(event)`; `anduril lua` worlds carry the
  pool so these work in standalone scripts.
  Slot records also decode (INFERRED slot, VERIFIED name targets):
  `builder` = fixed `FactionMemberBuilder` casting (cell ~7 alt);
  `spacing` = `AI/CombatSpacing` profile (`Execution1_01` runs
  `Olog` — the executioner *is* an Olog); `investigation` =
  `AI/Investigation` behavior (`LostTarget`/`WarlordSearch`/
  `Attract` on the ambient slots); `corpse` = `AI/Corpse` prop
  (`StealthKill`); `render` = `Character/RenderIDs` override
  (`CINE_Palantir`); `node_tags` = `AI/SpawnAmbient/NodeTags`
  (`Quest_Ambient`). All exposed on `event_participants` rows and
  `mission().roles[]`.
  Jobs are also **board-state selectors**: at spawn,
  `cast_mission_slots` casts slots whose job names a query —
  `*PlayerKiller*`/`LastPlayerKiller` → the living captain with most
  vendettas on the board, `SC_*` (`SC_Warchief_Vendetta`,
  `SC_DeathThreat`) → the board's top rank, `Ejected*` → a shamed
  captain. Selector results swap into their slot and never displace
  an earlier cast; the player participant is exempt. Applied to the
  rivalry (PowerStruggle/Revenge) and vendetta bands.
  A slot carrying `builders` casts the named
  `FactionMemberBuilder`s as real captains through the `Director`
  (`spawn_in_region`) on the mission's board: the job selector wins
  the slot when it resolves, `builders[0]` fills it otherwise, and
  remaining builders spawn as escorts on the cast captain's
  `bodyguards` — matching the real pattern where
  `FortDefense_Enemy_*` slots carry `FDInv*` jobs plus the
  `DLC5_Endless_Bodyguard_01/02` pair (MODELED). Builders are
  deduped per cast — six slots naming the same pair share one
  captain. This needs the `SimDb` world resource (an `Arc<MergedDb>`
  inserted alongside `NemesisEvents`); absent, or when the builder
  record is missing, the slot falls through to job/board casting
  unchanged. Cast participants carry their slot's decoded fields in
  `custom.cast_*` (`record/role/job/spacing/investigation/corpse/
  render/node_tags`). Slots with no job/builders but ambient data
  (`investigation`/`node_tags` — `Duel_03..05`'s crowd) spawn a bare
  entity: `Name` = slot record, `Fields.cast_ambient=1` + the slot's
  `cast_*` fields, no `Captain` — mods and ceremonies can see and
  drive them. `sim.cast_event(event, region)` is the mod-facing
  unseeded variant (sparse slot→entity map).
- Player simulation depth — power is a static number mods tune; a
  real campaign needs player progression and gear.
- Region partitioning — done: `Captain.region` is extracted from
  builder-name codes (`SW_Y\d_<CODE>_`, `DLC_Inv_<CODE>_,
  `Initial_Overlord_<name>`); `SotP_<Region>_`/`Gauntlet_<Region>_`
  names extract verbatim (`SotP_MinasIthil_*` → MinasIthil board).
  Slots, guards and rivals are per-board. The engine's six nemesis
  regions per `Regions/ZoneInfoNode` `*_Nemesis`: SnowMtn, Volcano,
  MtnPass, Island, Desert, MinasIthil. `region_name` maps:
  SM=Seregost, V=Gorgoroth, MP=Cirith Ungol, IS/Island=Núrnen
  (INFERRED — the sea's island fortress), Desert=Udûn (INFERRED —
  ash plain, `Desert_Landmark_Oasis`), MinasIthil/Morgul verbatim,
  VO=Lithlad (DLC5, INFERRED).
- Stock mission kinds resolve natively (provisional semantics — see
  table above); the real SimEvent records should calibrate effects,
  triggers and rates once their cells are decoded
  rather than band shorthand.
- Blood brothers (paired captains avenging each other).

Generation types carry three f32 cadence params in cells 0-2
(`sim.generation_params`): `PowerStruggle`/`Hunting` = [10,7,4-5],
outposts = [999,0,0], `Revenge` = [999,999,999] — large values read
as effectively-disabled. Field meaning is INFERRED; the numbers are
real. `Faction/Events/Replay/PlaybackEvents/Lists` maps outcome to
playback names (`PowerStruggle_SubjectWins_Duel`, `*_OthBrand`)
— exposed as `sim.playback(list)` (ordered outcome playback names)
and `sim.playback_steps(name)` (ordered `Steps/Definition` names).
`playback_for(kind,result)` composes the real grammar
(`<Outcome>[_<Method>][_<Extra>]`, plus `Duel_<W>Success_<L><Fate>`
alternates) and only returns names that exist in the data —
`mission_resolved` emits it as `playback`+`steps[]`, then a
`Ceremony` entity walks the script via `nemesis:replay_step`/
`nemesis:replay_done` events (one step/tick, MODELED).

`Steps/Definition` internals (`sim.step_info`): cell 1 is an f32
duration (0.95/0.5/0.25 — same float-as-ref collision as
generation params), cells 4-5+ hold `Replay/Presentation` action
refs plus `Traits/Marker`, `Predicates/TraitLists`, `AI/Corpse` and
`AudioHooks/Keys` refs, grouped by list. Class-`0x921` cells decode
as step guard predicates: the low u16 packs `(slot << 8) | ord` with
`ord` indexing `Faction/World/Predicates/Replay` — VERIFIED-STATIC by
ord-fit (1230/1258 in range, 0xFF = none sentinel, 0xFA-0xFE reserved
stay `raw`) plus name coherence (`*_Pred*` steps resolve `Pred*`/
`Replay_*`/`PS_*` records). `slot` (0-4, uniform across the corpus)
is the bound member slot — INFERRED, not yet driven at runtime.

The `Faction/World/Predicates/Replay` records those guards reference
are themselves *compositions* (`NemesisEvents::replay_predicates`,
`sim.replay_pred(name)`): class-`0x9DB` cells ord-index
`Faction/World/Predicates/FactionMember/Member` leaf predicates
(`Not_HasTrait_Mount`, `Overlord`, `Replay_IsSubject`), and other
tagged cells can ref the combat-state predicate lists
(`Other_DiedByGraug` → `AI_BH_PreDeath` + `Replay_IsSubject` +
`Replay_IsWinner`) — leaves kept in cell order. VERIFIED-STATIC:
`Replay_Subj_GainedMount` → `Not_HasTrait_Mount` + `HasTrait_Mount`
(before/after transition pair), `Subject_IsOverlord` → `Overlord`.
`nemesis:replay_step` emits `predicate_results[]` aligned with
`predicates[]`: the leaves are evaluated AND-wise against the cast
member at the guard's slot via `eval_member_pred` — `"t"`/`"f"`/
`"?"` (unevaluable or unknown leaf — tri-state, never fabricated
`false`), `"skip"` for `none` guards. A `gate` roll-up condenses
the beat's verdict: `"f"` if any guard decided false (the beat's
condition isn't met — INFERRED beat-gating semantics: guards are
per-step conditions like `BR_Winner_UI_GainedMount` ←
`Subject_Follower`, which is why the corpus carries decap/scale/
mount deltas), `"?"` if nothing decided false but some guard is
unevaluable (`PS_*`/`AQ_*` presentation-state/event predicates,
unbound slots), else `"t"` (unguarded counts as open). The sim
emits every beat regardless — `gate` is a verdict for
viewers/mods to act on, not a skip enforced by the engine side.
`sim.ceremony_plan(id)` (Rust: `nemesis::ceremony_plan`) exposes
the same roll-up up front — the full playlist as
`(pos, step, gate)`, letting a consumer compute the effective
presentation list (`gate != "f"`) without walking ticks. Guard
evaluation is shared: `eval_step_guards` backs both the tick
emission and the plan.
`eval_member_pred` maps the
evaluable vocabulary onto `Captain` (rank names, `HasTrait_*`,
`InHierarchy` = seat, `Shamed`/`Dead`/`Alive`), any `Not_*`/`Not*`
prefix negates an evaluable inner leaf (`NotWarChief`,
`NotOverlord`, `Not_HasTrait_*`, `NotInHierarchy` — one rule covers
the whole negated vocabulary), `Replay_IsSubject`/`IsOther`/
`IsWinner`/`IsLoser` compare against the ceremony's role bindings —
the `Ceremony` carries `subject`/`other`/`winner`/`loser` resolved
from the mission's role-aligned `slots` plus the playback's outcome
token (`SubjectWins`/`OtherWins`; draw playbacks leave both `None`).
Species decodes from `Character/Models` names (`Olog`/`Replay_Olog`
→ model contains `olog`; `Orc`/`Replay_Orc` → not-olog; empty model
list → `?`), `Bodyguard` scans the roster for membership in another
captain's `bodyguards`, and the allegiance leaves test `Captain.
allegiance` (`Enemy`/`Follower` — any branded designation counts —
`Spy`/`PlayerBodyguard`; `NotSpy` via generic negation). Allegiance
is MODELED: `*Dominated*` mission results brand the losing role's
cast member as `Follower` (vendettas cleared); `Spy`/
`PlayerBodyguard` are mod-set designations (`set_captain
{allegiance=...}`, persisted through snapshots). `Scale_Small`/
`Scale_Medium`/`Scale_Large` test `Captain.scale` — also MODELED:
the live db's `FactionMember/Scale` predicate list has zero inbound
refs from any builder/instance record (checked all 608 builders +
171 instances), so scale is a runtime property like the engine's
visual size randomization; we derive it deterministically from the
entity id at spawn (mod-settable via `set_captain {scale=...}`,
snapshot-persisted). `Prisoner` tests the `Shamed` pool — the same
pool `cast_by_job` routes captive/prisoner jobs to. The DLC4 leaves
decode from real name evidence: `DLC4_Easterlings` tests a `DLC4*`
builder or `Easterling_*`/`DLC4*` models/traits (all present in the
live db), `OutpostLeader_DLC4` tests a `DLC4_OutpostCommander_*`
builder (INFERRED — which field the engine keys on is unverified).
`Known`/`Unknown` model player awareness as `vendettas > 0` or any
branded allegiance (INFERRED — encounter state isn't tracked
separately). `Decapitated` tests `Captain.decapitated` — MODELED:
`*Decap*` result tokens or a `custom.decapitated` mission flag mark
the dying side at resolution (the `Replay_Oth_Decapitated` /
`Replay_Subject_*_OthDecap` guards read it; no playback or attack
name carries a decap token, verified by census). Transient combat
states live in `Captain.combat` (`on_fire`, `knocked_down`,
`stunned`, `dazed`, `pre_death`, `poisoned`, `weakened`, `wasps`,
`smoke_screen`, `pinned`, `vulnerable`, `blinded`, `bound`,
`cower`, `dashed`, `raised`, `grabbed`, `bleed_out`, `propagating`
— MODELED, mod-driven via `set_captain{combat={...}}`,
snapshot-packed as a bitset). Compound leaf names are parsed as a
boolean formula, not matched verbatim (INFERRED grammar): `_`
separates segments, CamelCase packs words inside one, combat-state
words (`OnFire`, `KD`, `Stunned`, `Dazed`, `PreDeath`, `Poisoned`,
`Weakened`, `Wasps`, `SmokeScreen`, `Pinned`, `Vulnerable`,
`Blinded`, `Bound`, `Cower`, `DashedOver`, `Raised`, `Grabbed`,
`BleedOut`, `Propagated`, `Normal`, `Death`) are OR-group
alternatives, resolvable *general* leaf words (`WarChief`, `Spy`,
`IsSubject`…) are required AND constraints, and `Not`/`And`/`Or`/
`Only` are infix operators (`Normal_NotWeakened`,
`StunnedAndQuickStunned`, `WarChief_NotSpy`,
`OnFire_PropagateOnly` = fire *while* propagating,
`Replay_IsSubjectOrOther` = subject OR other). `Wraith`/`Player`
prefixes scope following words to the `Player` entity (MODELED
`Player.combat`/`dead`, driven by `set_player_state`); position,
scope and namespace markers (`_L`, `_BR`, `_All`, `Replay`, `AI`,
`BH`, `Bip`, `Warg`…) are skipped. Result is Kleene tri-state —
the OR group short-circuits on a true term, constraints fail fast
on a decided false, and unresolved words keep the leaf `?` unless a
sibling already decided it. `Vendetta` tests `vendettas > 0`,
`Personality_Deranged` tests `deranged`, `Defender` tests
`guarding`, `Tribe_*` and `NotTorturuer` (misspelled in the data)
test `Captain.tribe` — the `TribeDef_*`/`DLC_TribeDef_*` record name
the captain's role selected at spawn (VERIFIED: the role's `Tribes`
struct link resolves through category `_StructuresFactionTribeEntry`
to `Faction/Tribes/Definitions`; the engine takes the first
DLC-valid entry — the role decides the tribe, not the models).
Captains with no role-derived tribe fall back to the INFERRED
model/trait name-decode (`Orc_Stk_Warmonger_*` pieces).
`DLC4_EasterlingArcher`
is the easterling+archer name pair, and `Replay_*RankedLoser_*`
compare the loser's `power` to the other's (INFERRED ranked =
power). `HasSimEvent` tests whether the member participates in an
in-flight `Mission` (MODELED). `OrdersDisabled` tests
`Captain.orders_disabled`, and `AssaultGroup_<Side>_Slot<NN>` tests
the member's posted assault-group slot (`Captain.assault_group =
(side, slot)` — both MODELED, mod-driven via
`set_captain{orders_disabled=, assault_group={side=,slot=}}`;
`assault_group=false` clears the posting). `Row_<n>`/
`Row_<n>_<lane>`, `Column_<n>`/`Column_<a>to<b>` (spanning), and
`SameRowOther`/`NotSameRowOther` test the member's pedestal
formation slot (`Captain.formation = (row, col, span, lane)` —
the `BR_Pedestal` grid; MODELED, mod-driven via
`set_captain{formation={row=,col=,span=,lane=}}`; unpositioned →
decisively `f`, lane-qualified on a lane-less member → `?`, and
bare `HigherRanked`/`LowerRanked` compare member vs `other`
power — INFERRED ranked = `power`). Whole-name leaf
semantics outrank compound decomposition — `AssaultGroup_Enemy_
Slot01` is one leaf, not `… AND Enemy…`.
`KilledByPlayer` tests `Captain.killed_by_player` — set when a
vendetta resolves in the player's favor, or via
`set_captain{killed_by_player=}` (MODELED). Leaf-context probes
placed the last singletons: `ScratchPosition` gates `Promote_*`
replays → `Captain.scratch_position` (pre-promotion staging slot);
`HasFortTrait` gates `Contract_Trait_*`/`Subject_HasFortTrait`
(fort-siege upgrade traits — no Fort-named trait records exist,
so it's a distinct flag) → `Captain.has_fort_trait`;
`Replay_IsSubjectPostPos` (post position + subject binding) →
`Captain.post_position`; `CannotSlotInAssaultGroup` (in
`PS_DarkHunt`) is *derived* eligibility — posted or
orders-suppressed members can't be slotted (INFERRED). All three
flags are MODELED mod-driven (`set_captain{scratch_position=,
post_position=, has_fort_trait=}`). Still `?`: `EmptyFactionPos`
(roster vacancy — no faction-slot-count model) and `FD_Enemy`
(unclear expansion — `PS_Raid_Caragors` context only), plus
monster-only state (`Troll_*`, `AI_Balrog_*`, `AI_BoG_*`,
`Wraith_HoG_*`, `Radiance_*`). `sim.eval_pred(id, leaf[, ctx])` exposes the
evaluator to mods (`nil` = unknown; `"@Name"` prefix ANDs a
composed Replay predicate's leaves).

Two ref classes moved to explicit ord decodes after the fixture
showed ord-fit bindings are unstable under list growth: `0x259`
(participant role ref) and `0xC00` (SimEvent lifecycle ref) both
index `Faction/Events/Events` by direct ord — deterministic and
immune to tie-steals (`0x259` tied `AudioHooks/Actions` at count 19;
`0x632`/`0x700` bindings were stolen when the participant fixture
grew).

`Replay/Presentation` internals (`sim.presentation_info`): cells 0-1
are f32 params (the `Store_Garrison_Unique_*` chain ramps cell 0
0.3→0.8; class ids `0x3e8`/`0x3f0` in `db analyze` are literally the
top bits of those floats — the same collision as durations). Cells
3-8 are optional `-1`-sentinel slots holding resolved condition refs:
`Combat/AttackPredicates` (`Ambient_DoNotEquipRanged`),
`AI/Sensors/Interpreters` (`KnownDeadBody`), `AI/Morale/Overrides`
(`Inspired_Presence_FleeBlocker`) and `Faction/Events/Events`
(`StorePresented` — an event fired by the presentation). Cell 11's
class-`0x4ee` refs bind PlaybackEvents globally but names don't
correlate — SUSPECTED MISBIND, exposed as `playback` anyway. Cells
9/10/12's class-`0x454` refs (1,020 total, dense ords 0..1019)
resolve to `Faction/HierarchyMenu/Animations/Choices` (1,129
records) — VERIFIED-STATIC by name-namespace correlation: every
sampled ord lands on an anim-choice name (`Replay_Duel_Cheer_*`,
`*_Idle`, `*_Fidget`) and `QS_*`/`Replay_*` prefixes track the
step's own name (a `PoseLoopSubject` step picks idle/cheer anims).
Exposed as `animations`; resolved by direct ord index since the
class has no global binding.

## Andúril model

This document *is* the Andúril-side model — it describes how
`anduril-sim::nemesis` represents the system (what's verified vs
heuristic inline). Retail truth for specific mechanics lives in the
sibling `faction-*` specs; this file stays the map between them and
the implementation.
