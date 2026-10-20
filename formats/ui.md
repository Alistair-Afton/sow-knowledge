# UI: Scaleform `.gfx` + HUD records

> **Status:** **verified**
> **Provenance:** Scaleform tag walks + HUD gamedb records — sow-rs.

## `.gfx` — Scaleform movies — VERIFIED

SWF superset (Scaleform GFx). All 19 install files parse with exact
tag walks (47,559 tags): all `GFX` uncompressed signatures
(`CFX` = zlib stream supported but unused), SWF versions 8–17,
**AS2 only** (`DoAction` 12; no `DoABC`).

- Header: 3-byte sig, u8 version, u32 total length, RECT stage
  (twips = 1/20 px; every HUD movie is 1280×720), u16 8.8 frame
  rate, u16 frame count.
- Tags: `u16 code=v>>6, len=v&0x3f` (0x3f → u32 length). Sprite
  bodies nest tag lists ending in their own `End`.
- Decoded tags (sow-rs): DefineShape1-4 (bounds + bitmap fill ids),
  DefineMorphShape(2) (health-bar drains), DefineSprite,
  PlaceObject2/3 (full flag sets), RemoveObject, DefineEditText
  (every HUD text field is HTML with font `$HydraHUD`),
  ExportAssets/SymbolClass, ImportAssets2 (imports `$HydraHUD` from
  `interface\fonts\gfxfontlib.swf`), FrameLabel, DefineFont2/3.
- Scaleform tags: `1000` ExporterInfo (per-file DoAction offset
  constants); `1009` DefineExternalImage2 — `u32 id, u16 fmt 13,
  u16 w, u16 h, str source_name, str exported` where
  `source_name` = the game's TEXR path and `exported` = `…tex.tga`.
  28/36 sphud externals resolve to `.tex` in the VFS; `UIRENDERTARGET`
  is the engine's render target (minimap).
- **Display structure:** root timelines are empty — every widget is
  an **exported symbol** (`SPH-*` 105 in sphud, `MM-*` 6 in minimap,
  `LIB-*` in componentlibrary) the engine instantiates at runtime.
  Empty sprites are slots the engine fills (health/might bar art is
  `.mat`-driven, not in the movie).

## HUD record map — VERIFIED records, PARTIAL layout

`Interface/HUD/WidgetList:All_HUD` aggregates everything:
`CombatBars:ComboPowerBar` (might bar; 3 levels → `fx\ui\menuitems\
powerbar{1,2,3}.mat` + backings, `FullScaleValue` 0.5/0.5/1),
`Widget:Combat` (→ `Elements:CombatWidget` = **`MM-Combat`**,
bottom-left), `:ComboCount`, `:ComboCrit`, `Crosshair`
(→ `LIB-HydraCrosshair`, centre, `GameplayRanged` state), prompts,
`Events/BossInfo`, `Events/CurrentTarget` (→
`Elements:TargetSelectedWidget`).

Inside the movies (VERIFIED tree reads):

- **`MM-Combat`** (minimap.gfx): `Bars` with `Health`/`MountHealth`
  (red-tinted) /`Energy` + pulse clips (empty `EndCap` slots =
  engine-drawn); `HealthPacks` icon+count; `MapRT` 512×256 render-
  target shape; `NorthIndicator`; `PlayerLevel`; `FocusBarClip`
  (`Arrow` = `ammo_arrownormal.tex`, empty `FocusBar` slot,
  `TokenCount` = arrow count); alarm icons; `StatusEffect`.
- **`SPH-PowerBar`**: two empty slots — engine draws the might bar
  from the powerbar materials.
- **`SPH-ComboCount`**: hit counter field (50 px `$HydraHUD`) with
  in/out label animation; **`SPH-ComboCrit`**: crit text + icon.
- **`SPH-HealthBar`/`SPH-CharacterMarkerCommon`**: enemy health —
  `bottomhealthbar.tex` frame + 99-frame `redbar.tex` drain morph +
  level text + `Warchief`/`Overlord` icons + `TurnTimer`.
- **`SPH-TargetSelected`/`SPH-EnemyNamePlate`/`SPH-BossInfo`**: name,
  objective, modifier icon, name+title plates.
- `HitCounterUI:Player` `ChargeColors`: 4 charge-level looks
  (colour/scale/rotation/shake, descending thresholds); the
  streak→level band mapping is HYPOTHESIS.
- `Interface/FloatingText` styles (`DefaultDamage` white/1000 ms/
  scale 3→0.7/X-offset range 20–80, `DefaultCrit`, `Healing`, …).
- `Combat/CounterType`: X/B/Y can counter, A cannot; Y needs target
  item `CSP_Impact_Parent`; nav markers `Combat_NormalCounter` /
  `Combat_SpecialCounter` / `Combat_Cannot{Parry,Block}` with
  Warg/Troll/Nazgul overrides.
- `Interface/GlyphReplacement:CounterPrompt/:DodgePrompt/:WASD` =
  prompt glyph ids.

**Not decoded:** how the engine anchors symbols on screen (AS2 or
`Interface/Elements` records), PlaceObject3 filters/clip actions,
ExporterInfo code-offset base.
