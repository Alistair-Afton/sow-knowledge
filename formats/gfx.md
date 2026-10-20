# `.gfx` — Scaleform GFx HUD movies

> **Status:** **verified** — container + tag walk confirmed on the real corpus
> **Provenance:** sow-rs/OnSight `formats/gfx.md` (exact tag walks over all
> 19 installed files); standard tags per Adobe SWF spec v19. Andúril has
> no GFX reader yet — this doc is the spec to implement against.

Scaleform GFx movies drive the HUD and menus. They are Flash SWF files
with Scaleform extensions: `GFX` = plain, `CFX` = zlib-compressed body
(`FWS`/`CWS` are plain SWF). Every installed file is `GFX`.

## Corpus (sow-rs, install scan)

- **19 `.gfx` files** (8 loose + 11 inside bundles), all parsed with
  exact tag walks; 47,559 tags total.
- SWF versions 8, 9, 14, 15, 17. ActionScript **2 only** (`DoAction`
  code 12) — no `DoABC` in any file.
- Every HUD stage is **1280×720 px**; frame rates 60 (`sphud`) / 30
  (`minimap`).

## Header

| Offset | Type | Field |
|---|---|---|
| 0 | 3 bytes | signature |
| 3 | u8 | SWF version |
| 4 | u32 | total file length after decompression, header included (= file size on all stock files) |
| 8 | RECT | stage in twips (1/20 px) |
| — | u16 | frame rate, 8.8 fixed |
| — | u16 | frame count |

RECT is a bit record: 5-bit `n`, then four `n`-bit signed values
`xmin xmax ymin ymax`, padded to a byte.

## Tag records

u16 header: `code = v >> 6`, `len = v & 0x3f`; `len == 0x3f` → u32
long length follows. Top level ends with `End` (0) at exactly file
length. `DefineSprite` (39) = `u16 id, u16 frame_count` + nested tag
list ending in its own `End` at the exact sprite-body end. A correct
reader rejects any record running past its parent and any decoded body
that doesn't consume its declared length.

Dominant tags on the corpus (of 47,559): `ShowFrame` 1 (18,581),
`PlaceObject2` 26 (17,509), `PlaceObject3` 70 (1,915),
`DefineSprite` 39 (1,364), `DefineEditText` 37 (1,135),
`CSMTextSettings` 74 (1,124), `RemoveObject2` 28 (1,041),
`FrameLabel` 43 (952), `ExportAssets` 56 (698), `DoAction` 12 (451),
`DefineMorphShape` 46 (441), `DefineShape` 2 (351),
`DefineExternalImage2` 1009 (246), `DefineShape3` 32 (132),
`ExporterInfo` 1000 (19 — one per file), `DefineFont3` 75 (only in
`interface/fonts/fonts_latin.gfx`, 9.6 MB).

### Decoded tag bodies (sow-rs exact-walk results)

- `DefineShape1-4` (2,22,32,83): id, bounds RECT (83 adds edge bounds +
  flags byte); first fill-style array — bitmap fills (types 0x40-0x43)
  give the bitmap character id. Colours RGB for 2/22, RGBA for 32/83.
- `DefineMorphShape(2)` (46,84): id, start/end bounds; a placement's
  `ratio` picks the blend (health-bar drain animations).
- `PlaceObject2` (26): flags u8 (0x80 clip actions, 0x40 clip depth,
  0x20 name, 0x10 ratio, 0x08 colour transform, 0x04 matrix,
  0x02 character, 0x01 move), depth u16, then flagged fields in that
  order.
- `PlaceObject3` (70): as 26 plus second flag byte (0x20 visible,
  0x10 image, 0x08 class name, 0x04 cache-as-bitmap, 0x02 blend mode,
  0x01 filters); class name right after depth. Filter lists undecoded.
- `DefineEditText` (37): id, bounds, flag bytes, font id/class, height
  (twips), RGBA colour, max length, layout, variable name, initial
  text — exact. Every HUD field is HTML with font `$HydraHUD`.
- `ExportAssets`/`SymbolClass` (56,76): `u16 count`, then
  `(u16 id, string)` pairs — exact.
- `ImportAssets2` (71): url string, two reserved bytes (1,0), count,
  `(id, name)` — exact. HUD movies import `$HydraHUD` from
  `interface\fonts\gfxfontlib.swf`.
- `FrameLabel` (43): string + optional anchor byte — exact.
- `DefineFont2/3` (48,75): id, font name.
- `DefineSubImage` (1008): u16 id, u16 image id, 4×u16 rect — spec
  layout; not present in this game.
- `DoABC` (82): flags, name, opaque bytecode — none in this game.

Matrix bit record: optional scale (5-bit n, two 16.16 values),
optional rotate/skew (same), then 5-bit n and two signed translations
in twips. Colour transform with alpha: has-add, has-mult, 4-bit n,
RGBA mult (8.8) and/or add.

## Scaleform tags (game-specific)

| Code | Tag | Layout as found in this game |
|---|---|---|
| 1000 | `ExporterInfo` | `u16 version` (0x401), `u32 flags` (0), `u16 bitmap_format` (13), `u8 len + prefix` (empty), `u8 len + swf name` (`sphud`), `u16 count` + `count × u32 code offsets`. Count equals the file's `DoAction` tags; offsets = body offsets + one per-file constant (sphud +1,506,562; minimap +1,126,537) — presumably offsets in the original exporter SWF (OPEN). |
| 1009 | `DefineExternalImage2` | `u32 character id`, `u16 bitmap format` (13), `u16 width`, `u16 height`, `u8 len + source name` (`interface\Textures\HUD\redawareness.tex`), `u8 len + exported file` (`…\redawareness.tex.tga`). Exact. The exported path minus `.tga` is the game's TEXR texture in the VFS — **the link from HUD art to `.tex` assets**. |
| 1001 | `DefineExternalImage` | Spec layout `u16 id, format, width, height, u8 len + file name`; not present. |

External-image characters are also exported via `ExportAssets` under
their source name. In `sphud.gfx` 28 of 36 resolve to a real `.tex`;
the 8 that don't (`menusheet`, `firebirdhud`, `firebirdhud2`,
`deathmark_short/long`, `hour_glass`, `stronghold`, `uirendertarget`)
are absent from the install — `UIRENDERTARGET` is the engine's render
target (the minimap's `MapRT`).

## Display structure

Root timelines are **empty**: every widget is an exported symbol
(`SPH-*` in sphud, `MM-*` in minimap, `LIB-*` in componentlibrary) the
game instantiates and positions natively. Inside a symbol, positions
are local twips. Empty sprites are slots the engine fills — the health
and might bars' `EndCap`, `FocusBar`, `PowerBar` clips — so bar fill
art is not in the movies. See `specs/hud-map.md` for the widget map.

## Open questions

- The per-file constant in `ExporterInfo` code offsets.
- `PlaceObject3` filter lists and clip actions (skipped, not decoded).
- How the engine positions exported symbols (native `Interface/Elements`
  records are the likely driver — PARTIAL).
