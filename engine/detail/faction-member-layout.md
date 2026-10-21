# Faction-member runtime layout — verified offsets

> **Status:** **verified-static** — field names from the engine's
> own member serializer `FUN_141bc45a8` (the save/load "Identity"/
> "Flags" blocks — it writes named keys, so the offset↔name binding
> is engine evidence, not inference); behavioral fields verified by
> their consumer functions.
> **Provenance:** `ShadowOfWar.exe` static disassembly (Ghidra),
> cross-checked against the member-domain consumers listed per row.
> Compiled 2026-10-22.

## Serializer-named fields (`FUN_141bc45a8`)

| Offset | Save key | Type |
|---|---|---|
| +0x150 | (container) | serialized sub-object |
| +0x158 | (container) | serialized sub-object |
| +0x168 | `RandomSeed` | u64 |
| +0x170 | `NameStringIDHash` | u32 |
| +0x174 | `FactionID` | u32 |
| +0x178 | `FactionRecord` | record ptr |
| +0x180 | `InitialFaction` | record ptr |
| +0x188 | `CombatTree` | record ptr |
| +0x190 | `PreSetRecord` | record ptr |
| +0x1c8/+0x1d0 | (vector, 0x8 stride) | serialized list |
| +0x1e0/+0x1e8 | (vector, 0x20 stride) | picker-history ring (16 entries, live-verified) |
| +0x1f8/+0x200 | (vector, 0x8 stride) | serialized list |
| +0x240/+0x248 | (vector, 0x8 stride) | serialized list |
| +0x260 | — | **stolen-item object** (`FUN_1404f2748`: when set the drop IS the item at `+0x38`, level `min(+0x24 + 5, 90)` or its own level with a `+0xb8` handle) |
| +0x268 | — | lookup key → `TraitLevelOffset_*` remap (`FUN_1401e4f54`) |
| +0x270 | `Personality` | record ptr |
| +0x278 | `CharacterModel` | record ptr |
| +0x280 | `PickerLevelRemap` | record ptr (`TraitLevelOffset_*` tier, remaps flag=1 XP-row bounds) |
| +0x288 | `RevengeType` | record ptr |
| +0x290 | `Purpose` | record ptr |
| +0x2c0 | `LastDeathType` | record ptr |
| +0x2d0 | `BossID` | u32 |
| +0x2d8 | (object) | serialized sub-object — equipped-item container (its elements' tags are harvested by `FUN_140646b40`) |
| +0x2e0 | (object) | serialized sub-object |
| +0x300/+0x308 | (vector, 0x10 stride) | serialized pairs |
| +0x318 | `Role` (name is a `DAT_` symbol, not a string) | record ptr — the member's CURRENT role |
| +0x320 | `PreviousRole` | record ptr |
| +0x328 | `Tribe` | record ptr |
| +0x330 | `Location` | record ptr |
| +0x338 | `LocationArea` | record ptr |
| +0x348 | `ResurrectionScarType` | record ptr |
| +0x350 | `BuilderInstance` | ptr |
| +0x358/+0x360 | (vector) | serialized list |
| +0x370 | `AppliedScarType` | record ptr |
| +0x3a0 | `InitialBuilderRecord` | record ptr |
| +0x498 | name string | inline/char* (used by `[LOOT]`/debug prints) |
| +0x4b8 | `XPLevel` | u32 — the member's level |
| +0x4bc | `LockedXPLevel` | u32 |
| +0x4c0 | `TraitLevel` | u32 |
| +0x4c4 | `PlayerKills` | u32 |
| +0x4c8 | `PlayerFleeCount` | u32 |
| +0x4f0 | — | purpose/event container ptr (0x40-stride keyed entries at `+0x18`, record key at `entry+0x28`, row blocklist at `entry+0x18`; `FUN_14034fb7c` reads it for XP-level draws; producer unresolved) |
| +0x4fc | `PurposeStringIDHash` | u32 |
| +0x500 | `ResurrectionCount` | u32 |
| +0x510 | — | flag byte (bit 3 gates the `+0x318`-list tag harvest — see below) |
| +0x512 | `PurposeTimeOut` | u16 |
| +0x514 | `CurEncounterIndex` | u16 |
| +0x516 | `PrevEncounterIndex` | u16 |
| +0x51a | `IntelDataLevel` | u8 |
| +0x51b | `ColumnNumber` | u8 |
| +0x520 | `InitializationType` | u8 |

## `+0x508` — the loot/mode bitmask (VERIFIED consumers)

- bit 12 → `Vow` boolean passed to `CalcLootRarity` (`FUN_1404f2748`
  arg → `FUN_14075a6f8`); the same bit the `MustHaveDeathThreat`
  table filter tests.
- bit 24 → `RevengeKill` boolean (same path).
- bit 29 → `FUN_140f87840` loot-mode selector returns `0xffffffff`
  (no setup fetched).
- bit 40 (`0x10000000000`) → `FUN_1404f2748` "has Loot disabled"
  early-out.
- bits 48/49/50 → setup-mode select 0/1/2 (first-set precedence;
  `FUN_140f84870` resolves the runtime `Global` setup field per
  mode — the slot→setup binding is runtime-resolved, not in the db).
- The `engine-spawn-pipeline` doc also maps bits 21/27/28/29 as
  eligibility flags from helper-plugin forensics.

## `+0x1b40` — pending role-select event

`FactionMember_RoleSelected_Notable` SimEvent object — the
role-selection path's pending marker (producer/consumer unresolved;
the RoleSelect consumer is the known gap behind role/seat
assignment weighting).

## Cross-checks

- `FUN_140646b40` (victim tag harvest for loot) reads `+0x318`
  (gated by `+0x510` bit3 — the role's tag list), `+0x328` (tribe
  record's tags), `+0x1b0/+0x198` nested lists, `+0x300` pairs,
  `+0x2c0`, `+0x48→+0x26a8`, `+0x50`, `+0x2d8` (equipped items).
- The row-number reader `FUN_1401e4d90` supplies `RowNumber`
  (board row lives outside the member blob; `ColumnNumber` is the
  inline byte at `+0x51b`).
