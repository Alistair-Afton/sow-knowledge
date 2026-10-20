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

`TAG0` + `40 00 00 10` + `SDKV20160200` (SDK version). Sections are
framed by markers `{0x40, 0x00, u16, tag[4]}` where tag = 4 uppercase
ASCII. The u16 does **not** encode span consistently across tags
(off-by-8), so section bodies are taken as marker→next-marker (or EOF).

Observed sections (corpus histogram):

| Tag | Count | Notes |
|-----|-------|-------|
| SDKV | 444 | tagfile header registers as first marker |
| DATA | 339 | navmesh binary payload — `hkai.data()` |
| TPTR | 444 | type pointers |
| TSTR | 444 | type strings |
| FSTR | 444 | ? strings |
| TBDY | 444 | type body |
| THSH | 444 | type hashes |
| TPAD | 444 | pad |
| ITEM | 437 | items (absent on 7 files) |
| PTCH | 437 | patch/fixups |
| FFFF | 4   | pad/fill marks on 1 file |
| VVVV | 4   | pad marks on 1 file |

Section sequences are deterministic — `SDKV [DATA] TPTR TSTR FSTR TBDY
THSH TPAD [ITEM PTCH]` — zero false-positive markers even in the 51 MB
desert file, so the scan is safe on this corpus. `DATA` is the navmesh
proper (hkaiNavMesh serialization); inner decode is future work.

## Verified

444/444 files parse through the VFS. CLI: `hkai <path>` lists every
section with offset/size; `hkai scan` → 444 ok, 0 failed.

## Unknowns

- Marker u16 semantics (per-tag span vs payload length).
- DATA inner structure (Havok hkaiNavMesh objects — vertices/faces/
  edges streaming sets; standard hk serialization, needs a full tagfile
  object walker to resolve).
- ITEM/PTCH fixup tables.
