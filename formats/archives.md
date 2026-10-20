# Archives and bundles

> **Status:** **verified**
> **Provenance:** LTAR/EMBB/SLVT layout — Andúril + sow-rs.

How assets are packaged and found at runtime.

## LTAR v4 — `.arch06` — CONFIRMED

The game's archive container. Every stock file lives in one of ~30
`*.arch06` archives; the format is also the mount mechanism for
loose-file mods.

- Every entry is **chunked**, including tiny files; chunks are at
  most **64 KiB** (VERIFIED, sow-rs; matches Anduril's reader).
- A chunk is **Oodle-compressed when its stored size is smaller than
  the raw size** — compare stored vs. declared size per chunk; raw
  chunks pass through.
- The entry directory maps normalized lowercase VFS paths to chunk
  lists (hashed with the engine name hash).
- Verified by sow-rs against build 3095001: all 48 HotChunk archives
  decompress; 18 optional DLC/UWP paths absent from the Steam install.
- Andúril: `crates/anduril-ltar` reader + `anduril-vfs` mounting;
  `docs/formats/arch06.md` has our own byte-level notes. Do not copy,
  move, or hash these files casually — several are multi-GB.

### `default.archcfg` — CONFIRMED

`x64/default.archcfg` lists archive paths relative to `x64/`, one per
line, `;` comments. **File search order is bottom-up** — the last
archive that contains a path wins. This is the retail layering
mechanism (patch archives override base ones).

**Mod implication:** a loose `.arch06` in the install root mounts
after every listed archive, so its files override stock ones without
editing `default.archcfg`. This is how existing archive mods ship.

## EMBB v5 — `.embb` + `.bndl` — CONFIRMED

Bundles carry region and character content. A `.bndl` manifest lists
entries (path, offset/size into the sibling `.embb` payload); the
`.embb` is the concatenated resource data.

- Region bundle example: `worlds/retail/minasithil/minasithil.embb`
  = 1.42 GB; 425 `.inst`, 1 `.objres`, 1 `.wldmsh`, 1 `.wldclnt`,
  189 `.mesh`, 1,069 `.mat`, 1,818 `.tex`, 138 `.texa`, 42 `.anix`
  (uncompressed entry sizes; VERIFIED, sow-rs).
- Streaming stamps: `minasithil.ss_*.bndl/embb` (named in the region
  OBJR GADB) hold additional prefab/material/texture resources.
- Character bundles: `<name>_g.embb` (geometry) + `<name>_m.embb`
  (materials/textures) per character body/head/gear piece.
- Indexing note (VERIFIED, sow-rs): some bundle manifests come from a
  newer DLC patch and disagree with the payload; walking a bundle's
  own `u32 size, data` records in one forward pass over the LTAR
  chunks indexes every bundle reliably (sow-rs: +8,961 files found
  that way, 0 skipped).
- VFS should resolve a path across loose files **and** all bundle
  contents — many `.inst` props only exist inside `.embb` payloads.

## SLVT v4 — inside `.tex` — VERIFIED

Some `.tex` entries are `SLVT` containers, not TEXR:

```text
+0x00 "SLVT", +0x04 u32 version=4, +0x08 u32 entry_count,
+0x0C u64 entry_offset[count]
```

Each offset points at an embedded `TEXR` record; offsets are
monotonic and the last runs to EOF (15 SLVT files in the install;
the sample is a menu atlas: 3 entries).

## Save / misc containers

`.save`, `.subb`, `.vib`, `.vid`, `.pck`, `.bnk`, `.uis`, `.uiwd`,
`.pmo_rcv`, `.vegtile/.veggrid` have structural readers in Andúril —
see `docs/formats/*.md` (PARTIAL; not re-verified for this corpus).
