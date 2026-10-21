# The Engine — canonical map
> **Status:** **partial** — Project Nemesis source. Spot-verified by Andúril: `CreateFactionMemberBuilder`, `FactionMemberBuilder`, `SimEvent` strings confirmed in `ShadowOfWar.exe`; the claimed `ArchivePatchLooseFiles.txt` string **byte-confirmed at exactly `0x141e6b1b0`** (title-cased on disk — a lowercase search misses it). All sampled code addresses begin with standard x64 function prologues, consistent with real function entries. Remaining internals are Nemesis-only evidence.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


> Status: **MODEL**. Subsystem map of Kraken (the engine) on build
> `0x5B7F5782`. Per-subsystem evidence lives under `docs/evidence/engine/`.

## Architecture sketch

```
ShadowOfWar.exe
├── VFS / archives      LTAR (.arch06) + loose tree + per-archive loose index
├── GameDB              static content db (gamedb.md)
├── WBScript            HavokScript VM + manager (scriptvm.md)
├── Nemesis system      Faction.* model, events, hierarchy — data + VM logic
├── Console/cvars       589 native command nodes, ~49 real cvars
└── Render/D3D11        overlay surface (Kraken_D3D11_C.pdb era symbols)
```

## Nemesis (the system we exist for)

- Data: ~1,300 `Faction.*` schema types — builders, traits, appearance,
  hierarchy, sim events, ambushes, rewards, contracts. Census:
  `docs/evidence/engine/faction-model.md`,
  `docs/evidence/gamedb/data-domains.md`.
- Captains are *constructed*, not spawned: `FactionMemberBuilder`
  records + `CreateFactionMemberBuilder` → hierarchy placement.
  Runtime path: `sdk.exec` + `DB*` lookups (see `runtime/sdk/spawn.lua`).
- Events: `Faction.SimEvent*` generation + resolution pipeline; engine
  event surface catalogued in `docs/evidence/engine/events.md` and
  `nemesis-events.md`.
- Roster debug dump format: `docs/evidence/engine/faction-dump-format.md`.

## Static anchors (read-only, verified)

| Anchor | Points to |
|--------|-----------|
| `0x1426ffaa8` | subsystem registry |
| `registry+0x6ce8` | script manager → `+0x58` lua_State |
| `registry+0x6db0` | script-state leak tracker |
| `.idata 0x270xxxx` | `DB*` closure slots (patchable data) |
| `0x141e6b1b0` | `archivepatchloosefiles.txt` string, used by the loose-index lookup at `0x140c28a57` |
| `0x142a58bc0` | console registry singleton — `vtable+0x60` node-reg, `+0x10` fn-reg (console-commands.md) |
| `manager+0xa4e70` | name→record intern map (find-or-insert `0x140189898`; gamedb.md) |
| `0x141c3fefc` → `0x141c3ff20` | `database\game\game.gamedb` open (+ `.gamedb_patch` sibling) |
| `0x1417662b0` | GADB whole-file blob loader (startup `game.gamedb` path) |
| `0x140503694` | GADB stream parser (`rdx` = optional patch stream) |
| `0x141d23228` | SKDB string-table loader (stream in `rdx`) |
| `0x140504c70` | BNDL loader (stream in `rdx`) |
| `0x1403ADBC0` | Wwise `GetIDFromString` — **don't detour**: callers rely on `r9`–`r11` surviving (THaych) |
| `0x1403AD54C` / `0x1412F4AEC` / `0x1406C27CC` / `0x1403AEBCC` | Wwise `PostEvent` / `SetState` / `SetSwitch` / `SetRTPCValue` (THaych) |

The four loader entry points each start with a 15-byte prologue
free of RIP-relative code, so they're cleanly detourable
(`jmp [rip+0]` + trampoline) — THaych's in-memory patcher hooks all
four and verifies the prologue bytes first.

**Engine stream interface** (vtable of the file stream class at
`0x142301620`; loaders call it generically — confirmed against the
GADB loader decompiles, which use `+0x08`, `+0x18`, `+0x20`,
`+0x28`, `+0x30`, `+0x40`): `+0x00` release, `+0x08`
`int read(buf, u32 n)` (0 = ok; on failure zero-fills and sets the
error flag), `+0x10` error flag, `+0x18` valid, `+0x20` seek (file
stream: to the given position — the blob loader rewinds to a saved
`tell`), `+0x28` `u64 tell`, `+0x30` `u64 size`, `+0x38`/`+0x40`
wide/narrow string readers (`u16` length + chars via `+0x08`),
`+0x48` deleting destructor, `+0x50` write. A plugin can feed any
loader an in-memory stream by implementing this table.

## Content layer

- ~225k `.bndl` bundles, ~190k `.embb`, 3.3k `.anix`, 2.7k scripts,
  305 `.bvr` behavior graphs, 10 `.gamedb` across archives
  (`extracted/archive_census.tsv`).
- STRDB string tables (SKDB v9, hash-indexed UTF-16): 63k enus strings.
- The exe is metadata-rich: 171 source paths, ~3.4k command strings,
  full schema names — mine it before disassembling blind
  (`nem strings`, `nem xref`, `docs/reference/exe-*.txt`).

## Existing mod ecosystem (the install we live inside)

Mod Menu (GENETICZZ2K) resolves 52 offsets at runtime; plugins load via
`bink2w64` proxy → `ShadowOfWarDllLoader` → `x64/plugins/*.dll`.
The loader's `DllMain` starts a thread that `LoadLibrary`s every
plugin, so plugins are resident long before the database load
(~8–10 s after launch) — early enough to hook the GADB/SKDB/BNDL
loaders (THaych: hooks installed at 0 ms in the plugin log). Loose
`.arch06` mods mount from the install root by wildcard. Details:
`docs/evidence/engine/initial-recon.md`, `modding-ecosystem.md`.
