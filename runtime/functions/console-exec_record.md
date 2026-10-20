# 0x1415cfe88 — command record executor
> **Status:** **partial** — Project Nemesis source. Andúril byte-check: the cited function addresses begin with standard x64 prologues (consistent with real entries); instruction-level decode remains Nemesis-only evidence.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC
**Addr:** `ShadowOfWar.exe+0x1415cfe88`   **Evidence:** disasm
2026-09-30; reached from exec verb `0x1415cff44`.

## Signature

`exec_record(obj, R, argc, argv)`

- `rcx` — console registry object.
- `rdx` — runtime command record `R` (executor thunk at `R+0x10`).
- `r8d` — argc; `r9` — argv string table.

## Behavior

- Converts each argv string with `0x1404dc114`, builds a converted
  argument vector, and calls the record's executor at `R+0x10`.
- No null check on `R` — callers must not pass a failed resolution.

## Callers / callees

- `0x1415cff44` (canonical exec verb).
- Calls `0x1404dc114` (argv conversion) and `R+0x10` (per-command
  executor — cvar adapter, code handler, or stored list runner).

## Quirks

- `R` is the *runtime* command record, not the static registration
  blob — the two are linked at registration time (see
  `docs/model/console-commands.md`).

## OPEN / UNKNOWN

- Exact `0x1404dc114` conversion semantics per argument type.
