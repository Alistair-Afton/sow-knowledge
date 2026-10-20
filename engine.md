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
