# Traversal — movement constants, CADB context actions, locomotion

> **Status:** **verified** — constants + CADB layout; state-graph
> semantics PARTIAL
> **Provenance:** sow-rs/OnSight `moveset-map.md` §M6 (record chains +
> `sow_tool probe` validation of edge placement against our own scene
> geometry). Andúril-side context: `../formats/detail/objres.md` (CADB
> section location), `../formats/detail/bvr.md` (blend graphs).

All units cm unless noted. Attribute names not in the dictionary are
quoted by hash.

## Movement constants — VERIFIED

`Character/Models:PlayerSkin_DarkTalion_Main.PlayerMovement` →
`Client/PlayerMovement:PlayerMovement` (shared by every player skin):

| Field | Value | Notes |
|---|---|---|
| `PhysicsCharacterGravity` | −2000 cm/s² | gravity is **20 m/s²**, not 9.81 |
| `StandCapsuleHeight`/`Radius`/`StepHeight` | 125 / 35 / **40** | 1.25 m capsule, 0.40 m step-up |
| `FallMinHeight` | 100 | drops under 1 m aren't falls |
| `FallDamageMin/MaxHeight` | 500 / 1200 | fall-damage band 5–12 m |
| `FallDamageMin/Max` | **0 / 0** | **no fall damage** in the data |
| `CamLandMinHeight`/`CamLandMoveDist` | 100 / −3 | landing dip: 150 ms down, 250 ms up, 5° roll/pitch |
| `PlayerYawInterpolateTime` | 400 ms | facing blend |
| `BaseSpeedMultiplier` | 1 | speeds come from clips |

Model record adds `CapsuleRadius` 30, `CapsuleMinimum` −60 /
`CapsuleMaximum` 50 around `Pelvis`, `CapsuleMaxClimbAngle` 90,
`DeadlyEdgeForwardCheckDistance` 200, `DeadlyEdgeLedgeCheckDistance`
500, `LockPhysicsContextActionRecords` `Block_Movement`, `Edge_Hang`
(radius 35, height 75, distance 22). `LocomotionIK:Player`: foot trace
20, `GroundOffset` −104 (pelvis 1.04 m above feet — the
(0,−104,0) offset originates every `.bvr` context-action and jump
query).

## Input — VERIFIED

`Predicate.Command.ActionId` (kind 0x05) is a heap offset to the
command name → `Profile/Commands` record. Traversal bindings:

| `.bvr` predicate | Command | Keyboard | Gamepad |
|---|---|---|---|
| `Jump_Button` | `CombatPrimary04` | Space | A |
| `Climb Up` | `LedgePullUp` | Space | A |
| `Dash` | `TriggerLeft` | — | LT |
| — | `LedgeDropdown` | E | B |
| — | `StealthToggle` | Left Shift | RT |

**Sprint is not a command**: `Run.To_Sprint` (blend 0.3 s) needs
`MoveThreshold` (`RawMoveMag` ≥ 0.25) **and `Jump_Button` held**, no
mud/wall/sprint-blocked. So WASD = jog, **Space held = sprint /
free-run**, Space tap at an edge = vault/mantle, E = drop.

## Locomotion clips and speeds — VERIFIED (clips), PARTIAL (weights)

`BlendState:Run` → `RunToWalk_Master_PA` blends `SuperSprint_PA`
(`PL_Sprint_Super`/`Sprint_Hills`) / `PL_Jog` / `PL_WalkFast`;
`BlendState:Sprint` → `SprintBoostModerator` → `SprintMaster_PA`.
The dampener parameters are native — which point plays when is PARTIAL.

| Clip (`player/pl_movement/…`) | Length | Root travel | Speed |
|---|---|---|---|
| `PL_Walk` | 2466 ms | 316 cm | 1.28 m/s |
| `PL_WalkFast` | 2466 ms | 444 cm | 1.80 m/s |
| `PL_Walk_Standard`/`_Combat` | 2233/2066 ms | 509/523 cm | 2.28/2.53 m/s |
| `PL_Jog` (= `Jog_Weapon`) | 1466 ms | 677 cm | **4.62 m/s** |
| `PL_Sprint85`/`_Combat_Sprint85`/`_Hustle85` | 1000 ms | 640/640/637 cm | **6.40 m/s** |
| `PL_Sprint_Super` (skill) | 1000 ms | 1166 cm | 11.66 m/s |
| `PL_Sprint_Boost_City` | 900 ms | 1166 cm | 12.95 m/s |
| `PL_JogStop_R`/`_Combat_R` | 1200 ms | 30 cm | stop |

Turn rate: `Parameterization.TurnRate` records scale a native rate by
`TurnRateScalar` — no deg/s value in the data (OPEN).

## CADB context-action markup — VERIFIED layout, PARTIAL placement

Traversal runs on **authored edge markup**, not collision-derived
edges. `Character/Models/ContextActions` (124 records) are edge tags:
`Edge_Hang`, `Edge_Pullup`, `Edge_NoHang`, `Edge_Hang_Dangle`,
`VaultOver`, `Hurdle`, `Scramble`, `Block_Movement`, `Rail_Running`,
`Ladder_Rung`, `ZipLine`, `Mantle 3M`, …

Stored in `.objres` section 10 `CADB` (`worlds/retail/<r>/<r>.objres`):

| Part | Layout | Minas Ithil |
|---|---|---|
| Header | `"CADB"`, u32 3, counts: actions, edges, planes, volumes, groups | 75, 181,578, 829, 85, 569 |
| Actions | `count × u32` = `name_hash` of a `ContextActions` record | 74/75 resolve |
| Edges | 52 B: `vec3 a, vec3 b, vec3 n` + `u32[4]` bitmask over actions | every `n` unit length |
| Planes | 16 B: `vec3 normal, f32 d` | 829 |
| Volumes | 32 B: `vec3 min, max, u32 plane_start, plane_count` | 85 |
| Groups | 52 B: `u32 key, vec3 min, max, u32 edge_start, edge_count, vol_start, vol_count, u32, u32` | 569, key-sorted, contiguous ranges summing to edge_count |

Section 9 `CAWR` = `{"CAWR", u32 1, u32 569}` then 569×
`{u32 key, u32 0, u32 key}` — the group-key mirror.

Tag counts (MI): `Edge_Hang` 71,444, `Edge_NoHang` 42,533,
`Edge_Pullup` 29,468, `Block_Movement_RailRun` 29,141,
`Block_Movement` 19,614, `Hurdle` 18,098, `VaultOver_Mount` 17,507,
`Edge_Hang_Dangle` 12,347, `VaultOver` 1,136. Edge lengths: median
0.6 m, p90 2.75 m.

Placement: 563 groups are prefab-local (centred at origin, 1–11 m);
the region group is key `0x6B9D05D0` (122,245 edges, x −498..514,
y −112..459, z −670..650 m); `0xD9D58386..8C` are per-stamp groups
matching `minasithil.inst` object ids in OBJR section 12. **CADB uses
the game's frame — negate X for Andúril's.** Verified by probing the
mirrored edge midpoint moved 0.3 m along `n`: sampled `Edge_Pullup`
edges land on scene surfaces within 2 cm on the `+n` side only →
**`n` points onto the walkable top**. 5/9 sampled edges have no
surface at their height (upper building parts).

OPEN: how the 563 local groups map to placed prefabs — the key is not
the name hash (nor CRC32/FNV-1/1a/djb2) of any of 424,559 VFS paths,
not a PRFB `self_hash`, not an OBJR section-12 id.

## Player blend database (`player_elf.bvr`) — VERIFIED records

270 categories; 2,047 `BlendState`, 15,704 `StateTransition`, 2,301
`Animation.SimpleAnimation`. Traversal query records:
`Predicate.FoundContextAction` (3,425) →
`LogicNodeData.ContextActionData` (3,493); `Modifier.FindContextAction`
(24: `Mantle_3M_ContextFinder`, `Vault_2M_ContextFinder`,
`FindVaultOver`, `FindShimmyDown`, `AutoJumpToHangContext`,
`AutoDropToHang`, …); `Predicate.CanJump` (1,966);
`Predicate.ShapeCastHit` (1,311); `Predicate.RayCastHit` (235);
`Modifier.StepHeightDetector` (`SH_VaultHeight`, `SH_LedgeJump_Height`,
`PredictData:PR_VaultR_Data` offset (0,−104,25), extents [0,200]);
`DetectEdgeData_Standard` (radius 100, height 75, offset −104, tag
`Edge_Hang`); `SlopeData` (55, 50, 500, 150).

`ContextActionData` fields (hash → meaning from use): `0x9D05D4BD`
first tag, `0xB6CAD59C` third tag, `0x20418C39` edge height window vs
query point, `0x095BB326` search radius, `0x7963B113` 67.5 and
`0xF4D5BF50` 35/45 (angles, PARTIAL), `0xFD290924` query offset
(0,−104,0):

| Data | Tags | Height window | Radius |
|---|---|---|---|
| `GetUp_1M_Data_EdgeHang` | `Edge_Hang` | [−150,−50] (edge 0.5–1.5 m up) | 200 |
| `GetUp_1M_Data_EdgeNoHang` | `Edge_NoHang`, `ForceWallCover` | [−150,−50] | 200 |
| `Mantle_3M_Data` | `Edge_Hang` | [−350,−150] (1.5–3.5 m up) | 100 |

`BlendState:VaultAndMantleAutoPicker` holds 131 height/distance-named
transitions (`GetUp_1m_EdgeHang_D150`, `To_Mantle_3M_Autopicker_D150`,
`To_StepUp_50cm`, `StepDown_1m`, `To_WallRun_To_Hang_3m..7m`, …).
Example chain: `GetUp_1m_EdgeHang_D150` →
`Predicate.And:GetUp_1m_EdgeHang_AND_wClimb` = (Jump held or not
stealth) ∧ `FoundContextAction:GetUp_1M_EdgeHang_New` ∧
`CanJump:CJ_GetUp_StartPosGetUp2m_withClimb` (range [50,200],
width [−100,100], vertical [−50,50]) ∧ ¬`SCH_HopUp` →
`BlendState:Vault_1M` (`PL_Idle_2_Mantle_1M`, rate 1.2, blend 0.1 s).

## Traversal clips — VERIFIED root motion, PARTIAL selection

| Move | Clip | Length | Root (x,y,z cm) |
|---|---|---|---|
| Mantle 0.5/1/2/3 m | `PL_Base_L_Fist_Mantle50/100`, `PL_Base_R_Fist_Mantle200`, `PL_Base_L_Fist_Mantle300` | 1233/1233/1200/1200 ms | (0,50,464)/(0,100,464)/(0,200,385)/(−18,300,518) |
| Mantle from idle 1/2 m | `PL_Idle_2_Mantle_1M`/`_2M_orig` | 1566/2033 | (14,100,165)/(−6,200,113) |
| Run mantle 0.5 m / step 0.2 | `PL_Mantle_Up_50cm_Run_L/R`, `PL_Mantle_Step_20cm_L/R` | 866/1233 | (±13,50,332)/(±15,20,225) |
| Mantle 3 m | `PL_Mantle_3M`, `_Settle` | 1800/1233 | (−29,301,264) |
| Hurdle 0.25/0.5/1 m | `PL_Hurdle_25/50cm/1M` (+_L) | 1100/1100/1133 | (−53,0,869)/(−53,0,869)/(−52,0,950) |
| Stand jump | `PL_Base_L_Fist_StandJump_Start/_Land`, `PL_JumpUp_2m` | 866/933/1400 | (−11,−125,1)/(12,0,11)/(−16,195,204) |
| Running jump 3/5 m | `PL_RunJump_L_3M/_5M` (+_R) | 1066/1266 | (0,0,617)/(0,0,809) |
| Fall loop / land | `PL_Base_L_Fist_RunJump_Fall_22a`, `PL_StandingJump_Land`, `PL_HardLandStand` | 1733/1100/1966 | — |
| Wall climb 1.5/2/3 m | `PL_Climb_Up_L_150/200/300cm` (+_R) | 1766/1400/1766 | y 150/200/300 (0.85–1.7 m/s) |
| Speed climb (Agility) | `PL_SpeedClimb_Up_L_700..1100cm` | 1666–1866 | y 700–1100 (~5.9 m/s) |
| Hang / shimmy | `Bat_Base_L_Fist_LedgeIdle`, `PL_Shimmy_L_125/200/300cm` (+_R) | 3866/2100/2133/2400 | x 125/200/300 (0.6–1.25 m/s) |
| Pull-up | `Bat_Base_R_LedgePullup`, `PL_Ledge_Pullup_Cmbt`, `PL_Hang_NarrowPullup` | 2033/2000/1200 | (−2,208,126)/(7,208,141)/(0,207,54) |
| Catch / drop | `PL_LedgeCatch`, `PL_Base_L_LedgeDrop/_Land`, `PL_Hang_2Drop_Hang_8M` | 900/900/1033/2866 | (−2,−124,−8)/(−2,−980,−75)/—/(0,−800,0) |
| Wall plant / kick | `PL_Base_L_Fist_WallPlant`, `PL_Base_L_Fist_WallKick500a` | 1133/1133 | (5,−19,57)/(23,−381,−465) |
| Death from above | `Elf_StlhKill_Above_Jump` (event `BSLOWMO Elf_DiveBomb_Slomo`) | 1666 | (0,−2516,735) |

Root y signs are not uniform (`PL_Base_L_Cmbt_StandJmp_Start` reads
y −200 for an upward jump) — use the clip name's stated distance, not
the sign.

## Falls — PARTIAL

No fall damage. `BlendState:Falling` loops `…_Fall_22a`; landing →
`RunningJump_Land_AutoPicker` on `PhysicsSupported` (levels 0–2 by a
native rhythm level). `PSS_Death_From_Above` and `PSS_Agility` exist;
no `PSS_PrecisionFall` record. Trigger heights / slow-mo / roll-on-
landing thresholds: OPEN (native).
