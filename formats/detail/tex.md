# `.tex*` — TEXR/TEXA cooked textures + CPUT raw copies

> **Status:** **proven** — all ~32.5k textures decode; renders verified
> **Provenance:** all ~32.5k real `.tex*` resources decoded; rendered output verified.

All ~32.5k texture resources are a small cooked header + tail wrapped
around a standard `DDS ` payload. Implemented in `anduril-tex`;
`anduril tex <path>` / `tex dump` / `tex scan` in the CLI.

Extension variants (all the same container):

- `.tex` — mip chain minus the top mip(s) of the source
- `.tex0` — same layout (minus-top-mip chain); the streaming strip
  carried via `HighMip.Arch06`
- `.texuhd` — includes the top mip; UHD upgrade layered on `.tex0`

Verified on stock data: `balrog_body_d.tex0` (source metadata 4096²,
embedded DDS chain starts at 2048²) is the chain *minus the top mip*;
`baranor_desertt1_d.tex` (2048² source) embeds the chain starting at
1024² while its `.texuhd` adds the 2048² top mip;
`orcbarricade01_1_proxd.tex0` (256² source) is the *full* chain — small
textures are not stripped. An external spec (Universal Modder KB)
describes `.tex0` as "the top three mips"; that does not match these
retail files. What *is* consistent is that the engine prefers
`.tex0`/`.texuhd` at full detail, and a `.tex` alone is a downgrade.
- `.texa0`, `.texauhd`, `.texa` — 2D/cube texture *arrays*
- `.texcpu` — NOT this format; a `CPUT` raw-pixel copy (below)

## Header (TEXR / TEXA)

```text
+0x00  char[4]  "TEXR" (plain, v8) or "TEXA" (array, v1)
+0x04  u32      version
+0x08  u8       log2(source width)   ─┐ describe the SOURCE texture,
+0x09  u8       log2(source height)   │ not the embedded slice:
+0x0a  u8       0                     │ a .tex0 strips top mips but
+0x0b  u8       source mip count      │ keeps source metadata
+0x0c  u8       kind: 1 = 2D, 3 = volume, 4 = array
+0x0d  u8       class: 0 = color/generic, 2 = normal map
+0x0e  u16      0
+0x10  u16      source-name length (no NUL)
+0x12  name     source .tex*-style path
```

**Retail read path (Ghidra `FUN_1405030ec`, byte-verified):**
accepts `{TEXR, ver==8}` **or** `{TEXA, ver==1}`, and the same
`SLVT` v4 wrapper as CRES. After the magic the loader reads a
**u64** (the packed +0x08 source-metadata bytes), then the source
name as a fixed **0x104-byte** slot, then the kind byte `t` —
`1,2` → 0x20-header records, `3..7` → 0x2C-header records, `5` →
its own 0x24 shape. `t==4` (array) is **only legal under TEXA** —
a TEXR with kind 4 is rejected. Array layer source names are
`count × 0x104` slots skipped after the tail (the "name pool" in
the tail spec is that fixed-slot array).

## Tail (2D kind 1/4, 23 + name bytes)

```text
u8    tag: 3 plain, 4 array
u16   source width  (pixels)
u16   source height (pixels)
u8    array depth (1 plain; cubeproj_array has 92)
u8    array sub-kind (0 slices; 3 = cube array, observed once)
u8    source mip count (repeats +0x0b)
u8    1
u8    DXGI format (71 BC1/DXT1, 77 BC3/DXT5, 87 BC7, 98 BC7_SRGB...)
u8[7] 0
u16   2
u8    sRGB flag (1 = decode sRGB — _d diffuse; 0 = linear — _n/_s/_b)
u8    2
u8    0
u8    depth duplicate
then  u16 len + name per entry — a name pool running to end of tail
      (arrays; for cube arrays count > depth)
```

## Tail (volume kind 3, 22 bytes)

```text
u8    5
u16   width, u16 height, u16 depth   (pixels; volumes use u16 dims)
u8    1
u8    DXGI format (26 R11G11B10_FLOAT and 87 BC7 observed)
u8[13] 0
```

The embedded DDS then sets `DDSCAPS2_VOLUME` with `depth` in its
header — used by `.lso.*.tex` lightmap volumes and colorgrade LUTs.

## `.texcpu` — CPUT v1

```text
"CPUT" u32 ver=1  u16 name_len  name  u32 w  u32 h  u32 bpp  pixels
```

Raw uncompressed BGRA (0xff alpha + B,G,R observed). File size is
`10 + name_len + 12 + w*h*bpp/8` exactly. Rare — two files in the
install, used where the engine needs CPU-readable pixels.

## Verification

`anduril tex scan` parses **all 32,505 texture files** in the retail
VFS (every `.tex`, `.tex0`, `.texuhd`, `.texa`, `.texa0`, `.texauhd`,
`.texcpu`) with zero failures. Fields cross-checked against embedded
DDS metadata (dims, mip counts, DXGI, volume flag).

## Pixel decode

`Tex::decode()` decodes the embedded DDS top mip to RGBA8 for the two
formats that dominate the corpus: **BC1** (DXT1 / dxgi 71-73) and
**BC3** (DXT5 / dxgi 77-78), including BC1 punch-through alpha and BC3
interpolated alpha. Other formats return
`TexError::UnsupportedFormat`. Used by `anduril-render` for textured
software rendering.

## Map conventions — sow-rs verified

- `_n` normal maps: **BC3 with X in green, Y in alpha**; blue ≈ 255,
  red is a cavity-like channel. Re-encode to BC5 losslessly: keep the
  alpha block (BC4-Y) and re-encode green→BC4-X.
- `_s` specular maps: R ≈ specular intensity, B ≈ roughness,
  metallic 0 (approximation — the game's BRDF differs).
- `_d` diffuse alpha on 4-layer shaders = height/cavity map used by
  the layer blend.
- See `formats/mat.md` for which material property names each.

## `SLVT` containers

Some `.tex` entries are `SLVT` (a container of embedded TEXR records),
not TEXR — 15 files on the corpus. See `formats/slvt.md`.

## Still unknown

- `+0x0a` byte (0 on 2D, 7 on volumes) and `+0x0b` on volumes (1) —
  for volumes the log2 fields are constant-ish rather than literal.
- Tail bytes `[17..19]=u16 2`, `[20]=2`, `[21]=0` — constant markers.
- Whether tail `tag` (3/4/5) is a field count or a variant enum.
- Name-pool indexing on cube arrays (which name maps to which face).
