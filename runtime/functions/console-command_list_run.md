# 0x14077f560 — stored command-list runner
> **Status:** **partial** — Project Nemesis source. Andúril byte-check: the entry address lands on valid code, and **1/3** cited sibling addresses appear as direct `call rel32` targets inside it — call graph consistent. Instruction semantics remain Nemesis-only evidence.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC
**Addr:** `ShadowOfWar.exe+0x14077f560`   **Evidence:** disasm
2026-09-30; registered as executor `R+0x10` for stored-command records.

## Signature

`run_list(obj, str_or_ctx, argc, argv)` — same executor ABI as
`0x1415cfe88` invokes (`obj, args…`).

## Behavior

1. Resolves the record via `0x14077f614`.
2. Replays `R+0x10..+0x18` — a vector of per-line argument vectors
   (`{vec begin,end}` pairs; `argc = (q[1]-q[0])>>3` per line) —
   invoking a per-line executor callback stored at `[ctx]+8`.

## Callers / callees

- Invoked through `R+0x10` by `0x1415cfe88` for stored commands.
- Calls `0x14077f614`; the per-line callback `[ctx]+8` re-enters the
  executor for each captured line.

## Quirks

- This is the machinery a cfg/source-file loader would populate:
  each file line becomes one `{argv}` entry in the record's replay
  vector. The loader that builds `R` from file text lives in the
  console-subsystem init (`0x14117cf30` family, shell object at
  `mgr+0xac0`).

## OPEN / UNKNOWN

- Exact file-loader entry that parses cfg text into `R+0x10..+0x18`.
