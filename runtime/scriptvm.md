# Script VM — canonical model
> **Status:** **verified** — ported from Project Nemesis (`Status: **MODEL**. Everything here is `VERIFIED-LIVE` on build`)

> **Provenance:** Project Nemesis runtime-model documentation (native `nemesis.dll` live-game forensics), mirrored and maintained in the corpus. Their `VERIFIED-STATIC`/`MODEL` tags carry over; unverified entries are marked inside each doc.


> Status: **MODEL**. Everything here is `VERIFIED-LIVE` on build
> `0x5B7F5782` unless tagged otherwise. Layout/address archaeology:
> `docs/evidence/scriptvm/index.md`.

## What it is

The engine embeds **HavokScript** — Havok's Lua-5.1-ABI fork — wrapped
in a "WBScript" manager layer. ~1,400 generated binding stubs expose the
game API; 201 `DB*` globals expose GameDB record lookups. Retail scripts
ship as **precompiled bytecode**, not text.

## The two-VM architecture

```
nemesis.dll                          ShadowOfWar.exe
┌─────────────────┐                  ┌──────────────────────┐
│ plugin VM (mlua)│  \\.\pipe\nemesis │ game VM (HavokScript)│
│  sdk.* surface  │◄──── commands ───►│  System:*, DB*, Faction│
│  runtime/, mods │                  └──────────┬───────────┘
└─────────────────┘                             │ game thread
                                       gamethread queue (wndproc)
```

- **Plugin VM** — our own `mlua` state inside `nemesis.dll`. Runs
  `runtime/` + `Mods/*.lua` freely, any thread, stock semantics.
- **Game VM** — the engine's `lua_State`. Reached *only* through
  `sdk.exec.*`, which marshals work onto the script thread and never
  parses foreign text.

## The three safety laws (each cost us a crash)

1. **Game-thread only.** All game-VM calls go through `gamethread` — a
   `SetWindowLongPtrW` subclass on the game window that drains a job
   queue when `stack_depth() == 0`. Foreign-thread writes tear the
   shared VM (this was the ThreadFreeze crash class).
2. **Parse-free.** `dostring`/`lua_load` on foreign source crashes
   deterministically inside the lexer (`+0x819113` sentinel assert) —
   the retail parser was never exercised on text. Use
   `exec.call`/`callm`/`eval`/`fields` which resolve functions via
   `lua_getfield` and invoke the game's own `lua_pcall`. `pcall` does
   not protect against native AVs.
3. **Never patch `.arch`.** Three detour attempts died — the code
   section is self-verifying or consumed by a generated-code pass late
   in loading; *any* byte modification is detected asynchronously.
   Reach state through read-only pointer chains or data-section hooks
   (vtables, `.idata` closure slots).

## Reaching the state (read-only anchor chain)

```
*(u64*)0x1426ffaa8          → subsystem registry
*(u64*)(registry + 0x6ce8)  → script manager
*(u64*)(manager  + 0x58)    → lua_State (vm_create result)
L + 0x48 / +0x50            → stack top / base
L + 0x70                    → globals table TValue
```

Rust: `scriptvm::state()`. Lua: `sdk.exec.state()`, `sdk.exec.depth()`.

**Manager privacy (VERIFIED-STATIC 2026-09-30m):** a full `.arch`+`.xtext`
sweep finds **six** `world+0x6ce8` accesses — zero-init, three
init/serialize field-setter `lea`s, and the lifecycle pair
`0x1417acb90` (teardown: clears console `+0x6cf0`, `mgr->vt+0x28`) and
`0x1417acc1c` (create: `mgr->vt+0x20` → stores result at `+0x6cf0`,
deleting any previous via `vt+0`). Only the lifecycle sites *read* the
manager — everything else reaches the VM through `mgr+0x58` or a cached
pointer, so the manager singleton is effectively private.

## Working in the game VM

- `sdk.exec.call("System:GetPlayer")` — resolves via chained
  `lua_getfield` (O(1), follows `__index`), invokes `lua_pcall`.
- `sdk.exec.callm(recv, method, ...)` — receiver can be a returned
  `{tag,val}` handle → chained calls stay parse-free.
- `sdk.exec.fields(obj)` — enumeration for struct/table handles.
- Lua-side, `sdk.call`/`sdk.resolve` wrap results in `GameObj` —
  `obj:Method()` / `obj:get()` / `obj:fields()` (see
  `runtime/sdk/obj.lua`).
- `DB*` accessors are `WBDBRecordLookup` tables: `DBFactionMemberBuilder`
  `:get("Captain")` → record handle.

## Known failure signatures

| Symptom | Meaning |
|---------|---------|
| `+0x819113` mid-parse | parser sentinel pop — stop feeding it source |
| `+0x819113`, rcx→`"Script state was leaked somewhere."` | state-leak audit; tracked at `registry+0x6db0`. Not triggered by call/callm/fields — suspect `dostring` or 3rd-party menu |
| `WATCHDOG: main loop stale` | a mod tick/load handler is doing heavy work — profile the handler |
