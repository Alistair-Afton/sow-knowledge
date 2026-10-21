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
- **Forthog Orc-Slayer assets (VERIFIED present).**
  `orc_head_forthog.bndl`, `orc_head_forthog.embb`, and the
  `dx_pr_sh_savior_*` / `dx_pr_sh_ambsavior_*` presentation bundles
  ship in retail. The `Savior_ForthogOrcSlayer` role list exists in
  GameDB but never feeds the board pool (VERIFIED — no
  FactionMemberBuilder draws it). Historical context: the Forthog
  character was a memorial DLC pulled from sale and later made
  free; the data remains in the base install.

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
