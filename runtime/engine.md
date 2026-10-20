# The Engine — canonical map
> **Status:** **partial** — Project Nemesis source doc; not independently verified by Andúril.


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
`bink2w64` proxy → `ShadowOfWarDllLoader` → `x64/plugins/*.dll`. Loose
`.arch06` mods mount from the install root by wildcard. Details:
`docs/evidence/engine/initial-recon.md`, `modding-ecosystem.md`.
