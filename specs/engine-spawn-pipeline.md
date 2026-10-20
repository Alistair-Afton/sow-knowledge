# Engine spawn/lifecycle model — evidence from helper plugins

> **Status:** verified — plugin forensics + our own exe disassembly (see "Binary verification pass")
> **Provenance:** third-party `Helper/` plugin DLL forensics (RoamingCaptains/Elvenrun hooks in the
> live game) cross-checked against our own `ShadowOfWar.exe` disassembly; Project Nemesis
> signature maps (`extracted/roamingcaptains_sigs.tsv`). Compiled 2025; consolidated 2026-10-19.

The `Helper/` DLLs (third-party plugins, not ours) prove a concrete
actor materialization pipeline by hooking it in the live game.
Project Nemesis mapped their signatures
(`Project Nemesis/extracted/roamingcaptains_sigs.tsv`,
`../Project Nemesis/docs/ideas/roaming-captains-analysis.md`); this file is the
*model* distilled for our reimplementation — semantics, not
addresses. All claims are evidence-tagged by plugin behavior, not
re-verified by us.

## Materialization is a request queue, not a direct call

An actor is *requested*, not spawned:

1. A request node is constructed (`0x1a90560`-ish) and enqueued
   (`0x1a91360`). The queue lives in a native spawn manager.
2. The engine later materializes the request through a
   "physical-spawn" routine (`0x1a927f8`, request-node vtable+8)
   which returns the actor pointer — or `NULL` on failure.
3. A post-spawn finalize (`0x1aa20f4`) completes the link.
4. The faction **member record gains an actor pointer** — the
   "actor link" RC polls for ~5 s to confirm materialization
   (`actor->member resolver` at `0x18d920`, a `member+0x98` keyed
   map). Request success ≠ actor existing; the link is the
   ground truth.
5. Requests can be cancelled while still queued
   (`0x1a9118c`, `0x1b3678` unlink).

Design consequence for us: spawn APIs should be *request →
pending → materialized(actor link)*, not synchronous returns.
Our `sim` should model the queue so scripts/mods see the same
"waiting for actual actor link" semantics the plugins cope with.

## Placement goes through nav projection

The requested position is not used raw — it is projected onto
navmesh first (`nav projection (navobj, pos, out)` at
`0xf2bc80`; log: `AUTO native nav projection`). On projection
failure the raw target is queued anyway and "native
terrain/nav placement can shift the final actor".

## The member record (observed fields)

- actor pointer — via the `+0x98` keyed map resolver
- eligibility bitfield at `+0x508` (bits 12, 21, 27, 28, 29
  observed by RC eligibility checks)
- bool state words at `+0x30`, `+0x38`
- member-identity / bodyguard-master guard at `+0x2d0`
  (`0x40f5b8`: `cmp [rcx+0x2d0],-1`)
- spawn-component validity fields `+0x25a0`/`+0x2578`
  (`0x176f0c`)
- relation arrays at `+0x240`-ish (`member relation-array op`
  sites: bodyguard/party relations — "snapshotted live same-side
  Warchief/bodyguard relations")

## Roster + mission state

- Roster enumeration: faction manager + `NemesisRuntimeList`;
  the army-menu roster-card fill (`0x1c6eca`) is a second
  enumeration point.
- Standalone world-root path (NazgulSpawn/UndeadPatrols
  `STANDALONE_WORLD_ROOT`): world global at
  `ShadowOfWar.exe RVA 0x26FFAA8` (a second global at
  `0x0266CB70`), quest system at `+0x6C60`, mission-active byte
  `+0x599`, and the nemesis runtime list as a begin/end pair
  `+0x3D8`/`+0x3E0` (vector-style range walk).
- Mission-active detection: `QuestSystem+0x599` byte /
  `+0xdd66c0` query (`cmp byte [rcx+0x599],0; setz`). RC
  suspends new requests while set, keeps a 10 s post-mission
  stability window, and holds mission-busy captains 60 s so
  stale HUD targets clear — mission teardown visibly churns
  roster/lifecycle state.
- Wrong path warning: `+0xF6B7A0`/`+0xFCB3A0` create *garrison
  requests* (inventory), not world actors.

## Actor structure fields (observed offsets)

From the NazgulSpawn/UndeadPatrols diagnostic vocab (shared
codebase — identical diagnostic strings in both DLLs):

- actor flags `+0x1128`/`+0x1129`/`+0x112A`, state byte
  `+0x2EE0`
- `actorAI` `+0x26A8`, `actorModel` `+0x2B30`, `actorRoot`
  `+0x2A00`, owner pointer `+0x5A0` (verified equal to actor),
  subobject `+0x120`
- actor vtable read + compared as RVA (`actorVtRva`,
  `a1VtRva`) — plugins sanity-check vtables before trusting
  objects
- spawn-side: `SpawnFactionMemberMount`, `actorSpawnComponent`,
  `actorFollowerComponent`, `nativeMountDescriptor`,
  `nativePlayerSide` (allegiance check for friendly/enemy
  roaming split)
- a 15-qword window at `+0x29C0` scanned for descriptors
  (`startOff=0x29C0 qwordCount=15`)

## Spawn hook taxonomy

The plugins install up to three signature-verified detours:
**OUTER** (natural spawn observer), **FACTORY** (actor factory
trace), **POSTINIT** (post-spawn event). Each skips cleanly on
signature mismatch (`WARN: postinit signature mismatch` /
`factory entry not pristine`), and `FOREIGN_FACTORY_HOOK_PRESENT`
detects another plugin's detour — i.e. the community hook chain
is cooperative, not exclusive. Staged gates appear as
`V92_SETUP`, `V96_DISPATCH`, `V111_BUNDLE_GATE`,
`V116_BUNDLE_READY`, `V118_ANCHOR_*`, `V127_NATIVE_MISSION_READY`.

Request budgets: `nazgulMaximumSpawnPerRequest=1` (one native
call per actor), `undeadBatchMaximum=5` (4/5/6-soldier patrol
groups rotate), mission pause is hard-wired ON.

## Corpse/eviction constant

`PersistentBodies.dll` patches a single `mov eax, <limit>` at
`ShadowOfWar.exe+0x4053df` (vanilla `15`), signature-checked at
`+0x40544e` — the corpse-eviction cap is an inline immediate,
and cutscene/world-transition cleanup is a separate path.

## NazgulSpawn's pipeline vocabulary

Its diagnostic strings document the same pipeline at finer
grain: spawn **descriptors** (qword0-typed — `isildur` vs
`generic` descriptor pointers compared directly), a **resource
gate** (`WORLD_RESOURCE_GATE_MATCH`), a **factory** whose
callers are classified (`v102FactoryOuterContext`,
`ExpectedCaller`), **postinit events**, and a **lifecycle**
state read that goes stale (`naturalNazgulLifecycleStale`) —
i.e. descriptor → resource gate → factory → postinit →
lifecycle, with an anchor-selection pre-stage
(`V118_ANCHOR_*`: validated normal-Orc anchor vs experimental
placement).

## SharedExecutions

`SharedExecutions.arch06` is content, already mountable by our
VFS: paired synchronized-kill animation variants —
`*_sync_chord` / `*_sync_counter` / `*_se` ANIX clips for
olog/orc + Talion blade/dagger sets plus `nemesis_talion.bndl`
and `player_base.skel`. Direct evidence of the paired-sync
naming scheme executions use; useful when M7/M9 reaches
execution playback.

The DLL (not packed — strings legible) documents the swap
machinery in detail:

- Execution clips are grouped per character and replaced
  wholesale: `group of %d Talion moves: %d replaced, %d kept`,
  `clip %s not available on this %s; original used` — a
  fallback-per-clip model.
- Character selection runs through predicate conditions:
  `condition PC_IsBaranor %p -> %s %p` — conditions are
  replaceable objects keyed by predicate name.
- Sync playback is a two-actor component pair:
  `attacker sync at %d, victim sync at %d`, `stored variation
  %s+%x`, `playback setup: variation %p -> %p` — execution
  clips carry indexed attacker/victim sync slots and named
  variations.
- Equipment sockets are hash-keyed: `hand socket %08x ->
  c07ebcb6 (bow %p)`, `sheath %08x` — and the dagger mod
  tracks draw/sheath state to hold Talion's dagger during
  Eltariel/Baranor executions (`INI: 0=Mixed, 1=Talion only,
  2=Eltariel and Baranor only`).

## Other plugins in Helper/

- `PersistentBodies.dll` — inline byte patch (corpse cap 15→N).
- `Blacklist.dll` — PDB path `ShadowOfWarNoSlowdown.pdb`;
  reads `./plugins/blacklist.txt`. A name-blacklist filter —
  presumably blocks listed effects/animations.
- `PacketLoader/` — a find/replace binary patch format:
  `PLG1Packets/<mod>/{Find,Replace}/*.mesh` + `Config.ini`
  (`signature_check_length`, `mode = remove`). The
  first-person packet removes two `MMSH`-magic meshes — i.e.
  runtime asset deletion by signature match. Direct precedent
  for a "patch packets" mod layer in Andúril.
- `AggressiveGrunts.dll` — region→element table
  (Gorgoroth=Fire, Seregost=Ice, Nurnen=Poison,
  Cirith Ungol=Poison, Lithlad normal) applied through a
  guarded `Troll_Cave` selector shared by bait/cage/ambient/
  post-fast-travel Graug spawns; Talion's Call Graug excluded
  by exact identity.
- `SoW_RewardMultiplier`/`SoW_ArmorHealth` — simple scalar
  multiplier patches (XP ×2, mirian ×8, armor ×10).
- `Middle-earth ... Trainer (+37)` — Qt/asmjit CE trainer;
  confirms gameplay rules worth modeling: Iron Will blocks
  recruitment, recruit fails if captain level >> player,
  legendary captains drop level-80 legendary gear.
- `UndeadPatrols.ini.example` — same schema with different
  defaults.

## Binary verification pass (disassembly, verified)

Everything above came from plugin strings — third-party observation.
This section is what *we* verified by disassembling
`x64/ShadowOfWar.exe` (image base `0x140000000`, ~104 MB of code
across 7 executable sections) using `tools/exe_sigscan.py` (AOB
patterns extracted from the Helper plugins at runtime) and
`tools/exe_xref.py` (call-site + RIP-relative-reference scan).

All 10 embedded plugin signatures are UNIQUE in the exe, so each
hooked function is pinpointed exactly:

- **`AISpawnManager` exists and owns the queue** — the name string is
  referenced by the command-constructor function at `rva 0x1bb1468`
  which also embeds the command tag `"SummonNPCs"` (`0x1dcce00`,
  `0x2302f50`). Sibling constructors embed `"TalkAdvance"`/
  `"TalkClear"`/`"NPC"` — a command-tag dispatch table for AI ops.
- **Batch spawn loop confirmed**: `rva 0x1a93064` (Summon's workhorse,
  sole caller = the `SummonNPCs` constructor region) reads the spawn
  manager singleton at `rva 0x26ffbe8`, then loops `edi` times:
  allocate a `0x78`-byte request, initialize it (`rva 0x1a90634`),
  push it onto the manager's list at `+0x98` (`rva 0x1161ed4`).
- **Request record layout (0x78 bytes)**: `+0x00` vtable
  (`0x1422fe510`), `+0x08` intrusive list node (self-linked pair),
  `+0x18` spawn descriptor pointer, `+0x20` parameter/variant object
  (ctor `0x1407fd96c`), `+0x40` 0x28-byte spawn transform
  (two movups + movsd — position+quat+scale block; a helper at
  `rva 0x1a9068c` fills default/zero transforms), `+0x68` flags
  zeroed, `+0x70` owner/result pointer.
- **The `+0x98` queue is heavily trafficked**: the manager global at
  `0x26ffbe8` is referenced from ~50 sites, with a dense cluster at
  `rva 0x1a858xx`–`0x1a92xxx` — the `AISpawnManager` subsystem proper
  (request lifecycle, drain, materialization all live there).
- **`rva 0x1554ec`** (one hooked site) runs inside a system that
  dereferences `0x1426ffab0` — 24 bytes below the plugin-reported
  `0x26FFAA8` world global; consistent with a world object whose
  subsystem pointer sits at `+0x18`, then walks `+0xf58`. Virtual
  dispatch through `*+0x88` — a manager-call.
- **`rva 0x6fbda0`** constructs/destructs a subsystem record ending
  `+0x1c0`/`+0x1c8` with a TLS lookup (`0x1406fbe10` reads `gs:0x58`)
  — a per-thread spawn context.

Tooling: `tools/exe_sigscan.py` + `tools/exe_xref.py` are generic and
runtime-only (patterns/targets passed as args, nothing from the
binary persisted). Disasm scratch dumps live in `Helper/disasm/`
(gitignored, outside the repo payload).

## Live-memory verification pass (runtime, verified)

Everything in this section was read out of a **running** game
(pid-scoped `ReadProcessMemory` via `tools/procdump.py`,
`tools/roster_live.py`, `tools/spawn_watch.py`). No inference —
these are live structures observed in a loaded save.

### The nemesis roster is a keyed faction table — decoded

```
world  = [exe+0x26ffaa8]               (the "world root" global)
mgr    = [world + 0x6c68]              faction manager
arr    = mgr+0x98,  n = mgr+0xa0       keyed open-addr container
td     = mgr+0xa8                      map-owner tag
curkey = mgr+0x1b8                     current faction key
rec    = arr + i*0x290  (key @ rec+0, body = rec+8)
inner  = [body+0x20]  = {member-vec @ +0x18, end @ +0x20}
subid  = u32[body+0x168]              current-target index (-1 = none)
member = vec[j]                        board member object
```

- 38 keyed records; keys are faction-record elements —
  `Faction_Player`, `Faction_<region>`, `Real_Faction_*`,
  `InvasionFaction_*`. Key name = cstr at `key+0x20`.
- **Shared inner vecs**: all live `Faction_<region>` records share
  ONE member vec (511 slots — the union army pool); invasion
  factions share a 120-vec; `Faction_Demovolcano` has its own
  27-vec. Empty/quest-replay keys hold 0-length inners.
- **Seat model**: seat = position in the shared pool; ordering
  packs region blocks (hierarchy first — e.g. seats 0-11 =
  volcano's overlord+warchiefs+bodyguards).
- Actor→member resolver `0x14018d920(actor)`:
  `reg=[actor+0x2708]`, `+0x08` roster index, `+0x18` owner tag;
  engine fatals (`0x14109e110` = `mov [0],0`) on tag mismatch —
  the precondition contract matters if we ever call it.

### Member object — verified fields

| Offset | Contents |
|--------|----------|
| `+0x048` | actor link (0 when unspawned; verified by RC + the spawn-finalize path) |
| `+0x0a0` | gear-kit vec (`<Item>`/`AD_<Item>` element pairs) |
| `+0x110` | recent-events surface (`Active*`, `CharEvent` names — script `AddFactionEvent` writes here) |
| `+0x198` | applied resists/fears/weakens vec (`Resistant_*`/`Vulnerable_*`/`Immune_*`/`Fear_*`/`Hate_*` + quirks) |
| `+0x1b0` | applied `Marker_*`/`ClassMarker_*`/`PersonalityMarker_*` vec |
| `+0x1e0` | picker-history vec — `Picker_*[_Perm]` + outcome element pairs |
| `+0x170` u32 | **namehash of the captain's genkey name** (`OrcNameGen1st_NNN`/`OlogNameGen1st_NNN`) — resolved by hashing pooled genkeys |
| `+0x178` | **faction key object** → per-member region (name at `key+0x20`) |
| `+0x188` | element `0xa8a` |
| `+0x1c8/+0x1d0` | pending-event vec (8-byte event ptrs; push `0x14073b9dc`, drain `0x1407a61c4`) |
| `+0x1f8/+0x200` | committed-reader log — record elements `{key@+4, name cstr@+0x20}` = the member's trait defs (`Default`, `Default_Act1`, `Traits_*`, `BER_Hate`) |
| `+0x268` | **optional row-seat element** (id `0xa29`, ~half the roster). Name at `element+0x20` = `<Region>_Row_<Seat>` authored seat. NULL on procedurally filled members — the authored/procedural split is directly observable |
| `+0x270` | personality record-element array (id `0xa03`) |
| `+0x278` | combat-tree record-element array (id `0xa66`) |
| `+0x280` | `TraitLevelOffset` element (id `0xa01`, 479/511 members) |
| `+0x390` | rarity/predator element array (id `0xa06`) |
| `+0x470`,`+0x4b8` | persistent-log counters; **`+0x4b8` u32 = level** (FM→Actor copy loop: `FM+0x4B8` → `Actor+0x234`; `FM+0x170` → `Actor+0x23C`) |
| `+0x538` | `Tag_*` body/appearance tag vec |
| `+0x550..+0x5c8` | appearance-slot record-element arrays (gear: `Smp_*`, `Orc_*_Chest/Legs/Arms/Helmet/Body_LOD`) |

Validation rule (`looks_member`): `+0x1c8` and `+0x1f8` each either
`(0,0)` or a bounded ptr-vec whose non-null slots are readable
pointers; at least one populated. 562/562 live members pass.

### Live GameDB: name→record map + string pools

- The whole GameDB is resident: interned-name string pools (raw
  cstrs, e.g. `Row_Captain_*`, `OrcNameGen1st_*`) plus a sorted
  `0x28`-stride lookup map in a ~29 MB private arena:
  `+0x10` name cstr, `+0x18` = `(namehash<<32)|slot`,
  `+0x20` packed record payload — **key verified = `namehash(name)`**
  (the engine's `h*0x397 + fold[c]` hash; fold table recovered,
  mirrored as `anduril_strdb::name_hash`).
- Record payloads in memory are the same packed cell format our
  on-disk decoder already reads (`u64` key + `{cnt,ord,type}` u32
  cell headers). Live records decode with existing logic.
- The `+0x18` key's low dword is a slot/index; entries with
  aliased/duplicate names share the payload ptr.

### Corrections to plugin claims (verified, do not re-chase)

- `world+0x3D8/+0x3E0` is **not** the nemesis list — it is a
  `{ptr,hash64}` pair list of scene objects (destruction/force
  objects like `Wall_14`, `Tower_129`). The actual roster chain is
  `world+0x6c68` above. The plugins' "nemesis runtime list begin/end"
  claim applied to a different object/older layout.
- The `AISpawnManager` `+0x98` request queue is **demand-driven**:
  idle world state is reliably empty (3.1M samples @2ms, zero
  non-empty during idle play). Ambient/spawn traffic does not queue
  there; `SummonNPCs` requests land only on explicit summons.
- `tools/procdump.py` iter_regions previously stopped at the
  NULL-base free region — every early "0 hits" scan was the bug,
  now fixed and re-validated against known image strings.

### What this enables

- `tools/roster_live.py` dumps the live census
  (`Helper/roster_live.tsv`): per member — addr, pool, seat, actor,
  genkey name, level, faction, seat record, trait names. First
  parity-grade live dataset; 530/562 captains fully resolved.
- Static seat→spawn ordering is now observable: the board is the
  shared member pool, ordered as region-hierarchy blocks.

## What this means for Andúril

- `spawn` in our sim should be: create request → nav-project
  → materialize → set actor link on the member → emit
  `member_spawned`. Polling/pending semantics are authentic.
- Faction members need an `actor` field + eligibility
  bitfield + relation sets (bodyguard links both directions).
- Mission state is a queryable byte on the quest system —
  our `mission_busy` flag already exists; keep it readable.

## Andúril model

`anduril-sim` reproduces the materialization pipeline as a request
queue drained before tick listeners (promote/duel modes, seat
occupancy) — semantics only, not addresses. Rank/seat casting comes
from `Faction/RowRanks` refs on builders. Parity: hierarchy shape
checked in faction-parity census.
