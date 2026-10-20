# `0x140189b50` — record-reference resolver (crash site)
> **Status:** **partial** — Project Nemesis source. Andúril byte-check: the entry address lands on valid code, and **5/12** cited sibling addresses appear as direct `call rel32` targets inside it — call graph consistent. Instruction semantics remain Nemesis-only evidence.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC + VERIFIED-LIVE (null-guard detour live
2026-09-28; 551K+ hits clean)
**Addr:** `ShadowOfWar.exe+0x189b50`
**Evidence:** dbwatch detour (`nemesis-sdk::dbwatch`);
`docs/model/gamedb.md`.

## Signature

`bool f(manager, payload, ctx)` — `rcx` = GameDB manager,
`rdx` = the `&bucket+8` payload from `0x140189898`, `r8` = sweep
context object.

## Behavior

1. `rbx = manager+0xa1158` — embedded deferred-list head;
   `[rbx+8] = [rbx]` (rewind for drain).
2. **CRASH SITE**: `cmp [payload+0x10], 0x15` at `0x189b79` —
   `payload` is null when `0x189898` missed → null+0x10 deref.
   The `0x189b50` detour returns `false` on `a2==0`, converting the
   crash into the sweep's graceful missing-record path.
3. Tag must be `0x15` (record-entry); else early-return.
4. `r8` → `0x140190cf0` vtable-tag downcast (`*[obj]==0x14054b100`,
   else `vt[0]` chain-walk) → `[casted+0x428]` must be non-null.
5. Iterates `payload+0x48 .. payload+0x50` (sub-elem span from
   `0x141872800`): `0x140188c80` filter → namehash each sub-name
   (`h*0x397 + fold[c]`, LUT `0x141db50d0`) → `0x140278540` bsearch
   into registry `0x142701d98` → `0x140350e90`/`0x140350e44` compare
   against `casted+0x428`.

## Quirks

- `r8` is supplied by the sweep's caller — not intern-owned.
- Guard covers ALL call paths (sweep `0x189950` family and the
  parallel `0x14018ecxx` sweep both call this).
- Second crash class: a `+a4de0` miss passes null into
  `0x140d18880` (`mov rbp,[rcx]` at `0x140d18890`) — unguarded.

## Callers / callees

- Called per spec-elem from the resolution sweeps after
  `0x140189898`/`0x140d23e75` fill.
