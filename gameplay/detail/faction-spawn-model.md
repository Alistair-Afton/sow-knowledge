# Spec: Faction member spawn / leveling model

> **Status:** partial — accrual window verified; spawn distribution open
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

## OPEN

- True `spawn` distribution — needs the native spawn path read or a
  live spawn-watch dataset (`tools/spawn_watch.py` exists for exactly
  this).
- Whether flag=1 band rows evaluate eligibility on *current* level
  (GELO) or *all* rows always (ALLBAND) — see xp-rewards spec.
