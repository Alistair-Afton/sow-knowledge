# Combat system

> **Status:** **verified****
> **Provenance:** combat records/formulas/clips — sow-rs evaluator + record walks.

How stats, formulas, attacks, meters, and the fight circle actually
work — all traced from `game.gamedb` records (VERIFIED by sow-rs;
chains followed and formulas evaluated at level 1, Normal
difficulty). Clip names verified against the full `.anix` index.

## Stat formulas — VERIFIED evaluation rules

`Combat/Stat/Formula/{Float,Damage}` records: `InitialValueRange`,
`ClampOutput`, `ClampRange`, `Nodes` (`Predicate`/
`VariablePredicate`/`DamagePredicate`, `Operation`,
`ReturnImmediateIfPassPredicates`).

1. Start at `InitialValueRange.min` (else 0).
2. Run nodes in order; apply a node's `FloatOperation` only when its
   predicates all pass; a passing `ReturnImmediateIfPassPredicates`
   node ends evaluation.
3. `FloatOperation`: `OperationType` (i32) + summed term groups —
   type 0: `(prev + A − S) × M / D`; type 1: `prev × M / D + A − S`
   (A,S,M,D = sums of `_Variables`/`_Formulas`/`_Constants`; empty
   MUL/DIV = 1).
4. Side 0 = source, side 1 = target; formulas reached from side 1
   evaluate swapped.
5. `ClampOutput` → `ClampRange` at the end.

Variable predicates: bool `RequiredValue`; int `Min/MaxValue`; float
bounds as value options with `EvalMin`/`EvalMax`/`AllMustPass`.
Damage predicates (attack/defender state, categories, items) are not
modelled by sow-rs's evaluator — a constraining one fails (their
documented assumption).

Vanilla numbers (VERIFIED, lvl 1): Talion HP 298, orc infantry 300;
Talion strike 30, orc hit 60.48; hit-streak scalar 1 / 1.3 / 1.65 /
2 at streaks 0/14/29/44; orc post-action delay 2 s → 0.25 s over
streaks 0–49.

## Attacks and moves — VERIFIED record chains (selected)

`Combat/Attack` records carry `AnimationData.name` (the clip),
`AttackData.DamageFormula`, hit-counter flags, `AnimationData.Sync`
→ `Combat/Sync` (`AttachSocket`, `InterpolationPos/Rot` — sync pairs
align via socket-node interpolation).

| Move | Chain highlights |
|---|---|
| Light strikes | `PC_HC_Lowest_*` / pool by distance band `PC_PoolAttacks_Distance_0..3` (0–75/75–250/250–450/450–1000 cm — likely body-gap, PARTIAL); clips `PL_Lt_L1_D0_*` in `pl_d0_l1.anix`; damage at STRI `APPLYDAMAGE` (0.333 s), chain window `EHITWIN` (0.633 s), end `AEE` (0.938 s) |
| Execution | needs predicate `CanExecuteLvl1Not2_OR_Vndta_HealthForMight` = one **full Might bar** (not a streak): `Int_ChargeMeter` 100–199 ⇒ threshold −1. `PC_Bip_Standing_InstaKill_F` spends `PC_Execution_SpendMight`, `DmgForm_All_Instakill` = 3000 |
| Ground finisher | `PC_Bip_GroundTakedown_{F,B,L,R}` + `PC_Bip_KD_GroundTakedown_*` syncs (socket `Attach`, 0.2/0.1 s interp); `DmgForm_All_Instakill` |
| Counter | `PC_Melee_Counter_Generic`; sync pair `pl_sync_counter` + `orc_sync_counter`; orc counterable until `EPW` (~0.75 s), lands at `STOP_ALIGN` (0.30 s); **0 damage** |
| Multi-counter | Celebrimbor companion model plays `CB_Bip_Sync_Cntr_Stun_*` on a second winding attacker; `PC_STN_Counter_HC_*` stuns by hit counter (1.5–3 s); `AutoTriggerHitCounterValue` 25 OPEN |
| Stun L1/L2 | `PL_St_L1_*` → `STN_100PCT_ShortDur` (2–5 s); L2 needs `PS_Wraith_Punch_2` → `QSTN_100PCT_StunPunch` (3–4 s + freeze ease) |
| Shoulder bash | `PC_DashAttack_ShouldRam`; KD length `PC_Dash_Attack_ShoulderRam_KD_HC_*` bands 1.5–3.5 s; `DmgForm_Plyr_A_ShoulderBash` 7.5 |
| Vault | `PC_Bip_Sync_JumpOver_{F,B}` + `PC_Bip_DashOver_*` syncs; `DmgForm_ZERO`; i-frames during clip |
| Evade | `PC_Dash` → `PL_Roll_2_Combat_F`; ignores queued hits |
| Wraith Flash | `PC_WraithAoE` (input R / `WraithAOE_KBM`) → `WraithAoE_Lvl1`: radius 650 cm, `DmgForm_Plyr_A_WraithAOE_Lvl1` = 78, low-band `Stunned` 2.5–3 s; costs one full Might bar |
| Wraith flurry | `PC_Flurry_*` chain (`pl_cmbt_flurry.anix`); needs `Quick_Stunned`; INTO 3 dmg, ender 150 |
| Drain | hold E: `PC_HoG_Standing_*` — 3 s loop (`Formula_HoG_Duration`), complete = `DmgForm_Plyr_A_HoG_Base` 200 + heal 50% HPMax |
| Brutalize | `PC_Bip_SneakInstaKill_Loud_F` — unaware target + `CSP_Terrorize`; `DmgForm_Plyr_A_LoudKill_DmgNone` |
| Shadow strike | `PSS_ShadowStrike_Kill_Parent` (not owned); X while aiming; `PC_Bip_WraithDash_Kill_Standing_{F,B}` sync pair; **costs 2 arrows**; `DmgForm_All_Instakill` |

## Ranged (bow) — VERIFIED chain

- Bow "Azkâr": `Primary_3Bow_1Default` → `PC_Bow_Default*` →
  wraith-model `player_rangerbowt1.skel` on `LeftHand` — **exists
  only while aiming** (no back socket; `ActivateWeaponSlot` on the
  ranged mode). Dark Talion instead gets "Helm's Hammer" shotgun
  mode.
- Draw (hold LMB/RT) → fire on release; `RangedDrawPct` maps draw to
  `Ranged_ChargeShot` 0.5–1; ~1 s full draw (PARTIAL).
- Projectile: velocity 300 m/s, gravity weight 75, range 117 m,
  1 s life, homing turn 200°/s, `FireNode` `ArrowSocket`.
- Damage `DmgForm_Plyr_Range_Light(Head)` = `(150 flat) ×
  (charge + PCT mods) × difficulty × ranged defence` → **75/150**
  vs lvl-1 grunt (HP 300). Headshots have **no damage multiplier**
  in the data (`Orc_Infantry_29` x2 stat is unreferenced) — PARTIAL:
  `AlwaysApplyBonusDamageOnHeadshotOrStealth` is native, unknown.
- Reactions by hit node (`AttackOverrides`): head/shield/body
  overrides; recoil/death clip families mapped.
- Arrows (`Int_AmmoCount`, resource type 2): max 5 base (bow affix
  `Float_MaxAmmoElfshot`); 1/shot; `Ammo_Small` pickups +4; drains
  give 0 at base.
- **Focus (slow-mo):** owned at start; `Float_Energy` max 100 (103
  w/ starting ring); drain −23/s aiming, +40 regen after 12 s
  cooldown; `SlowMo:WraithMode` world 1/10 speed, player full
  (PARTIAL reading), ramp 0.5 s, exits on damage.

## Meters — VERIFIED records, PARTIAL activation/gain-order

- **Might (`PowerMeter`/`Int_ChargeMeter`):** granted by the
  Execution skill chain; capacity 100 → 200 with `PSS_PowerMeter`
  (Brutal Strike); levels at 100/200/300. Gains: +8 default hit
  (light/heavy/stun/bash/GTD/vault), +4 flurry, +12 held heavy, 0
  counters (without `PS_Impact_Child3`), 0 executions; losses: −10
  miss, −20 hit taken (−300 olog/stun-grenade); upgrades modify.
  `OnHitAdjustments` first-match order assumed (HYPOTHESIS).
- **Elven Wrath (`CBMeter` — the rage meter, not "wraith"):**
  Domination-tree skill; max 100; +2/kill; −10/s while active.
  Activation input and rage-mode effects: OPEN.
- **Hit counter:** 3 s window (`BuffCounters:PC_HitCounter`),
  paused/reset per attack flags and on being hit (−9999 `OnRecoilAdd`).

## Fight circle — VERIFIED records, PARTIAL policy

`Combat/FightCircle/Participant:Infantry`: ring names encode radius
cm — `Rings:250to400` `Infantry_In` (weight 0.25, CanAttack),
`450to600` Mid, `850to1000` Out, `1250to3000` Outmost;
`AlwasyAvoidRadius` 75, `HysteresisTime` 1.5, `NextAttackDelay` 1,
cancel 3 s / 3 m / 20 m. `PlayRole:PC_BaseMelee` picks the attacker:
`DistanceToTarget` 0–1900 cm, weights `DistanceWeight` 12,
`CameraWeight` 10, `OffScreenWeight` 10, `LastAttacker` −2 (combine
rule OPEN). Orc attack bands: basic chain 0–150 cm. One attack
token at a time is the observed/implemented model; ring fill policy
(In 3, Mid 4, then out) is sow-rs's PLACEHOLDER.

## Perception — VERIFIED records

`AI/Perception/Types:Biped_Vs_Player`: notice within 20 m inside a
70° cone, or 5 m omnidirectional; alert tracks to 25 m. Awareness
state machine beyond "has seen" is OPEN.

## Capsules and movement — VERIFIED

`Character/Models` `CapsuleRadius`: Talion 30, orc 50 (cm);
`PlayerMovement`: gravity **−2000 cm/s²**, capsule 125×35, step 40,
fall bands 5–12 m with `FallDamageMin/Max` = **0** (no fall damage).
