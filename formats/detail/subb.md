# `.subb` — `SUBB` v1 subtitle cue tables

> **Status:** **verified**** — cue tables decoded on all 121 files
> **Provenance:** cue tables decoded on all 121 real cinematic `.subb` files.

One file per cinematic video under `cinematics/videos/**` (121 files).

## Layout (verified — exact EOF on all 121; string count == cue count)

```text
+0x00  char[4] "SUBB"
+0x04  u32     version = 1
+0x08  u32     cue_count
+0x0c  u32     flags (always 8)
+0x10  u32     cue_table_bytes = 8 * cue_count
+0x14  cues × { u32be start, u32be end }   ← BIG-ENDIAN (unique!)
+...   strings × { u16 len, utf8[len] }    (not NUL-terminated)
```

The cue times are **big-endian u32s** — the only BE fields seen in the
whole corpus so far; all other formats are LE. `start <= end` holds on
every cue in the corpus.

Units appear to be **milliseconds on the video timeline** — max
observed 599040 ≈ 10 min (finale cinematic `f_10_20`, matching its
676 MiB `.vib`). Not independently verified against `.vib` duration.

String ids are `cn_<scene>_<idx>_<speaker>` (`cn_SOP_10_10_040_Cb`,
`cn_D3FIN10_20_080_Talion`) — the `cn_` prefix matches the
string-database (`SKDB`) cinematic namespace, so these resolve through
`anduril-str` at runtime.

## Known unknowns

- The `flags` word (constant 8).
- Exact time units (ms assumed; could be 1/1000-scale ticks).
