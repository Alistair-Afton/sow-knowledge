# `.uiwd` — UIWD v4 (world-section UI data)

> **Status:** **verified**** — record decode; rendering semantics open
> **Provenance:** paired record decode across the real `.uiwd` corpus.

Paired with [`uis.md`](uis.md) — same basename, same `@0x08` entry count.

## Layout

```text
+0x00  char[4] "UIWD"
+0x04  u32     version = 4
+0x08  u32     entry count (== paired .uis count)
+0x0c  u32     field (0/1)
+0x10  u32     offset-like field
+0x14  u32     field
+0x18  [body]  binary records interleaved with u16-len strings
```

Unlike `.uis`, names are **not** a contiguous tail pool: consecutive
u16-length strings are separated by 0/2/4-byte trailers (per-string
auxiliary fields), so `Uiwd::names` is a heuristic printable scan, not
an exact record walk.

Observed strings: region ids (`Island_Beast_Food`, `Island_01_Monument`),
slot/class names (`Ammo_Large`, `Ammo_Small`, `Default`, `mcCara`), map
layers (`MapCanvasCave_*`, `Terrain_Low`, `Terrain_Prp`,
`Terrain_Arc_Low`).

## Corpus

189/189 files parse; e.g. `island.uiwd` = 247 KiB, ~490 names.

## Open

- Record framing around each string (variable trailers 0–4B).
- Whether `@0x10` points at a sub-region inside the body.
