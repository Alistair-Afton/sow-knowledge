# Traversal and movement

> **Status:** **verified**
> **Provenance:** CADB markup + movement records — sow-rs probes.

VERIFIED by sow-rs (records + root-motion checks on `player_elf.bvr`
and `pl_movement/*.anix`); centimetres throughout.

## Constants — VERIFIED

`Client/PlayerMovement:PlayerMovement` (shared by every player skin):
gravity **−2000 cm/s²**, capsule 125 × 35 cm, step-up **40 cm**,
`FallMinHeight` 100, fall-damage band 500–1200 but `FallDamageMin/Max`
= **0** (no fall damage), yaw blend 400 ms. `LocomotionIK:Player`:
foot trace 20, `GroundOffset` −104 (pelvis 1.04 m above feet — also
the origin offset of every `.bvr` context query).

## Input — VERIFIED

`.bvr` `Predicate.Command.ActionId` → `Profile/Commands` record →
`PCKeyboard`/`PCGamepad`: `Jump_Button` = `CombatPrimary04` = Space;
`LedgePullUp` Space; `LedgeDropdown` E; `StealthToggle` **Left Shift**
— Left Shift is stealth, NOT sprint. **Sprint is not a command**:
`Run.To_Sprint` needs move magnitude ≥ 0.25 **and `Jump_Button`
held** — WASD jogs, holding Space sprints/free-runs, a Space press at
an edge vaults/mantles, E drops. `Dash` = `TriggerLeft` (gamepad LT
only).

## Speeds — VERIFIED (clip root motion)

`PL_Walk` 1.28 m/s · `PL_WalkFast` 1.80 · `PL_Walk_Standard/Combat`
2.28/2.53 · `PL_Jog` 4.62 · `PL_Sprint85`/`PL_Combat_Sprint85`
6.40 · `PL_Sprint_Super` 11.66 · `PL_Sprint_Boost_City` 12.95 m/s.
Blend selection (which point plays when) is native — PARTIAL.

## Context-action edges — `CADB` — VERIFIED layout

Traversal is **authored edge markup**, not derived collision edges.
`Character/Models/ContextActions` (124 records) are the tag
vocabulary: `Edge_Hang`, `Edge_Pullup`, `Edge_NoHang`,
`Edge_Hang_Dangle`, `VaultOver`, `Hurdle`, `Scramble`,
`Block_Movement(_RailRun)`, `Rail_Running`, `Ladder_Rung`,
`ZipLine`, `Mantle 3M`, ...

Per region, OBJR section `CADB` (v3, in `.objres`):

```text
"CADB" u32 3, u32 actions, u32 edges, u32 planes, u32 volumes, u32 groups
actions: u32 name-hash list (ContextActions record names)
edges:   52 B = vec3 a, vec3 b, vec3 n + u32[4] bitmask over actions
planes:  16 B = vec3 normal, f32 d
volumes: 32 B = vec3 min, max, u32 plane_start, u32 plane_count
groups:  52 B = u32 key, vec3 min, max, u32 edge_start/edge_count,
         vol_start/vol_count, u32, u32  (sorted by key; ranges tile)
```

Minas Ithil: 75 actions, 181,578 edges, 829 planes, 85 volumes, 569
groups; tag counts `Edge_Hang` 71k, `Edge_NoHang` 42k, `Edge_Pullup`
29k, `Block_Movement_RailRun` 29k, `Hurdle` 18k, `VaultOver_Mount`
17k. Median edge 0.6 m. `CAWR` section mirrors the group keys.

Semantics: **`n` points onto the walkable top** (checked: surface
0.3 m along n, ground lower on −n). 563 groups are prefab-local
(origin-centred); world groups are the region key `0x6B9D05D0`
(122k edges) and per-stamp keys. **The local-group key is OPEN** —
not the name hash (nor CRC32/FNV/djb2) of any VFS path, PRFB
self_hash, or OBJR id. This is the biggest gap in traversal:
connecting authored edges to placed prefabs.

## Player blend DB (`player_elf.bvr`) — VERIFIED records, PARTIAL

270 categories; 2,047 `BlendState`, 15,704 `StateTransition`, 2,301
`SimpleAnimation`. Query records: `Predicate.FoundContextAction`
(3,425) → `ContextActionData` (3,493) with hash-mapped fields: first
tag `0x9D05D4BD`, third tag `0xB6CAD59C`, height window
`0x20418C39`, search radius `0x095BB326`, query offset
`0xFD290924` = (0,−104,0).

Example verified chain: `GetUp_1m_EdgeHang_D150` → predicates
(`FoundContextAction:GetUp_1M_EdgeHang_New` tag `Edge_Hang`, height
[−150,−50] rel. query, radius 200; `CanJump:...` distance [50,200])
→ `BlendState:Vault_1M` (`PL_Idle_2_Mantle_1M`, rate 1.2, blend
0.1 s). `VaultAndMantleAutoPicker` holds 131 named height/distance
transitions.

## Traversal clips — VERIFIED (root motion)

Mantles 0.5–3 m (`PL_*Mantle50/100/200/300`, `PL_Idle_2_Mantle_*`,
`PL_Mantle_3M`); run mantle/step; hurdles 0.25–1 m; stand/running
jumps; wall climbs 1.5–3 m and speed-climb 7–11 m; hang/shimmy/
pull-up; ledge catch/drop (8 m hang-to-hang); wall plant/kick;
death-from-above (`Elf_StlhKill_Above_Jump`, y −2516). Full clip→
file list in OnSight `moveset-map.md` M6. **Signs in root y are not
uniform** — use the clip name's stated height, not the sign.

OPEN: fall/precision-fall/death-from-above trigger heights, roll-on-
landing thresholds (native or unnamed `.bvr` fields).
