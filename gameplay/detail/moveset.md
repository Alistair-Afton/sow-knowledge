# Talion moveset — skills, meters, ranged, and move chains

> **Status:** **verified**** record chains for most M3/M4 moves;
> several M2 chains PARTIAL
> **Provenance:** sow-rs/OnSight `moveset-map.md` (538-line record map:
> every chain followed in `game.gamedb` + clip index over all `.anix`).
> Condensed here; keep the OnSight file as the detailed lookup.
> Formula values are level-1 Talion (`Combat/InventorySetup:Player`,
> Normal) vs `Orc_Infantry`. `—` = not determined.

See `combat.md` for the formula evaluator and fight circle;
`traversal.md` for CADB/locomotion; `hud-map.md` for the
HUD widget map.

## Meters

### Might (`PowerMeter` / charge meter) — VERIFIED (gain order PARTIAL)

- Variables `ChargeValueFloat`→`Float_PowerChargeMeter`,
  `ChargeValueInt`→`Int_ChargeMeter` (initial 0).
- Granted by `PS_Execution_Parent_UNLOCK` → buff
  `SkillB_ChargeMeterGrant` → `ChargeMeterSetup:PowerChargeMeter_Default`
  (`ChargeCost` = `Check_PowerMeter_Cost` = **100**).
- Capacity = `Float_PowerChargeMeter_MaxValue` + 100 when any Might
  ability is unlocked, clamp [0,200]: **100 = one bar**; upgrade
  `PSS_PowerMeter_Parent` adds +100 → two bars. Levels:
  `Int_ChargeMeter` 100..199 = L1, 200..299 = L2, 300+ = L3.
- Starts 0; base regen 0 in/out of combat (child-3 "Risky" drains −200
  OOC; Elven Wrath active +40). Units presumed per second (PLACEHOLDER).
- Gain on hit (first-match category assumed): default **+8** (light,
  heavy, stun, shoulder bash, ground finisher, vault); counters 0 (+4
  with `PS_Impact_Child3_SKILL`); executions 0; flurry +4; held heavy
  +12; quick throw +4.
- Loss: miss **−10**; hit by orc **−20**; olog/stun-grenade −300.
- OPEN: first-match vs all-match on `OnHitAdjustments`; `duration` 3;
  float→int conversion.

### Elven Wrath (`CBMeter`) — VERIFIED records, PARTIAL activation

- Variable `Float_CBWraithMeter`, capacity `Check_CBMeterAll_Max` =
  **100**. Granted by `PS_CBMeter_DEFAULT_SKILL` →
  `WraithMeterSetup:CBMeter_Parent` (skill `PSS_CBMeter_Parent`; not in
  starting inventory).
- Gain: `ProcDef_CBMeter_Default_WrathGain_KillEnemy` →
  `Check_CBMeter_AnyKill` = **+2 per kill** (0 while active).
- Drain while `IsGameBreakerActive`: `Check_CBMeter_Parent` −10
  (`Check_CBMeter_Default` −15).
- OPEN: activation input and what rage mode changes; HUD art.

### Focus / slow-mo energy (`PC_EnergySetup`) — VERIFIED records

- Owned at start via `PSS_SloMo` → `S_SloMo`. Value
  `Float_Energy`; max = `Float_SlowMoFocus_MaxValue` + 100, clamp
  [0,200] → **100** bare, **103** with the starting ring.
- Passive drain while `IsFocusEnergyDrainNormal`: **−23**/s
  (PLACEHOLDER unit); otherwise **+40** regen after a **12 s** cooldown
  (`Passive_FocusRegenCooldown`). `MinEnergyToEnter/Remain` = 1.
- `SlowMo:WraithMode`: `SimulationTimeScale` [1,10],
  `PlayerTimeScale` [1,1], ramp up/down 0.5 s — read as world at 1/10
  speed, Talion full speed (PARTIAL, by `VehicleSloMoWeak` analogy).
  Exit on damage. Bow `SlowMoVelBias` 0.5.

### Arrows (Elf-shot)

- `Int_AmmoCount` (types 2..6 use `Int_AmmoCount1..6`). Max =
  `Elfshot_MaxFormula` = `Float_MaxAmmoElfshot` → **5** with Azkâr at
  level 1 (+4 with a DLC4 skill). Starting count OPEN (assume full).
- Cost **1 per shot** — draw/shoot require `Int_AmmoCount ≥ 1`; no
  record deducts (native, PARTIAL). Resource type 2.
- Refill: `Ammo_Small` pickup (auto-pickup 250 cm) +4; drain gives 0 at
  base. Empty crosshair message "Elf-shot Needed".

## Input bindings (VERIFIED)

`.bvr` `Predicate.Command.ActionId` / `Input.ActionId` values are heap
offsets to command names = `Profile/Commands` records; `PCKeyboard` /
`PCGamepad` links give defaults (all 382 commands dumped by sow-rs).

| Action | Command | Keyboard | Gamepad |
|---|---|---|---|
| Enter aim (hold, not toggle) | `CombatSecondary03` | RMB held 0.2 s | LT |
| Draw (while aiming) | `CombatKBM05` | LMB | RT |
| Fire (release) | `CombatKBM05` | LMB release | RT release |
| Focus slow-mo toggle | `CombatKBM13` | Space (in aim) | unbound (`CombatHidden06`) |
| Shadow strike (while aiming) | `CombatKBM06`/`CombatPrimary01` | X | X |
| Drain (hold on orc) | `CombatPrimary03` | E | B |
| Wraith Flash | `CombatKBM03` | R | — |
| Jump / climb up | `CombatPrimary04`/`LedgePullUp` | Space | A |
| Ledge dropdown | `LedgeDropdown` | E | B |
| Stealth toggle | `StealthToggle` | Left Shift | RT |

`InputType` (PARTIAL, from use): 0 press, 1 release, 2 press-and-hold
start, 3 held. **Sprint is not a command** — see traversal spec.

## Move chains

### Combat core (M2) — mostly PARTIAL

- **Evade/roll** — clips `PL_Roll_2_Combat_F`, `PL_Roll_2_IdleStand_F`
  (`pl_combat/pl_cmbt_block.anix`); no skill→attack chain proven.
- **Vault over** — skill `PSS_Dash_Over_SC`; attacks
  `PC_Bip_Sync_JumpOver_F/B`, syncs `PC_Bip_DashOver_F/B`
  (`AttachSocket=Attach`, interp 0.2/0.1); clip `PL_Bip_Sync_JumpOver`
  (`pl_sync_jumpover.anix`); damage `DmgForm_ZERO`.
- **Stun** — clips `PL_St_L1_*` (`cmbt_pool_stun/pl_stunattackpool.anix`;
  STRI events `CMSADD/CMSREM QuickStun`); record chain open.
- **Shoulder bash** — `PC_DashAttack_ShouldRam`; formula
  `DmgForm_Plyr_A_ShoulderBash` = 7.50 at L1; clip chain open.
- **Multi-counter** — `Combat/PlayerCompanion/Group:Counter_Double`;
  candidate attacks `PC_Bip_Sync_Cntr_TimedKD_F/B`, clips candidate
  `Elf_OrcInf_Sync_Cntr_KD_F`; identity unproved.
- **Ground finisher** — attacks `PC_Bip_GroundTakedown_F/B/L/R`, syncs
  `PC_Bip_KD_GroundTakedown_*` (socket `Attach`, 0.2/0.1); clips
  `PL_Bip_Sync_GTD_Gr_*` (`pl_sync_gtd*.anix`); damage
  `DmgForm_All_Instakill` = 3000; predicates include `GroundTakedown`,
  `Not_OnTheGround`, 150-units vertical.
- **Critical strike** — skills `PSS_CriticalStrike_*`,
  `PSS_HC_Crit_Threshold`; variable mods + `Check_BonusDamage_Chance` /
  `Check_TimingCrits_Chance`; attack/clip chain open.

### Wraith / meter moves (M3) — VERIFIED chains

- **Execution** — `CanExecuteLvl1Not2_OR_Vndta_HealthForMight` tests
  `Float_CurrentHealth_PCT` ≥ threshold; threshold −1 when
  `IsPowerMeterLevel1` → **one full bar executes**, not a hit streak.
  `PC_Bip_Standing_InstaKill_F` spends the meter
  (`PC_Execution_SpendMight` → `Remove_Charge_OnExecute`).
- **Wraith flash** — node `PC_WraithAoE(_KBM)` (R) → attack
  `WraithAoE_Lvl1`, clip `PL_WraithAOE` (`pl_wraithcast.anix`); costs
  the whole meter (`Spend_PowerMeter_OnExecute` −1000). AOE tick:
  radius 650, arc 180, growth 1 s, `DmgForm_Plyr_A_WraithAOE_Lvl1` =
  **78**; `HC_Low` applies `Stunned` 2.5–3 s.
- **Wraith flurry** — `PSS_WraithFlurry_Parent`; pool
  `PC_Flurry_Standing_INTO` → `PC_Flurry_INTO`/`_1..6`/enders; clips
  `pl_cmbt_flurry.anix`; needs target `Quick_Stunned`; damage INTO 3,
  ender 150; +4 Might per hit.
- **Drain (standing)** — E held on an orc: `PC_HoG_Standing_Start` →
  hold loop `PC_HoG_Standing_Hold_Normal` (`HOG_Duration` = 3 s) →
  `PC_HoG_Standing_Complete` (`PL_Wraith_HandOfGod_Complete`,
  `DmgForm_Plyr_A_HoG_Base` = **200**, heals 0.5×HPMax = 149, 0 arrows /
  0 Focus at base). Grab variant `PC_Grab_HOG_*` shares categories.
  Combat-brand `PC_Bip_InstaHOG_F` spends Might.
- **Brutalize** — `PC_Bip_SneakInstaKill_Loud_F` etc., category
  `PC_AutoCat_StandingBrutalize`, clips `pl_sync_chordkill_brutal.anix`;
  needs unaware target + dagger + `CSP_Terrorize`;
  `DmgForm_Plyr_A_LoudKill_DmgNone`.
- **Stun (wraith)** — `PSS_Wraith_Stun_1/2`, `PSS_Wraith_Punch_1` have
  no unlock item (Mordor leftovers); the live stun is the M2 one.

### Ranged (M4) — VERIFIED chains

- **Bow "Azkâr"** — `Primary_3Bow_1Default` → `PC_Bow_Default` /
  `PC_Bow_Default_Light` → model `player_rangerbowt1.skel` with wraith
  materials; affixes: `Float_DmgMod_R_FLAT` = **150**, `Float_MaxAmmoElfshot`
  = **5**. Bow exists **only while aiming** (`ActivateWeaponSlot`,
  `WeaponSlot` 2; only `LeftHand` active node — not carried on the
  back). Sockets: `Arrow_Draw_Socket`, `Arrow_Spawn`, `Rune01..06`.
- **Dark Talion** — `Player_DarkTalion` uses `PrimHam_3Bow_1DarkRanger`
  "Helm's Hammer" → shotgun mode `PC_Ranged_Shotgun`, not the bow.
- **Aim** — `PC_WraithRanged_Default` mode: `ExitOnTakeDamage`,
  `AllowedModes` `PC_SlowMo`/`PC_WraithPing`/`PC_Ranged_Zoom_Default`.
  Free aim: `MaxTargetDistance` 11700 cm, projectiles from camera,
  auto-target 0.5 s within 0.15 of screen centre ≤ 117 m. Camera
  `Ranged_Ground`, 0.25 s sine-out transition.
- **Shot** — draw → hold → fire node chain
  (`PC_Ranged_Ground_Normal_*`); draw power `RangedDrawPct` 0..1 →
  `Ranged_ChargeShot` 0.5..1 linear; ~1 s to full draw (PARTIAL).
- **Projectile** — `Velocity` 30000 cm/s, `GravityWeight` 75,
  `MaxRange` 11700 cm, `ImmediateImpact`, homing `MaxTurnSpeed` 200,
  `RequireLOS`, fire node `ArrowSocket`, wraith-arrow model + FX.
- **Damage** — `DmgForm_Plyr_Range_Light` = `(150 + mods) ×
  (PCT + ChargeShot) × difficulty × defence` → **75 no-draw / 150 full**
  vs grunt (300 HP). Head and body share the formula — **no headshot
  multiplier**: the `Orc_Infantry_29` stat entry isn't referenced by any
  Defence record; 2 full-draw arrows kill a grunt (VERIFIED against
  the data, counterintuitive — recheck before shipping).
- **Reactions** — body/head/shield/leg `AttackOverrides` pick
  `AI_Bip_Recoil_Ranged_*` / `AI_Bip_Ranged_Death_*` clips.
- **Shadow strike** — `PSS_ShadowStrike_Kill_Parent` (not owned);
  X while aiming; `PC_WraithDash_Kill_Start` →
  `PC_Bip_WraithDash_Kill_Standing_F/B` sync pairs; needs ammo ≥ **2**,
  `ShadowStep_Spend` −2 arrows; `DmgForm_All_Instakill` = 3000.
- **Charge shot** — skill not owned at start; base draw interp already
  scales damage; what the skill adds is OPEN.
- **Quick throw** — candidate `PC_Dagger_QuickThrow_base` /
  `Elf_QuickThrow_1` doesn't match the `pl_quickthrow.anix` clips —
  mapping OPEN.

## HUD anchors

- Might bar: `Interface/HUD/Widget/CombatBars:ComboPowerBar` → symbol
  `SPH-PowerBar`, levels 1–3 `powerbar{1,2,3}.mat`,
  `k_fResourceScale`, `FullScaleValue` 0.5/0.5/1.
- Focus gauge = crosshair `Energy` arc (`LIB-HydraCrosshair`,
  `MM-Combat`); `TokenCount` = arrow count; `Reticle` labels
  `None`/`Drawing`.
