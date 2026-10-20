# Nemesis parity baseline

> **Status:** living — census + scorecard, updated per parity run
> **Provenance:** endgame save `07010497_1` (534 WHDC captain records) via `anduril save <f> captains`
> + `tools/nemesis_parity.py` scorecard vs `anduril-sim` output; live-roster census via
> `tools/roster_live.py`. Reproduce instructions inline.

Ground truth extracted from a real endgame save (`07010497_1`, 534 captain
records in WHDC) via `anduril save <f> captains`. Used to validate
`anduril-sim` captain generation and hierarchy behaviour. Reproduce with
`ANDURIL_HASHDB=<hashdb>` + `save … captains` against any save; numbers
below are save-specific (this is a late-game state — levels skew high).

## Per-captain fields (all decoded)

`NPC_<n>` GUID string · level (u32 stored twice, 26B before the `Role_*`
epithet hash) · `Faction_*` + `<Region>_Row_Captain_<n>` board slot ·
`Role_<epithet>` display epithet · `TribeDef_*` · `AI_<class>` · spawn
anchor · gear/visual-kit name refs (`AD_*` appearance defs) · trait lists.

## Distributions (n = 534)

- **Level**: 427/534 records carry the `<lvl u32> 04 <lvl u32>` pair;
  values observed to 84 (above the nominal 80 cap — pool/forced records).
  The remaining ~107 use an alternate `0f 02`-style header with no level —
  consistent with never-spawned pool rows (several carry a 10-byte raw
  GUID instead of the NPC string).
- **Resists**: median 9–10 `Picker_Resist_*` per captain (the full damage
  channel block is rolled per captain; range 3–20 with outliers).
- **Specialization** (`Marker_Specialization_*`): BeastMaster 72, Tank 64,
  Commander 59, Assassin 58, Tracker 57, Destroyer 57, Berserker 55,
  CombatMaster 47, Trickster 41, Marksman 30.
- **Class** (`ClassMarker_*`): Infantry 119, Hunter 91, Olog 91,
  Savage 89, Archer 72, Defender 70.
- **Potential**: Medium 249, High 98, Low 94, VeryHigh 48, VeryLow 42.
- **Rarity**: Common 345 (65%), Epic 129 (24%), Legendary 55 (10%) —
  plus forced/chance variants (~9%).
- **Factions**: desert_Real 134, MtnPass 127, volcano 85, island 68,
  snowMtn 54, MinasIthil 40, Demovolcano 26.
- **AI class** (record's `AI_*`): Infantry 109, Spearman 82,
  Olog_Infantry 82, Berserker 78, Defender 61, Archer 61 (+45 records
  carry a stale `BIP_BleedOut_TimedOut` animation name — dead/reserve?).
- **Traits/record**: median ~46 (Marker/Picker/Traits/ClassMarker/
  PersonalityMarker names), p90 ≈ 55.

## Parity gaps vs `sim nemesis` (as of 07ef568)

- Sim captains roll only the builder's TraitLists/PickerListRaw entries
  (0–3 traits) — the real record carries the *resolved* block: ~10
  resists, bodyguard count, gang, mount, TribeAttack, BaseStrength per
  class+spec, hate/fear/betrayal pickers, appearance, learn speed,
  potential, rarity.
- Sim has no level field (power 1–20 stands in); real records store
  level u32 twice.
- Roles absent on ~half of sim spawns vs always-present epithet in save.
- Real board seats are named `<Region>_Row_Captain_<n>`; only 177/534
  records carry one (seated in hierarchy), rest are unseated pool/dead.

## Sim progress (post-PickerList integration)

`generate_captain` now rolls every `Faction/Traits/PickerList` record per
captain (Setup Default's 19 linked lists first, then the remaining 22 in
record order — the unreferenced lists are list-driven, not ref-driven).
Each list draws `min..max` members without replacement per its
`_StructuresFactionTraitPickerQuantity` range.

A second pass rolls the `Faction/Rewards/XPLevel` bundles (the
level-driven reward pipeline): each bundle's
`_StructuresFactionRewardXPLevel` rows are weighted alternatives pointing
at `Faction/Rewards/Lists` pools (`[pick_count, Rewards/Definitions
refs]`), and each definition contributes its direct marker/trait refs
plus its `RewardPickerList` / `PickerListRaw` outcomes. Rolled per captain:
`Default_OrcRarity` (feeds the
rarity markers), `Default_Resistances`, `Default_*_Weaknesses`,
`Default_Nemesis`, status-effect/gang/mount/base-class rewards,
`GrowInPower`, `IronWill`, and `Default_EpicTraits_{Epic,Legendary}Orc`
gated on the rolled rarity.

`PickerGroupsToGive`/`PickerGroupsToRemove` rows are per-row draws: every
`_StructuresFactionRewardPickerList` struct is ONE independent weighted
draw over its own cells, and every cell is an alternative — verified rows
are all-refs (no `[weight, refs]` shape exists; picker weights are 0 so
the draw is uniform). Single-cell rows are guaranteed grants
(`AdvancedClass_Berserker`: live spec-berserkers hold
LifeSteal/PrimalRoar/Tackle at 96-98%); multi-cell rows are choice pools
(`AdvancedClass_Berserker` row ord 2365 = 26 `Picker_Hate_*` → ~4% each
live; `BER_Hate` def's row ord 2490 the same pool). `PickerGroupsToRemove`
draws one cell and deletes its outcome (live spec-berserkers:
`Picker_BodySlam` 1.8%). The earlier flat-pool read skipped cell 0 as a
"weight" — that made the 26-hate pool deterministically resolve to cell 1
(`Picker_Hate_Curse` → 37% vs live 4.7%).

Picker outcomes are resolved through
`_StructuresFactionTraitPickerEntry` rows to `Traits/Definitions` names,
and `Faction/Traits/Exclusion` groups (`Exclude_Fire`, `Exclude_Fears`,
…) are enforced: at most one picker-derived outcome per group survives.
Quantity rows are treated as weighted alternatives (not a uniform range),
reward-bundle rows are level-band gated (`lo == hi` = minimum level) and
every eligible row rolls its pool, and `ET_*` outcomes are dropped unless
the rolled rarity is Epic/Legendary.

## `_StructuresFactionTraitPickerList` member semantics (verified)

Member row = `[f32 weight, Marker gates…, Trait grants…,
Picker/PickerListRaw choices…]`:

- **Cell 0 = f32 weight** (misdecodes as refs — `0x3f800000` =
  `SnowMtn`, `0x40800000` = `BIP`, `0x42a00000` = `Spider_Idle`).
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
  gated rows are upgrades (`RoleMarker_ExtraHates`, `ClassMarker_Olog`
  rows); when no row's gate is satisfied the full row set is used.
- `MarkerList` predicate cells are positional (VERIFIED on
  `MarkerList_Potential`/`LearnSpeed`): a `Predicates/RoleLists` ref in
  cell 1 = role must be a member (`Roles_who_grow_fast`), cell 2 = role
  must NOT be a member (`Roles_who_grow_slow`). Rows pre-filter the
  weighted pool — potential/learnspeed match the census exactly.
- `XPLevelSetup` flag semantics (VERIFIED on census): `flag=1, lo==hi`
  rows are single-tier checkpoints — only the highest passed tier rolls
  once (flat retail `HasMount` rate proves they don't accumulate);
  `flag=1, lo<hi` rows keep rolling per level `>= lo` (`hi` is a tier
  label, not a cap — retail `Marker_StatusEffect_*` rising to ~70% by
  lv75 matches the cumulative bands only under this reading).
- `Rewards/Lists` cell 0 IS the def pick count (CORRECTED — the
  earlier `Str`-provenance reading under-covered every resist channel
  ~63% vs live ~91%): `Default_Resistances`=12, `Default_
  ExploitableWeaknesses`=3, `Extra_ExploitableWeaknesses`=2,
  `Default_SpecialWeaknesses`=2, `Default_OrcRarity`=1. Draws are
  weighted WITHOUT replacement; when pick count ≥ pool size every def
  fires but the weighted draw order decides same-channel conflicts —
  `Default_Resistances` draws all 4 defs and the shared
  `PickerGroup_Resist_{Fire,Poison}` lets the first channel def win
  (Resistant w20 vs Immune w5 → ~80:20, matching the live grade mix).
- `Picker_` records' `Traits/Marker` cells are *blocking* gates and
  apply on every path (VERIFIED: all 9 `Picker_TribeAttack_*` carry
  `ClassMarker_Olog`; retail Ologs get tribe attacks only 3/83 while
  non-Ologs get them 429/451 — the marker excludes Ologs, matching
  `picker_allowed` semantics). `Marker_NoStatusEffect` is special: the
  picker requires an element to be held.
- Tribe attacks link via the `Tribes/Definitions` record's direct
  `Traits/Picker` cell — NOT name-derived (`Sorcerous`→`Mystic`).
- `Role` records' `Traits/Marker` cells grant markers (VERIFIED:
  `Role_TowerGuard` → `Marker_NoTerrors` + `Picker_IronWill_Perm`).
- Orphan/scripted picker lists are dormant: `PickerList_Hates_Extra`
  has zero referencers and all-gated qty rows — the ambient sweep must
  not roll lists that only exist for script contexts.
- Tribe coverage: retail captains are effectively all tribal — the `?`
  census column is unresolved hash names; per-tribe tribeattack rate is
  ~78-91% (Terror 45/58, Feral 47/58, Dark 43/53, Slaughter 38/53,
  Undead 44/52, Warmonger 40/50, Marauder 42/46, Sorcerous 40/46,
  Outlaw 37/45). **Undead DO carry tribe attacks** (44/52) despite
  `TribeDef_Undead` having no `Traits/Picker` cell — they were raised
  post-spawn and keep their prior attack (consistent with the
  `Marker_Resurrected_*` evidence).
- `PickerList_OrcRarity` members list all three rarity markers — the
  row matching the already-rolled rarity fires (rarity is decided by
  the `Default_OrcRarity` reward bundle first: `Rewards/Definitions`
  cell 1 = f32 weight, Common 85 / Epic 5 / Legendary 5).
- Class selection: builders lacking a `Combat/Tree` roll
  `Faction/CombatTreePicker::Default` — weighted
  `_StructuresFactionClassEntry` rows `[f32 weights…, Combat/Tree]`.
  Only ~28% of builders link a tree directly.

## Scorecard (`tools/nemesis_parity.py`, n_real=534 / n_sim=500)

Latest run — **procedural mode** (`sim nemesis <seed> <n> [t]
procedural`; ClassMemberSetup-driven, no builder record), seed 1,
n=500:

Nailed: `tribeattack` 80/81, `gang` 45/44, `weakened` 42/47,
`giveup` 43/47, `slowadapt` 85/83, `mines` 13/11, `mannerism` 19/16,
`hate` 87/84, `loyal` 4/5, resist_* ~96-100/99 (Curse 34/43),
potential/learnspeed near-exact, class mix close
(Infantry 25/22, Olog 20/17, Archer 17/13, Hunter 13/17,
Savage 12/16, Defender 11/13).

`rarity` fixed by construction: 87/12/6 Common/Epic/Legendary vs real
64/26/10 — the Legendary skew was 100% authored-builder pollution
(scripted `*_Leader`/`TribeBattle_*_Legendary` records force
`Marker_OrcRarity_Legendary`). Creation-time Epic is rare in retail
too (0% among lv0-9 captains); the 26% Epic is post-spawn promotion.

Still divergent (procedural):

- `fear` 68/55 — per-captain quantity hot; **mass tail fully gone**
  (max 4 fears; retail max 5). Residual suspect: ambient list rolls.
- `specmarker` coverage 100/99 but the *mix* skews hard:
  Berserker/Marksman/Tank/Commander/Assassin/Tracker all ~39-44% each
  vs real's flat ~10-13% per family — specialization source pool is
  too wide (exactly 1 spec marker per captain, correct count).
- `mount` 15/33, `epictrait` 12/34, `ironwill` 18/28,
  `humiliator` ~0/11 — under-generated; humiliator's pickers lived
  only on authored builders, so a generic grant path is still missing
  (also true for builders mode).
- `marker_other` 22/57 — post-spawn history markers (expected gap).
- `personality` 100/81 — retail's 19% marker-less captains are
  raise-dead transformations (verified `Marker_Resurrected_…` +
  `Marker_Deranged`), a post-spawn path not generation.
- Immune-tier mix skews Resistant-heavy — `PG*_Immune` upgrade defs
  likely need level/potential gates we don't feed.

Prior builders-mode scorecard (kept for reference): fear 66, hate 91,
rarity Legendary 50%, tribeattack 87, epictrait 27, mount 20 —
procedural mode beats it on every metric that matters, confirming the
retail board path does not consume `FactionMemberBuilder` records.

Appearance (now ~100% vs real 99%): `Marker_AppearanceTag_{Buff,Slim,
Stocky}` mirrors the `Models[]` body-variant draw on the captain's
`Model/SimpleModel` (per-entry f32 weight; body class read off the
`SimpleModelPieces`/`SimpleModelHeads` names — `_Buff_`/`_Slim_`/
`_Stk_`). Builders without a `Character/Models` ref (~85%) fall back
to the class model (`Orc_Defender`, `Orc_Archer`, `Olog_Bruz`, …).
Sim split 42/46/12 vs real 45/35/19. `Marker_AppearanceTag_BigNose`
(real 20%) has no piece-name source in the entries seen — unsourced.

Remaining known approximations: `PickerLevelRemap` level scaling,
`FactionPredicateClass` gating, DLC4 Easterling variants not
region-gated, `RewardWeightTribe` structs unapplied, member-gate
polarity inferred from correlations (limiter/spec/class/SE markers
verified; `Marker_HasHypeMan`/`NeverHumiliator` assumed).

## Board-population gap (resolved — procedural mode)

`sim nemesis` draws uniformly from all 608 `Faction/FactionMemberBuilder`
records; ~30% of generated captains come out tribeless because their
builders are scripted/story specials (`SW_*` 230, `DLC5_*` 128,
`Gauntlet`, `OnlineArena`, `Finale_*`, `Follower`, …) that carry no
tribe ref, no `Tribes/Lists`, and a role-less `RoleLists` pool. Retail
board captains are effectively all tribal, so the real spawn path must
restrict the builder set.

Progression of the eligibility model (each step verified, seed-1 n=500):

1. Role/tribe resolution (476 builders): `tribeattack` 57→77.
2. `+` role pool must intersect the generic epithet set — the union of
   `All_Roles{,2,3,NonJarring,NonJarring_2}` from
   `Faction/ClassMemberSetup::Default`'s `_StructuresFactionClassDataMap`
   rows (153 builders): `tribeattack` 87.
3. `+` zero authored `Faction/Traits/Picker` cells (97 builders):
   VERIFIED bimodal census — 508/608 builders carry 0 Picker cells;
   the 100 that do are all named scripted casts (`SotP_*` 18-57,
   `BladeSide*` 37-41 including ~30 `Picker_Terror_*` entries,
   `Captain_Assassin` 14). Builders that DO have pickers are authored
   fixed captains; builders with none are procedural templates whose
   traits come from `Setup::Default` rolls. This removed the
   mass-terror tail entirely (0 captains ≥6 fears; retail max 5).

Ruled out:

- `FactionMemberBuilderInstance` refs mark *scripted placement*, not
  board eligibility — every bearer (`Gauntlet_*`, `SW_*_Leader`,
  `DLC5_*`, `Captain_Assassin`) is an authored cast; the 46-builder
  instance pool produced Legendary 65% vs real 10%.
- Wholesale `Quest/Jobs/Definition`-reference exclusion over-filters
  (476→58): legit gameplay builders are also job-referenced.

Resolution (this session): the "fully procedural" hypothesis was
tested directly. `spawn_procedural` synthesizes a captain from
`Faction/ClassMemberSetup::Default`'s `_StructuresFactionClassDataMap`
rows — family (Orc/Olog; Forthog excluded, it's a scripted savior)
→ `Combat/Tree` → `All_Roles*` role → tribe → `Setup::Default`
rolls — through the same `realize` tail as builder spawns, so every
verified roll (pickers, markers, rewards, rarity cleanup, tribe
attack, personality table) is shared. 500-captain census: the
Legendary skew vanished (6% vs the builders-mode 50%), mass-terror
tail gone, `tribeattack` 80 vs real 81. Modeled constants: Orc/Olog
17% split (no weight cells in the class map), power bands, and the
region draw are census-derived, not record-derived — flagged in
`spawn_procedural`'s comment. The builder-filter machinery stays
for scripted casts (`with_list`, `spawn` by name).

`Faction/Rows` = exclusions only, `PresetMembers` = scripted presets,
`Faction/Pool::MainGame` has no builder refs — the real seat→spawn
mapping is still unidentified, but the procedural class map now
explains the population data well enough that the gap is downgraded
from `blocked` to an open question about *seating order*, not
*generation content*.

## Live roster census (verified in-game, this session's ground truth)

Beyond the save snapshot, `tools/roster_live.py` walks the **live**
nemesis roster in a running game (read-only `ReadProcessMemory`).
Chain: `mgr=[world+0x6c68]` → `mgr+0x98` keyed faction records
(`0x290` stride) → shared inner member vecs. Full member layout is
in `engine-spawn-pipeline.md`.

Observed live (endgame save, idle world):

- **562 members** across 3 shared vecs: the 511-slot union army
  pool (all `Faction_<region>` keys), a 120-slot invasion pool
  (`InvasionFaction_*`), and a 27-slot `Faction_Demovolcano` pool.
- **530/562 named** via `member+0x170` namehash →
  `OrcNameGen1st_*`/`OlogNameGen1st_*` genkeys. Level at `+0x4b8`.
- **Seat model**: the board IS the shared pool, ordered as
  region-hierarchy blocks (overlord+warchiefs+bodyguards first).
- **`member+0x268` seat-record element** marks authored seats
  (`<Region>_Row_<Seat>` name) — NULL on procedurally filled
  members: the authored/procedural split is directly visible.
- `member+0x1f8` committed log = per-member trait defs
  (`Traits_*`, `Default_Act1`, `BER_Hate`) — decodes every
  member's class spec live.
- Unnamed members (32) cluster as lv9 `Faction_MinasIthil`
  5–11-trait scripted casts (name overridden outside the genkey
  pool; `MinasIthil_Row_Overlord` carries 11 traits).

Export: `Helper/roster_live.tsv` (gitignored) —
addr/pool/seat/actor/name/level/faction/seat_rec/traits per member.
This dataset complements the save census: the save is a static
snapshot, this is the engine's own live state — usable for
live-vs-sim distribution diffs on real captain composition.

### Save ↔ live identity bridge (verified)

Each WHDC captain record embeds the **namehash of its genkey name**
(`OrcNameGen1st_NNN`/`OlogNameGen1st_NNN`). Scanning decrypted
plaintext for the hashed genkey space recovers the save's captain
sequence verbatim:

- `tools/save_roster_correlate.py <save.pt>` → **530/530 WHDC
  genkey records match live members 1:1** (0 unmatched) — the save
  census IS the live roster, identity bridge proven.
- **Save order is NOT pool-seat order** — 237/529 adjacent seat
  inversions, 423 faction runs (no positional grouping), no
  correlation with member allocation order (r = −0.15). The WHDC
  sequence is an internal iteration/insertion order that
  **reshuffles as the roster churns**: two same-roster snapshots
  preserve only ~57% of pairwise order.
- Consequence: the exact initial board-fill sequence cannot be
  recovered from a late-game save; it needs either a fresh-game
  live capture or the fill code path read statically. The seat
  → member mapping itself, however, is now fully decodeable live.

### Full applied-trait surface (verified live)

Beyond `+0x1f8`, members carry their complete applied record set
in ptr-vec fields (mapped by `tools/member_surface.py`):

| Offset | Applied names |
|--------|---------------|
| `+0x0a0` | gear kit (`<Item>` + `AD_<Item>` pairs) |
| `+0x198` | resists/fears/weakened (`Resistant_*`, `Vulnerable_*`, `Immune_*`, `Fear_*`, `Weakened_*`, `Hate_*`, quirks) |
| `+0x1b0` | `Marker_*`/`ClassMarker_*`/`PersonalityMarker_*` |
| `+0x1e0` | picker history — `Picker_*[_Perm]` + outcome pairs |
| `+0x1f8` | `Default*` + `Traits_*` class/spec defs |
| `+0x538` | `Tag_*` body/appearance tags |

`roster_live.py` emits all of them as a 10th TSV column — every
live member now decodes the same feature set the save census shows,
but at *current* state (the save's record is a historical
snapshot; these are the engine's live applied sets).

### Live-vs-sim scorecard (first live-truth run)

`nemesis_parity.py Helper/live_traits.txt Helper/sim_out.txt`
(live n≈220 board members, early-game save state — levels 20–43;
sim n=500 procedural):

Nailed: giveup, loyal, ironwill 18/13, mount 16/19, slowadapt,
weakened, learnspeed/potential/class mix.

Newly visible divergences (the save census showed only applied
*history*, masking these):

- `statuseffect_marker` **sim 61% vs live 24%** — over-generated.
- `weapon_elem` **sim 61% vs live 25%** — over-generated (same
  root suspicion: dormant-at-spawn pickers firing in the ambient
  sweep, or a gate we don't model).
- `gang` sim 45% vs live 27%; `tribeattack` 80% vs 69%;
  `fear` 69% vs 59%; `resist_Curse` 34% vs 17%;
  `resist_Frost` always-Resistant vs live 86% (live has vuln/
  immune tail).
- `epictrait` sim 12% vs **live 19%** — the save census's 34%
  was post-spawn promotion on a lv80 board; at lv20-43 live the
  creation-time rate is ~19%, still above sim.
- `humiliator` live 6% vs sim ~0-1% — still the sharpest deficit.
- `marker_other` live 43% vs sim 22% — post-spawn history markers
  (expected gap at t=0).
- Sim emits some categories at 100% where live shows ~90%
  (potential/learnspeed/rarity/personality markers) — ~10% of
  live members lack the marker vec entirely (placeholder/unnamed
  members, or genuinely marker-less board members).

**Level-matched correction** — the live board was lv20-43 while
sim's power model (`51% → 76-84, else 8-75`) was tuned to the lv79
endgame save. Restricting sim to `power ≤ 45` (n=131) vs live
(n=216) nulls most of the raw divergences:

- `statuseffect_marker` 24→**29/24**, `weapon_elem` 25→**29/25**,
  `epictrait` 19→**19/19** — the earlier 2.5× gaps were level
  artifacts, not roll bugs.
- Remaining true divergences at matched level: `fear` sim 76 vs
  live 59, `hate` 90/76, `gang` 44/27, `tribeattack` 81/69 (hot);
  `ironwill` 2/13, `mount` 11/19, `humiliator` 0/6,
  `resist_Curse` 7/17 (cold — interesting inversion: Curse-immune
  OVER-fires at high level, under-fires at low level).

**Methodology rule**: live-vs-sim parity must always be computed
level-matched; raw board-vs-board comparisons confound level
distribution with roll logic. The sim power-band model itself
needs region/state-aware bands (the 51% high-level skew only fits
late-game boards).

### `Faction/Traits/PickerGroup` mutual exclusion (verified, implemented)

Every `Faction/Traits/Picker` record can carry a
`Faction/Traits/PickerGroup` ref naming a shared exclusion family:
`PickerGroup_LowHealth_Fear_Hate` (the four `Terror_LowHealthN`
tiers + Fear/Hate variants), `PickerGroup_Fire_Fear_Hate`
(Fear_Burning / Terror_Fire / Hate_Burning alternates),
`PickerGroup_Resist_<channel>` (incl. `_Perm` scar pickers),
`PickerGroup_GangOrPet`, etc. Live `+0x1e0` picker history shows
~99.5% of members hold ≤1 picker per group — it is retail's
mutual-exclusion structure, distinct from `Faction/Traits/Exclusion`.

Sim now enforces `pg:{group}` dedupe in `apply_picker_mode`, with
`PickerGroup_Resist_*` groups exempted (already covered by the
`resist:{channel}` rule; `_Perm` scars legitimately share the group).

Also verified: `SW_Terror_LowHealth1..4` reward defs all resolve to
the same `PickerListRaw_FearHate_LowHP` — raw-list outcomes are now
deduped per captain (`rl:{list}`) since retail resolves each raw
list at most once (~10% live across all four tiers vs ~47% when
stacked independently).

### Live picker-history vs sim draws (level-matched, lv~28)

`Helper/live_pickers.txt` (n=223, `+0x1e0`) vs debug `PUSH via
Picker_*` counts (n=200, `Helper/sim_pickers.txt`):

- `WeakBy` family calibrated: BerserkWearsOff 21%/23%,
  PinningShot 12%/9%, Headshots 9%/9%, StealthAttacks ~9%/7%.
- `GrenadedRetreat` is class-skewed in live (Hunter 54%, Infantry
  20%, Archer 17%) — sim's ~20% overall is in range; the parity
  category conflates class prevalence.
- Residual ~1.5-2× uniform over-fire across ALL fear/terror
  triggers (not one structural defect): per-group probabilities
  are individually too hot → points at pool draw *frequency*
  (pick counts / per-level accumulation in `reward_bundle`),
  not picker logic. Needs the retail XP-row interpreter read.

### Live perk accumulation vs level (verified)

Perk counts correlate only weakly with level (perks r=0.39,
pickers r=0.25); per-band rates for fear/terror/gang are nearly
flat (fear_p ~30% at every band, terror_p ~25-30%, gang 10→37%
between lv<10 and lv20-29 then flat). Retail's creation-time
weakness pool draws dominate; level-up accumulation is modest —
consistent with spawned captains never having "played" their levels.

### XP-row checkpoint semantics revised (verified vs live bands)

`flag=1, lo==hi` rows (`Default_HasMount`, `Default_HasGang`,
`Default_SeismicStrike` …) are level-tier checkpoints — live
`+0x1e0` rates rise stepwise with band (`Picker_HasMount` 2%
lv10-19 → 10% lv20-29 → 27% lv30-39; `Picker_HasGang` 15% → 37%
→ 30%), inconsistent with the earlier "highest tier only" model.
Sim now rolls **every reached checkpoint once**. Result at lv28:
mount 16→22 (live 19), ironwill 1→4 (live 13), statuseffect 31→24
(live 24 — now exact).

### Live `+0x1e0` picker history is a bounded window

191/223 live members carry exactly 16 picker-history entries —
`+0x1e0` is a bounded log (likely ring/most-recent), so per-picker
live rates are *undercounts* for members whose spawn picks were
evicted by later level-up picks. Treat picker-history diffs as
directional, not absolute truth; outcome vectors (`+0x198`,
`+0x1b0`) remain the authoritative applied set.

### Residual gang/pet over-fire — source isolated

`PickerList_GangsPets` stays in the ambient sweep and yields
~43% gang-or-pet alone; `Default_HasGang` checkpoint draws add
~14% → sim ~48% vs live ~35% combined. Either the list's qty
semantics still over-draw, or `Default_HasGang` flag=0/flag=1
rolls are level-up-only in retail (not spawn). Next step
unchanged: decode the native XP-row/list interpreter.

### Second live census — endgame board + direct accumulation measurement

A second full live walk (562 members, 530 named) captured an
**endgame board** (lv0–84; dominant cohort lv76+, n=232) —
complementing the earlier lv20–43 capture (n=223). Splitting live
members by level band measures trait accumulation across a
captain's life directly (no sim involved):

| category | lv≤50 (n=90) | lv≥70 (n=232) |
|---|---|---|
| epictrait | 21% | 51% |
| fear | 43% | 48% |
| gang | 23% | 34% |
| hate | 63% | 84% |
| ironwill | 12% | 19% |
| mount | 10% | 28% |
| statuseffect_marker | 18% | 69% |
| tribeattack | 54% | 81% |
| weapon_elem | 21% | 69% |
| unbreakable | 10% | 17% |
| humiliator | 7% | 11% |

Every category grows steeply — retail **does** accumulate rolls as
captains level/promote (level-up reward path, `GrowInPower`, etc.),
consistent with the per-level/per-checkpoint accumulation model.

**Re-baseline under the VERIFIED draw rule** (BAND-cap+backfill,
from `FUN_14034f9f0` — replaces the empirical FINAL default;
`sim nemesis 1 200 0 '' level=N`, n=200):

| category | live~lv28 | sim lv28 | live~lv80 | sim lv80 |
|---|---|---|---|---|
| fire_resist | 58% | 65% | 48% | 69% |
| fire_immune | 15% | 12% | 31% | 21% |
| fire_vuln | 18% | 18% | 13% | 6% |
| statuseffect | 24% | 19% | 60% | 37% |
| weapon_elem | 24% | 19% | 60% | 36% |
| curse_immune | — | 20% | ~45% | 46% |
| grenade | — | 28% | ~36% | 28% |
| mount | — | 16% | ~28% | 19% |

Reading: fire grade mix stays near-live at lv28; `curse_immune`
lands ON target at lv80 (the FINAL model's over-hot stale-band
draws were masking it). The `statuseffect`/`weapon_elem` lv80 gap
(37/36 vs 60) is now attributable to the decoded-but-unmodeled
**event/purpose reward surfaces** (`member+0x4f0` row-exclusion
list, `purpose_levelremap_record` remap) — element weapons arrive
through event rewards, not XP rows. `fire_vuln` lv80 6% vs 13%:
the scar/downgrade regeneration path partially compensates;
`fire_resist` +21pt suggests upgrade defs over-fire without the
`+0x4f0` exclusions suppressing them.

**Level-matched scorecard at lv80** (live n=232 vs sim n=200,
`sim nemesis 1 200 0 '' level=80`):

| category | live | sim |
|---|---|---|
| statuseffect_marker | 69% | 73% |
| weapon_elem | 69% | 73% |
| hate | 84% | 89% |
| tribeattack | 81% | 86% |
| ironwill | 19% | 22% |
| mount | 28% | 36% |
| fear | 48% | 64% |
| gang | 34% | 65% |
| epictrait | 51% | 24% |
| unbreakable | 17% | 15% ✓ |
| humiliator | 11% | 2% |

The accumulation model is now validated at **both** bands — the
aggregate categories that were 2.5× off at creation-conflation land
within ~4–8pt at lv80. Remaining gaps are specific, not systemic:

- `Quirk_Unbreakable` — **SOLVED (15% sim vs 17% live)**: the grant
  path is the prestige-role draw, not XP rows or resurrection. Stock
  GameDB has exactly three random-captain routes to the quirk:
  `Role_Unbreakable` (`All_Roles_3`) and `Role_Pretender`
  (`All_Roles_2` / `RolesWhoShouldNotBeLowLevel`) grant
  `Picker_Unbreakable_Always`; `PickerListRaw_Unbreakable` is
  Minas-Morgul-rewards only. The decisive decode: **role draws are
  weighted by `Faction/Roles/Definitions` cell-1 f32** — Pretender
  carries weight 150.0 against a ~0.9 baseline (Coward_Ratbag 440,
  Berserker 80). In `RolesWhoShouldNotBeLowLevel` (99 roles,
  Σw=1060), Pretender alone is **14.1%** of the draw. Board fillers
  spawn from `Roles_who_are_unimportant` (369 roles, 0% ub weight)
  and veteran members re-draw from the prestige pool — modeled as a
  level≥30 re-draw until the `Faction/Rows` seat tier is simulated.
  Live correlation confirmed: 58/562 (10.3%) roster ub is flat
  across class/spec/level/region — only a global re-draw pool can
  produce that profile.
- `epictrait` now lands ~on target at lv80 (38 vs live 34) once
  the prestige pool feeds rarity: 68/99 prestige roles carry
  `Marker_OrcRarity_{Epic,Epic_Chance,Legendary_Forced}` and
  rarity→ET is near-1:1 in live records (all 129 Epic + 54/55
  Legendary hold ETs ≈ 34% overall).
- **Rarity marker resolution (implemented)**: `Marker_OrcRarity_
  {Tier}_Chance` is a *transient input* consumed by the rarity
  resolution — live records keep it on only ~5% of captains, and
  the base tier is drawn by `Default_OrcRarity` alone (Common
  64/Epic 26/Leg 10 live). `Marker_OrcRarity_{Tier}_Forced`
  always upgrades the base marker. Sim now resolves both at
  finalization → **rarity Common61/Epic27/Leg13 vs live
  64/26/10** at lv80; the universal ≥30 prestige re-draw is
  gated to a ~2/3 seat share, which jointly reproduces
  ub 10.7% (live 10.3) AND the rarity mix — evidence that the
  prestige pool exposure is per-seat, not per-captain.
- **Applied-picker persistence (implemented)**: retail saves store
  every *applied* `Picker_*` record name in the member's marker
  set — `Picker_Follower_GrabAndThrow` on ~99%, `Picker_Bodyguard
  Count_2` on ~64% — regardless of outcome. The picker is
  persisted when it survives veto checks (group/marker/exclusive),
  NOT only when an outcome row hits: `Picker_BodyguardCount_*`'s
  rows are warchief-gated quirks yet the picker name persists on
  64% of ALL captains. Sim now inserts the `pk:` key at veto
  survival and harvests `pk:` → markers at finalization →
  `bodyguards` 18→100% and `follower_grab` 37→100% at lv80
  (live 98–99), betray/morale/inhierarchy all ~100%.
- `fear`/`weakened` presence ~2× hot (fear 97 vs 55, weakened
  71 vs 47): **the level-0 question is RESOLVED** —
  `FUN_14034f9f0` iterates `trait_level ..= menace_level`
  INCLUSIVE, so level 0 is a real draw level: every flag=0 row
  is stored `[1,1]` → effective `[0,0]`, i.e. the creation pass
  is just level 0 of the same uniform band loop (`flag` selects
  the `TraitLevelOffset` remap predicate, not creation-vs-
  levelup). Pick count = the `Rewards/Lists` `Count` attr
  (FUN_14034f980, default 1). The weakness pools already carry
  `EW_/SW_Extra_NothingChance_*` absorber defs (w=200) — the
  arithmetic lands ~2 weaknesses/captain matching live; sim's
  residual over-production comes from additional ambient
  TraitPickerList draws + GrowInPower per-decade picks feeding
  the same families, not the weakness rows themselves.
  Implemented: cross-bundle dedup now applies to flag=0 rows
  at their match level (`Extra_Special_Weaknesses` shares the
  `Default_SpecialWeaknesses` list → draws once), a dedup skip
  marks the row band-matched so the no-band fallback can't
  re-roll the pool (was double-drawing shared lists 2×),
  `PickerGroupsToRemove` drops the applied `pk:` history, and
  def gates re-evaluate per pick (WeakBy conflict lists).
  Result: fear 100→80 (live 55), weakened 72→56 (live 47) —
  residual is ambient TraitPickerList feeds + GrowInPower
  grant/remove balance, not the weakness rows.
- `mount` 26 vs 33: pipeline decoded — `Default_HasMount` →
  `PickerListRaw_HasMountExcludedTraits` (creation p=0.25 +
  checkpoint rows at lv9/19/29/39/49/59, p=0.5, two-chance),
  rarity `*EliteGang` defs → `PickerListRaw_HasMount_HasEpicMount`,
  and `Default_HasGang_*` defs roll `PickerListRaw_PetOrGang`
  where `Picker_HasMount` is a 1-of-37 side draw. The 91%-mounted
  live BeastMasters flow through the spec draw, not a mount
  picker — `AdvancedClass_BeastMaster` carries guaranteed
  `RewardPickerList` grant rows (`HasPets_Caragors`,
  `Summoner_Caragors`, `CaragorTamer`, `BeastialRoar`, …) plus
  `WeightMarker [6.0, Prob_BeastMaster]` + `WeightCharType`
  dynamic-weight rows (`FUN_1403505f4`: base + first-matching
  row weight per `Weight*` attr family, now implemented).
  Residual = BeastMaster spec rate (sim 8% vs live 13%) —
  role-granted `Prob` markers land at ~half the live rate.
- Row probability is **two-chance** (`FUN_1403513dc`:
  `roll <= p || f(roll) <= p`, effective `1-(1-p)²`) —
  implemented at all three roll sites; mount 22→26 alone.
- `gang` 41 vs 44, `epictrait` 37 vs 34, `rarity` 62/28/9 vs
  64/26/10 — at parity. `mannerism` 40 vs 16, `marker_other`/
  `statuseffect_marker`/`weapon_elem` still 10–20pt hot
  (ambient TraitPickerList feeds suspected).
- `humiliator` 0 vs 12%: `PickerList_Humiliator` exists with two
  members (weights decoded) but its `Picker_Humiliator` is
  gate-blocked in sim — live 62/534 carry `Picker_Humiliator`
  plus 25 bare `Humiliator`. Gating condition unresolved —
  likely the vendetta/humiliation lifecycle path, not spawn.
- `gang` 41 vs 44 — at parity after the dedup/fallback fix; the
  earlier "~2× hot" measurement predates per-level dedup.
- `humiliator`/`epictrait` share the shape "live grows with member
  age beyond what creation+level rolls give" — the member-aging
  promotion path is the unmodeled piece. (`unbreakable` is now
  solved — see the prestige-role entry above.)

**Live-call attempts (for the record):**
`Faction:CreateFactionMemberBuilder(DBFactionMemberBuilder.<rec>)`
and `Faction:CreateEmergencyPlayerFollower()` are callable via
game-VM eval but are job-source registrations — no new member
appeared in the roster (562→562). Fresh-member generation on
demand needs a real faction event in flight (assault aftermath,
recruitment), not menu calls.

## Spec-marker recount + `IncludedCharacterType` gates (verified)

Line-vs-record recount of the live save dump: **525/528 captains
hold exactly one `Marker_Specialization_*`** (99%); 6 records hold
two. Per-spec share: BeastMaster 13.7%, Tank 11.8%, Commander 11%,
Assassin 11%, Tracker 10.9%, Destroyer 10.7%, Berserker 10.1%,
CombatMaster 8.8%, Trickster 7.6%, Marksman 5.7%.

Reward defs gate on `Included/ExcludedCharacterType` attrs
(`Combat/Predicates/CharacterType` records → member
`Combat/CharacterType` names): `AdvancedClass_Marksman` requires
`Trait_Archers`. Implemented as `GateCheck::CharType*` via the
existing `member_char_types` map.

`Champion<Spec>` defs are "epic trait for your spec" — excluded by
`MarkerList_AC_CannotBecome<X>` = all spec markers except own +
`RoleMarker_SpecializationNever_X`. Their `WeightMarker` rows are
+3 own spec-or-prob, -3 every other Prob marker. Prob/Never
markers are transient generation inputs (never persisted), same
class as `OrcRarity_*_Chance`.

Remaining undecoded def gates: `Included/ExcludedRole`,
`Included/ExcludedTribe`, `Included/ExcludedScarType`,
`MustHaveAvailableRole`, `IncludedRowRank`,
`IncludedAppearanceTags` (partially handled), `GlobalVarPredicate`,
`SetRoleFromFilter`.

## `Included/ExcludedRole` + `Included/ExcludedTribe` def gates (verified)

Reward-def attrs `IncludedRole`/`ExcludedRole` hold refs to
`Faction/Predicates/RoleLists` records (expanded to member
`Role_*` names via `expand_refs`) or direct
`Faction/Roles/Definitions` refs; `Included/ExcludedTribe` resolve
`Predicates/TribeLists` or direct `Tribes/Definitions` names the
same way. Implemented as `GateCheck::{Role,Tribe}{Included,Excluded}`
evaluated against the member's actual epithet/tribe —
`EW_Fear_Burning` excludes `Roles_accustomed_to_fire` (14 fire
epithets: Blacksmith, FlameMonger, Infernal, …). 231 defs carry
non-empty role-list refs. Member role/tribe are now threaded
through `realize`/`reward_bundle`/`roll_list`/`reward_def`
(including the shame/return/event-pool lifecycle paths), the
`roles` scratch vector is seeded with the member's role, and
`WeightPred::{Role,Tribe}` receive real context (role compare is
`Role_`-prefix tolerant).

### `MustHaveAvailableRole` is the ONLY role-transform slot (verified)

`RoleLists` cells on a def serve two different purposes by slot:
`IncludedRole`/`ExcludedRole`/`WeightRole` are eligibility gates,
while `MustHaveAvailableRole` (+ `SetRoleFromFilter` flag) marks a
role transform (`PS_*`, `ResReward_TransformRole_*`,
`Reward_PinnicleScar_*`) whose `RoleCategory_*` first member is the
new epithet role. `reward_def` previously ran the transform path —
applying the member role's `Traits/Picker` refs — for EVERY
`RoleLists` cell. `Reward_IronWill` carries `ExcludedRole` →
`Special_Butterfly_Roles`, whose member roles (`Role_Fool`,
`Role_Giggles`, `Role_Gluttonous`, … — 29 defs grant
`Picker_AudioMannerism_*` directly) were granting their pickers as
a side effect of an eligibility test. Generation-only mannerism
dropped 32%→20% (live 17.6%), and the bogus feed was masking real
undercoverage in ironwill/statuseffect/weapon_elem. Now gated on
`slot_name(i) == "MustHaveAvailableRole"`.

### Measurement artifact + two-chance correction (verified)

The parity scorecards reported in recent rounds were computed on a
STALE `Helper/sim80f.jsonl` — re-running the census on a fresh
binary at the pre-change commit shows board-wide inflation that
predates the role-gate work. Root causes, now verified:

1. **The two-chance reading was wrong.** Re-decompiled
   `FUN_140352dc4` (the per-row matcher inside `FUN_1403513dc`):
   `fVar14 = DAT_14202e5a0` is the CONSTANT **1.0**, and
   `FUN_1401b3c6c` is a fresh uniform draw — LCG
   `state = state*0xbb38435 + 0x3619636b`, top 24 bits scaled by
   2^-24 (`DAT_14227f634` = 5.96e-8). So the row test is
   `p >= 1.0 || U(0,1) <= p` — a PLAIN single roll, no second
   chance. The `1-(1-p)²` model double-fired every row and has
   been reverted.
2. **`RoleMarker_*` are transient generation inputs** — live
   saves carry zero; they now strip at the persist point
   (`SpecializationProb_*`/`Never_*`/`IsPathetic`/`ExtraHates` feed
   gates/weights only).
3. **`OrcRarity_*` trait/marker upgrades are exclusive** —
   `Default_EpicOrc_*`/`LegendOrc_*` defs roll
   `Picker_OrcRarity_{Epic,Legendary}` mid-generation, promotions
   (`set_rarity`), vendetta epicify, and undead-raise Epic bumps
   all REPLACE the held rarity marker+trait. Implemented at the
   grant sites (`reward_def` marker/trait cells, `PickerItem`,
   picker-member grants, `apply_picker` outcome) plus highest-tier
   squashes at realize/level-up commits. Multi-rarity captains:
   0/200.
4. **Level-up was rolling the epic-trait bundles ungated** —
   `level_up_rewards` now applies realize's rarity `want` filter
   to `Default_EpicTraits_*` (non-PickRemovable).

5. **`rarity_of` counted chance markers as rarity** —
   `Marker_OrcRarity_Epic_Chance`/`Epic_Forced` are lottery flags
   (live holds 12 `Epic_Chance`+Common captains), but the old
   substring match admitted them to the `Default_EpicTraits_*`
   gate; each rolled def then granted real `Picker_OrcRarity_Epic`
   and promoted the holder. Fixed to exact-tier matching via
   `rarity_tier` (+`Picker_OrcRarity_*` history names). Tick-80
   epictrait fell 59→50.

Ticked-run diagnosis (n=200, 80 ticks, post-fix): the flat
`NemesisRules` d100 bands (death 3 / promotion 4 / shame 5 /
vendetta 7 per Alive roll) are placeholder heuristics and the
dominant remaining skew, not reward logic:

- **Mortality**: only 46/232 captains end Alive (~91% churn over
  80 ticks); the trait census counts dead captains' promoted
  markers. Alive-only rarity 16C/21E/9L on n=46 still skews high
  because promotions precede deaths.
- **Rank seats are enforced** — `rank_slots` caps 1 overlord +
  5 warchiefs per region key and holds; no over-seat promotions.
- **Region fragmentation** (FIXED): `region_of` returned raw
  codes for `Y*` builders (`IS`, `SM`) but CamelCase names for
  `Gauntlet_`/`SotP_` (`Island`, `SnowMtn`) — same board, two
  seat pools. `realize` now canonicalizes via `region_name`
  (extended to be idempotent: canonical names map to
  themselves). Board keys consolidated ~9 → 7 canonical +
  `""`/`R`/`Other` off-board spawns; epictrait 50→46,
  ironwill 36→33, mount 47→42 at t80.
- **Rarity-marker promotions are direct writes**, not picker
  draws: `Picker_OrcRarity_*` counts are frozen gen→tick
  (38E/22L) while `Marker_OrcRarity_*` grows (E 38→83, L 22→46)
  via rank promotion (`set_rarity`), undead raises, vendetta
  epicify.
- Real undead `Quirk_Undead` 40/232 ≈ 17% matches live ~17%.
- The retail tick is event-driven, not band-driven: db carries
  `Faction/Events/SimEvent` (431 recs), `SimEventPicker` (80),
  `SimEventChance` (34), `TimedEvents` (4: CompletelyEmpty,
  HalfEmpty, Resurrection, FillUpCompletely_LongDelay),
  `SimEventGenerationType` (68), `PromotionEventSetup` (4),
  `SimEventCooldown`, `Difficulty/SimEventLifeSpan` — the next
  parity axis is modeling that scheduler, not retuning bands.

### Board refill (IMPLEMENTED)

- `Faction/Rows` seat math: per base region **1 Overlord + 1
  Inner_Warchief(2) + 1 Outer_Warchief(4-5) + ~19 captain-row
  members** — `*_Row_Captain_{0,1,2}` count cells 4/8/7.
- `nemesis_tick` now refills canonical boards (hole ≥
  `refill_min_holes`=2 per retail `TimedEvents/HalfEmpty`
  `MinNumHoles`) back to `captain_seats`=19 via
  `spawn_in_region` with region-matched builders — the
  `Promote{Empty,Dead}_Builder_Transform` semantic. Requires
  `SimDb`; `""`/`R`/`Other` boards never refill.
- Alive-only tick-80: 46→397 survivors, rarity 51/29/20 vs the
  pre-refill 35/46/20 (live 64/24/10).
- `nemesis_parity.py` now filters `/Dead|/Shamed` records (live
  census is alive-only) and excludes `*_Chance`/`*_Forced`/
  `NotLegendary_*` flags from the rarity category (`rarity_flag`
  split out).

### Level-up checkpoint re-fire + level cap (FIXED)

- Checkpoint rows (`flag=1, lo==hi`) must lie in the NEWLY
  gained level range `(old_power, new_power]` on level-up —
  the native loop covers `trait_level..=menace_level`, each
  level fires once ever. The earlier `ck <= level` clause
  re-rolled every past checkpoint on every level-up
  (`Default_HasMount` p=1.0 @lv20..60 re-fired ~40x/captain).
  Fix: `creation || ck > spawn` (`spawn` = old power on
  level-up). Mount 58→47, gang 72→46 at t80.
- Menace level caps at 80 — live roster has ZERO captains at
  power >= 80 (394/562 in the 70-79 band); `gain_power` was
  uncapped (sim powers reached 120). Capped `.min(80)`,
  matching the player-award clamp. Mount 47→41, ironwill
  37→33, statuseffect 41→36.
- `Default_HasMount` xp rows decoded: flag=0 p=0.25 creation
  + flag=1 p=0.5 @lv10 + p=1.0 checkpoints @20/30/40/50/60 →
  `Rewards/Lists` draw [def w20, Nothing_Weight_100 w100]
  (1/6 per draw). Def gates: ExcludedRole
  `Roles_With_Unqiue_BodyAnimations`, ExcludedTraitMarkerList
  `MarkerList_NoMountMarkers` (Berserker/Olog/NeverMounted/
  HasHypeMan/Deranged), ExcludedTraitPicker x3,
  ExcludedCharacterType `Trait_Olog`, IncludedTraitMarkerList
  `MarkerList_AllButBerserkerMarkers`.
- Mount composition: sim HasMount 39%/HasPets 11%/DireMount
  7% vs live 22%/19%/2% — BeastMaster mount rate ~correct
  (BM ~100% sim vs 64% live); non-BM HasMount ~37% vs ~12-25%
  live — residual likely pool-side/event-side (live captains
  gain mounts via hunt SimEvents, `Reward_GainedMount` defs).

### Event-scheduled tick (IMPLEMENTED — first scheduler run)

- `NemesisRules.region_jobs` switches `nemesis_tick` to the
  decoded SimEvent model: per-region job spawns via
  `schedule_board_jobs` (PowerStruggle/Revenge/Prisoner/Vendettas
  contexts, `cast_mission_slots` casting), capped-tier vacancies
  via `board_promotions` (live seat counts; the earlier draft
  re-promoted from a stale roster snapshot — first run produced
  1990 warchiefs), ambient Hunting stays per-captain, Dead/
  Shamed returns keep their per-captain rolls (TimedEvents
  resurrection model).
- Two spawn-side bugs surfaced and fixed: (1) the captain-row
  refill admitted `*Warchief*`/`Gauntlet_*`/`DLC5_*_Bodyguard`
  builders whose `RowRanks` ref is a capped seat — they seated
  as warchiefs but never filled the captain hole, respawning
  every tick (pool now requires `RowRanks=Captain`); (2)
  `enter_hierarchy` counted the just-seated newcomer against its
  own slot check and never demoted a duel challenger — a full
  board could still seat it. Challenger now drops to Captain and
  duels for the seat.
- Tick-80 alive-only (n=316, `region_jobs=2`): epictrait 35 vs
  34, ironwill 26 vs 28, humiliator 12 vs 12 — at parity; mount
  40 vs 33, fear 72 vs 55, weakened 61 vs 47, gang 50 vs 44,
  statuseffect 30 vs 53 (under), rarity 70/36/8 vs 64/24/10 —
  Epic still elevated via promotion rarity upgrades.
- Not yet modeled (spec: faction-lifecycle.md): SimEventChance
  outcome tables on resolution, per-event cooldowns,
  SimEventLifeSpan durations, per-rank generation allow-lists
  (context choice is state-driven), Lithlad/Minas boards starve
  (their builder pools are capped-tier only — bodyguard/
  warchief builders; captain-row refill finds no pool).
