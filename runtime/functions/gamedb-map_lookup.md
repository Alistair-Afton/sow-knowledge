# `0x140189898` (family) — dense pool-indexed map lookup
> **Status:** **partial** — Project Nemesis source. Andúril byte-check: the entry address lands on valid code, and **8/15** cited sibling addresses appear as direct `call rel32` targets inside it — call graph consistent. Instruction semantics remain Nemesis-only evidence.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC + VERIFIED-LIVE (callva-proven index math,
2026-09-28)
**Addr:** `ShadowOfWar.exe+0x189898` (0x70-stride variant); siblings
below. **Evidence:** `docs/model/gamedb.md` intern-map section;
`runtime/recon/intern_probe.lua`.

## Signature

`payload* f(map*, elem*)` — `rcx` = map `{+0 buckets, +8 count,
+0x10 pool}`, `rdx` = name-element pointer. Returns `&bucket[idx]+8`
on hit, `0` on miss. **Lookup-only — never inserts.**

## Behavior

1. `0x1401898e8(elem)` computes the pool index:
   - `pool = elem+0x18` (back-pointer); must equal `map+0x10`
     (mismatch → advisory `"Bad record index"` log via
     `0x14109e256`, still proceeds).
   - `byte[elem+1] bit1` set → `idx = (elem − pool+0x38)/0x28`
     (vecA, 0x28-stride elems)
   - clear → `idx = (elem − pool+0x48)/0x20` (vecB, 0x20-stride;
     `0x1898e8` uses a `/0x20` shift path `0x189910`).
2. Bounds: `idx >= map+8` → miss.
3. `bucket[idx].key == elem` → return `&bucket[idx]+8`, else miss.

## Sibling family (same shape, different bucket stride)

| fn | map (manager+) | bucket stride | consumer |
|----|----------------|---------------|----------|
| `0x14018f164` | `+a4dc8` | 0x80 | `0x14018eab4` |
| `0x140189898` | `+a4e70` | 0x70 | `0x140189b50` |
| `0x140638370` | `+a4ea0` | 0x50 | `0x14066adec` |
| `0x140d17110` | `+a4de0` | 0x90 | `0x140d18880` → `+a4e10` chain |
| `0x14018f110` | `+a4e10` | 0xc0 (`idx*3<<6`) | parallel sweep `0x14018ecxx` |
| `0x140189860` | (unidentified) | 0x40 | — |

## Quirks

- Buckets are `{key@0, payload@8}` — the returned payload pointer
  aliases `bucket+8`; payload size = stride − 8.
- The bucket array is exact-sized `{ptr,count}` — no capacity field.
- Index space is shared across flag classes when vecs are contiguous
  (biased-base hypothesis for vecB — `docs/model/gamedb.md`).

## Callers / callees

- Called from resolution sweeps (`0x189950` family), the public
  resolver `0x140d23a40`, payload populator `0x141872800`.
- Calls `0x1401898e8` (index), `0x140189920` (magic-div helper),
  `0x140189940` (flag read).

## OPEN / UNKNOWN

- vecB index bias model (three hypotheses in `docs/model/gamedb.md`).
- `0x140189860` 0x40-stride family's map/consumer.
