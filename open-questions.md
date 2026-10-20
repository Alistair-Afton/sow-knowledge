# Open questions

> **Status:** **living**
> **Provenance:** consolidated open questions across all docs.

What we know we don't know, consolidated. Each item states what's
missing and where to start. **This list is not exhaustive** — large
areas of the game haven't been examined at all yet, so there is more
unknown here than is written down. Resolved items are removed, not
kept.

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
- **TERR**: whether non-region terrain exists; world origin of the
  quadtree grid; `0x60` header word; group tail u32s; `tile_size`.
- **`.wldclnt`** tail: the packed index data after the 17 region
  AABBs (list framing; u16 lanes are ~51% valid placement indices).
- **CADB**: group tail fields; edge action/mask semantics;
  `PFDG` record meanings (`n1`/`n2` split, `a`/`b` index space).
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

- Streaming/stamp system semantics (`minasithil.ss_*`, `StreamingStamp`
  objects), LOD switching to proxy meshes at distance.
- Audio: `.bnk`/`.pck` formats barely mapped.
- Networking/MP: none (single-player title).

## Residual decode / parity gaps

- ~30/407 corpus `.inst` files decode with geometric outliers in
  Andúril — residual flavors + stream-tail truncation (see
  geometry.md). Whether sow-rs's decode handles these files is
  untested — worth cross-checking.
- No public viewer yet renders with correct far-shell/LOD selection,
  decal/alpha handling, or real shader states — expect "sharded" or
  placeholder visuals until the flavor/brdf questions above close.
- Combat/traversal/gameplay docs above are **record-level evidence** —
  verified against gamedb/`.bvr` records, not yet exercised inside a
  running engine. Treat them as the spec the runtime should
  reproduce, then check parity in-game.
