# `0x141872800` — `+a4e70` bucket payload populator
> **Status:** **partial** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC (2026-09-28f)
**Addr:** `ShadowOfWar.exe+0x141872800`
**Evidence:** disasm; consumer in `0x140d23e75` find-or-fill tail.

## Signature

`f(ctx, elem)` — `rcx` = a context object whose `ctx+0xb0` **is the
target map** `{buckets,count,pool}` (callers pass `manager+0xa4dc0`
so `+0xb0` → `manager+0xa4e70`); `rdx` = pool element.

## Behavior

1. Runs `0x140189898(ctx+0xb0, elem)` — **early-returns on miss**
   (the bucket must already exist; this is NOT a seeder).
2. On hit, fills the `&bucket+8` payload (0x68 bytes) entirely from
   the elem via per-field accessors:
   - `+0x00 = elem` (redundant key copy)
   - `+0x08 = 0x14018bdb0(elem)`
   - `+0x10 = u32 tag` = `0x14186f744(elem)` — the domain-derived
     type tag that `0x189b50`/`0x189ec0` dispatch on (`0x15` =
     record). Never stored raw in the file.
   - `+0x18 = 0x14186f160(elem)`, `+0x20 = 0x14186f114(elem)`
   - `+0x28`/`+0x2c` = f32 `0x1418701dc`/`0x14187013c`
   - `+0x60 = byte` `0x141870e28(elem)`
   - loop `i < 0x141870dd8(elem)`: `sub = 0x141870d90(elem, i)` →
     `0x140188c80(sub)` filter → `0x141872b4c(&payload+0x48, &sub)`
     pushes the sub-elem span `0x189b50` iterates; when tag==0x74 a
     `0x1411549b0`/`0x141169ccc` side-list fills `payload+0x30`.

## Why it matters

Runtime interning does not need to replicate the payload contract:
seed `bucket[idx].key = elem` + zeroed payload, then call
`0x141872800` with a stack-built `ctx` (`ctx+0xb0` = the map) — the
engine derives every payload field itself.

## Quirks

- The `ctx+0xb0` indirection means the "map" is embedded in a larger
  object; a synthetic ctx whose `+0xb0` overlays the real map is a
  valid call.

## OPEN / UNKNOWN

- `0x141872b4c` span-push growth semantics (realloc? fixed cap?).
- `0x14018bdb0` / the `0x141870xxx` accessors' internal elem fields
  (elem layout beyond `+0x00/+0x01/+0x18/+0x20` — live template via
  `sdk.debug.dbname` pending).
