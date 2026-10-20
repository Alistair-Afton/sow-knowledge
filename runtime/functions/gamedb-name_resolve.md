# `0x140d23a40` — public name→record resolver (parsed-token input)
> **Status:** **partial**** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC (2026-09-28f)
**Addr:** `ShadowOfWar.exe+0x140d23a40`
**Evidence:** disasm; `docs/model/gamedb.md`.

## Signature

`f(manager, token, ctx, out, …)` — `rcx` = manager, `rdx` = a parsed
name **token** (NOT a raw cstr), `r8`/`r9` = ctx/out.

## Behavior

1. `0x140d18ed0(token)` → name-ish sub-object (early-out if `*p==0`).
2. `0x140d1a480(token)` → kind tag; dispatch 1/2/3.
3. Kind 3: `0x140d19e20(token)` fetches the bound name **elem** via
   the arg-descriptor pattern — global table `[0x14264c660]`
   (`+0x300` count gate, `+0x310` desc idx), then
   `token+0x10[desc+0xe]` → `0x140d1a160` tail through
   `[0x14264c520]`. The elem was interned at parse time.
4. `0x140189898(&manager+0xa4e70, elem)` → payload →
   `0x140189ec0` secondary fill (tag-dispatched, `0x18b944` table).

## Why it matters

No raw-string→elem bridge exists on this path — the elem is bound
inside the token by the (VM'd) parser/loader. `sdk.db.intern` must
therefore mint pool elements directly (template via
`sdk.debug.dbname`), not reuse this resolver.

## Callers / callees

- Called from `0x1402d0axx` region consumers (e.g. `0x1402d0afd`).
- Calls `0x140d18ed0`, `0x140d1a480`, `0x140d19e20`,
  `0x140d17110` (`+a4de0` holder lookup), `0x140189898`,
  `0x140189ec0`, `0x140190bc0`.
