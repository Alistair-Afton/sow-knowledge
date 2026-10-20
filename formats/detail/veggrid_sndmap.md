# `.veggrid` + `.sndmap` — small index formats

> **Status:** **verified** — index formats decoded
> **Provenance:** vegetation grid/sound-map cells decoded on the real corpus.

## `.veggrid` — VEGG v1 (4 files)

Vegetation grid manifest; lives next to `.vegtile` content:

```text
+0x00  char[4] "VEGG"
+0x04  u32     version = 1
+0x08  u16 len + utf8  section path ("worlds\retail\…\spawncondition*")
       …             pad + f32 grid params (~20000.0, ~25000.0) — opaque tail
```

`VegGrid` in `anduril-veg`.

## `.sndmap` — SMAP v1 (1 file)

`sound/_eventmap.sndmap` — Wwise event-hash index:

```text
+0x00  char[4] "SMAP"
+0x04  u32     version = 1
+0x08  u32     entry count (1899)
+0x0c  u32     field (0xb)
+0x10  u32×n   **sorted** id/hash table — binary-search index
       …       opaque delta/varint-coded record data
```

`SndMap` in `anduril-bnk`. The id table is verified strictly monotonic
— it exists for O(log n) event lookup.

**Retail read path (Ghidra `FUN_141d2b488`, byte-verified):**
`{SMAP, ver==1}` exact, then **three** u32 sizes (not two) —
the loader bulk-reads `n1×u32`, `n2×u32`, and `n3×u64` arrays,
so the "opaque delta/varint data" is a second u32 table plus a
u64-element table (Wwise event id → event-ref pairs, consistent
with the AKPK event-id domain).
