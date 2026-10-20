# cvar adapters — bound-command executor family

> **Provenance:** Project Nemesis function-decode documentation (native `nemesis.dll` forensics), mirrored into the corpus.

> **Status:** **partial** — Project Nemesis source doc; per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE / VERIFIED-INGAME / MODEL / UNVERIFIED) preserved inline. Not independently verified by Andúril.

**Tag:** VERIFIED-STATIC (bool family VERIFIED-LIVE via console
GET/SET round-trip)
**Addr:** `0x14117d4cc` (bool tail), `0x141cb6554` (int variant,
storage `node+0x3c`), `0x141cad994` (bool variant, storage `node+0x39`,
forwards to `0x14117d4cc`), `0x141c8b704` (resolved-program variant)
**Evidence:** disasm 2026-09-30; live GET/SET via the Nemesis plugin API.

## Signature

`adapter(node, argc, argv)` — executor ABI; `node` is the runtime
command record / bound storage object.

## Behavior

Common convention: `id = min(argc, 0x40)`; dispatch on argv[0] —
empty/GET path reads the bound cell; SET path writes it. A
generation counter gates lazy rebind (record `+0x39`/`+0x3c` storage
mirrors the bound value when generation matches).

- `0x14117d4cc` — bool cvar tail: GET returns the bound bool, SET
  parses and stores.
- `0x141cad994` — bool variant storing directly at `node+0x39`.
- `0x141cb6554` — int cvar variant storing at `node+0x3c`,
  generation-gated mirror.
- `0x141c8b704` — resolved-program variant: forwards `[node+0x38](id,
  argv)` — the program slot resolves to the real handler.

## Callers / callees

- Registered as `R+0x10` executors on cvar-backed command records
  (~590 decoded command nodes; 21 "minor family" render/video
  commands are all adapter-B variants).
- `0x141cad994` tails into `0x14117d4cc`.

## Quirks

- In-node storage means these are callable without the lazy-binding
  path ever having run — safe for direct `R+0x10` invocation.
- Generation check uses a shared counter; forcing a stale generation
  re-runs the bind.

## OPEN / UNKNOWN

- Float/string adapter variants — same family likely, not yet located.
