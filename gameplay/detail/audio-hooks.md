# Audio hooks — how voice lines and sounds are chosen

> **Status:** **verified** — record shapes and attr names CONFIRMED on the installed `game.gamedb`; runtime behavior per THaych (external notes; his own reports, not our observations).
> **Provenance:** Andúril gamedb dumps + THaych's Player Skins Overhaul writeups (external — shared as notes/Discord messages; not all mechanisms may be in the shipped mod).

All context-sensitive audio — barks, dialog, presentation lines,
mix changes — is selected by data in the `AudioHooks/*` categories.

## Record model — CONFIRMED

| Category | Records | Role |
|---|---|---|
| `AudioHooks/Keys` | 18,510 | a trigger point: **ordered** list of hooks |
| `AudioHooks/Hooks` | 29,616 | `Actions` + `Conditions` (all must pass) |
| `AudioHooks/Actions` | 29,709 | what plays: sound event/dialog, mix on/off, cooldowns, priority, playback character |
| `AudioHooks/Conditions` | 3,345 | `{EvaluationMode, VariableConditions, Character, GameConditions}` |
| `AudioHooks/GameConditions` | 2,188 | the actual test (`SubStringCompare`, `MinValue`/`MaxValue`, `StringValue[]`, …) |
| `AudioHooks/DialogEvents` | 17,239 | dialog line containers |
| `AudioHooks/Characters` / `Variables` / `MixStates` / `DialogCooldowns` / `Conversations` | | supporting tables |

`EvaluationMode` is `Normal` or `InvertPassFail`. Example:
`PR_Dialog_FA_OutroSpeech_Bundled` (fortress-capture outro) is a key
of 7 hooks — Talion recapture lines first, then Celebrimbor's
`BrightLordOutro*` — gated by conditions such as
`QuestComplete_GP_Finale40` and `CurrentWorld_IsNotDesert`.

## Runtime behavior (THaych's reports)

- **Model tests:** a GameCondition with `SubStringCompare = 1`
  matches by substring ("contains"), so use full model names to avoid
  accidental matches; model tests **fail while loading**.
- **Bundled keys play one line.** A hook whose dialog can't play is
  skipped and evaluation moves on to the next hook.
- **A presentation that waits for a line hangs if nothing plays** —
  always end a key with a safety-net hook that can always play.
- **Dialog events** are one-off or repeatable (THaych: "header
  `0xf/2` one-off, `0xd/0` repeatable"). In attr terms (CONFIRMED
  names): field 0 is a bool-flag word (`FacialAnimationLoop`,
  `PlaybackHistorySaved`, `PlayFacialAnimationWithSameName`,
  `TriggerPresentationCallbacks`) and field 5 is `PlaybackBehavior`;
  `0xf/2` = the `PlaybackHistorySaved` bit + `PlaybackBehavior 2`.
  Observed on retail: `dx_pr_fo_BrightLordOutroB_CB` is `0xf/2`, while
  `…OutroA_CB` and `…OutroE_CB_01` read `0xd/0` — so not every
  Celebrimbor outro carries the one-off marker; the in-game one-off
  behavior of those may come from elsewhere (OPEN).
- Fast travel fires `menu_map_zone_exit_fast_travel`, then later
  `b_sbx_Rez_Delay`.
- Global audio mixes set at spawn are reset by loading hooks and
  menu previews — re-assert them on the gameplay event itself.

## Dialog event and action fields — CONFIRMED names

`AudioHooks/DialogEvents`: 0–3 flag word (above), `4 DialogCategory`,
`5 PlaybackBehavior`, `6 PlaybackOrder`, `7 SubtitleBehavior`, then
two same-length string arrays of the lines — `CustomBodyAnimationList`
(lip-sync/body names) and `SoundEventList` (audio names) — followed by
`SwitchGroupFilter`, `FacialAnimation`, `ExternalSourceEvent`,
`ResponseAudioHookKey`, and a `FacialBlend` ref
(`Character/FacialBlend`). `AudioHooks/Actions` field `14` is
`DialogEventList`. The voice-FX event a dialog plays (e.g.
`dx_b_sbx_CB_All`) is what Wwise `PostEvent` receives — not the line
name (THaych).

## Gameplay audio flows (THaych's reports)

- **Spawn hooks** run on key `CharacterSpawns_Delay` (~6 s after spawn,
  list order: off-hook before on-hook).
- **Wraith grunts:** Celebrimbor's are always requested; the Dark
  Talion mix (`mix_wra_dark_talion_on/off`) mutes them. Dash sounds use
  keys `wra_swiftness_in`, `wra_gc_wrath_dash`. The "on" mix is
  switched **at spawn** by the hook `mix_wra_dark_talion_on_item`
  (CONFIRMED: action `mix_wra_dark_talion_on` + conditions
  `Source_IsPlayerAny` + `HasItem_GP_PlayerIsDarkTalion`); the
  endgame skins otherwise get the mix from their `PC_IsDarkTalion`
  buff's apply/remove. Because model tests run at spawn **before**
  the skin model is applied, a model test alone fires the mix for
  the wrong skin — THaych disables the item hook outright, leaving
  the buff-driven on/off for the endgame skins (a skin without the
  buff is simply never muted).
- **Pre-ring vs endgame dialogue** is picked by
  `HasNotCompleted_GP_Finale10` / `QuestComplete_GP_Finale10` — both
  test "is the final quest done". The pre-ring banter lines are
  never removed: the recordings ship in the sound packages and the
  ending only changes which hooks' conditions pass — so restoring
  them is a data-only fix (THaych). His rewire changes the
  *question itself*, not the hooks: the spare
  `CharModel_GraugOrCarnan` GameCondition (record CONFIRMED in
  `AudioHooks/GameConditions` — it CharModel-tests `Troll_*` model
  strings and is unused in retail per THaych) gets its model list
  rewritten to every *other* skin's endgame model (Talion, Dark
  Talion, Baranor, Serka, Eltariel, … incl. Prologue Talion —
  Celebrimbor's deliberately left out), so it asks "is the player
  wearing another skin's endgame model?". Both Finale10 conditions
  then point at that check, giving: pre-ending any skin → banter
  plays; post-ending Celebrimbor → banter plays; post-ending any
  other skin → endgame lines. Two pitfalls he hit: the condition
  must use **the same matching mode the game's own
  `CharModel_Player*` checks use** — the default mode also matched
  the Celebrimbor player and muted him again; and model tests run at
  spawn before the skin model applies (above).
- **Silencing a sound for one skin** (THaych's dash-mute for Prologue
  Talion): insert a mute hook **first** in the key's ordered list —
  hooks are evaluated in order, so a first-position hook gated on the
  skin's CharModel wins before the sound plays. He used the spare
  `fts_ply_step_bamboo` hook + `CharModel_PlayerOrcBerserker`
  condition inside keys `wra_swiftness_in` / `wra_gc_wrath_dash`.
- **Player identity tests:** `CharModel_PlayerTalion`,
  `CharModel_Celebrimbor`, `CharModel_PlayerAny`; dead slots
  (`PlayerSkin01/02`) name no model.
- **Fortress capture speech** (`PR_Dialog_FA_OutroSpeech_Bundled`):
  Talion's `dx_pr_fo_RecaptureOutro_Talion(_d4)` (6 lines, cycling)
  once `QuestComplete_GP_Finale40`, else Celebrimbor's
  `BrightLordOutroA–E_CB`, picked by `PlayerOwnedForts` float ranges
  (0 / 1–2 / 2–3 / 3–4; E = the desert fortress). Talion's hooks also
  gate on `CurrentWorld_IsNotDesert` (CONFIRMED). Lip-sync
  presentations are `Combat/Attack` records named after the lines
  (pool `PC_FortIntros`).
- **Fast travel / respawn:** `menu_map_zone_exit_fast_travel`, then
  respawn key `b_sbx_Rez_Delay`. After Finale40 both play
  `dx_sbx_SW_Respawn` (Witch-King/Talion Shadow Wars lines);
  `b_sbx_Rez_Delay` also plays `b_sbx_RezGenBucket` (Talion +
  Celebrimbor banter) while `HasNotCompleted_GP_Finale10`.
- **Throne room:** `PR_OverlordShowdown_Bundled` → overlord taunt
  `dx_pr_fk_OvTaunt###A_<personality>`, whose action chains through
  `_StructuresAudioHookNextKey` to `PR_Dialog_OvTauntResp_Bundled` —
  31 Talion replies filtered by camera variant, overlord personality,
  "not the keep's own overlord" and "overlord killed you 0 times". No
  passing reply is fine (the fight starts). No Celebrimbor replies
  exist; Serka has her own `dx_pr_fk_TauntRejoinder_Serka`.

## Wwise IDs

Wwise object IDs are 32-bit **FNV-1 of the lowercase name**
(CONFIRMED: `.bnk` `BKHD` bank IDs equal FNV-1 of their bank file
names, e.g. `amb_rain` → `0xC4D128AC`). Hashing every GADB pool
string (plus key/event names) builds a reverse ID → name table that
resolves dialogue (`dx_*`), mixes (`mix_*`, `st_glb_mix_*`) and music
(THaych). See `formats/detail/bnk.md` and the Wwise anchors in
`runtime/engine.md`.

**Debugging a voice line:** log Wwise `PostEvent`/`SetState`/
`SetSwitch`/`SetRTPCValue` with timestamps from a plugin, line it up
against gameplay video (frames every few seconds + per-second audio
level to find speech), then hash candidate line names and read the
subtitle from the English `string.strdb` (THaych's workflow).

## Duplicate line keys — the same recording under two names

**VERIFIED (strdb 2026-11):** the line "Be my sword!" is keyed twice:

- `dx_pr_dm_CommDefend_cb_08` — in the `pr_dm_CommDefend`
  (Domination "command defend") Celebrimbor family. Verified set:
  `cb_04` "Fight by my side!", `cb_05` "Slay my enemies!", `cb_06`
  "To war!", `cb_07` "Fight for the Bright Lord!", `cb_08` **"Be my
  sword!"**, `cb_09` "Protect your lord!", `cb_10` "Gird yourself
  for battle!" (`cb_01..03` have no strdb entry).
- `dx_sbx_CeleActivate_cele_02` — in the `sbx_CeleActivate`
  (Celebrimbor "Activate" lines) family, a 20-entry set
  `cele_01..20` of domination-activation barks ("You shall abide!",
  "Bring Light!", "Serve my will!", … "My eyes are yours!").

The line is registered under both the command-defend context (a
cut "Stay and fight for me" set — per THaych/Mach-Tei, EXTERNAL)
and the activate-bark pool. Practical consequence: strdb text
collides across `dx_*` families — the key names the *hook
context*, not a unique recording, so a subtitle match alone cannot
identify which context played.
