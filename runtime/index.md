# Runtime — index

> **Status:** **living** — regenerated as docs are added.

> **Provenance:** Andúril corpus index.

## Digests

- [`console-commands.md`](console-commands.md) — Console commands — the engine's `*.ConsolePrograms` system
- [`engine.md`](engine.md) — The Engine — canonical map
- [`gamedb.md`](gamedb.md) — GameDB — canonical model
- [`nemesis-memory.md`](nemesis-memory.md) — Nemesis memory — the CharacterEvent system
- [`scriptvm.md`](scriptvm.md) — Script VM — canonical model
- [`spawning.md`](spawning.md) — Spawning, summon templates, and NPC asset bundles
- [`ui.md`](ui.md) — UI model — Scaleform GFx + data-driven menus
- [`vfs.md`](vfs.md) — VFS & Asset Loading — canonical model

## functions/ — native function decodes

- [`functions/console-command_list_run.md`](functions/console-command_list_run.md) — 0x14077f560 — stored command-list runner
- [`functions/console-cvar_adapters.md`](functions/console-cvar_adapters.md) — cvar adapters — bound-command executor family
- [`functions/console-exec_record.md`](functions/console-exec_record.md) — 0x1415cfe88 — command record executor
- [`functions/console-exec_verb.md`](functions/console-exec_verb.md) — 0x1415cff44 — canonical console registry exec verb
- [`functions/console-name_resolve.md`](functions/console-name_resolve.md) — 0x14077f614 — console name/file resolver
- [`functions/gamedb-bucket_resize.md`](functions/gamedb-bucket_resize.md) — `0x141873430` — bucket-array exact-size resize
- [`functions/gamedb-map_lookup.md`](functions/gamedb-map_lookup.md) — `0x140189898` (family) — dense pool-indexed map lookup
- [`functions/gamedb-name_resolve.md`](functions/gamedb-name_resolve.md) — `0x140d23a40` — public name→record resolver (parsed-token input)
- [`functions/gamedb-payload_fill.md`](functions/gamedb-payload_fill.md) — `0x141872800` — `+a4e70` bucket payload populator
- [`functions/gamedb-record_resolve.md`](functions/gamedb-record_resolve.md) — `0x140189b50` — record-reference resolver (crash site)
- [`functions/gamedb-resolve_sweep.md`](functions/gamedb-resolve_sweep.md) — `0x140189950` (family) — reference-resolution sweep
