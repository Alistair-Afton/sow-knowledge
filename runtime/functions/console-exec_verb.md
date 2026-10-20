# 0x1415cff44 — canonical console registry exec verb
> **Status:** **partial** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-LIVE
**Addr:** `ShadowOfWar.exe+0x1415cff44`   **Evidence:** registry vtable
`0x141ec6488` at `+0x58`; pipe invocation via `runtime/sdk/console.lua`;
disasm 2026-09-30.

## Signature

`exec(obj, str, argc, argv)`

- `rcx` — console registry object (`[0x142a58bc0]`).
- `rdx` — command line / name string.
- `r8d` — argc; `r9` — argv (string table).

## Behavior

1. `0x14077f614(obj, str, …)` resolves `str` to a runtime command
   record `R` (file paths return null — see `name_resolve.md`).
2. Passes `R` and `(argc, argv)` to `0x1415cfe88` which converts and
   invokes `R+0x10` (the record's executor thunk).

## Callers / callees

- Vtable slot `+0x58` on registry `0x141ec6488`; engine console sinks
  dispatch through it. Nemesis calls it from
  `runtime/sdk/console.lua` `exec`.
- Callees: `0x14077f614`, `0x1415cfe88`, plus dead-channel print
  plumbing (`0x14018bdb0`-family) used for diagnostics.

## Quirks

- Name path only — `*.cfg`-style file strings take a different caller
  branch after `0x14077f614` returns null.
- Live-verified on `Faction.Ambush.PrintDeferred`,
  `Faction.TriggerAmbush`, `Faction.Ambush.AlwaysPassChance`.

## OPEN / UNKNOWN

- Whether the file-path null return is consumed by a sibling exec-file
  verb on the same registry object or by a higher-level driver.
