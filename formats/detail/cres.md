# `.cres` — CRES v5 (typed resource container)

> **Status:** **partial**** — envelope decoded; payload semantics vary by type
> **Provenance:** envelope decoded on the real `.cres` corpus.

Generic cooked-resource envelope used for cloth physics, lighting, and
sector streaming data. 122 files.

## Container header

```text
+0x00  char[4] "CRES"
+0x04  u32     version = 5
+0x08  char[4] type tag
+0x0c  u32     type schema version
+0x10  …       type-specific header + body
```

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
{u32 count, u32 stride, u32 0xa15bf781, count×stride record data}
```

`stride` is the record size in bytes — 8/12/16/20/24/32/36 observed.
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
