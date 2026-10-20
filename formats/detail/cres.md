# `.cres` — CRES v5 (typed resource container)

> **Status:** **partial** — envelope decoded; payload semantics vary by type
> **Provenance:** envelope decoded on the real `.cres` corpus.

Generic cooked-resource envelope used for cloth physics, lighting, and
sector streaming data. 122 files.

## Container header

```text
+0x00  char[4] "CRES"
+0x04  u32     version = 1..5 accepted (v5 in the corpus)
+0x08  char[4] type tag        (v>=3 only)
+0x0c  u32     type kind       must equal the tag's ordinal below
+0x10  …       type-specific header + body
```

**Retail read path (Ghidra `FUN_14072e4a8`, byte-verified):**

- `CRES` + version `1..5` accepted; the "type schema version" u32 is
  actually a **kind ordinal** checked against the tag:
  `none→0, SSTR/LOCT→1, GPUC→2, HAIR→3, PRTS→4` (mismatch → reject).
- An `SLVT` v4 **wrapper** also exists: `{SLVT, 4, u32, align-pad,
  u64 back-offset}` — the loader skips ahead then re-reads the
  embedded `CRES` header (versions 1–4 inside SLVT).
- v>=3: tag check is followed by a fixed **0x104-byte** type header;
  v>=4 adds a size-prefixed name blob; then `u32 record_count`.
- Per record: `v<2 → {u32 size>>2, stride 4, kind 1}`;
  `v2..4 → {u32 size, u32 count, u32 kind}`; `v5 → {u32 size,
  u32 count, u32 typeHash}` where `typeHash` resolves through the
  engine's vertex-element hash table: `Float32, UInt32, SInt32,
  UInt16x2, UInt8, UInt8x4, UNorm8, UNorm8x4, FloatR11G11B10,
  Float16x2, UNorm16x4, Float16x4, Float32x2, Float32x4,
  Structured` → kinds 0..13 (unknown hash → reject). The
  `0xa15bf781` "sentinel" in sentinel sections is one of these
  element-type hashes — not a marker.

## Type tags

| tag    | ver | files | contents |
|--------|-----|-------|----------|
| `PRTS` | 4   | 23    | cloth/flair particle sims (`bnnr*`, `*_ps.cres`) |
| `LOCT` | 1   | 52    | `*.lso_*` lighting/sector data |
| `SSTR` | 1   | 39    | `*.osc_*` sector/overlay streams (up to 34 MiB f32) |
| `GPUC` | 2   | 7     | `*.gpucloth` GPU-cloth resources |
| `HAIR` | 3   | 1     | `hairlookup_array.cres` hair table |

## Sentinel sections (PRTS / GPUC / HAIR)

Bodies are sequences of tagged record blocks:

```text
```text
{u32 count, u32 stride, u32 0xa15bf781, count×stride record data}
```

`0xa15bf781` is `name_hash("Structured")` — the v5 element-type
hash for raw byte-structured records, not a sentinel (verified
against the FOLD-table hash). `stride` is the record size in bytes — 8/12/16/20/24/32/36 observed.
PRTS example (`bnnrregalstnd01_red_fp`): 2×16B + 4×12B + 4×8B —
cloth-particle constraint triples (`{u32,u32,u32,f32}`), vec3f rest
positions, and pin/weight pairs. GPUC files carry 5 sections (e.g.
561×20B verts + 226×{32,24,12}B cloth params + 4×36B) with small
inter-section blobs. Sentinel framing validates on all 31
sentinel-bearing files; no false positives.

`LOCT`/`SSTR` bodies are dense f32 streams without sentinels —
`Cres::body` keeps them opaque.

## Corpus

122/122 parse. `Cres::section_data` borrows record slices zero-copy.
