# Engine fundamentals

> **Status:** **verified**
> **Provenance:** engine frame, archives, VFS semantics — Andúril + sow-rs retail evidence.

Frame, units, hashing, and the virtual filesystem — the facts every
other document assumes.

## Coordinate frame and units — CONFIRMED (sow-rs, consistent with ours)

The game's data is **left-handed**: **+X right, +Y up, +Z forward**,
in **centimetres**. Evidence (sow-rs, task 052):

- Talion's bind pose has `R_Hand` at x = +64 cm, `L_Hand` at −64;
  `R_Toe` z ahead of `R_Foot`; spine along +Y. Facing +Z with right
  hand at +X and +Y up is left-handed.
- Front faces are **clockwise** in the file's frame (D3D convention):
  `(b−a)×(c−a) · vertex_normal > 0` on ~99.8% of Talion and Minas
  Ithil prefab triangles.
- Quaternions are `xyzw`, Hamilton product, active rotations — skel,
  MMSH, anix, PRFB and WDSC all compose identically.

A consumer that wants a right-handed world mirrors X once on import:
point `(x,y,z) → (−x,y,z)`; quaternion `(x,y,z,w) → (x,−y,−z,w)`;
triangle winding `(a,b,c) → (a,c,b)`; bounds swap X ends; bitangent
handedness must then be taken per-vertex (`w = sign((N×T)·B)`).
A consumer can also render in the game's native frame directly (no
mirror) — that works if the whole pipeline is self-consistent; mirror
at the boundary only if the renderer is right-handed.

Distances in gamedb records are centimetres unless stated; HUD "m"
figures are cm/100. Durations are milliseconds.

## Name hash — VERIFIED

One case-insensitive 32-bit hash identifies records, attributes,
files, nodes, and bones everywhere (gamedb record names, anix track
nodes, PRFB `self_hash`, CADB action names, OBJL object names).
Algorithm: the CRC-style hash shared by the community tools
(`sow_gamedb::name_hash`; the same function hashes `LTAR`/`EMBB` entry
paths and string-table lookups). Implementations hash the
**lowercased** string; path strings hash with **backslashes** as
stored (`prefabs\gameplay\props\questtrigger\qt_warcouncil.inst` →
`0x093D8B7A`, `worlds\retail\minasithil\minasithil.inst` →
`0x22FB37E9`). Attribute names are NOT stored in gamedb files — they
must be recovered by hashing candidate names (dictionary approach;
sow-rs matched 34% of nemesis attribute occurrences this way).

### Implementation reference — CONFIRMED (Andúril)

`h = h*0x397 + FOLD[b]` over the UTF-8 bytes of the string,
`h` starting at 0 (wrapping u32). `FOLD` is a 256-entry byte→symbol
LUT recovered from `ShadowOfWar.exe` — a character fold+map table,
derived data, not code. Nonzero entries (every other byte maps to 0):

```text
0x20:25 0x21:3a 0x22:33 0x23:3c 0x24:3d 0x25:3e 0x26:40 0x27:32
0x28:42 0x29:43 0x2a:41 0x2b:28 0x2c:36 0x2d:27 0x2e:37 0x2f:34
0x30:1b 0x31:1c 0x32:1d 0x33:1e 0x34:1f 0x35:20 0x36:21 0x37:22
0x38:23 0x39:24 0x3a:31 0x3b:30 0x3c:38 0x3d:29 0x3e:39 0x3f:35
0x40:3b 0x41..0x5a: 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
10 11 12 13 14 15 16 17 18 19 1a   (A–Z fold to 1..26)
0x5b:2c 0x5c:2a 0x5d:2d 0x5e:3f 0x5f:26 0x60:44
0x61..0x7a: same as 0x41..0x5a   (a–z fold identically → case-insensitive)
0x7b:2e 0x7c:2b 0x7d:2f 0x7e:45
```

Check vectors: `"Predicate.Not"` → `0x929DB77B`,
`"WorldInitializer"` → `0x6B9D05D0`, `"Script"` → `0x2F61ECD3`.
Bytes ≥0x80 and unlisted controls map to 0 (skipped in effect — they
multiply the accumulator without adding). Path strings fold `/` and
`\` to different symbols (`/`→0x34, `\`→0x2a): always hash the stored
backslash form.

## Virtual filesystem / mount order — CONFIRMED

- Loose files ship inside LTAR archives (`*.arch06` in the install
  root and `x64/`). `x64/default.archcfg` is the mount list:
  `;`-comments, paths relative to `x64`, **bottom-up precedence** —
  a later entry overrides earlier files of the same path. Loose
  `.arch06` files dropped in the root mount after the listed ones —
  this is the game's own mod mechanism.
- Inside archives, entries are named by normalized lowercase VFS
  paths (`worlds/retail/...`, `characters/...`, `database/game/game.gamedb`).
- **Compression:** Oodle via the install's own
  `x64/oo2core_5_win64.dll` (never redistribute it; load it from the
  user's install at runtime — the seam exists for other platforms).
- Bundles: `.embb` payloads indexed by sibling `.bndl` manifests give
  a second namespace layer (region and character content). Entries
  inside bundles use the same path namespace; a good VFS resolves
  `path` across loose files and every bundle.

## Executable landmarks — VERIFIED (sow-rs)

Build 1.0.8874. Format loaders found by unique `cmp dword, 'XXXX'`
sites in `.arch` code: PRFB `0x14058A26C`, OBJR `0x1407A75C8`, TERR
`0x1407AC0F4`, WDSC `0x1406B3510`, KEYF `0x1404BB804`, ANIX loader
`sub_140480AEC`. ANIX codec decoders register in tables at
`0x1411E1940` (SSE4.1/SSE3/scalar variants) and `0x1411DDC00`
(composition). The exe embeds Havok 2016.2 (incl.
`hkaSplineCompressedAnimation`).

## Install layout (this machine's install; modded)

```text
ShadowOfWar/
  *.arch06            data archives (+ loose mod archives)
  x64/                exe, oo2core, default.archcfg, plugins/
  game/               engine cfg (SKU.cfg, projectconsole.cfg)
  database/           (inside archives: game.gamedb, string.strdb)
  worlds/retail/<r>/  per-region bundles: <r>.embb, sectionsandlayers/
  characters/         meshes, skels, anix (loose + character bundles)
  prefabs/            PRFB .inst props/architecture
  interface/          .gfx movies, HUD textures
```

## Performance notes for implementers

- Region bundles are 1.4 GB+; mmap/stream, never copy wholesale.
- Corpus sizes (VERIFIED counts, full install): ~5,832 `.inst`,
  ~4,531 `.mesh`, ~29,464 `.tex` + 1,137 `.texa`, ~24,728 `.anix`,
  4,117 `.skel`, ~1,198 `.mat` (loose), 19 `.gfx`, one 57 MB
  `game.gamedb` (~695k records).
