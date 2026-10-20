# Faction generation — captain build rules, event grammar, live roster

> **Status:** **verified** — verified against real-install `game.gamedb`
> records, a 534-captain save census, and the live in-game roster.
> **Provenance:** real-install `game.gamedb` walks, endgame-save captain
> census, live roster memory reads; Project Nemesis research notes
> (evidence, not authority). Distilled from internal model/parity specs
> — this doc keeps only the retail-game facts.

The nemesis system is **event-driven, not rate-driven** (VERIFIED-STATIC,
`Faction/Events/*` lists): `SimEventPicker` records (~80) map triggers —
`Quest/Jobs/Definition` refs like `PlayerKiller`, `EjectedWarchief`,
`DLC3_Overlord` — onto `Faction/Events/SimEvent` definitions (~431). The
engine picks *which* event fires when a context arises; per-roll
probabilities do not exist in the data (they live in engine code/jobs,
undecoded).

## SimEvent families

| SimEvent family (count) | Role |
|---|---|
| `PromoteEmpty*` (29), `Promote*` (11) | board ops — fill open seats |
| `PromoteDead*` (28) | return events for dead-held seats |
| `Eject*` (7) | shame/eject |
| `DuelIntoHierarchy*` (18), `Feast*` (4) | rivalry/duel missions |
| `Guard*` (12) | bodyguard graph |
| `Execution*`/`Trial*`/`Kill*` families | death missions |
| `Transform*` (8) | cheat-death/deranged transforms |
| vendetta chains | player-killer arcs |

## Board entry (SimEventPicker)

A builder carrying a `Faction/Events/SimEventPicker` ref routes spawn
through the hierarchy grammar — real data: `Builder_EnterHierarchy` →
`EjectedWarchief` job + `PromoteEmpty_Builder` / `PromoteDead_Builder` /
`DuelFromBuilderCreate`. The builder name picks the seat tier
(`*Overlord*` / `*Warchief*` → that rank, else `Captain`). The picker
names the SimEvent by seat state: open seat → `PromoteEmpty`, dead-held
seat → `PromoteDead`, full seat → `DuelFromBuilderCreate` spawns a `Duel`
mission against the seated holder.

## SimEvent record structure (VERIFIED-STATIC)

- Cell 13 → `Quest/Jobs/Definition` participant-slot job (`PS_Other` on
  most records, `PS_Subject` also appears — not a single "driving job").
- Cells 16+ → `SimEventParticipant` cast records.
- Cell 12 → `<E>_Simulate` lifecycle sub-event (name-verified across
  Duel/Feast/Promote*/BeastHunt); its cell-0 name is the choreography
  attack (`PL_St_L1_D1_L_LL_LL_R2L_2` on Duel/Feast/Promote*/Trial).
- `<E>_Clear` / `<E>_End` appear at varying later cells (28/29, 30/31,
  32/33 — position is NOT fixed).
- Refs to other SimEvent records exist (Duel →
  `DuelIntoHierarchy_TribeBattler_03`) but their slot varies and
  semantics are undecoded — do NOT read as success/fail chains.
- Cell 25 → `Combat/AOE/Picker`; cell 31 → `Combat/Stat/Variable/Bool`;
  cell 19 → `Character/RenderIDs/Setup`; cells 14/21 → `AI/CombatSpacing`;
  cell 23 → `AI/Simulation/OffMesh`. Class-`0xA8F` cells →
  `AudioHooks/Actions` (the event's audio/shout pool).
- **Cell 0 is a type enum** (INFERRED): 24 = encounter
  (Duel/Feast/BeastHunt/Rescue/Trial/RaidGondor), 25 = board op
  (PromoteEmpty/Eject/SendToGarrison), 16 = kill (Execution1),
  27 = resurrect (`PromoteDead_*`).
- `SimEventChance` (34 records: Duel, Feast, Execution1/2, Trial,
  RaidGondor, CampRaid, Convoy, Rescue, Arena, Invasion…) each carry an
  8-float vector — e.g. Duel = `[0.15, 0.02, 0.0025, 1.0, 0.5, 0.003,
  1.1, 0.15]`; `SubjectAlwaysWins`/`FortAssault` use -1.0 sentinels
  (scripted outcomes). Field meaning not yet mapped — likely base chance
  + modifiers (cell 5, 1-based, reads as the subject's win chance).
- `SimEventParticipant` (572) carries per-event actor setup;
  `SimEventAbortCondition` (3) names real abort cases;
  `SimEventGenerationSetup/Type/TypeWeight` (18/68/4) hold
  spawn-selection weights.
- `SimEventChance`/`SimEventParticipant` bind to events **by name** —
  19 of 34 chance profiles and 38 of 572 participant records share a
  SimEvent name (the rest are generic profiles like
  `Default_1orc_general`).
- `SimEventGenerationType` cell 7 (class 0x7ab) lists the candidate
  events per generation type — 478 refs across 68 records.
  `SimEventGenerationSetup` adds the regional half: 18 region_tier
  tables (`Desert_Captain`, `Default_Warchief`…) — e.g. `Desert_Captain`
  → `{Trial_Gondor, CaptainBombs, Trap_Vendetta, DT_SustainedAlarm,
  Caravan1_Revenge, CampVendetta, GetGang_Ghuls, Rescue,
  KillArchers_Outpost_01/02, KillCaragors_Vendetta, RescueG_Origin}`;
  `Default_Warchief` → 5 incl. Arena. Names are
  `Quest/SimEventPredicates` records.
- `SimEventGenerationTypeWeight` (4 records — `PowerStruggle`,
  `PowerStruggle_Act2`, `Hunting`, `Hunting_Act2`) binds each
  concurrent-job slot to a `Difficulty/VariableFloat`: cells are
  `catIdx<<20|recIdx` links to unnamed
  `{f32 job_slot, VariableFloat ref}` rows. The authored curve is the
  type's inline `Weights` attr (`PowerStruggle`=[10,7,4,0],
  `Hunting`=[10,7,5,0], `SpawnConditions`=[4,3,2,1,1,1],
  `Vendettas`=[1,0], `Prisoner`=[1×6], `Gauntlet`=[1000×3,0]) —
  decreasing, and a 0 slot caps concurrency. (VERIFIED — the row's
  f32 field is a 1-based slot selector matching the `WeightN` var it
  binds, not a weight; reading the rows as a `[1,2,3,4]` curve is a
  known misread.)
- Difficulty remap lane (VERIFIED): `Difficulty/VariableFloat` names
  (`SimEvent_PowerStruggle_Weight{1,2,3,4+}`,
  `SimEvent_Hunting_Weight{1,2,3,4+}`) are resolved per difficulty
  tier through unnamed
  `_StructuresDifficultyVariableFloatAdjust` rows
  `{Variable ref → Combat/Stat/Constant/FloatArray ref}` — each of
  the five unnamed `Difficulty/Settings` records (ordered by
  `DifficultyLists`; `Difficulty/WeightTables` names confirm tiers
  `1Easy/2Normal/3Hard/4Brutal`, fifth = post-launch Gravewalker)
  links its own adjust rows. SimEvent weights only ship
  `*_Diff4+`/`_4Brutal` arrays, i.e. only Brutal+ overrides:
  PowerStruggle `[10,7,4,0]`→`[5,2,0,0]`, Hunting `[10,7,5,0]`→
  `[2,0,0,0]` — the hardest tier allows roughly one concurrent
  nemesis job per region. Below Brutal the inline `Weights` apply.
- `Difficulty/SimEventLifeSpan` exists (cells empty in both records —
  table-driven elsewhere) — corroborates missions having lifespans.

## `PL_*` attack-name grammar (INFERRED)

`actor_context_size_direction_limbs..._swing_variant_move` — actor
prefix (PL player, AI, Def, PC, PLBar), context (St standing, KD,
Sync…), size class (L1/L2/Sm/Med/Lrg), direction (D0–D3), limb sequence
(L/R/B/LL/RL/LH/RH…), swing (R2L/L2R/U2D/D2U — anim-variant selector,
verified by paired-name diffs), numeric variant, move name (`SpinSlash`).

## Per-captain record fields (all decoded)

`NPC_<n>` GUID string · level (u32 stored twice, 26 B before the
`Role_*` epithet hash) · `Faction_*` + `<Region>_Row_Captain_<n>` board
slot · `Role_<epithet>` display epithet · `TribeDef_*` · `AI_<class>` ·
spawn anchor · gear/visual-kit name refs (`AD_*` appearance defs) ·
trait lists.

## Census distributions (n = 534, endgame save)

- **Level**: 427/534 carry the `<lvl u32> 04 <lvl u32>` pair; values to
  84 (above the nominal 80 cap — pool/forced records). ~107 use an
  alternate `0f 02`-style header with no level — consistent with
  never-spawned pool rows (several carry a 10-byte raw GUID).
- **Resists**: median 9–10 `Picker_Resist_*` per captain (range 3–20).
- **Specialization** (`Marker_Specialization_*`): BeastMaster 72,
  Tank 64, Commander 59, Assassin 58, Tracker 57, Destroyer 57,
  Berserker 55, CombatMaster 47, Trickster 41, Marksman 30.
- **Class** (`ClassMarker_*`): Infantry 119, Hunter 91, Olog 91,
  Savage 89, Archer 72, Defender 70.
- **Potential**: Medium 249, High 98, Low 94, VeryHigh 48, VeryLow 42.
- **Rarity**: Common 345 (65%), Epic 129 (24%), Legendary 55 (10%).
- **Factions**: desert_Real 134, MtnPass 127, volcano 85, island 68,
  snowMtn 54, MinasIthil 40, Demovolcano 26.
- **AI class**: Infantry 109, Spearman 82, Olog_Infantry 82,
  Berserker 78, Defender 61, Archer 61.
- **Traits/record**: median ~46, p90 ≈ 55.
- Spec recount: **525/528 captains hold exactly one
  `Marker_Specialization_*`** (99%; 6 hold two).

## `_StructuresFactionTraitPickerList` member semantics (verified)

Member row = `[f32 weight, Marker gates…, Trait grants…,
Picker/PickerListRaw choices…]`:

- **Cell 0 = f32 weight** (misdecodes as refs — `0x3f800000` = 1.0).
- **`Marker` cells are presence prerequisites**, not grants. Verified
  against real records: `Marker_Specialization_CombatMaster`→SlowAdapt
  (47/47), `ClassMarker_<X>`→`BaseStrength_<X>` (534/534 match-own),
  `Marker_StatusEffect_Fire`→fire-offense (79/79),
  `Marker_Limiter_Grab`→`Grab_Limited` (80/80). Dormant-at-spawn rows
  (limiters, `NeverHumiliator`, `SpecialButterfly`) fire post-creation.
- **`Picker`/`RawList` cells = choice pool** — drawn members roll
  exactly one entry (the `Terror_LowHealth1-4` set yields one level).
- If **no member is gate-eligible**, the list falls back to one draw
  among all members (that's how non-limited captains get
  `Picker_Resist_Grab`).
- `TraitPickerQuantity` row = `[f32 weight, lo, hi, Marker gate?]` —
  gated rows are upgrades; when no row's gate is satisfied the full row
  set is used.
- `MarkerList` predicate cells are positional (VERIFIED on
  `MarkerList_Potential`/`LearnSpeed`): a `Predicates/RoleLists` ref in
  cell 1 = role must be a member; cell 2 = role must NOT be a member.
- `Rewards/Lists` cell 0 IS the def pick count:
  `Default_Resistances`=12, `Default_ExploitableWeaknesses`=3,
  `Extra_ExploitableWeaknesses`=2, `Default_SpecialWeaknesses`=2,
  `Default_OrcRarity`=1. Draws are weighted WITHOUT replacement; when
  pick count ≥ pool size every def fires but weighted draw order decides
  same-channel conflicts — `Default_Resistances` draws all 4 defs and
  the shared `PickerGroup_Resist_{Fire,Poison}` lets the first channel
  def win (Resistant w20 vs Immune w5 → ~80:20, matching the live mix).
- `Picker_` records' `Traits/Marker` cells are *blocking* gates
  (VERIFIED: all 9 `Picker_TribeAttack_*` carry `ClassMarker_Olog`;
  retail Ologs get tribe attacks only 3/83 while non-Ologs get 429/451 —
  the marker excludes Ologs). `Marker_NoStatusEffect` is special: the
  picker requires an element to be held.
- Tribe attacks link via the `Tribes/Definitions` record's direct
  `Traits/Picker` cell — NOT name-derived (`Sorcerous`→`Mystic`).
- `Role` records' `Traits/Marker` cells grant markers (VERIFIED:
  `Role_TowerGuard` → `Marker_NoTerrors` + `Picker_IronWill_Perm`).
- Orphan/scripted picker lists are dormant: `PickerList_Hates_Extra`
  has zero referencers and all-gated qty rows.
- Tribe coverage: retail captains are effectively all tribal; per-tribe
  tribeattack rate ~78–91%. **Undead DO carry tribe attacks** (44/52)
  despite `TribeDef_Undead` having no `Traits/Picker` cell — they were
  raised post-spawn and keep their prior attack.
- `PickerList_OrcRarity` lists all three rarity markers; rarity is
  decided by the `Default_OrcRarity` reward bundle first
  (`Rewards/Definitions` cell 1 = f32 weight, Common 85 / Epic 5 /
  Legendary 5).
- Class selection: builders lacking a `Combat/Tree` roll
  `Faction/CombatTreePicker::Default` — weighted
  `_StructuresFactionClassEntry` rows `[f32 weights…, Combat/Tree]`.
  Only ~28% of builders link a tree directly.

## `PickerGroup` mutual exclusion (verified)

Every `Faction/Traits/Picker` record can carry a
`Faction/Traits/PickerGroup` ref naming a shared exclusion family:
`PickerGroup_LowHealth_Fear_Hate` (the four `Terror_LowHealthN` tiers +
Fear/Hate variants), `PickerGroup_Fire_Fear_Hate`, `PickerGroup_Resist_
<channel>` (incl. `_Perm` scar pickers), `PickerGroup_GangOrPet`, etc.
Live picker history shows ~99.5% of members hold ≤1 picker per group —
it is retail's mutual-exclusion structure, distinct from
`Faction/Traits/Exclusion`. `SW_Terror_LowHealth1..4` reward defs all
resolve to the same `PickerListRaw_FearHate_LowHP` — each raw list
resolves at most once per captain (~10% live across all four tiers vs
~47% if stacked independently).

## `XPLevelSetup` checkpoint semantics (verified vs live bands)

`flag=1, lo==hi` rows (`Default_HasMount`, `Default_HasGang`,
`Default_SeismicStrike`…) are level-tier checkpoints: **every reached
checkpoint rolls once** — live rates rise stepwise with band
(`Picker_HasMount` 2% lv10-19 → 10% lv20-29 → 27% lv30-39;
`Picker_HasGang` 15% → 37% → 30%). `flag=1, lo<hi` rows keep rolling
per level `>= lo` (`hi` is a tier label, not a cap — retail
`Marker_StatusEffect_*` rising to ~70% by lv75 matches cumulative
bands only under this reading).

## Reward-def gates (verified)

- `Included/ExcludedCharacterType` attrs → `Combat/Predicates/
  CharacterType` records → member `Combat/CharacterType` names
  (`AdvancedClass_Marksman` requires `Trait_Archers`).
- `Champion<Spec>` defs are "epic trait for your spec" — excluded by
  `MarkerList_AC_CannotBecome<X>` = all spec markers except own +
  `RoleMarker_SpecializationNever_X`. Their `WeightMarker` rows are
  +3 own spec-or-prob, -3 every other Prob marker. Prob/Never markers
  are transient generation inputs (never persisted), same class as
  `OrcRarity_*_Chance`.
- `Included/ExcludedRole` hold `Faction/Predicates/RoleLists` refs or
  direct `Faction/Roles/Definitions` refs; `Included/ExcludedTribe`
  resolve `Predicates/TribeLists` or direct `Tribes/Definitions`
  (`EW_Fear_Burning` excludes `Roles_accustomed_to_fire` — 14 fire
  epithets). 231 defs carry non-empty role-list refs.
- **`MustHaveAvailableRole` is the ONLY role-transform slot**:
  `IncludedRole`/`ExcludedRole`/`WeightRole` are eligibility gates,
  while `MustHaveAvailableRole` (+ `SetRoleFromFilter` flag) marks a
  role transform (`PS_*`, `ResReward_TransformRole_*`,
  `Reward_PinnicleScar_*`) whose `RoleCategory_*` first member is the
  new epithet role. Treating gate slots as transforms wrongly granted
  role pickers (e.g. `Reward_IronWill`'s `ExcludedRole` →
  `Special_Butterfly_Roles` — 29 defs granting `Picker_AudioMannerism_*`
  as a side effect).
- `GlobalVarPredicate` refs resolve to `Quest/GlobalVarPredicates`
  records of shape `{AndTest:int, IntTests:struct-link*,
  BoolTests:struct-link*}`. Each `IntTests` target is an unnamed
  `{Max, Min, Variable}` record and each `BoolTests` target an
  unnamed `{RequiredValue, Variable}` record; `Variable` links a
  `GameData/Variables` global. VERIFIED bounds: `Act1` =
  `StoryState ∈ [0,199]` (a band — act-1-only, not a floor),
  `Act2` = `StoryState ∈ [200,9999999]`; `WarchiefsUnlocked` =
  OR over 4 bool vars (`WarchiefsUnlocked`, `InvasionModeActive`,
  two more). 37 records gate on `Act1` (the `Act1_AdvancedClass_*`,
  `DefaultAct1_*` defs and the bare `PowerStruggle`/`Hunting`
  generation types), 55 on `Act2` — this is the mechanism behind
  the `_Act{n}` table naming: gated JobLists and reward defs go
  dead outside their band, they do not fall back.
- Remaining undecoded def gates: `Included/ExcludedScarType`,
  `IncludedRowRank`, `IncludedAppearanceTags` (partially handled),
  `SetRoleFromFilter` semantics beyond the flag, `FloatTests`
  (schema attr exists; unused in the retail corpus).

## Live roster structure (verified in-game)

Read-only memory walk chain: `mgr=[world+0x6c68]` → `mgr+0x98` keyed
faction records (`0x290` stride) → shared inner member vecs. Observed
(endgame save, idle world):

- **562 members** across 3 shared vecs: the 511-slot union army pool
  (all `Faction_<region>` keys), a 120-slot invasion pool
  (`InvasionFaction_*`), and a 27-slot `Faction_Demovolcano` pool.
- **530/562 named** via `member+0x170` namehash → `OrcNameGen1st_*`/
  `OlogNameGen1st_*` genkeys. Level at `+0x4b8`.
- **Seat model**: the board IS the shared pool, ordered as
  region-hierarchy blocks (overlord+warchiefs+bodyguards first).
- `member+0x268` seat-record element marks authored seats
  (`<Region>_Row_<Seat>` name) — NULL on procedurally filled members:
  the authored/procedural split is directly visible.
- `member+0x1f8` committed log = per-member trait defs (`Traits_*`,
  `Default_Act1`, `BER_Hate`).

### Applied-trait surface (member ptr-vec fields)

| Offset | Applied names |
|--------|---------------|
| `+0x0a0` | gear kit (`<Item>` + `AD_<Item>` pairs) |
| `+0x198` | resists/fears/weakened (`Resistant_*`, `Vulnerable_*`, `Immune_*`, `Fear_*`, `Weakened_*`, `Hate_*`, quirks) |
| `+0x1b0` | `Marker_*`/`ClassMarker_*`/`PersonalityMarker_*` |
| `+0x1e0` | picker history — `Picker_*[_Perm]` + outcome pairs |
| `+0x1f8` | `Default*` + `Traits_*` class/spec defs |
| `+0x538` | `Tag_*` body/appearance tags |

- `+0x1e0` picker history is a **bounded window** — 191/223 live
  members carry exactly 16 entries (likely ring/most-recent), so
  per-picker rates are undercounts for members whose spawn picks were
  evicted. Outcome vectors (`+0x198`, `+0x1b0`) are the authoritative
  applied set.
- Unnamed members (32) cluster as lv9 `Faction_MinasIthil`
  5–11-trait scripted casts (name overridden outside the genkey pool;
  `MinasIthil_Row_Overlord` carries 11 traits).

### Save ↔ live identity bridge (verified)

Each WHDC captain record embeds the **namehash of its genkey name**
(`OrcNameGen1st_NNN`/`OlogNameGen1st_NNN`). Scanning decrypted
plaintext for the hashed genkey space recovers the save's captain
sequence verbatim: **530/530 WHDC genkey records match live members
1:1** — the save census IS the live roster.

- **Save order is NOT pool-seat order** — 237/529 adjacent seat
  inversions, 423 faction runs, no correlation with member allocation
  order (r = −0.15). The WHDC sequence is an internal
  iteration/insertion order that **reshuffles as the roster churns**:
  two same-roster snapshots preserve only ~57% of pairwise order.
- Consequence: the exact initial board-fill sequence cannot be
  recovered from a late-game save; it needs a fresh-game live capture
  or the fill code path read statically. The seat→member mapping is
  fully decodable live.

## Open

- `SimEventChance` 8-float field semantics beyond the win-chance cell.
- `SimEventGenerationType` cells 0–2 units (variable-length; undecoded).
- SimEvent→SimEvent ref slot semantics (not proven success/fail chains).
- Engine-side job/cadence logic that turns picker routing into rates.
