# Spec: Faction XP reward system (`RewardXPLevelSetup` / `RewardXPLevel`)

> **Status:** verified — data chain proven; flag=1 draw rule
> decoded from the native interpreter (see below)
> **Provenance:** real-install `database/game/game.gamedb` via `anduril
> db` (patch04+IncreasedCap merged), two live in-game roster censuses
> (`Helper/live_traits.txt` n=223 lv~20-43, `Helper/live_traits_hi.txt`
> n=391 lv≥70, captured via nemesis.dll memory walks), sim runs
> (`anduril sim nemesis`), `tools/nemesis_parity.py` scorecards.
> String/registration RVAs read from `x64/ShadowOfWar.exe` directly.
> Date: 2026-10-18/19. The native interpreter function itself is
> **undecoded** — this spec records the data model and the measured
> accumulation behavior.

## Data chain (VERIFIED — record contents read live)

```
Faction/Rewards/XPLevelSetup  (24 recs: Default, Default_Act1,
    Traits_<Spec>, BER_Hate, Default_DLC3, Default_EasterlingRewardSetup,
    DLC4_Archetype_*, …)
  cells: refs → Faction/Rewards/XPLevel bundles (52 recs)
    cells: refs → _StructuresFactionRewardXPLevel rows
      row = [flag:u32, w:f32, lo:u32, hi:u32, ref→Faction/Rewards/Lists]
        pool cells: [0]=Str provenance path (NOT a count — VERIFIED),
                    [1..]=refs → Faction/Rewards/Definitions
          def → marker/trait refs + one _StructuresFactionRewardPickerList
                weighted pick
```

Verified row examples (`ANDURIL_DEBUG_SETUP=1` dump, seed 1):

| bundle | rows (flag,w,lo,hi→pool) |
|---|---|
| `Default_GrowInPower` | 6× `flag=1, w=0.25`, bands 1-10/11-20/21-30/31-40/41-50/51-60 → `Default_GrowInPower_<band>` pools |
| `Default_StatusEffect` | `flag=1` bands 5-10 w=0.001, 11-20 w=0.002, 21-35 w=0.009, 36-60 w=0.01 → all → `StatusEffect_StatusEffect` |
| `Default_HasMount` | `flag=0` 1,1 w=0.25 + `flag=1` checkpoints lo==hi at 10(0.5),20/30/40/50/60(1.0) |
| `Default_HasGang` | `flag=0` 1,1 + `flag=1` checkpoints 10..60 all w=1.0 |
| `Default_SeismicStrike` | `flag=1` checkpoints 30/40/50/60 w=1.0 |
| `XPLevel_IronWill` | `flag=1` band 25-60 w=0.005 → `List_IronWill` |
| `BodySlam`/`ThroatGrab`/`Infantry_HeavyAttack` | `flag=1` bands 0-15 w=0.05, 15-60 w=0.08 |
| `Default_SniperShot` | `flag=1` checkpoint 15,15 w=0.2 |
| `Default_OrcRarity`/`Default_Resistances`/`…Weaknesses`/`Default_Nemesis` | `flag=0` single row lo=hi=1 → creation-time |

## Row semantics

- `flag=0`: creation-time row. Applies once at member generation;
  `w` = probability of the pool draw (VERIFIED vs save census for
  rarity/weakness pools).
- `flag=1, lo==hi`: level checkpoint. Fires once when a member's level
  *crosses* that tier (VERIFIED: live `+0x1e0` picker history shows
  stepwise band growth — `Picker_HasMount` 2% lv10-19 → 10% lv20-29 →
  27% lv30-39; `Picker_HasGang` 15% → 37% → 30%).
- `flag=1, lo<hi`: level-up accrual row. `w` is a small per-level
  probability. **The exact draw rule is OPEN** (see below).

## Spawn-level gating (INFERRED → partial VERIFIED)

Members are generated at some level `spawn <= current` and accrue
flag=1 rolls only for levels `spawn < l <= level` — captains spawned
at a level never retroactively see lower tiers. Evidence: checkpoint
categories match "crossed checkpoints only" (~1.5× better than
"all reached"); live picker-history is a bounded 16-entry ring
(`+0x1e0`, 191/223 live members at exactly 16 entries) so early
evidence is evicted — measured per-category rates are *undercounts*
for aged members.

## Accumulation measurements (VERIFIED — live census)

| category | live lv~28 | live lv≥70 |
|---|---|---|
| statuseffect_marker | 24% (0/1 per member — exclusive) | 60% |
| weapon_elem | 25% | 60-69% |
| resist_Curse | 17% | 45% |
| ironwill | 13% | 29% |
| mount | 19% | 34% |
| gang | 27% | 47% |
| resist_Fire/Poison (any: vuln+resist+immune) | ~94% | ~99% |
| epictrait | 19% | 37-51% |

Status-effect presence fits a **constant per-level rate q≈0.022**
(`1-(1-q)^L`: 0.27@L≈14 → 0.61@L≈40). Σ of the bundle's four band
weights is exactly 0.022 — the strongest single clue for the native
rule.


## Post-spawn level-up draws (VERIFIED cadence, implemented)

Retail members draw each bound bundle's flag=1 rows once per level
gained — generation-time backfill models the *history* of a
member spawned at level L, but a live captain that gains power must
draw the newly lived levels too. Andúril: `gain_power` routes every
lifecycle power gain (duel/raid/vendetta wins, feasts, transforms)
through `Director::level_up_rewards`, which re-runs `reward_bundle`
with `spawn = old_power` so band+backfill semantics draw exactly the
gained levels; a new `creation` flag suppresses flag=0 rows and the
sub-minimum fallback post-spawn (`lo==hi` checkpoints still fire
when crossed). This is where `GrowInPower_*` upgrades, the
`Picker_Resist_*_Upgrade` chains and epic-trait accumulation happen
in play.

## The flag=1 draw rule — VERIFIED-STATIC (native interpreter)

Found by following the menace_level setter `FUN_14035188c`
(`member+0x4b8`, field name `menace_level`/`XPLevel`): on every real
level change it calls `FUN_14034f9f0(member)` — the draw loop — then
commits `trait_level (+0x4c0) = menace_level`.

`FUN_14034f9f0` decoded:

- iterates `level = trait_level ..= menace_level` (inclusive, capped
  at a global max `FUN_140f847a0`)
- per level, `FUN_1403513dc` collects matching rows across the
  member's bound-bundle vector `member+0x1f8`
  (`levelchange_reward_list`), also honoring an injected exclusion
  list read from `member+0x4f0` (event/purpose-driven row blocklist)
  and a level remap from `purpose_levelremap_record` (`member+0x280`)
- per bundle slot `FUN_140352dc4` takes the FIRST row matching
  `lo-1 <= level <= hi-1` — **`hi` is a real band cap** — after a
  row-probability roll (`FUN_140353d54`)
- per matching row, `FUN_14034f980` returns the pick count
  (**binary-confirms `Rewards/Lists` cell-0 = pick count**) and
  `FUN_140350500` runs the weighted def draw (`[FACTION|REWARD]
  Picking Reward From List` / `Passed Filter - Weight` logs) —
  def gates `FUN_140105aec`, weight `FUN_1403505f4`, reservoir
  pick `FUN_1401b3b90` — applied via `FUN_140351af8` (ApplyEvent)

Generation consequence: the ctor zeroes the object, so `trait_level`
starts at 0 and the first menace_level set backfills all levels
1..=spawn — **the BAND-CAP+BACKFILL model is the verified rule**;
the earlier FINAL default (hi ignored) was an empirical stand-in and
is now behind `ANDURIL_XPROW_FINAL`. The lv80 StatusEffect shortfall
under band-cap (~38% vs live 60%) is therefore a MISSING SOURCE —
element weapons/SE markers arrive via event/purpose reward paths
(`member+0x4f0` extras, ApplyEvent pushes) that the XP-row system
never sees — not draw-rule slack. Level-delta rewards:
`FUN_140351220` computes `clamp(menace + delta, cap-1)` from the
reward's delta item and re-enters the setter — the shame
`ReduceLevel_*`/`IncreaseLevel_*` path.

Sim experiments (env knobs `ANDURIL_XPROW_MULT`, `ANDURIL_XPROW_GELO`,
`ANDURIL_XPROW_ALLBAND`, `ANDURIL_XPROW_FINAL`;
`sim nemesis 1 200 0 '' level=N`) — historical comparison, kept
for reference:

| model | lv28 statuseffect | lv80 statuseffect | lv28 curse | lv80 curse |
|---|---|---|---|---|
| live | 24% | 60% | 17% | 45% |
| band-only `lo<=l<=hi` (baseline) | 24-26% | 24% | ~17% | 22% |
| mult≈5 on band model | 36% | 60% | 53% | 44% |
| `level>=l>=lo` (GELO) | 17% | 59% | 30% | 72% |
| all rows always (ALLBAND) | 32% | 58% | 60% | 74% |
| **FINAL — eligibility at final level, one draw per lived level** | **26%** | **58%** | 40% | 74% |

**The empirical models are retired** — the native interpreter is
decoded (above), so the default is the verified band-cap+backfill
rule; `ANDURIL_XPROW_FINAL`/`GELO`/`ALLBAND` remain as comparison
knobs only. Historical note: FINAL fit lv80 StatusEffect best
because its stale-band draws accidentally supplied the element
grants that retail's *purpose/event* reward paths deliver — the
fit was modeling a missing source as draw-rule slack.

### Stored-bound semantics (VERIFIED, implemented)

`FUN_140352dc4` matches `stored_lo - 1 <= level <= stored_hi - 1` —
the stored bounds sit one above the effective band. Concretely,
`Default_StatusEffect`'s rows store `{5,10} {11,20} {21,35} {36,60}`
→ effective 4–9 / 10–19 / 20–34 / **35–59** (SE rows stop accruing
at level 60). A `lo == hi` checkpoint row matches exactly level
`lo-1` — fires once when that level is crossed. The sim applies
`saturating_sub(1)` to both bounds in band matching and checkpoint
detection (`reward_bundle`, `nemesis.rs`).

### Row probability (VERIFIED record decode)

XPLevel row cell 1 is an f32 per-level probability, not a weight —
`Default_StatusEffect` rows carry ≈0.001 / 0.002 / 0.0094 / 0.01
across the four bands. SE accrues slowly over a career (~0.4
expected picks by level 60 backfill), consistent with live 24%@28 /
60%@80 presence when purpose/event grants are added.

### Purpose level remap + row exclusion (VERIFIED-STATIC, unmodeled)

- `FUN_140352f8c` consumes `purpose_levelremap_record`
  (`member+0x280`): a `Faction/Traits/PickerLevelRemap` table of
  `{hi, lo, delta}` entries — first band containing the level
  applies `max(0, level+delta)`. Five tables exist:
  `TraitLevelOffset_{-2,-1,0,+1,+2}`. Applies only to rows whose
  `FUN_1403530f0` predicate is set. Who selects the table (which
  GameDB record carries the ref) is OPEN — `Faction/RevengeType`
  records are empty name-only markers.
- `FUN_14034fb7c` reads `member+0x4f0`: a 0x40-stride keyed
  container (key at entry+0x28 via `FUN_14016e240`) whose `+0x18`
  payload is a row blocklist — active purposes/events can suppress
  specific reward rows. The PRODUCER (what writes +0x4f0) is being
  scanned (`ScanWriters '4f0'`); OPEN.

Residuals under the verified rule (n=200): lv80 curse_immune ~52
vs live ~45 (pool-yield, not frequency), grenade ~35 vs ~36 ✓,
mount ~22 vs ~28; lv28 curse ~22 vs ~17, gang ~35 still hot.
`fire_resist` (~66-76 vs live 94-99) and epictrait/unbreakable
shortfalls remain attributed to missing grant paths (event/purpose
bundles, scar/downgrade member-aging), not draw frequency.

## Andúril model

`anduril-sim` implements the proven data chain (level bands, weights,
pool yield). The flag=1 draw rule defaults to **FINAL**: each ranged
`flag=1` row with `lo <= level` draws once per lived level
(`spawn+1..=level`), `hi` ignored. Env experiments:
`ANDURIL_XPROW_BAND` (band-cap+backfill `lo..=min(hi,level)`),
`ANDURIL_XPROW_GELO`, `ANDURIL_XPROW_ALLBAND`, `ANDURIL_XPROW_MULT`
(probability scale). All are models — none is canonical until the
interpreter is decoded; the OPEN stays open.
