# `0x141873430` — bucket-array exact-size resize
> **Status:** **partial** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC (2026-09-28)
**Addr:** `ShadowOfWar.exe+0x141873430`
**Evidence:** disasm; allocator chain `0x14114cc98`→`0x140602d20`→
`0x1401b0528`.

## Signature

`f(map*, u32 n)` — `rcx` = map `{+0 buckets, +8 count, +0x10 pool}`.

## Behavior

1. Reads `map+8` (old count) and `map+0` (old buckets); frees the old
   array via the engine allocator (`0x14178ab84`/`0x14178c598` free
   path).
2. `map+8 = n`.
3. `map+0 = 0x1401b0528(allocator, n*0x70, align=8)` where
   `allocator = *(0x1426fef88)` (singleton via `0x14114cc98`;
   fallback init `0x141113520`).
4. Initializes the allocation (memset-ish loop).
5. On alloc failure, `map+8 = 0`.

The stride `0x70` is baked to this map; sibling maps use their own
stride variants of the same idiom (`0x140d57f30` for the 0x90 map:
`n*0x90`, `pool->+0x28`-counted vecA seed).

## Why it matters

Any replicated intern insert MUST allocate through
`callva(0x1401b0528, [0x1426fef88], n*stride, 8)` — a plugin-owned
buffer would be `free`d by a later engine resize (dangling pointer
into our heap → crash).

## Callers / callees

- Called by the loader-time map builders (`0x141871xxx` family)
  inside `0x141872718` (temp-manager build, merge-swapped into the
  live manager by `0x14181ed58`).

## OPEN / UNKNOWN

- Whether any runtime (non-load) resize ever fires — watchpoints
  showed 0 mints in one session; assume load-time-only.
