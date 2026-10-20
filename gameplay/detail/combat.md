# Combat data model — formulas, attacks, fight circle

> **Status:** **verified** (record layouts + evaluation rules),
> semantics of damage predicates PARTIAL
> **Provenance:** sow-rs/OnSight `formats/combat.md` + `combat.md`
> (record walks + a working evaluator against real `game.gamedb`).
> Andúril consumes the same GADB records via `anduril-gamedb`; the
> eval rules below are reproduced so an Andúril evaluator can be
> checked against them.

All data lives in `database/game/game.gamedb` (GADB; see
`../../formats/detail/gamedb.md`). Links are `category_index<<20 | record_index`.
Attribute **names are absent** from serialized GADB — attributes are
identified by `name_hash` of candidate names (see `nemesis-data.md`).

## Records

- `Combat/Stat/Formula/Float` and `.../Damage`: `InitialValueRange`,
  `ClampOutput`, `ClampRange`, `Nodes` (structs of `Predicate` /
  `VariablePredicate` / `DamagePredicate`, `Operation`,
  `ReturnImmediateIfPassPredicates`). Range fields are GADB kind 0x07
  (heap offset to two f32 min/max; both zero = unset).
- `Combat/Stat/Formula/FloatOperation`: `OperationType` (i32) plus
  `ADD|SUB|MUL|DIV` x `_Variables` / `_Formulas` / `_Constants`
  (constants link to `Combat/Stat/Constant/Float`, field `value`).
- Variables: `Combat/Stat/Variable/{Float,Int,Bool}` (`InitialValue`);
  per-character tables `Combat/Stat/Variable/FloatIndex`
  (`IndexVariable`, `IndexOffset`, `IndexedValues`, `VariableToSet`).
- Characters: `Combat/InventorySetup:<name>` — `VariableBools`,
  `VariableLists` (FloatIndex tables), `Items` whose
  `ItemBuilder.AffixPicker` base affix contributes FloatIndex values.
  Difficulty: `Difficulty/Settings:Difficulty_2Normal.SetPlayerVariables`.
- Attacks: `Combat/Attack` (16,976 records): `AnimationData.name` = the
  clip, `AttackData.DamageFormula`, hit-counter flags
  (`IncrementHitCounter`, `ResetHitCounterOnMiss`, `PauseHitCounter`),
  `AnimationData.Sync` -> `Combat/Sync` (`AttachSocket`,
  `InterpolationPos`/`Rot`).

## Evaluation rules

Inferred from the data by sow-rs; every formula checked agrees:

1. Start at the initial range's **min** (else 0). Run nodes in order;
   apply a node's operation only when all its predicates pass; stop
   after a passing node with `ReturnImmediateIfPassPredicates`; clamp
   to `ClampRange` when `ClampOutput`.
2. Within each ADD/SUB/MUL/DIV bucket the terms are **summed** (A, S,
   M, D; empty M or D = 1). `OperationType` 0: `(prev + A - S) * M / D`.
   Type 1: `prev * M / D + A - S`.
   Evidence: `Override_Return_10` (type 1, `+10 *0`) returns 10 only
   under the type-1 rule; `Gem_GreenRing_DefScalar` = 0.9; a
   multiplicative reading of `*(Float_DmgPCT_HealthFull, 1)` would zero
   every hit at full health.
3. Side 0 reads the evaluation's source character, side 1 its target;
   a nested formula read from side 1 evaluates with source and target
   **swapped**.
4. Variable predicates: bool = `RequiredValue`; int bounds are i32
   `MinValue`/`MaxValue`; float bounds are value options (`RawValue` /
   `variable` / `Formula`) gated by `EvalMin`/`EvalMax`;
   `AllMustPass`. **Damage predicates are not modeled** — sow-rs treats
   any that constrain anything as failing; the real semantics are OPEN.

## Vanilla numbers (level 1, Normal difficulty)

| Stat | Value | Chain |
|---|---|---|
| Talion HPMax | 298 | `BaseHealth_Player` 300 + default armour affix −2 |
| Infantry orc HP | 300 | `BaseHealth_Orc_Inft` |
| Talion strike | 30 | default sword affix; base melee 0 |
| Orc hit | 60.48 | `BaseDmg_Uruk_Inft_M` |
| Hit-streak damage scalar | 1 / 1.3 (14+) / 1.65 (29+) / 2 (44+) | |
| Orc post-hit/miss/counter delay | 2 s ≤ 9 hits → 0.25 s at 49 | |
| Orc double-attack chance | 0 below 5 hits, up to 0.15 | |

## Fight circle (infantry vs Talion)

`Combat/FightCircle/Participant:Infantry`: `RingSettings` pairs rings to
settings — `Rings:250to400` → `Infantry_In` (`Weight` 0.25,
`CanAttack`), `450to600` → `Infantry_Mid` (0.2), `850to1000` →
`Infantry_Out` (0.08, `UseCombatMovement`, `TargetWeightAdjustment`
−10), `1250to3000` → `Infantry_Outmost`. Rings carry no radius field —
**the radius is the name (cm)**. Settings link action pools
(`Bip_Idle`, `Bip_Taunt`, `Bip_Flair_LR`, `Bip_Avoid`…).

Participant fields: `AlwasyAvoidRadius` 75 (sic), `FarDistance` 50,
`HysteresisTime` 1.5, `NextAttackDelay` 1, `AttackerCancelDelay` 3 /
`…Distance` 300 / `…AlwaysDistance` 2000,
`IgnorePlayDelayIfSingleAttacker` true, `AttackCountMin/Max` 1,
`RingWeights` (`DistanceWeight` 1, `MoraleWeight` 2, `Bias` 0).

Plays: `PlayBook:VS_Player_Default` → `Play:PC_BaseMelee`
(`SelectionWeight` 0.1; `PlayDelayOnHit/OnCounter/OnMiss` = the
`BipVsPlayer_Post*Delay` variables; stage `PC_Melee_Single`: one
attacker, role `PlayRole:PC_BaseMelee`). Role picks the attacker:
`DistanceToTarget` `Vs_Player_BaseMelee` (0–1900 cm),
`DistanceWeight` 12, `CameraWeight` 10, `OffScreenWeight` 10,
`LastAttacker` −2 — how these combine is OPEN.

Distances (`Combat/Predicates/Distance`, field `DISTANCE`, cm): orc
basic attack `AI_Infantry_Melee_Chain_Atk` 0–150; Talion's pool strikes
are banded `PC_PoolAttacks_Distance_0` 0–75, `_1` 75–250, `_2` 250–450,
`_3` 450–1000; each strike clip's attack record names its band
(`PC_HC_Lowest_L1_D0_*` → `Distance_0`). D0 clips lunge 0.7–1.9 m, so
bands are most likely **gaps between body capsules**, not centre
distances (PARTIAL — no flag says so).

## Clip catalogue (name conventions + verified files)

Clip names: `Lt`/`Hy` light/heavy, `L1`/`L2` level, `D0..D3` distance
band, `F/B/L/R` direction to target, then hands/arc. Paths under
`characters/animationcharacters/`.

| Use | File | Clips |
|---|---|---|
| Player light strikes | `player/pl_combat/cmbt_pool_low/pl_d0_l1.anix` | `PL_Lt_L1_D0_F_RL_LR_RL_1` 1.50 s, `…_LR_RL_LR_7` 1.80 s, `…_LL_RR_RL_5` 2.00 s, `…_LL_RL_ST_3_FrontKick` 1.47 s (+`_B_/_L_/_R_` sets) |
| Player counter (sync pair) | `player/pl_sync_counter/pl_sync_counter.anix` | `PL_Bip_Sync_BasicCounter_F` 1.50 s, `_B` 1.53 s |
| Player hit reaction | `player/pl_combat/pl_multiattack.anix` | `PL_MultiAttack_Recoil_L2R/R2L/U2D/D2U` 0.70 s |
| Player death | `player/pl_recoils/pl_recoils.anix` | `Death_In` 2.83 s, `Death_Loop` 6.13 s |
| Orc combat locomotion | `orcs/orc_movement/orc_spr_behavior.anix` | `Spr_Combat_Idle` 5.73 s, `Spr_Combat_Walk`, `Spr_Combat_Run` 0.73 s |
| Orc attack | `orcs/orc_combat/infantry/orcinf_cmbt_attack.anix` | `Bip_Cmbt_Attack_Front_1/2/3` 1.83 s, `OrcInf_Cmbat_AttackHeavy_R` 2.97 s |
| Orc countered (sync) | `orcs/orc_sync_counter/orc_sync_counter.anix` | `Bip_PL_Sync_BasicCounter_F` 2.30 s, `_B` 2.03 s |
| Orc hit / knockdown | `global/bip_recoil/bip_recoils.anix` | `Orc_Recoil_F_St_100cm` 1.30 s, `Orc_Recoil_F_KD_STR` 1.93 s, `Orc_Recoil_KDLoop` 6.03 s, `Orc_Recoil_KDGetup` 1.50 s |

STRI timed events mark hit frames in many clips: Talion strikes deal
damage at `APPLYDAMAGE` (0.333 s), chain from `EHITWIN` (0.633 s);
counter pairs end at `STOP_ALIGN`. Use them for damage timing where
present, else a fixed fraction. See `../../formats/detail/anix.md` for the
STRI record layout.

## Open questions

- Damage-predicate semantics (attack/defender state, categories, items).
- How role weights combine to pick an attacker.
- Whether distance bands are capsule-gap or centre distance.
- The attack token mechanics beyond `AttackCountMin/Max`.
