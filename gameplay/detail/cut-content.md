# Cut, removed, and leftover content

> **Status:** **living** — add entries as they are verified.
> **Provenance:** retail VFS listing, `string.strdb`, GameDB records.
> Every entry is tagged VERIFIED (present/checked in shipped data) /
> INFERRED (strong data signal, intent unconfirmed) / EXTERNAL
> (reported by a community source — name it).

Retail ships a large amount of dev-era material. This registry is
the place to record anything that exists in the files but is cut,
unused, demo-only, or debugging infrastructure — with the evidence,
not just the name.

## How to find more (method)

- **strdb orphan keys:** a `dx_*` line key with no entry in
  `string.strdb` usually means the line's subtitle was removed (or
  never shipped) — family gaps (e.g. `_cb_01..03` missing inside a
  numbered run) flag cut members of a line set.
- **Same text, two keys:** emit the strdb, group by text — the same
  localized string under two `dx_*` families marks a line that was
  moved between contexts (one context often cut).
- **Path names:** grep the VFS list for `demo`, `test`, `debug`,
  `deprecated`, `e3`, `dev`, `old`, `unused` — hundreds of shipped
  dev assets surface this way.
- **Unreferenced GameDB records:** records in `game.gamedb` that no
  other record references (builders never drawn, role lists never
  pulled, pickers gated by unreachable predicates) are cut or
  vestigial content. Cross-check against the live roster before
  calling anything "cut" — spawn-gated ≠ removed.

## Registry

### Dialogue

- **"Be my sword!" — keyed twice (VERIFIED strdb; context cut per
  EXTERNAL THaych/Mach-Tei).** `dx_pr_dm_CommDefend_cb_08` (the
  `pr_dm_CommDefend` domination "command defend" family) and
  `dx_sbx_CeleActivate_cele_02` (the 20-line Celebrimbor
  activate-bark set) resolve to the same text. The CommDefend
  context — Celebrimbor ordering a dominated captain to keep
  fighting ("Stay and fight for me") — is reported cut; the line
  was re-registered in the activate pool. See
  [`audio-hooks.md`](audio-hooks.md#duplicate-line-keys--the-same-recording-under-two-names).
- **`dx_pr_dm_CommDefend_cb_01..03` — no strdb entries (VERIFIED
  absent keys; cut lines INFERRED).** The shipped CommDefend set
  starts at `cb_04` ("Fight by my side!") — the first three
  numbered slots were dropped or never shipped.

### Regions / board content

- **`Demovolcano` and `InvasionMtnPass` board remnants (VERIFIED).**
  Live rosters carry members seated in `volcano_Row_*` rows under
  a `Demovolcano` region id and `MtnPass_Row_*` rows under
  `InvasionMtnPass`, including a `volcano_row_overlord_Demo` seat —
  leftovers of a demo/invasion board variant shipped with the
  retail data.
- **Forthog Orc-Slayer — USED, not cut (VERIFIED).** Forthog is
  the Savior: during the PreDeath/Last-Chance sequence a dice roll
  (`Calculate_Forthog_Chance` formula over
  `SaviorForthogChance`/`SaviorNonDLCForthogChance` value options,
  `Float_SaviorNonDLCForthogChance_PCT` difficulty var,
  `DLC_Forthog` game-data var, `Predeath_Savior_ForthogCooldown`
  timer items) can spawn `Savior_ForthogOrcSlayer` to rescue the
  player from an orc/olog. Full support chain ships: model
  (`Orc_Forthog` + `Forthog_Head`), AI tree (`AI_Forthog`),
  personality table, `Forthog_Despawn` combat state, dedicated
  sever actions, `dx_pr_sh_savior_*` dialog, `Denizens_Forthog`
  appendix entry, and `DisallowForthog` tags. The Savior system is
  generic — Forthog is one channel next to Serka (DLC4),
  Eltariel/Idril/Baranor NPC saviors, follower/bodyguard saviors,
  and GP story saviors (see `story-locked.md` for the mechanic;
  `Gameplay_Saviors` menu checkbox controls it). Historical
  context: the Forthog DLC was pulled from sale and made free —
  the `SaviorNonDLCForthogChance` var is the non-DLC fallback
  chance left in data.
  - Leftovers inside the feature (INFERRED dev residue):
    `PreDeath_Savior_SyncRefocus_Forthog_ELMbackup`, `DEMO_Savior_*`
    presentation predicates, `PreDeath_Savior_Debug_AlwaysSave*`
    item groups, and `RangedSaviorTestbed` find-perch predicate.

### Dev / demo assets (VERIFIED present, unused at runtime)

- `debug.bndl`, `debugline*.fxo`, `debugbox/capsule/cylinder/
  sphere.fxo`, `debuglinesystem*.mat`, `deferred_debug*.mat`,
  `debugshadowmaps.fxo`, `deferreddebug*.fxo` — debug rendering set.
- `d5demointro.subb` / `d5demointro.vid` — a demo-intro video
  package.
- `demo_*.fxp` (camera shakes, `demo_stormbringeridentify_*`,
  `demo_drakereleasehold_shake`, `demo_frontgate_dofshake`,
  `demoqueststart_smash`, `demo_onfire_muchless.mat`,
  `orcarrow_trail_demo_deadeye.fxp`) — demo-scene FX records.
- `camerarig_test.anix`, `bl_movetest_dcbl.anix` — animation test
  clips.
- `deprecated.mat`, `deprecated_d.tex{,0}` — assets literally named
  deprecated.
- `av-sync_90sec_30fps.vid` — A/V sync test clip.

## Not (yet) in scope

Region seat counts that differ from the standard grid (MinasIthil
having no `Inner_Warchief` occupants on the sampled roster) are
fill-path behavior, not cut content — keep them in
`faction-lifecycle.md`, not here.
