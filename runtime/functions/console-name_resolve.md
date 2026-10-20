# 0x14077f614 — console name/file resolver
> **Status:** **partial** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


**Tag:** VERIFIED-STATIC
**Addr:** `ShadowOfWar.exe+0x14077f614`   **Evidence:** disasm
2026-09-30; called by exec verb `0x1415cff44`.

## Signature

`resolve(obj, str, flag) -> R*`

- `rcx` — registry object; `rdx` — name or file-path string.

## Behavior

1. `0x140188c80` file-existence check on `str`. If it exists as a
   file: `xor eax; ret` — returns **0** (file execution is a separate
   caller-side branch, not this function's job).
2. Otherwise resolves `str` through the registry map into a runtime
   command record `R`, validating `R+0x28` (stored name field).

## Callers / callees

- `0x1415cff44` exec verb; `0x14077f560` command-list runner.
- Calls `0x140188c80` (FileExists) and the registry map lookup.

## Quirks

- The null-on-file contract means `exec` never accidentally sources a
  file — whoever calls exec must branch on the file path first.

## OPEN / UNKNOWN

- `flag` arg semantics; which sibling/caller consumes the file branch.
