# `.uis` — UISH v3 (world-section UI descriptor)

> **Status:** **verified**** — record decode; rendering semantics open
> **Provenance:** paired record decode across the real `.uis` corpus.

Paired with [`uiwd.md`](uiwd.md). Found under
`worlds/retail/<world>/sectionsandlayers/**/*.uis`; every `.uis` has a
sibling `.uiwd` with the **same `@0x08` entry count** (e.g. 117 for
`island`), so the pair describes the same region set, scene- and
data-side.

## Layout

```text
+0x00  char[4] "UISH"
+0x04  u32     version = 3
+0x08  u32     entry count N — always equals the tail-pool name count
+0x0c  u32     section offset/size field (opaque; correlates with body size)
+0x10  u32     N again
+0x14  …       header tail — zeroes, hash 0x0af33a4e, f32/AABB-like fields
       [body]  opaque binary records (float-dense; bounds/transforms?)
       N × (u16 len + str)   name pool — contiguous to EOF, no trailers
```

Names are region ids: `Island_RoadN_Execution`, `IS_Harbor_Supply`,
`IslandFort_Keep`, `mcOP_Shelob`, `Terrain_Low`, `MapCanvasCave_*`.

## Parser

`Uis::parse` exposes `count`, `region_field`, opaque `body`, and
`names`. The pool start is not stored in the header; it is recovered by
walking exactly `count` u16-length strings backward from EOF (chain of
`end == next.start`).

## Corpus

188/188 files parse; `count == names.len()` on all of them.

## Open

- Body record stride and semantics (bounds vs transforms vs UI quads).
- `@0x0c` field meaning; the gap between it and the pool is not a fixed
  stride of the record count.
