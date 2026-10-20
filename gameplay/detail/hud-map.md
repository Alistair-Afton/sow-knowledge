# HUD — gamedb records, Scaleform movies, widget map

> **Status:** **verified** — record links + movie structure; runtime
> screen placement PARTIAL (engine-driven, not in the movies)
> **Provenance:** sow-rs/OnSight `hud-map.md` (gamedb queries +
> `sow_gfx` tree walks of `sphud.gfx`/`minimap.gfx`). Format spec:
> `../../formats/detail/gfx.md`.

## Gamedb records — VERIFIED

- `Interface/HUD/WidgetList:All_HUD` — main aggregation: links
  `CombatBars:ComboPowerBar`, `Widget:Combat`, `ComboCount`,
  `ComboCrit`, `Crosshair:Crosshair`, prompt widgets, events.
  `Gameplay_ComboMessages` groups ComboCount+ComboCrit; `Reticle` →
  Crosshair; `ButtonPromptCluster` → `TopPrompts`,
  `TopKeyboardPrompts`, `StandardPrompts`, `KeyboardPrompts`.
- `Interface/HUD/Widget/CombatBars:ComboPowerBar` — element
  `Interface/Elements:Combo_PowerBar`; three configured levels use
  `fx\ui\menuitems\powerbar{1,2,3}.mat` + backings
  `powerbarback{1,2}.mat`/`powerbarbackfull.mat`, pulse FX, full scale
  values 0.5 / 0.5 / 1.
- `Combat/Buffs/HitCounterUI:Player` — `ChargeColors` 4 thresholds
  (desc): L3 color −6284541, scale 1.6, rot 20, shake 16/15; L2 same
  color 1.4/15/12/30; L1 −1220855, 1.2/10/8/50; L0 white (−1), 1/0/0/0.
  Hit-count→charge-level conversion is NOT encoded here (OPEN).
- `Combat/CounterType:{X,A,B,Y-Counter…}` — X/B/Y can counter, A
  cannot; Y can be blocked + needs `CSP_Impact_Parent` for the 200 ms
  special window; X/B/Y redirect cannot-counter to A. Nav markers:
  `Interface/NavMarkerTypes:Combat_CannotParry`, `_CannotBlock`,
  `_NormalCounter`, `_SpecialCounter` (+ Warg/Troll/Nazgul overrides).
- `Interface/FloatingText:{DefaultDamage,DefaultCrit,Healing}` (16
  styles) — color, shadow, lifetime ms, `scale`/`XOffset` ranges.
- `Interface/HUD/Widget/Events/{BossInfo,CurrentTarget}`,
  `Interface/NavMarkerTypes:EnemyHealthBar`,
  `Interface/ElementFXList:NamePlate`,
  `Interface/GlyphReplacement:{CounterPrompt,DodgePrompt,WASD}` —
  exist; composition PARTIAL.
- Meter records behind the bars: `ChargeMeterSetup:PowerChargeMeter_*`,
  `WraithMeterSetup:CBMeter_*` (see `moveset.md`); the **curse**
  overlay is a separate `CurseMeterSetup:Player` (max 300, on at 100.1,
  off at 100, `curse_buildupoverlay.mat`) — don't conflate with Wrath.

## Movie structure — VERIFIED (sow-rs tree walks)

Both root timelines are **empty**; widgets are exported symbols the
engine places at runtime. `sphud.gfx` exports 105 `SPH-*` sprites + 36
external images; `minimap.gfx` exports 6 `MM-*` + 16 images. Positions
below are local pixels inside the symbol; boxes are in symbol space.

- **`MM-Combat`** (bottom-left cluster): `Content` (123.3,−100.8) →
  `Bars` (scale 0.597) with `Health`, `MountHealth` (sprite #60, red
  tint mult[256,0,0,256]), `Energy`, `HealthPulse`/`EnergyPulse`; bar
  `EndCap` children are empty sprites → **engine-drawn bar art**.
  Also `HealthPacks` (icon + `X/X` text), `MapRT` (512×256 shape,
  `UIRENDERTARGET.tex` = minimap render target), `NorthIndicator`
  (82×21), `PlayerLevel` ("Lvl 99"), `FocusBarClip` (scale 0.797:
  `Arrow` `ammo_arrownormal.tex`, empty `FocusBar` slot, `TokenCount`
  "20" 19 px = arrow count, `PoisonMessage`, `InfiniteAmmo`),
  `Alarm`/`ElevatedAlarm`/`DecreasingAlarm`, `Last_Chance0..2`,
  `StatusEffect`.
- **`SPH-PowerBar`** = two empty slots `PowerBar` + `PowerBarPulse`
  (sprite #303, reused as `Health` in `SPH-BossInfo` and
  `HealthBarClip` in `SPH-QuestSelected`) — engine-drawn.
- **`SPH-ComboCount`** → `Content` → `Count` (scale 1.161 at
  (169.3,0.35)): 35 px right-aligned field + `Field` (50 px,
  `$HydraHUD`); box 180×84; labels `out`0/`off`9/`in`10/`on`19
  (slide-up fade). **`SPH-ComboCrit`** → `Field` 34 px centred +
  `Icon` (`confrontation.tex`) on label `Advanced`.
- **`SPH-HealthBar`** (enemy): frame `bottomhealthbar.tex` (52×19) at
  depth 1 + 99-frame drain at depth 2 (morphs, then `redbar.tex`
  fills); used by `SPH-HealthAndIcon` and `SPH-CharacterMarkerCommon`
  (health bar + `Level` text, `Warchief`/`Overlord` icons,
  `TurnTimer`).
- **`SPH-TargetSelected`** → `Target` at (14,4.5): `Name` 16 px,
  `Objective` 15 px, `Modifier` icon 70×70; `TargetClone` animates on
  `changed`. **`SPH-EnemyNamePlate`**: `PrimaryInfo` (`Header` 18 px +
  `Icon` 48 px) + `AuxInfo` (`Single`/`Double`/`FortDefenseWave`);
  `Member` shows `NameTitle`. **`SPH-BossInfo`**: `Name`, `Modifier`
  icon 45 px, `Health` slot (48.65,7.7).
- Crosshair: `Interface/HUD/Widget/Crosshair:Crosshair` →
  `LIB-HydraCrosshair` (componentlibrary.gfx), `POSITION` (0.5,0.5),
  fade 0.5 s, HUD state `GameplayRanged`; children `Energy` (left arc,
  200-frame drain = Focus gauge), `Stamina` (mirrored arc), `Reticle`
  labels `None`/`Drawing`.
- 28/36 sphud external images resolve to `.tex` in the VFS; the 8
  missing are engine targets or absent assets (`formats/gfx.md`).

## Textures seen (candidate roles, not proven usage)

`interface/textures/hud/` — `bottomhealthbar.tex` (128×128 DXT5;
confirmed the enemy-bar frame), `numbersheet.tex`,
`progressbarcircular.tex`, `backawareness.tex`,
`questicons/new/enemies/enemyeasyarrow.tex`, kraken icons
(`blade`, `dark_talion`, `damage`, `defense`, `health_packs`,
`objective_normal`, `redbar`), `ammo_arrownormal/_fire/_posion.tex`.
Filename ≠ role — verify via symbol/bundle usage.

## Open questions

- Where the engine anchors exported symbols on screen
  (`Interface/Elements` attributes hold `POSITION` pairs — partial).
- Hit-streak → `ChargeColors` band thresholds.
- Whether `Energy`/`Stamina` arcs are Focus and Wrath or Focus alone.
