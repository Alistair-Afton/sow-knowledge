# `.wldmsh` — MESH v5 world meshes

> **Status:** **proven**** — world-mesh geometry renders verified
> **Provenance:** real world-collision meshes; solid renders verified.

Chunked world-space geometry — sparse vertex/edge data used for world
bounds and collision. 47 files in the install (`worlds/`, `prefabs/`).
Unrelated to `MMSH` v17 (`.mesh` character geometry) despite the name.

## Declared-geometry interpretation (sow-rs — VERIFIED)

sow-rs decodes the file as a 0x20-byte header `"MESH", 5, 1,
submesh_count, decl_count, 16, size, 0` followed by **the same
`[vertex data][index data][declarations][descriptors]` block as PRFB
`.inst` embedded meshes** (see inst.md — identical element types,
52-byte descriptor records, min/base/StartIndex semantics). Under this
reading the file is the region's **merged render geometry** (rock,
cliffs, mountains — Minas Ithil: 17 submeshes, 4 spanning kilometres)
and its material indices select `.mat` names from the `.wldclnt` name
list in blob order.

Our older "region A / region B" decode below describes the **same
bytes through a different lens**: region A = vertex data (our
position runs + attr records = stream-0/1 blocks), the index pool +
attr groups + 52-B "chunk records" = the index data, vertex
declarations, and descriptor table. Keep both: the declared
interpretation is the authoritative semantics; the field observations
below still document real per-block structure (index pooling,
a_end-sharing, tail truncation) that complements it.

Corollary correction: the note "`.wldmsh` is collision, not visual
ground" in terrain.md is **wrong** — it is the region's merged render
geometry (though its verts sit above the prop cloud and it is not the
TERR heightfield).

## Layout (our chunked description — complementary field view)

```text
+0x00  char[4] "MESH"
+0x04  u32     version = 5
+0x08  u32     unk1 (always 1)
+0x0c  u32     chunk_count
+0x10  u32     unk10 (1..3 — likely LOD/group count)
+0x14  u32     unk14 (6..16)
+0x18  u32     payload_size (rest of file)
+0x1c  u32     0
+0x20  u32     chunk_count (duplicate)
+0x24  u32     region_a_size
+0x28  u32     region_b_size   (2 files under-report by 2 bytes — slice B to EOF)
+0x2c  …       header body: float vec3 soup (bound/AABB cells + outline
               polygon verts; record split not yet decoded)
hdr    u8[a_size]  region A: per-chunk {vec3 positions, packed edge
               records containing ffffffff marker, u16 index tail}
       u8[b_size]  region B (below)
```

`file_len - payload_size` = header end = region A start. Region B
follows A. Header body's vec3s are world-coordinate bound vertices
(e.g. desert AABB cells x≈5710–5734); `B`'s u16 index pool can index
past region-A positions into this vertex soup.

## Region B

```text
u16 index pool            (triangle list; size derived, no count field)
attr groups {             u32 group_count
  group_count × {         u32 n_attrs, n_attrs × u32 attr_code }
}
u32 chunk_count           (== header chunk_count)
chunk_count × 52B records (ends exactly at EOF)
```

Attr codes observed: `0x200, 0x10201, 0x40a01, 0x1040a01, 0x20201,
0x30201, 0x50401` — low u16 looks like a semantic id, high u16 an index.
Corpus: 1–4 groups per file, group sizes 2–7.

### Index pool semantics (verified on volcano)

- The pool is a flat u16 array forming a **triangle list** — consecutive
  triples share edges (e.g. `296,292,290 / 296,297,292 / 297,298,292`).
- Per chunk `i`: slice `pool[n3 .. n3 + n5*3]` — `n3` is a u16-index
  offset, `n5` is the chunk's **triangle count**. Verified: `n3` deltas
  equal the previous chunk's `n5 × 3` in `flag` order, all 27 volcano
  chunks.
- Indices are **block-local**: a pool value `v` addresses
  `block_vertex_base + v` where the base is the cumulative position
  count of all preceding region-A blocks (99.3% of volcano tris land on
  finite position data; small shared chunks index into a neighbour
  block's positions).
- `n4` tracks a cumulative vertex offset in a related but different
  space (chains of `n4 += count` run within flag-groups, overlapping
  other groups) — exact semantics open.
- Long **index ramps** live inside chunk windows and are *legitimate
  triangles*: `(i, i+1, i+2)` triples mean those triangles never share
  verts (consecutive-vert ordering — typical of cooked collision soup).
  Baraddur c0's window contains ramp runs covering local indices
  `0..1443` contiguously, followed by edge-shared tris up to `count−1`.
- Windows can **overflow the pool end**: baraddur c0 claims
  `[2418, 4530)` vs a 4432-u16 pool — the last 98 u16s (49 tris) are
  absent. `tris()` clamps to the pool rather than dropping the chunk;
  whether the missing tail is an encoding trick or intentional
  truncation stays open.
- Indices can overshoot the block's decoded position count by a small
  amount (~16 in baraddur: `count=1603` vs 1587 decoded verts) — the
  extras resolve into the *next* block's positions, producing a handful
  of cross-block smear triangles. Either the position prefix is
  occasionally 12-192B longer than marker-anchoring detects, or the
  overshoot is intentional.

## Chunk record (52 B = 13 × u32/f32)

| # | Semantics |
|---|-----------|
| 0 | unk (0; 3 once) |
| 1 | **record count** — region-A record count for the block (see below) |
| 2 | duplicate of #1 |
| 3 | `n3` — u16-index offset into the B pool |
| 4 | `n4` — cumulative vertex offset (open; chains within flag groups) |
| 5 | `n5` — triangle count in the pool |
| 6 | `flag` — block rank in ascending `a_end` order (equal values mark
      groups; also used for pool layout) |
| 7–9 | zero |
| 10 | **a_end** — end of this chunk's region-A block |
| 11 | f32 world-space bound (huge values, e.g. 11737.6) |
| 12 | f32 small value (0.25 … ~1e-7) |

Chunks with equal `a_end` **share** one block (e.g. baraddur chunks
0 and 2, volcano chunks 4 and 22 — identical `n3`/`n4`/`n5` too).

## Region A

Blocks are laid end-to-end in ascending `a_end` order. Each block:

```text
[lead 0..16B][V × vec3 positions][count × stride records][tail]
```

- `V` may equal `count` (volcano: every block is exactly
  `count×12 + count×stride`) or differ (baraddur: 1587 pos vs 1603 recs
  + 192B tail).
- Positions are world-space f32 vec3s (verified: volcano coords
  ~−173000, baraddur tower silhouette renders correctly).
- Record strides observed: **48, 44, 8, 4**.
  - 44/48B records carry a `ffffffff` marker at offset +40 plus
    unit-vector data (normals/edge attributes).
  - 8B records are `{u32 value, ffffffff}` or `{ffffffff, u32}` pairs.
  - 4B records are packed data (f16-pair-like values, `fefefe`-style).
- Record detection is **marker-anchored** (longest `ffffffff` grid at
  the stride); blocks without markers use the positions-first exact fit
  `span == count*12 + count*stride`. Blocks fitting neither expose the
  whole span as `verts` (`rec_stride == 0`) — treat as unresolved.
- `rasterize`-able output: `WldMesh::tris()` resolves the pool into
  global triangle indices over the concatenated block `verts`.

## Known variant

`prefabs/ui/worlduimenuws.wldmsh` (c=3, B=0x60) uses a different region-B
layout — no `{count, count×52B}` record tail. Currently rejected.

## Verified

46/47 files parse through the VFS; chunk count, attr-group walk, and the
52B-record run all land on exact boundaries (`attr groups end → count →
records → EOF`).

## Unknowns

- Header body record structure (bound polys/AABB cells).
- Chunk-record fields 0, 3, 4, 6–9 and the two trailing floats.
- Exact index-pool split into edge vs face indices per chunk.
- The u16 index space crossing A positions ↔ header bound verts.
