# `0x140189950` (family) — reference-resolution sweep
> **Status:** **partial**** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC
**Addr:** `ShadowOfWar.exe+0x189950`; parallel sibling at
`0x14018ecxx`. **Evidence:** `docs/model/gamedb.md`; disasm.

## Signature

`f(manager, spec, ctx)` — `rcx`/`rdi` = GameDB manager, `rsi`/`rbp` =
spec object carrying name-element vec pairs, `r8`-chain = caller
context (forwarded to `0x189b50`).

## Behavior

Iterates spec name-element vectors and resolves each elem through the
manager's dense maps, then dispatches the payload to per-map
consumers:

| spec vec | map | lookup | consumer |
|----------|-----|--------|----------|
| `+0x10/+0x18` | `manager+0xa4dc8` (0x80) | `0x14018f164` | `0x14018eab4` |
| `+0x40/+0x48` | `manager+0xa4e70` (0x70) | `0x140189898` | `0x140189b50` |
| `+0x90/+0x98` | `manager+0xa4ea0` (0x50) | `0x140638370` | `0x14066adec` |

Find-or-fill per elem: lookup miss → `0x140d23e75` → `0x141872800`
populates the payload only if a bucket already exists; the eventual
`0x189b50(payload=null)` on persistent miss is the G41 crash site
(now guarded).

The parallel sweep `0x14018ecxx` adds the `+a4de0` (0x90) →
`0x140d18880` sub-elem fetch → `+a4e10` (0xc0) chain; a `+a4de0`
miss null-derefs inside `0x140d18880` — **second unguarded site**.

## Quirks

- Iteration uses `0x140189b40` (intrusive-list iterator over `rbx`
  node cursors).
- Early-exits per-map when a consumer returns false.
- Manager+0xa1158 deferred-list is rewound (`[rbx+8]=[rbx]`) on
  entry — sweep is the drain.

## Callers / callees

- Caller chain reaches `0x140d23a40` (public name→record resolve,
  parsed-token input) and load-time registration.
