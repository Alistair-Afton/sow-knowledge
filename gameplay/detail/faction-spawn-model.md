# Spec: Faction member spawn / leveling model

> **Status:** **partial** — accrual window verified; spawn distribution open
> **Provenance:** live census level distributions (223-member lv20-43
> board, 562-member endgame board), `Faction/PlayerBasedXPLevel/*`
> records (166 `XPLevelInit` Delta recs, 12 EventAwards, 1
> InitialXPLevelClampSetup), sim `sim nemesis` scorecards.
>

## Member levels (VERIFIED)

- Captain level = player level + `PlayerBasedXPLevel/Delta` draw —
  Delta rows carry relative-level ranges (e.g. `-1_to_1_levels`,
  `10_to_20_levels`); `CLI level=N` sims the board baseline.
- Board mixes levels widely (live endgame: lv0-84, dominant cohort
  lv76+; earlier capture lv20-43).

## `spawn` level — the accrual window (INFERRED)

- XP (flag=1) rows fire only for levels the member **lived through**:
  `spawn < l <= level`. A member generated at level 40 never saw the
  lv10/20/30 checkpoints.
- Sim derives `spawn` from the **entity id hash** (splitmix-style mix,
  NOT the PRNG) so the deterministic draw stream is untouched by the
  model. Distribution: currently `uniform(1..=level)`.
- Prior model (min-of-2-uniforms triangular, mean ≈ level/3) was
  reverted — uniform matched live accumulation better at lv80 but
  UNDER-fires lv28 band rows under the GELO rule (see
  `faction-xp-rewards.md` OPEN). **UNVERIFIED:** the true spawn
  distribution — live suggests it may skew lower than uniform (more
  levels lived at lv28) or that low-level members accumulate via an
  extra path.

## Accumulation shape (VERIFIED)

- Live trait presence grows ~linearly with levels lived and saturates
  by pool composition — e.g. StatusEffect 27%@28 → 61%@80 fits
  constant q≈0.022/level; fear/hate grow weakly (r=0.39 perks,
  r=0.25 pickers) since creation-time pools dominate those.
- Live `+0x1e0` picker log is a 16-entry ring; applied vectors
  `+0x198`/`+0x1b0` are authoritative.

## Role/seat layer (PARTIAL)

- Seat assignment (Captain/Soldier/Warchief/Overlord/Bodyguard) gates
  several behaviors — `Follower_GrabAndThrow` strip is the proven
  example (see `faction-lifecycle.md`).
- **Role draw weights — VERIFIED (GameDB)**: `Faction/Roles/Definitions`
  cell 1 is an f32 selection weight (`Role_Coward_Ratbag`=440,
  `Role_Pretender`=150, `Role_Berserker`=80, baseline ~0.9). Sim draws
  weighted; uniform draws under-produce heavyweight roles ~20×.
- **Prestige pool — VERIFIED (GameDB + census)**: board fillers draw
  `Roles_who_are_unimportant` (369 roles, zero ub weight); the
  veteran/prestige re-draw pool is `RolesWhoShouldNotBeLowLevel`
  (99 roles; `Role_Pretender`=14.1% of weight → `Quirk_Unbreakable`).
  Retail trigger is seat-tier (only `*_Row_Captain_2` rows in
  MtnPass/MinasIthil carry it as a direct `RoleLists` cell) or a
  career "no longer low level" re-draw — exact trigger OPEN (needs
  the RoleSelect consumer or row-seat modeling); sim approximates
  with a level≥30 re-draw and lands at 15% vs live ~10–17%.
- **`Faction/Rows` = the board seat template — NEW, verified shape**:
  each row record carries region, `RowRanks` seat, a
  `SimEventGenerationSetup` ref (the member's purpose — the likely
  producer of the runtime SimEvent object at member+0x4f0), ambient
  `SpawnAmbient/Locations`, reward lists (`Default_BloodBrother`),
  optional seat `RoleLists` (exclusion filters
  `Roles_disallowed_*` for Warchief/Overlord seats; draw pools for
  `_2` captain seats), and `_StructuresFactionInitialXPLevel` entries
  — gated (`Act1`/`ShadowWarUnlocked`), player-relative level bands
  (`PlayerLevel_Plus5_To_Minus15`, `GruntKillsPlayer_CompareXP`) that
  are the OPEN spawn-level-distribution source.
- `bodyguards` category under-fires (sim ~13% vs live 26%@80):
  warchief seat share may be too low, or ExtraBodyguards has a
  promotion-time grant path.

### RoleSelect decode (VERIFIED — binary)

- **`FUN_141bbf928` = the member builder.** Its ctx (`param_3`)
  carries an explicit role override at `ctx[0x29]`; otherwise
  `FUN_141bd6a48` builds the role-pick context and
  `FUN_141bd7934`/`FUN_141bd7b00` drive the draw into `SetRole`.
- **Candidates** come from two ctx lists (primary `ctx+0x130/0x138`;
  secondary `ctx+0x188` behind a flag — the seat RoleList). On
  total failure `ctx+0x150` is cleared and both passes re-roll —
  a guaranteed fallback.
- **`FUN_141bd8144` = the per-candidate scorer**, gated on:
  ctx exclusion vector/list, a record-condition gate, a
  **global role census** (`{count, decrement}` per role), a
  min-level/prestige integer gate vs `ctx+0x180` (matches
  `RolesWhoShouldNotBeLowLevel`), four predicate gates, and two
  optional ctx-flagged gates.
- **Effective weight = role weight penalized by board census**:
  `w_eff = base − count×decrement`, clamped — roles already on
  the board lose weight (diminishing returns). Selection goes
  through `FUN_1401b3b90` on the `DAT_14202e8xx` RNG family;
  exact accept semantics (reservoir vs sequential) INFERRED.
- **`FUN_141bc63f8` = `SetRole`** — the only direct
  `member+0x318` writer outside save/load/ctor. Swaps
  `PreviousRole←Role`, grants the role's child records into
  `member+0x158` (trait/marker container — how roles carry
  markers), bumps the global census, applies a role-carried
  Personality record to `member+0x270`, and broadcasts a
  SimEvent. 7 call sites: the builder, a wrapper, three
  `SetRole` command handlers, and `FUN_140f859a0`.

## OPEN

- True `spawn` distribution — needs the native spawn path read or a
  live spawn-watch dataset (`tools/spawn_watch.py` exists for exactly
  this).
- Whether flag=1 band rows evaluate eligibility on *current* level
  (GELO) or *all* rows always (ALLBAND) — see xp-rewards spec.
