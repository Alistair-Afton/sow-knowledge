# GameDB intern-map structures — map / pool / name-element
> **Status:** **partial**** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC + partially VERIFIED-LIVE (elem fields
`+0x18`/`+0x01`/vecA stride confirmed on `0x4ef93708`, 2026-09-28)
**Evidence:** `docs/model/gamedb.md`; live dumps via
`sdk.debug.dbpool`/`dbname`.

## Map `{0x18 bytes}` at `manager + 0xa4d98 .. 0xa4f10`

| Off | Type | Name | Meaning |
|-----|------|------|---------|
| +0x00 | ptr | buckets | exact-sized bucket array |
| +0x08 | u32 | count | bucket count (= pool elem count) |
| +0x10 | ptr | pool | pool descriptor this map indexes |

Bucket = `{key@0 (elem ptr), payload@8}`; stride varies per map
(0x50/0x70/0x80/0x90/0xc0/0xe8 observed). No capacity field — resize
is exact realloc through the engine allocator (`0x141873430`).

## Pool descriptor (e.g. live `0x4ef93708` for `+a4e70`)

| Off | Type | Name | Meaning |
|-----|------|------|---------|
| +0x20 | ptr | domain-sel | static domain-selector cfg (`0x14264c720`) |
| +0x28 | u32 | count | vecA element count (loader seed bound) |
| +0x38 | ptr | vecA | 0x28-stride flag-set elems |
| +0x48 | ptr | vecB | 0x20-stride flag-clear elems (possibly biased base — see model) |

## Name element — vecA stride 0x28

| Off | Type | Name | Meaning |
|-----|------|------|---------|
| +0x00 | u64 | id | `{hi,lo}` key/id — NOT a cstr |
| +0x01 | u8 | flag | bit1 selects vecA(0x28)/vecB(0x20) index space |
| +0x08 | ptr | link | sibling-container link (0x14-spaced nodes) |
| +0x10 | ptr | link | sibling-container link |
| +0x18 | ptr | pool | back-ptr to owning pool descriptor — lookup validates `elem+0x18 == map+0x10` |
| +0x20 | ptr | payload | `.idata` static object (name content lives separately; string-table elems ARE their cstr) |

## Invariants

- `bucket[i].key == vecA[i]` after loader seeding; key pointers must
  be rebased on any vec realloc.
- `elem+0x18` must equal the map's `+0x10` or the lookup
  (advisory-logging) misses.
- `map+8` must cover `idx` or the lookup misses.

## Lifespan / ownership

Pool vecs + bucket arrays are engine-allocator owned; resized/freed
by engine paths only. Never store plugin-allocated pointers where a
later `0x141873430`-family call may free them.
