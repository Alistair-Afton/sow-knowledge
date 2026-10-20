# Open questions

> **Status:** **living**
> **Provenance:** consolidated open questions across all docs.

Everything we know we don't know, consolidated. Each item states
what's missing and where to start.

## Formats

- **PRFB descriptor fields** `+0x1C`, `+0x20`, `+0x2C`, `+0x30`, and
  header counts `b`, `g`, `h`; node-table runtime semantics.
  Start: follow the PRFB loader past the mesh blob (`0x14058A26C`).
- **PRFB flavor variants:** `fishandguts`-style 52-byte tables that
  pass structural checks but decode wrong; ~30/407 files still show
  geometric outliers. Need per-file flavor discrimination beyond
  structural validity.
- **PRFB stream tails:** last ~1–5% of some blocks' stream-0 decode
  as attribute-like data — is stream 0 really `nv` records long, or
  does a second count exist?
- **MMSH `MeshIndexPairs` fallback:** what the game does when a
  selected `MinIndex..MaxIndex` range matches no mesh group;
  `FlairMin/MaxIndex` semantics.
- **Mesh→material binding field:** which record/field binds a mesh
  connection to a `MaterialSet` entry (the LOD-group material index
  path is proven, but is it *the* path?).
- **SKEL**: the `0x40f8d776` sentinel in 7 files; Torvin shield's
  alternate rig encoding.
- **ANIX**: `param` track → facial-control/camera-property mapping;
  block-B packed fields; `0x210C8F5A` track type.
- **OBJR sections** `ONLS`, `NLFE`, `SCRB`, `VOLM` detail; `.veggrid`
  body; `.objres` section-12 id space.
- **TERR** quadtree child-mask bit order (patch placement); whether
  non-region terrain exists.
- **`.wldclnt`** tail sections after the placement table
  (`u32 17, 5, 6` + floats).
- **CADB group keys** for prefab-local groups — not any tested hash
  of VFS paths/PRFB hashes/OBJR ids. Blocks prefab-edge linking.
- **`.mat`** GPU resource-binding tokens' runtime semantics; `.fxo`
  shader container format (SRHR/DXBC blocks readable but
  undocumented here).
- **`.gfx`**: on-screen anchoring of exported symbols; PlaceObject3
  filters; ExporterInfo code-offset base.
- `.save` full schema; `.scri` deep semantics; `ONLS/NLFE/...`.

## Behavior

- **Might meter** `OnHitAdjustments` first-match semantics;
  `duration` field meaning; `Int_ChargeMeter` derivation (floor?).
- **Elven Wrath** activation input + rage-mode effect set.
- **Headshot bonus:** `AlwaysApplyBonusDamageOnHeadshotOrStealth` is
  native — actual bonus unknown (records suggest none for a grunt).
- **Arrow spend:** the record that decrements `Int_AmmoCount` isn't
  linked in data — assumed native, 1/shot.
- **Fight circle:** attacker-score term combination, ring fill
  policy/caps (In≤3, Mid≤4 is a placeholder).
- **Awareness state machine** beyond "seen once".
- **Crit rolls:** timing window and how `BonusDamage`/`TimingCrits`
  chances combine (not in data).
- **Sprint/traversal selection:** blend-parameter semantics
  (`SituationModeDampener`, `Game_NodeLifeTime`) are native.
- **Captain generation:** first-valid-tribe claim is a field note;
  DLC eligibility, name tables' weights, builder logic unmapped —
  no faithful data-only generator exists.
- **Weather:** full pattern selection (quest/time/random), the
  transition mechanics, light-probe usage.
- **Shaders:** real alpha-test/blend states (only name heuristics
  exist); 4-layer blend formula; `k_fSkyLerp` sky patch maps.

## Engine

- Precise name-hash algorithm (documented in upstream tooling; the
  Shareable corpus describes its properties — publish an
  implementation reference with care for clean-room).
- Streaming/stamp system semantics (`minasithil.ss_*`, `StreamingStamp`
  objects), LOD switching to proxy meshes at distance.
- Audio: `.bnk`/`.pck` formats barely mapped.
- Networking/MP: none (single-player title).

## Andúril-specific residuals

- ~30/407 corpus `.inst` files decode with geometric outliers —
  residual flavors + stream-tail truncation (see geometry.md).
- `sim view` renders the world but far-shell/LOD selection is
  approximate; decals/alpha not rendered.
- Combat/traversal/gameplay docs above are **sow-rs evidence** —
  verified against records, not yet exercised in our runtime.
