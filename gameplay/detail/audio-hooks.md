# Audio hooks — how voice lines and sounds are chosen

> **Status:** **verified** — record shapes and attr names CONFIRMED on the installed `game.gamedb`; runtime behavior from THaych's Player Skins Overhaul (shipped mod, tested in-game).
> **Provenance:** Andúril gamedb dumps + THaych's Player Skins Overhaul notes (external, in-game).

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

## Runtime behavior (THaych, in-game)

- **Model tests:** a GameCondition with `SubStringCompare = 1`
  matches by substring ("contains"), so use full model names to avoid
  accidental matches; model tests **fail while loading**.
- **Bundled keys play one line.** A hook whose dialog can't play is
  skipped and evaluation moves on to the next hook.
- **A presentation that waits for a line hangs if nothing plays** —
  always end a key with a safety-net hook that can always play.
- **Dialog events** are one-off or repeatable by header: one-off
  `0xf / 2` (e.g. Celebrimbor's outro events), repeatable
  `0xd / 0`.
- Fast travel fires `menu_map_zone_exit_fast_travel`, then later
  `b_sbx_Rez_Delay`.
- Global audio mixes set at spawn are reset by loading hooks and
  menu previews — re-assert them on the gameplay event itself.

## Wwise IDs

Wwise object IDs are 32-bit **FNV-1 of the lowercase name**
(CONFIRMED: `.bnk` `BKHD` bank IDs equal FNV-1 of their bank file
names, e.g. `amb_rain` → `0xC4D128AC`). See `formats/detail/bnk.md`
and the Wwise anchors in `runtime/engine.md`.
