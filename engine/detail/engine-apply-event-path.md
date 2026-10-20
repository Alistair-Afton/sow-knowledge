# Spec: Native member-reward application path (binary evidence)

> **Status:** **verified** — member-reward write-path anchors located in binary
> **Provenance:** `x64/ShadowOfWar.exe` static analysis (custom
> xref/sigscan tools + Ghidra 12.1.4 project `C:\Tools\ghidra-proj\sow`),
> live in-game verification via nemesis.dll `callva` probes
> (journal 2026-09-28..10-04, `Project Nemesis/docs/model/
> nemesis-memory.md`). Addresses are build-8874 VAs.
> This spec names functions/addresses as *evidence anchors* — no
> decompiled code is reproduced here.

## ApplyEvent write path (VERIFIED-INGAME)

- `Faction.Reward` accessor-element application to a member:
  `callva(0x1403511f4, member, element, x, y)` → `0x140351af8` →
  `0x140351220` log tail. Args 3/4 ignored by the callee. Verified on
  a live roster member (`ResReward_Terror_Caragor`: ApplyEvent ran,
  `member+0x51c` notify flag set; `Reward_FromFlee` consumed the
  `member+0x2e0` container).
- The virtualized gameplay producer `0x140f7cadc` is byte-identical
  (`ApplyEvent` + `0x140351220` per element).
- **Persisted "events" ARE `Faction.Reward` definitions**
  (`Faction/Rewards/Definitions`): `Reward_(Fear|Terror|Hate)_*`,
  `EW_*` weaknesses, `Reward_FromFlee`, `Reward_PinnicleScar_*` scars.

## Pending queue vs committed log (VERIFIED-STATIC)

- `member+0x1c8` = pending event queue, 8-byte stride, walked by the
  drain `0x1407a6118` calling `0x1403511f4(member, *slot, 0, 0)`.
  Elements are pointers to live 0x28-byte event-record objects
  `{+0 id, +4 name-key, +8 items-array(16B groups), +0x10 bitmask,
  +0x18 ctx, +0x20 name cstr}`.
- `member+0x1f8` = bound EventList elements — holds `Default_Act1`,
  `Traits_Defender`, … — **VERIFIED as `levelchange_reward_list`**
  (the bound `RewardXPLevelSetup` bundle list; see field-name map).
- Committed log = per-type circular buffer written by `0x140351220`
  via `0x14035188c(member, wrapped-idx)`; counters `+0x470`/`+0x4b8`,
  dirty flag `+0x474`, picker-history vec `+0x1e0` (16-entry ring),
  applied vectors `+0x198`/`+0x1b0`.

## Resolution helpers (VERIFIED)

- `0x141b090bc(name-cstr)` — the engine's own name→record-element
  resolver (matches `sdk.db.entries` `e.addr`).
- `0x140423e70` namehash → `0x140278540(array, &key, count)` bsearch
  over `0x28`-stride accessor arrays (`count@+0x28`, `array@+0x38`).
  Arg order `(array, &key, count)` — swapped args crash at
  `exe+0x278560`.
- `0x14025180c` generic value→typedesc resolver; `0x141169594` accepts
  typedesc tag `+0xa&0x3f==3` — reusable "is a valid engine object"
  primitive.

## Class-registration table (VERIFIED-STATIC, this session)

`.idata` records, 0x40-byte entries `{type_name, vtbl?, class_name,
0,0,0, accessor_global, flags}`:

| class | string VA | accessor global |
|---|---|---|
| `Faction.RewardXPLevelSetup` | `0x141f54560` | **`0x14271f230`** |
| `Faction.RewardXPLevel` | `0x141f54580` | `0x14271f1e0` |
| `Faction.RewardList` | `0x141f54560`-region | `0x14271f280` |
| `FactionRewardPickerList` | `0x141f542e8` | `0x14295c5f0` |
| `Faction.XPLevelInit` | `0x141f563c0` | (record at `0x2668150`) |

The accessor globals are written at init (the only direct code refs,
at `0x8a0da1-0x8a0e71` / `0xa8d19-0xab549`); consumers reach them
through the bound table — the interpreter is findable via xrefs to the
*value* or via `member+0x1f8` writers/readers.

## Ghidra project findings (2026-10-19, `C:\Tools\ghidra-proj\sow`)

Headless query recipe (project already analyzed; `-noanalysis` opens
in seconds — do NOT re-import): `analyzeHeadless.bat C:\Tools\ghidra-proj
sow -process ShadowOfWar.exe -noanalysis -scriptPath Helper\ghidra-scripts
-postScript FindXprow.java <hexaddr>...` (MAXMEM=16G, JDK 21).
Java `.java` scripts only — `.py` needs PyGhidra. Script `println` output
goes to the INFO log stream.

- `0x14271f230` accessor: registered via `FUN_1408a0db8` →
  `FUN_140c6da00(&global,"Entries",0xb,...,"FactionRewardXPLevel",0xffffffff,6,&LAB_1411564fc)`.
  Only direct code refs are init-time (zero-init stub at `0x1400ab540`
  clears `0x14271f230..0x14271f270` (~0x48 B) then tail-jumps the
  accessor; the accessor's fn-ptr sits in a DATA table at
  `0x142b21668`). Consumers are fully indirect — **VERIFIED no direct
  consumer xref exists.**
- `FUN_14018de04(mgr+0x98, rec)` = keyed faction member-record lookup:
  validates index (`FUN_1401898e8`), bounds vs count, returns
  `base + 8 + idx*0x290` — **binary-confirms the live-walk `mgr+0x98`
  keyed table + 0x290 stride.**
- `FUN_14016e128(mgr, elem, x, y)` = resolve-element
  (`FUN_1401e5088`) → `mgr+0x98` lookup → `FUN_1401e4c88`
  element-indexed getter (u32 array `ctx+8..0x10` /4).
- `FUN_141c20820` iterates a `+0x18..+0x20` pointer array matching
  `item+0x190` (0x190) — a sub-record keyed lookup.
- Console `SetXPLevel` (`0x141c5ba9c`) is a DATA-table
  (`0x14267c9b8`) trampoline → `JMP FUN_141c5b5ec`, a 4-op console
  dispatcher resolving via `world+0x6c68`; op handlers funnel through
  `FUN_14016e128` / `FUN_141b5c314`→`FUN_141c20820` — **the console
  path does record lookup only, no level-application loop**.

## ApplyEvent `FUN_140351af8` — full decode (VERIFIED-STATIC)

The verified ApplyEvent callee is the event-item dispatcher. Per
event record it runs ordered item-category walks (each
`FUN_1407a*/FUN_14034*` iterator + `FUN_1402774a8` end-check):

- **Trait blocks gated by flag `0x26` DisableTraitChanges**: marker
  items → push to `+0x1b0` (`FUN_14079a1d4`); marker-match eval
  (`FUN_1407d3ea4(member+0x1b0, found-set)`); picker items → match
  `+0x1e0` picker ring, apply `FUN_140353454`, consume slot
  (`thunk_FUN_1407a7378`); trait-pick eval chain
  `FUN_1407dabfc`/`FUN_1406b34d8`/`FUN_1407d4890` → flag set
  `FUN_141156740(member,0x22)`.
- **RewardSetup items → push into `member+0x1f8`**: two item
  categories (`FUN_141bb73c0`, `FUN_141bb73a4` item getters) each
  find-if (`FUN_1405ece9c` / `FUN_1401b32a8`, `FUN_1405ee768`) then
  insert (`FUN_14019f830`, `FUN_14114c2f4`) — **events can add
  levelchange_reward_list bundles to a member at runtime**.
- `PendingRewardOnSpawn` push into `+0x1c8` (`FUN_140350490` item,
  param_4-gated, `FUN_14115ff60`).
- TraitLevel sync item: `FUN_140350424` → `member+0x4c0 = +0x4b8`.
- Other dispatched subsystems: menu/relationship (`FUN_141bb7910`,
  `FUN_141bb9100`, `FUN_141bc6698/60a0/6718/409c`, `FUN_1407bb8c4`),
  inventory (`+0x50`), interaction score (`+0x150` → dirty `+0x474`),
  faction-record fn (`+0x178` via `FUN_14034f6d0`), notify `+0x51c=1`.

**So ApplyEvent is a producer/populator, not the row interpreter** —
the `lo/hi/flag/prob` draw loop is elsewhere.

## `FUN_140f9bb20` — job-assignment processor (VERIFIED-STATIC)

3000-line promotion/job pipeline ("Process Job, Reject Job %s,
FactionMember %s — <predicate name>" rejection logs). Verified
predicate gates per candidate member `uVar18`:

- RowRank predicate (`FUN_140797118` on `member+0x268` record)
- PIS predicate (`FUN_14059516c` over a job list)
- Included/Excluded personality lists (`FUN_141bd24c4` vs
  `member+0x270` personality_record)
- **Level band: `local_248 <= (member+0x4b8 + member+0x470) <=
  local_268 − 1`** — i.e. jobs gate on `XPLevel + field+0x470`
  (a per-member bonus/offset term, likely the committed-log counter
  seen earlier at `+0x470`). Marked INFERRED for the `+0x470`
  semantics.

False positives recorded: `FUN_140f51e10` (unrelated +0x4b8 read),
`FUN_1415806ac` (Havok cloth solve — `0x1f8/0x200/0x4b8` are
physics-object fields, same offsets). Displacement scans hit any
same-shaped struct; co-field checks alone aren't enough.

## Member deserializer `FUN_141bc21b4` — the bind site (VERIFIED-STATIC)

Load-side twin of `FUN_141bc5044`: reads each named key from the
serialized member blob (`FUN_140507c38` key lookup +
`FUN_1406330b0`/`FUN_1407acf34` scalar/bool decode). Confirmed
bindings:

- `"levelchange_reward_list"` → per element `FUN_1411fd05c` then
  `FUN_1417a76f4(elem, DAT_142701c58, &out)` resolves to a record of
  accessor-type **`DAT_142701c58`**, pushed via
  `FUN_14114c2f4(member+0x1f8)`. **`DAT_142701c58` is the bound
  `RewardXPLevelSetup` record-type descriptor** — its other
  consumers are the draw-path readers.
- Sibling lists bind with their own type descriptors:
  `trait_marker_list`→`DAT_142701d38`, `trait_picker_list`→
  `DAT_142701d40` (+ `FUN_1407cd03c` insert), `faction_event_list`→
  `DAT_142701a20`, `relationship_list`→`FUN_1406379dc`.
- `"menace_level"` → `+0x4b8` — **the persistence name for XPLevel
  is `menace_level`** (save-format name is `XPLevel`; both map to
  the same field).
- Flag names → `FUN_141156740(member, bit, val)` — confirms the
  bit map; adds `is_player_follower`→`0x1b`, `is_spy`→`0x1c`.
- `FUN_141bb936c` = member copy/clone (field-copy + re-add of
  markers/pickers/events) — not the draw.

## Binding layer — fully mapped (VERIFIED-STATIC)

- `DAT_142701c58` @ `0x142701c58` = the **accessor global** for
  `Faction.RewardXPLevelSetup` records (`count@+0x28`, `0x28`-stride
  element array `@+0x38`, same shape as `0x14271f230` family).
- `FUN_140f81fe0(owner, name)` = generic name-hash→bsearch binder:
  resolves a record element in an accessor array and pushes into
  `owner+0x1f8` with manual growth — **reveals the vector layout:
  `{begin+0x1f8, end+0x200, capacity+0x208(u32), owns+0x20c(byte)}`**
  (elements are pointers into accessor element arrays).
- `FUN_141bc2d8c(member, srclist)` = list binder: iterates a source
  list (`FUN_1417a9528(src, DAT_142701c58)` per element) →
  `FUN_14114c2f4(member+0x1f8)` push_back.
- `FUN_141bc0f2c(member, src)` = the **spawn-time binder**:
  sequential `FUN_1417a9528(src, type)` record-field binds into
  `+0x168`..`+0x508` (incl. `+0x4b8` XPLevel, `+0x4c0` TraitLevel),
  plus list binders `FUN_141bc2c70`/`FUN_141bc2cfc` (marker/picker,
  wrapped in flag-0x26 save/restore), `FUN_141bc2d8c` (RewardSetup),
  `FUN_141bc2024` (relationships), `FUN_141c1115c` (inventory).
- `FUN_141b09bbc` = **persistence-schema registrar**: ~1000×
  `LEA field; CALL 0x14115b52c/0x14045a0e0/0x141c3fed0` triplets —
  registers every member field offset (incl. +0x1f8/0x200/0x4b8/
  0x4bc) for the serializer framework. Not the draw.

## RESOLVED: the XP-row draw loop (VERIFIED-STATIC)

Found via a full-binary store scan for `+0x4b8`
(`ScanWriters.java`): `FUN_14035188c(member, newLevel)` is the
**menace_level setter** — on change it stores the level then calls
`FUN_14034f9f0` (the draw), `FUN_140353c44` (level-flag sync +
`XPLevel` notification), `FUN_14061ba74` (inventory/gear level
rescale), then the warchief-present notify block.

`FUN_14034f9f0(member)`:

- `for level = member+0x4c0 (trait_level) ..= member+0x4b8`,
  capped by global `FUN_140f847a0`
- `FUN_1403513dc(member, member+0x1f8, level, out)` collects rows:
  per bound bundle element `FUN_140352dc4` — plus an exclusion
  list from `member+0x4f0` and a remap from
  `purpose_levelremap_record` (`member+0x280`, resolved via
  `FUN_141156510`)
- `FUN_140352dc4` per bundle slot, full decode (VERIFIED-STATIC):
  iterates the bundle's rows; per row —
  1. `FUN_140353154(row)` = the row's `Rewards/Lists` ref; skip if
     that ref is already in the output list (`FUN_1402d6fd4`
     search) — **cross-bundle dedup: the same list can't be drawn
     twice across bundle slots** (first occurrence wins).
  2. **SimEvent exclusion**: if the member's `+0x4f0` object has a
     participant entry (see below), each `Reward` ref in that
     entry's payload is compared against the row's list ref
     (`FUN_141bb9320`) — a match SKIPS the row. Confirmed
     blocklist semantics: participants in an active SimEvent get
     specific reward lists suppressed.
  3. Bounds: `FUN_140353084`/`FUN_140353014` (hi/lo getters) each
     return `stored`, then the code compares `param_2` (level)
     against `stored-1` on both sides — `lo-1 <= level <= hi-1`.
     When the row predicate `FUN_1403530f0` passes, BOTH bounds
     are remapped through `FUN_140352f8c` (the
     `TraitLevelOffset` table) before comparing.
  4. Probability `FUN_140353d54(row)`: passes when
     `fVar14 <= prob || FUN_1401b3c6c(fVar14) <= prob` where
     `fVar14 = DAT_14202e5a0` (a float seeded per draw loop —
     two chances per row).
  5. On pass, the row's list ref is pushed to `param_5` (the
     collection out-list via `FUN_14114c42c`) and the row loop
     breaks — first match wins per slot.
- per row: `FUN_14034f980` = pick count; `FUN_140350500` =
  weighted pick (log strings `[FACTION|REWARD] Picking Reward
  From List %s` / `Reward '%s' Passed Filter - Weight = %0.3f`);
  `FUN_140351af8(member, def, 0, 1)` applies
- commits `+0x4c0 = +0x4b8`; queued rewards then re-enter
  `FUN_140351220` (delta applies: `clamp(menace + delta, cap-1)`
  — the `ReduceLevel_*`/`IncreaseLevel_*` path)

`FUN_140350500` internals: `FUN_140350f80` list iterator,
`FUN_140105aec` def gate, `FUN_1403505f4` def weight,
`FUN_1401b3b90` reservoir pick.

## `member+0x4f0` — the member's active SimEvent instance (VERIFIED-STATIC)

The save serializer `FUN_141bc45a8` writes `member+0x4f0` under the
key **`"SIMEvent_NOTLOADEDFROMHERE"`** via `FUN_141bf05d8` — the
SimEvent object's own serializer names every field:

| offset | field |
|---|---|
| `+0x18`..`+0x20` | **`Participant[]`** — the 0x40-stride entry vector |
| `+0x30`/`+0x38` | `GUID_Part_1`/`GUID_Part_2` |
| `+0x40` | `Faction` record |
| `+0x48` | **`SimEvent`** record |
| `+0x50`/`+0x58` | `Quest`/`QuestOverride` records |
| `+0x60` | `SimEventGenerationType` record |
| `+0x68`/`+0x70` | `SubjectTraitPicker`/`SubjectTrait` records |
| `+0x80`/`+0x88` | `SequenceGroup`/`SequenceItem` records |
| `+0x94` | `SimEventID` (u32) |
| `+0x98` | `LifeSpanTimeStamp` |

Each `Participant` entry is 0x40 bytes (serializer
`FUN_141bf1114`): `+0x00` Participant ref, `+0x08` (unnamed ref),
`+0x18` **`Reward`** record ref, `+0x20` `FortUpgrade`,
`+0x28` `Faction` record, `+0x30` `FactionID` (-1 = non-faction
participant), `+0x38`/`+0x39`/`+0x3a` unnamed/`Column`/`Flags`.

`FUN_14034fb7c(obj, member)` / `FUN_140596bc4(obj, member)` find
the member's participant entry (key compare `FUN_14016e240`
against `entry+0x28` — the participant's faction/name identity);
miss returns `FUN_14034fbe9()` = NULL. The lookup has **eleven
callers** — a general purpose/event state surface; the XP row
collector `FUN_1403513dc` is the one relevant here: it takes the
member's participant entry's `+0x18` `Reward` ref and passes it
(as a small ≤8-entry context list via `FUN_141158264`) into
`FUN_140352dc4`, where matching reward LISTS are **excluded**
from row collection. So: **an active SimEvent can suppress
specific reward lists for its participants** — e.g. a captain
locked in a vendetta/rivalry event stops accruing certain
level-change rewards until the event resolves.

The object is **runtime-only** (never deserialized; the save key
literally says so) — created by the sim-event system when an
event involving this member activates. Bind-to-member at spawn:
`FUN_141bc2fd4` → `FUN_140596bc4`. No direct store to
`member+0x4f0` exists in .text (full-binary scan).

`FUN_140353ec4` is a level-change notifier on a wrapper object
(byte level at `+0x51a`): on level increase it updates the level
and pokes the `+0x4f0` object's internals (`+0x48`/`+0x50`)
via `FUN_1404fa9d4`/`FUN_1404fa558` — consistent with sim-event
state reacting to level ups.

Additional fields verified from `FUN_141bc45a8` (the save
serializer — complements the inspect-dumper map below):
`+0x4bc` LockedXPLevel, `+0x4c8` PlayerFleeCount,
`+0x500` ResurrectionCount, `+0x320` PreviousRole,
`+0x348` ResurrectionScarType, `+0x350` BuilderInstance,
`+0x370` AppliedScarType, `+0x3a0` InitialBuilderRecord,
`+0x2e0` HierarchyMenuOverrides, `+0x50` Inventory.

## OPEN

- What populates a participant's `Reward` exclusion refs — i.e.
  which SimEvent records/scripts carry the per-participant
  reward blocklist (the `+0x290` purpose-assignment path is the
  prime suspect; store scan `ScanWriters 290` in flight).
- **SimEvent producer — new lead (VERIFIED GameDB)**: `Faction/Rows`
  seat records each carry a `Faction/Events/SimEventGenerationSetup`
  ref (e.g. `MtnPass_Row_Captain_2` → `MtnPass_Captain`), which is a
  list of `Faction/Events/SimEventGenerationType` entries
  (`Hunting`, `Location`, `MP_Outpost_OP1/OP2`, …) — the board seat
  assigns each member its event-generation profile, the most likely
  path that activates the member's `+0x4f0` SimEvent instance. Rows
  also carry seat `RoleLists` (disallowed filters for
  Warchief/Overlord seats; draw pools `RolesWhoShouldNotBeLowLevel`
  on `*_Row_Captain_2`) and `_StructuresFactionInitialXPLevel`
  act-gated player-relative level bands.
- `DAT_14202e5a0`: the float seed driving the two-chance row
  probability test — where it is re-randomized (per draw loop?
  per level?) determines whether retail rolls are per-row
  independent or correlated.
- Which GameDB record selects the `PickerLevelRemap` table
  (`TraitLevelOffset_±{0,1,2}`); `Faction/RevengeType` records
  are empty name-only markers.
- `flag`/`prob` field indices inside the row record (the
  `FUN_140353014`/`FUN_140353084`/`FUN_1403530f0` item getters'
  element indices) — needed to confirm our `flag=0/1` split
  maps to retail's row-kind field.

## Member layout — from the serializer `FUN_140594788`

Verified member field offsets (serializer writes each field in order):

| offset | content |
|---|---|
| `+0x150` | sub-struct, 3× `{u32,u32,u32,pad,pad}` entries at `+0x10` |
| `+0x158` | vector of 16-byte elems `{qword, u32, byte, byte}` (>>4 count) |
| `+0x168`,`+0x170` | qword, u32 |
| `+0x174` | u32 (member id lo; `+0x178` hi — matches live dumps) |
| `+0x1b0`/`+0x1b8` | qword-ptr vector |
| **`+0x1f8`/`+0x200`** | **qword-ptr vector — bound record refs (the XPSetup bundle slot)** |
| `+0x240`/`+0x248` | vector of optional ptrs (null flag + payload 0x50/0x60/0x68) |
| `+0x300`/`+0x308` | fixed 32× 16-byte array |
| `+0x358`/`+0x360` | qword-ptr vector |
| `+0x1c8`/`+0x1d0` | qword-ptr vector |
| `+0x190` (0x190) | key field used by `FUN_141c20820` sub-record lookup |
| `+0x270`..`+0x400` | assorted qword refs, u32s, bytes (full map in serializer) |

## Member field-name map — from inspect dumper `FUN_141bc5044`

`FUN_141bc5044(member, writer)` serializes the member to a named-field
writer (`FUN_1411f783c` = write-named-field, `FUN_1417a6e48` =
record→string-id). Verified names ↔ offsets:

| offset | engine name |
|---|---|
| `+0x168` | `random_seed` (qword) |
| `+0x170` | `name_string_id_hash` |
| `+0x174` / `+0x178` | `faction_id` / `faction_record` |
| `+0x188` | `combat_tree_record` |
| `+0x190` | `preset_record` |
| `+0x1b0`/`+0x1b8` | **`trait_marker_list`** (ptr vector) |
| `+0x1e0`/`+0x1e8` | **`trait_picker_list`** (0x20-stride structs — the 16-entry picker ring) |
| `+0x1f8`/`+0x200` | **`levelchange_reward_list`** — the bound `RewardXPLevelSetup` bundles |
| `+0x240`/`+0x248` | `relationship_list` |
| `+0x270` | `personality_record` |
| `+0x278` | `character_record` |
| `+0x280` | `purpose_levelremap_record` |
| `+0x288` | `revengetype_record` |
| `+0x290` | `purpose_record` |
| `+0x2c0` | `last_deathtype_record` |
| `+0x2d0` | `boss_faction_id` |
| `+0x2d8` | `SMCStorage` |
| `+0x300`/`+0x308` | `faction_event_list` (0x10-stride) |
| `+0x318` | `role_record` |
| `+0x328` | `tribe_record` |
| `+0x330` | `location_record` |
| `+0x348`/`+0x370` | `resurrect_scartype_record` / `resurrect_applied_scartype_record` |
| `+0x3d0` | `npc_profile_guid` |
| `+0x400`/`+0x420` | `unpack_product_slug` / `unpack_guid` |
| `+0x468` | `npc_next_state_id` |
| `+0x4b8` | `menace_level` |
| `+0x4c0` | `trait_level` |
| `+0x4c4` | `player_kills` |
| `+0x4fc`/`+0x512` | `purpose_string_id` / `purpose_timeout` |
| `+0x514`/`+0x516` | `encounter_current_index` / `encounter_previous_index` |
| `+0x51a`/`+0x51b` | `interrogation_level` / `faction_col` |
| `+0x50` | `Inventory` sub-struct; `+0x150` player_interaction_score |
| — | `trait_base` = `FUN_141bbbd94(member)`; `is_total`/`is_warchief`/`is_spy`/`is_player_follower` = fns; `faction_row` = `FUN_1401e4d90` |

`FUN_14114b55c(member, bit)` = flag getter; bit map: `0`
is_resurrected, `2` is_dead, `3` killed_by_player, `4` can_resurrect,
`5` is_player_presented, `7` is_menu_presented, `0xc` has_vow, `0x11`
is_severed, `0x15` is_mockdead, `0x16` can_interrogate_about,
`0x18` is_revengetarget, `0x19` was_ever_preset, `0x29`
cant_change_xp_level.

**Key confirmation:** the `+0x1f8` vector serializes under the name
`levelchange_reward_list` — it IS the bound `RewardXPLevelSetup`
bundle list (`Default_Act1`, `Traits_Defender`, …), each element a
refcounted record-bound wrapper.

Member lifecycle (verified by decompile):

- **Ctor** `FUN_141bb4c1c` — installs vtable `PTR_FUN_141fd5ea0`,
  zeroes the whole ~`0x520`-byte object incl. the `+0x1f8` vector;
  `+0x174` and `+0x2a8`-region ids init to `0xffffffff`.
- **Dtor** `FUN_1401f786c` — walks `param_1[0x3f]` (= `+0x1f8`)
  releasing **each element** via `FUN_14012f0b0`, then frees the
  buffer. `FUN_14012f0b0` is the engine's generic intrusive
  refcount release (atomic hi-word count at `obj+0x10`; virtual
  destroy at `vtable+0x10` on zero; `FUN_14012f080` is the AddRef
  sibling): **the +0x1f8 vector holds refcounted objects** — bound
  record-bundle wrappers, not raw record pointers.
- `FUN_14014db34`/`FUN_1401f7808` use a same-shaped object for an
  unrelated system — `+0x1f8`-offset alone doesn't prove member.
