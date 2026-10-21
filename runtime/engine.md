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

Slot targets on the file stream (THaych): `0x146f59e40` release,
`0x141b50de8` read (forwards to an inner buffered-file object, vtable
`0x1422c1e58`, Win32 handle at `+8`; error byte at stream `+0x28`),
`0x1407f8410` error, `0x1407f8400` valid, `0x141b50ecc` seek,
`0x146f59240` tell, `0x141b50d5c` size (`GetFileSizeEx`),
`0x1411a363c` / `0x140503628` generic wide/narrow string readers,
`0x141b50d18` deleting dtor (frees 0x30 bytes), `0x141b50e4c` write.
File-stream ctor `0x141b50cd8`, open `0x141b50dbc`. The blob loader
uses `+0x30/+0x18/+0x28/+0x20/+0x08/+0x40`; the parsers only `+0x08`,
`+0x10`, `+0x40` — never stream fields.

**Loader prologues** (first 15 bytes; no RIP-relative operands, so
they relocate verbatim to a trampoline followed by
`FF 25 00 00 00 00 <abs64>`) — check before hooking (all CONFIRMED
byte-exact on the installed exe, file offset = RVA − `0xA00`):

| Function | Bytes |
|---|---|
| `0x140503694` GADB parser | `48 8b c4 48 89 58 10 4c 89 48 20 4c 89 40 18` |
| `0x1417662b0` GADB blob loader | `48 8b c4 4c 89 48 20 4c 89 40 18 48 89 50 10` |
| `0x141d23228` SKDB loader | `48 89 5c 24 18 55 56 57 41 56 41 57 48 8b ec` |
| `0x140504c70` BNDL loader | `48 8b c4 48 89 58 08 48 89 70 18 48 89 78 20` |

Wwise `GetIDFromString` begins `48 83 EC 28 33 C0 48 85 C9` and
immediately loads `0x811C9DC5` — the FNV offset basis (CONFIRMED),
matching the FNV-1 IDs in `formats/detail/bnk.md`.
Signatures: `PostEvent(u32 id, u64 obj, u32 flags, cb, cookie, ext,
u32 playingId)`, `SetState(u32 group, u32 state, …)`,
`SetSwitch(u32 group, u32 switch, u64 obj)`,
`SetRTPCValue(u32 id, float, u64 obj, …)`.

**Re-finding anchors after an update without Ghidra** (THaych's
method): disassemble the code section with `llvm-objdump`, take
function bounds from `.pdata` (12-byte `RUNTIME_FUNCTION` entries;
tiny thunks have none, and MSVC hot/cold splitting can jump into far
chunks), and grep for magic compares — `cmpl $0x42444147` (`GADB`),
`$0x42444b53` (`SKDB`), `$0x4c444e42` (`BNDL`) — or for `lea` users of
a string's VA. The blob loader is the function that calls stream
`+0x30` and then compares `GADB`. Note some real code lives in the
`.bss`-named section (~`0x146f5xxxx`); section map: code VA `0x1000`
→ raw `0x600`, read-only data/vtables `.xtext` `0x1d92000`, `.idata`
`0x25ac000`, `.pdata` `0x2aa5000`, `.bss` `0x2c89000` (raw
`0x28d6a00`); the `.bind` section is the Steam DRM stub. Build: final
Steam exe, 117,732,784 bytes, dated 2021-01-06.

**Third-party anchors** (from a mod menu's log on the same build, via
THaych; unverified here): LocalPlayer access `0x14051CAE6`, entity
access `0x140145569`, camera `0x1406E1FA0`, health write
`0x1402E4D29`, summoning `0x14060B2D4`, DebugMenu open/close
`0x141890924` / `0x14188B120`, NPC spawn context `0x141015560`,
captain world-spawn preload `0x140F6B7A0`.

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
The patched `bink2w64.dll` (imported by the exe) carries the string
`ShadowOfWarDllLoader.dll` and loads it at process start; the
loader's `DllMain` (entry `0x1800055b8`) starts a thread
(`0x180001000`) that enumerates `.\plugins` and `LoadLibraryA`s every
plugin once the loader lock frees, so plugins are resident long before the database load
(~8–10 s after launch) — early enough to hook the GADB/SKDB/BNDL
loaders (THaych: hooks installed at 0 ms in the plugin log). Loose
`.arch06` mods mount from the install root by wildcard. Details:
`docs/evidence/engine/initial-recon.md`, `modding-ecosystem.md`.
