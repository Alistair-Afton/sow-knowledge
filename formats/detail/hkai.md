# `.hkai` — HKAI v3 navmesh (Havok tagfile)

> **Status:** **partial** — tagfile structure decoded; query semantics open
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
(hkaiNavMesh serialization); inner decode is future work.

## Verified

444/444 files parse through the VFS. CLI: `hkai <path>` lists every
section with offset/size; `hkai scan` → 444 ok, 0 failed.

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

- **`ITEM`** — `n × 0xC` records `{u32 type_idx(flags|idx &0xffffff),
  u32 data_offset, u32}`; record 0 is skipped (null item). Each binds
  a type (via `TPTR[type_idx]`) to the object at `DATA + data_offset`.
- **`PTCH`** — u32 stream of `{u32 type_idx, u32 n, n×u32 offsets}`
  groups, bounds-checked ("Corrupt patch index body at index {}").
  Each offset points into `DATA`; the u32 slot there holds either
  `0` (patch to null) or an **ITEM index** (`slot×0xC` into the item
  table) resolved to that item's address — i.e. the pointer fixup
  table linking the object graph.

## Unknowns

- TBDY bytecode encoding — the gate to a full object walker (parsed
  lazily by the type index, not in the TYPE dispatch loop).
- ITEM record shape; THSH hash function; TCID/FRCT compendium
  resolution (which library the u64 ids reference).
