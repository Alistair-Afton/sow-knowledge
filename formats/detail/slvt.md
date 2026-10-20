# `SLVT` — multi-TEXR container inside `.tex`

> **Status:** **verified** — confirmed on the full `.tex` corpus by sow-rs
> **Provenance:** OnSight `formats/slvt.md`; one file decoded + full
> corpus scan (all 15 SLVT files). Andúril has no reader yet.

Some `.tex` entries are `SLVT` containers, not `TEXR` textures — a
container holding several embedded TEXR records (observed on
`prefabs/ui/worlduimenuws.hier.lsv_00color.tex`, 11,864 B, 3 entries).

## Layout

```text
+0x00  char[4]   "SLVT"
+0x04  u32       version = 4
+0x08  u32       entry_count
+0x0c  u64[count] entry offsets
```

Each offset points at an embedded `TEXR` record (each containing a
`DDS ` header at +40). Offsets are strictly increasing; the last record
runs to EOF. The table itself ends at the first entry's offset.

## Reader contract

Check the complete offset table, enforce monotonic entry boundaries,
and parse each bounded entry with the ordinary TEXR/DDS reader. On the
install corpus all 15 SLVT files parse this way; malformed entries
remain failures. Whether all SLVT variants use this exact table layout
is **OPEN** (single-version sample; the community ImHex patterns have
no SLVT entry).
