# `.vid` — `VIDO` v12 video descriptors

> **Status:** **verified** — descriptors decoded; playback untouched
> **Provenance:** descriptors + Bink headers decoded on the real corpus.

Tiny sidecar files next to `cinematics/videos/**` and
`interface/videos/**` `.vib` streams (152 `.vid` paths in the install;
150 are VIDO — the 2 `gon3_gate` files under `realizations/` contain
non-VIDO data beginning `8d 40 bd 48` / `a5 40 b1 48`, i.e. not this
format; treated as foreign stragglers).

## Layout (verified — every string offset/len resolves corpus-wide)

```text
+0x00  char[4] "VIDO"
+0x04  u32     version = 12
+0x08  u16     flags (always 0)
+0x0a  u16     entry_count (1 or 2; 113 files have 2, 37 have 1)
+0x0c  u8      flag = 1
+0x0d  entry0: { u32 off, u32 len }
+0x15  [if entry_count == 2] u8 = 1, u8 = 1, then
       entry1: { u32 off, u32 len }
...
at off:  NUL-terminated path, len includes the NUL
```

Entry 0 is the standard `.vib` path (`cinematics\videos\…\name.vib`);
entry 1, when present, is the same path with `uhd` appended
(`name.vibuhd`) — the high-resolution variant. The u8 markers are
constant `01` in every observed file; semantics unknown.

## Known unknowns

- Meaning of the u8 markers and the u16 flags.
- How the engine picks `uhd` (likely display-res dependent).
- Why `gon3_gate01/02.vid` are non-VIDO (mislabeled foreign blobs).
