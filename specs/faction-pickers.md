# Spec: Faction picker system — gates, groups, dedupe

> **Status:** verified — gates, groups, dedupe vs live data
> **Provenance:** `_StructuresFactionTraitPicker` /
> `_StructuresFactionMarkerPicker` / `Faction/PickerList*` records
> read via `anduril db`; live `+0x1e0` picker-history census
> (`Helper/live_pickers.txt` n=223) vs sim `PUSH via Picker_*` debug
> counts (`Helper/sim_pickers.txt` n=200). Date: 2026-10-19.

## Picker-row model (VERIFIED)

- `PickerList_*` records hold `_Structures…Picker` rows; each row is a
  gated, weighted candidate. The picker evaluates gates, then draws
  once weighted across eligible rows.
- `IncludedTraitPicker` refs on a `Reward/Definition` are a
  **per-ref AND gate** — every ref must independently match (trait/
  marker held), NOT a merged OR. VERIFIED 2026-10-18: merging made
  any weakness satisfy every `PG*_Remove_EW_*` def → GrowInPower pool
  stayed ~40-def eligible → per-def share ~1% (Immune_Curse 4% sim).
  Per-ref AND shrinks eligibility to real matches → `PG_Curse_Immune`
  ~10%/draw; lv80 scorecard `resist_Curse` 23→live 45 (was 4),
  `ironwill` 29 == live 29.
  - Known side effect (expected, honest): `resist_Fire/Poison` dropped
    ~15-25pt vs the merged-OR model — the old inflation was spurious
    cross-weakness eligibility.

## Mutual exclusion / dedupe (VERIFIED)

- `pg:{group}` dedupe in `apply_picker_mode` — one outcome per
  `PickerGroup_*`; `PickerGroup_Resist_*` groups are exempt (covered by
  the `resist:{channel}` rule; `_Perm` scars legitimately share).
- Raw-list dedupe `rl:{list}`: retail resolves each `PickerListRaw_*`
  at most once per captain — e.g. `SW_Terror_LowHealth1..4` all
  resolve to `PickerListRaw_FearHate_LowHP`; stacking them gave ~47%
  vs live ~10%.
- `PickerList` qty semantics: each `Rewards/Lists` pool draws cell-0
  defs per invocation — cell 0 IS the pick count (CORRECTED: the
  `k=0x03` string-provenance reading was wrong; `Default_Resistances`
  cell0=12 draws its whole 4-def pool, and weighted draw order +
  `PickerGroup_Resist_*` mutual exclusion reproduces the live ~80:20
  Resistant:Immune grade ratio and ~91% channel coverage).

## Live-vs-sim calibration (VERIFIED @lv28 cohort)

- `WeakBy` family calibrated: BerserkWearsOff 21/23, PinningShot 12/9,
  Headshots 9/9, StealthAttacks ~9/7 (live/sim %).
- `GrenadedRetreat` is class-skewed live (Hunter 54, Infantry 20,
  Archer 17) — parity category conflates class prevalence; sim ~20
  overall is in range.
- `+0x1e0` picker history is a **bounded 16-entry ring** (191/223 at
  exactly 16) — per-picker live rates are undercounts for aged
  members; applied vectors `+0x198`/`+0x1b0` are the authoritative
  applied set.

## OPEN

- Residual ~1.5-2× uniform over-fire across ALL fear/terror triggers —
  points at pool draw *frequency* (see `faction-xp-rewards.md` OPEN),
  not picker logic.
- `gang`/`mount` ambient `PickerList_GangsPets` yield (~43%
  gang-or-pet alone) — verify its true invocation path (def-referenced
  only? gated?).

## Andúril model

`anduril-sim::nemesis` implements per-reference AND gating, group
pools, and dedupe as specced. Creation-time pool draws are calibrated
against live census (`tools/nemesis_parity.py`); most lv28 categories
within ±5pt. Deviations: none intentional — residual category deltas
trace to the OPEN spawn-level/XP-row questions, not picker logic.
