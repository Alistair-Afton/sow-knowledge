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
  lv76+; earlier capture lv20-43). The endgame histogram is
  **bimodal at the player level**: 144/562 sit exactly at 79 and
  123/562 at 84 — the census player was level 79, so the mass is
  `player_level + delta` (deltas top out at +5, matching the 84
  spike; `XPLevelInit` draws clamp at 85).
- The menace cap is ~85, not 80 — INFERRED: the roster reaches 84
  and the delta records clamp at 85.
- **Per-seat `InitialLevels` — VERIFIED (real `Faction/Rows`)**:
  each row record refs ordered `_StructuresFactionInitialXPLevel`
  struct entries; first match wins:
  1. `GlobalVarPredicate`-gated (`ShadowWarUnlocked` — the late-game
     flag) → resolves a `PlayerBasedXPLevel` delta record instead
     of a fixed band (`snowMtn_Row_Overlord` → `PlayerLevel_Plus5`,
     `desert_Row_Captain_0` → `PlayerLevel_Minus5ToPlus5`);
  2. `MustHaveKilledPlayer`-gated entries (post-kill respawns);
  3. `Act1`-gated early-game band (e.g. 4–8);
  4. ungated fixed `LevelMin..LevelMax` fallback (verified:
     `snowMtn_Row_Overlord` 34..34, `desert_Row_Captain_0` 34..41).
  All entries carry `DifficultyVarLevelOffset` =
  `Difficulty_Captain_BonusLevels` (DefaultValue 0 at stock
  difficulty).

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
  RESOLVED (2026-10-22): the pool is a **seat-tier draw pool** —
  `*_Row_Captain_2` rows carry it as their `RoleLists` cell, which
  the pick ctx consumes as its secondary (`ctx+0x188`) candidate
  list. No career re-draw exists; members seated on `_2` rows draw
  prestige roles at spawn/fill. Note `Role_Pretender` (the pool's
  headline weight) is a **unique story boss**, not a filler epithet:
  its record carries a full `Tower_*` armor set,
  `ForcedPersonality=Orc_Professional1`,
  `Picker_GangOfDoppelgangers` and `ForceCharacterAudioUpdate` —
  retail's condition gates hold it outside its story window, and
  live boards show 0 copies.
- **`Faction/Rows` = the board seat template — verified**: each
  region carries `{Captain_0, Captain_1, Captain_2, Outer_Warchief,
  Inner_Warchief, Overlord}` rows; `ColumnMax` caps the seats per
  row (~4/8/7/4/2/1). A live board census (n=562) matches ColumnMax
  almost exactly per rank AND shows **63.3% of members hold no row
  seat** — most nemesis members are unseated roamers. Seat shares
  of the total board: Captain_0/1 17.1%, Captain_2 10.0%, WC_Outer
  5.9%, WC_Inner 2.3%, Overlord 1.4%. Builder `RowRanks` refs are
  tier CONSTRAINTS (36% of builders are Outer-tier but only 5.9%
  of the board sits Outer), not seat assignments — seats fill from
  matching-tier candidates. Rows also carry region, `RowRanks`
  seat, a `SimEventGenerationSetup` ref (the member's purpose —
  the likely producer of the runtime SimEvent object at
  member+0x4f0), ambient `SpawnAmbient/Locations`, reward lists
  (`Default_BloodBrother`),
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
  through `FUN_1401b3b90` — VERIFIED a weighted reservoir
  sampler: accept candidate with `w/(sum+w)`, last accepted wins
  (distributionally = a weighted roulette).
- **`FUN_141bc63f8` = `SetRole`** — the only direct
  `member+0x318` writer outside save/load/ctor. Swaps
  `PreviousRole←Role`, grants the role's child records into
  `member+0x158` (trait/marker container — how roles carry
  markers), bumps the global census, applies a role-carried
  Personality record to `member+0x270`, and broadcasts a
  SimEvent. 7 call sites: the builder, a wrapper, three
  `SetRole` command handlers, and `FUN_140f859a0` — the
  **fill-unroled** path (early-returns when `+0x318 != 0`).
  No caller re-draws a roled member's role: the prestige pool
  is a seat-tier pool (`*_Row_Captain_2` seat RoleLists feed the
  ctx `+0x188` candidates) — prestige members are drawn prestige
  at spawn/fill, not re-drawn.

## OPEN

- True `spawn` distribution — needs the native spawn path read or a
  live spawn-watch dataset (`tools/spawn_watch.py` exists for exactly
  this).
- Whether flag=1 band rows evaluate eligibility on *current* level
  (GELO) or *all* rows always (ALLBAND) — see xp-rewards spec.
