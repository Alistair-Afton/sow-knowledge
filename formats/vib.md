# `.vib` — Bink video (BIKi / KB2i / KB2j)

> **Status:** **verified** — Bink headers decoded; playback untouched
> **Provenance:** descriptors + Bink headers decoded on the real corpus.

Cinematic videos referenced by [`vid.md`](vid.md) `VIDO` descriptors.
741 files, three codec magics:

| magic   | files | format |
|---------|-------|--------|
| `KB2j`  | 659   | Bink 2 (shipping) |
| `BIKi`  | 71    | Bink 1 |
| `KB2i`  | 7     | Bink 2 intermediate |

## Header (both families)

```text
+0x00  char[4] magic
+0x04  u32     payload size (== filesize − 8)
+0x08  u32     frame count
+0x0c  u32     largest frame size
+0x10  u32     frame count (repeat)
+0x14  u32     width
+0x18  u32     height
+0x1c  u32     fps dividend
+0x20  u32     fps divisor
+0x24  u32     flags
+0x28  …       audio-track table + frame-offset index, then codec payload
```

e.g. `d3fin10_20.vib` = 3357f @ 1920×1080 30fps (134 MiB);
`brand.vib` (BIKi) = 345f @ 512×256 29.97fps.

## Corpus

737/741 parse. Failures: the two `gon3_gate` foreign blobs (same
corrupt pair as their `.vid` siblings) and two truncated legal-splash
stubs — documented stragglers, not parser gaps.

## Out of scope

The compressed frame payload is proprietary Bink codec data; only the
container header is decoded. Playback needs a licensed Bink decoder or
an external extractor (e.g. ffmpeg-bink for Bink1).
