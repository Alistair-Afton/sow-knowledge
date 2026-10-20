# `.hkai` — HKAI v3 navmesh (Havok tagfile)

> **Status:** **verified** — tagfile structure, type schema, and the
> DATA object graph are fully decoded and verified on all 444 shipped
> files (815,651 typed items; navmesh arrays resolve with sane counts,
> e.g. island = 48,244 faces / 258,632 edges / 177,157 vertices).
> **Provenance:** Havok tagfile structure decoded on all 444 real `.hkai` files.

Navigation meshes per world/section. 444 files, 13 KB – 51 MB.

## HKAI header (0x14 B)

```text
+0x00  char[4] "HKAI"
+0x04  u32     version = 3
+0x08  u32     0xffffd00d (sentinel — also seen in skel cloth blobs)
+0x0c  u32     tagfile size, little-endian  == file_len - 0x14
+0x10  u32     tagfile size, big-endian    (same value byte-swapped)
+0x14  …       Havok TAG0 tagfile
```

**Retail read path (Ghidra `FUN_14048427c`, byte-verified):**
`{HKAI, ver==3}` exact, then the `0xffffd00d` sentinel and the
little-endian size are checked in code; the tagfile is handed to a
dedicated parser (`FUN_1404bc7f0`) and a **trailing `0xffffdead`
sentinel** must follow it (`return local_res8[0] == -0x2153`) —
so the file is `{HKAI,3,0xffffd00d,size,TAG0…,0xffffdead}`.

## Embedded tagfile

`TAG0` + `40 00 00 10` + `SDKV20160200` (SDK version). Block frames
are `{u32be size|flags, u32be tag}` — **big-endian**, top 2 bits =
flags, `& 0x3fffffff` = span including the 8-byte header (what earlier
looked like a `u16` was the low half of the big-endian size). Blocks
nest: the reader (`FUN_1405a3404`) walks headers inside each parent
span on a stack.

Top-level section dispatcher `FUN_1404bd5dc` accepts exactly:

| Tag | Meaning |
|-----|---------|
| SDKV | sdk version `20160200` |
| DATA | serialized object graph |
| INDX | item index |
| DICT | **TCID** — compendium type-id list `{0x20 hdr + n×u64}` referencing an external type library |
| TYPE | embedded type dictionary (nested blocks below) |
| FRCT | compendium reference (errors "COMPENDIUM_REFERENCE wrong size") |

`TYPE` iterates nested blocks (`hkTagfileReadFormat.cpp`): `TSTR`
type-name strings, `ASTR` a second string array, `FSTR` field-name
strings and `TBDY` packed type-descriptor bytecode (sub-reader taken,
parse deferred to the lazy type index), `THSH` type hashes, `TPTR`
zeroed type-pointer relocation slots (`span/8` u64s, all must be 0 or
"Buffer already inplace loaded"), plus `TNA1`/`TNAM`/`TSEQ`/`TPAD`.

Marker-scan histogram (all nesting levels flattened):

| Tag | Count | Notes |
|-----|-------|-------|
| SDKV | 444 | top-level |
| DATA | 339 | navmesh object payload (absent on 105 files) |
| TPTR/TSTR/FSTR/TBDY/THSH/TPAD | 444 each | TYPE-section children |
| ITEM | 437 | item block (absent on 7 files) |
| PTCH | 437 | patch/fixup offsets |
| FFFF | 4   | pad/fill marks on 1 file |
| VVVV | 4   | pad marks on 1 file |

Sequences are deterministic — `SDKV [DATA] TPTR TSTR FSTR TBDY
THSH TPAD [ITEM PTCH]` at marker level — zero false-positive markers
even in the 51 MB desert file. `DATA` is the navmesh proper
(hkaiNavMesh serialization) and is fully decodable: each `ITEM`
record binds `{type_idx, data_offset, count}` to a typed array of
objects in DATA, and DATA pointer slots hold u32 **item indices**
(0 = null) — on disk the object graph is already fully wired, no
fixup application needed to read it. `hkStringPtr`/`char*` slots
index a char item; `hkRefPtr`/`hkRefVariant` slots index the target
object item; `hkArray` members point at an item holding the element
objects strided by element byte size.

## Verified

444/444 shipped files parse cleanly through this structure —
zero false-positive section frames even in the 51 MB desert file.

## Serialization layer (Ghidra-verified)

The tagfile is **stock Havok 2016 serialization** — a leaked assert
path names `External\hk2016_2_0_r1\Source\Common\Base\Serialize\
hkSerialize.cpp` (`hkSerialize::Load::toVarInplace`). Chain:
`FUN_14048427c` (HKAI header + `0xffffdead` tail) →
`FUN_1404bc7f0` (blob read + decompress fallback) →
`FUN_1404bc8e4`/`FUN_1404bc9a0` (`toVarInplace`, reader vtable) →
vtable `+0x28` = `FUN_14482c2e0` → section dispatcher `FUN_1404bd5dc`
(errors cite `hkTagfileReadFormat.cpp`).

The type vocabulary is fully self-describing — 111 types on the
smallest file: `hkRootLevelContainer{NamedVariant}` → `hkaiNavMesh`,
`hkaiNavMesh::{Face,Edge}`, `hkaiAnnotatedStreamingSet`,
`hkaiStreamingSet::{NavMesh,Graph,Volume}Connection`,
`hkaiNavMeshClearanceCache*`, `hkaiNavMeshGenerationSettings*`,
`hkaiCarver`, `hkaiMaterialPainter`, `hkaiVolume`, plus Monolith
customs `CHkStringTable`, `CHavokAIObstacleData{Array}`,
`CHavokAIDynamicNavMeshPiece{Array}`, `CHavokAIUserEdgeData{Array}`,
`CHavokAIContextActionInfo`.

## INDX section (Ghidra-verified, `FUN_1405a2b54`)

Children `ITEM` + `PTCH`, read after TYPE populated the type-pointer
array at `reader+8`:

- **`ITEM`** — `n × 0xC` records `{u32 type_idx(&0xffffff) |
  isPtr<<28, u32 data_offset, u32 count}`; record 0 is skipped
  (null item). Binds `TPTR[type_idx]` to `count` objects at
  `DATA + data_offset`, strided by the type's byte size.
- **`PTCH`** — u32 stream of `{u32 type_idx, u32 n, n×u32 offsets}`
  groups, bounds-checked ("Corrupt patch index body at index {}").
  Each offset points into `DATA`; the u32 slot there holds either
  `0` (patch to null) or an **ITEM index** (`slot×0xC` into the item
  table) resolved to that item's address — i.e. the pointer fixup
  table linking the object graph.

## `hkaiNavMesh` layout (verified on real objects)

```text
Face (16 B)  i32 startEdgeIndex | i32 startUserEdgeIndex |
             i16 numEdges | i16 numUserEdges | i16 clusterIndex | u16 pad
             → boundary edges = edges[start .. start+numEdges]
Edge (20 B)  i32 a, b (vertex indices) | u32 oppositeEdge |
             u32 oppositeFace | u8 flags | u8 pad | f16 userEdgeCost
             → opposite* are hkaiPackedKey = plain indices in a
               single-section mesh; 0xffffffff = boundary
NavMesh      hkArray faces/edges/vertices/streamingSets/faceData/
             edgeData + stridings + aabb (hkAabb) + flags +
             clearanceCacheSeedingDataSet ref
```

`hkVector4f` stores xyz world position (Y-up) + unused w; faces are
convex polygons (triangles in practice); `userEdgeCost` is `hkHalf`
(f16, 0.0 in observed retail data). Face adjacency for path queries
comes from `edge.oppositeFace`; user edges are a parallel lane via
`startUserEdgeIndex`/`numUserEdges` into `CHavokAIUserEdgeData`.

## Unknowns

- THSH hash function (u32 per type).
- TCID/FRCT compendium resolution — which `TCM0` library blob the
  u64 ids address.
- Retail's runtime query internals (which Havok hkai calls the game
  issues, cost/shape filters, streaming-set traversal order) — the
  decoded structure supports face-graph pathfinding, but the engine's
  own query code path is unexplored.

## Packed integers

All TYPE/INDX metadata uses a 1–4 byte varint:

```text
0xxxxxxx                            → value = b0
10xxxxxx b1                         → value = (b0<<8 | b1) & 0x3fff
110xxxxx b1 b2(>H)                  → value = (b0<<16 | H) & 0x1fffff
111xxxxx b1 b2b3(>H)                → value = (b0<<24 | b1<<16 | H) & 0x7ffffff
```

## TYPE section grammar — decoded (verified on real files)

```text
TPTR  zeroed u64 slots — per-type pointer array, patched at load
TSTR  NUL-separated type-name pool
TNAM/TNA1  packed: n_types (types[0] = null); per type:
           name = TSTR[p], then n_templates × {name = TSTR[p], v = p}
           (template name 't…' → v is a type index, 'v…' → literal)
FSTR  NUL-separated member-name pool
TBDY  packed record stream; loop until section end:
        type_idx = p;  if 0 → skip (empty slot)
        parent   = p (type index)
        flags    = p   (TagFlag bitmask below)
        if flags&0x01: subTypeFlags = p
        if flags&0x02 and (subTypeFlags&0xf) >= 6: pointer = p (type idx)
        if flags&0x04: version = p
        if flags&0x08: byteSize = p; alignment = p
        if flags&0x10: abstractValue = p
        if flags&0x20: n = p;  n × {name = FSTR[p], flags = p,
                                   byteOffset = p, type = p}
        if flags&0x40: n = p;  n × {iface_type = p, offset = p}
THSH  packed: n;  n × {type_idx = p, hash = u32le}
TPAD  pad (optional)
```

TagFlag: `0x01` SubType · `0x02` Pointer · `0x04` Version ·
`0x08` ByteSize · `0x10` AbstractValue · `0x20` Members ·
`0x40` Interfaces · `0x80` reserved.

TagSubType (`subTypeFlags & 0xff`): `0` void · `1` invalid · `2` bool ·
`3` string · `4` int · `5` float · `6` pointer · `7` class ·
`8` array · `0x28` tuple (count = `subTypeFlags >> 8`).
Int width flags: `0x2000` i8 `0x4000` i16 `0x8000` i32 `0x10000` i64;
`0x200` signed.

Verified end-to-end: the smallest `.hkai` decodes **180 types** —
`hkaiNavMesh` (208 B; `faces, edges, vertices, streamingSets,
faceData, edgeData, faceDataStriding, edgeDataStriding`),
`Face` (16 B; `startEdgeIndex, startUserEdgeIndex, numEdges,
numUserEdges, clusterIndex, padding`), `Edge` (20 B; `a, b,
oppositeEdge, oppositeFace, flags, paddingByte, userEdgeCost`),
`hkArray {m_data, m_size, m_capacityAndFlags}`, `hkVector4f`,
`hkRefPtr`, `hkEnum`, plus Monolith `CHavokAI*` customs.

### In-memory view (Ghidra)

The loader expands TBDY into a pool of tagged records —
`{u64 tag_bits, u64 next_sibling, u64 values[popcount(tag_bits)]}`,
field n at `node+16+8·rank(bit)` — accessors `FUN_1405a23f0`,
`FUN_1405a222c`/`FUN_1405a37c4` (sibling chain, `&1` terminator).
The runtime type index is Havok `hkReflect` (assert path
`hkReflectTypeDetail.cpp`).
