# Injection & Runtime — canonical model
> **Status:** **partial** — ported from Project Nemesis (`Status: **MODEL**. How `nemesis.dll` lives inside the game.`)

> **Provenance:** Project Nemesis runtime-model documentation (native `nemesis.dll` live-game forensics), mirrored and maintained in the corpus. Their `VERIFIED-STATIC`/`MODEL` tags carry over; unverified entries are marked inside each doc.


> Status: **MODEL**. How `nemesis.dll` lives inside the game.

## Chain

```
ShadowOfWar.exe
 → bink2w64.dll (patched proxy — pre-existing, not ours)
 → ShadowOfWarDllLoader.dll (Survivalzeed generic loader)
 → x64/plugins/*.dll  — nemesis.dll alongside mesow_patches, Elvenrun, RC
```

`nemesis.dll` is a sibling plugin in an existing ecosystem — we do not
own injection and should not try to; the loader chain is stable and
already battle-tested by the other mods.

## Lifecycle

1. `DllMain` → spawn host thread (no work in loader lock).
2. Host: `nemesis.ini` config, `Nemesis.log`, signature resolution
   (fail-closed per build), `gamethread` wndproc subclass, pipe server
   `\\.\pipe\nemesis`, overlay console (`console_key`, F8 reload).
3. `nemesis-lua` creates the plugin VM, loads `runtime/init.lua`
   (explicit module order, each `require` pcall-protected), then
   `Mods/**/*.lua` via the mod registry.
4. F8 → full runtime rebuild in the same VM; per-module load errors are
   logged and skipped, never fatal.

## Deployed layout

```
x64/plugins/nemesis.dll      ← target/release/nemesis-host.dll
x64/plugins/nemesis/         ← runtime/ tree (recursive, reconciled)
x64/plugins/nemesis.ini      ← config (created only if absent)
Mods/<name>/                 ← user mods, loose files
x64/plugins/Nemesis.log
```

`tools/deploy.ps1` reconciles the deployed runtime tree to match the
repo exactly (stale files removed), skips the DLL when the game holds a
lock, and only ships `runtime/recon/` with `-Recon`.

## IPC

`\\.\pipe\nemesis` — one command per connection, FIFO worker:

- `! <chunk>` — run in the plugin VM
- `=<expr>` — eval in the game VM (marshal path)
- `!f8` / `! __rt:F8()` — hot reload
- sdk commands registered by `runtime/sdk/cmd.lua`

Client: `python tools/nem.py pipe "<cmd>"` (or `tools/nemesis.ps1`).

## Failure isolation

- Unknown build → signatures fail → host logs and degrades, never
  guesses offsets.
- Lua module error → that module skipped, rest of runtime lives.
- Pipe flood → FIFO queue; heavy scans log to `Nemesis.log` when they
  drain (~3 s reply timeout ≠ dead command).
- DLL file-locked while game runs → deploy skips it; Lua/mods still
  hot-deploy.
