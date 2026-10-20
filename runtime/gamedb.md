# GameDB — canonical model
> **Status:** **partial** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


> Status: **MODEL**. Verified live on build `0x5B7F5782` unless tagged.
> Deep layouts and decode history: `docs/evidence/gamedb/index.md`.

## What it is

The game's static database: `database\game/game.gamedb` (~58 MB, `GADB`
v17) inside the `Patch_*.arch06` archives. At runtime it becomes ~1,884
named lists / ~270k records — the entire content model: factions,
captains, quests, AI, loot, names, events.

## The model

- **Lists** are named registries (`Faction/Builder`,
  `AI/SpawnAmbient/AITypes`) each bound to a schema type
  (`AI.SpawnAmbientAIType`). Full census: `extracted/gamedb_lists.tsv`,
  `nem db lists`.
- **Records** are named entries in a list, hash-keyed
  (`OrcName_000_Akoth`). Strings are **interned objects**, not raw
  cstrs — the engine resolves them through a name→record intern map at
  `manager+0xa4e70`. This is why structural writes must go through the
  engine (see G41).
- **Attributes** are positional cells — `cell = data + index*8`; blocks
  are variable-sized and *sparse* (a cell prefix in attr-index order;
  attrs past the extent are simply absent = default).
  **CAVEAT (VERIFIED-INGAME 2026-09-30o, G50-adjacent):** this holds
  for flat scalar/ref records, but `Faction.AmbushDefinition` stores a
  **packed/reordered layout**: scalars first (`+00`=3 followers,
  `+08`=1.0f), then single refs (`+30` Job→`PC_MarkedTarget_AmbushJob`,
  `+40` EventPreSpawn→`Ambush_Presented`), then var-array element runs
  inline (`+68` ExcludedRowRanks×3, `+80` Results×5, `+a8`
  AmbushChance×4 — the `pklo=-1` attrs expand in place), then trailing
  refs (`+d8` CategoryList, `+e8` Buff). `attr.index` is the *declared
  ordinal*, not the physical slot — `sdk.db.get`/`cells` therefore
  mis-type values for these types (`Job` reads the Event, `Retry*`
  reads the RowRanks). The file's own cell order (see
  `extracted/gamedb_binary.jsonl`) matches the live block. The real
  index→offset map is the record's **hash-accelerator**
  (`entry+0x08` → 0x10-stride rows — VERIFIED-INGAME 2026-09-30o,
  all 22 attrs namehash-matched):
  ```
  accel row j (0x10) — doubles as prop-entry for declared attr j
  AND as the j-th hash-sorted bsearch index row:
    +0x00 u32  own attr's namehash     (this row = decl j)
    +0x04 u32  sorted-key namehash     (bsearch key)
    +0x09 u8   declared index of the +0x04 key's attr
    +0x0a u8   kind                    (1=bitfield, 0x0a=cell-ref,
                                      2/3/7 = scalar/struct flavors)
    +0x0c u16  element count           (array attrs; count=5 Results,
                                      count=4 AmbushChance — matches)
    +0x0e u16  cell_index              → cell = data + cell_index*8
  ```
  Verified physical map for `MarkedAsTarget_Ambush`:
  `Job`→cell6, `FactionEventOnSuccess`→cell8 (`Ambush_Presented` is
  the *OnSuccess* event, not PreSpawn — decl18 points at empty
  cell28), `ExcludedPlayerTags`→cell12 (`DisallowShowdown`),
  `ExcludedRowRanks`→cells13-15 (`Overlord`/`Warchief_*`),
  `Results`→cells16-20, `AmbushChance`→cells21-24,
  `ExcludedPlayerAttackCategoryList`→cell27 (`PC_DoNotAmbush`),
  `ExcludedPlayerBuff`→cell29 (`PC_Buff_GameChanger`).
  `kind==1` attrs bit-pack (`cells + (idx>>5)*8`, bit `idx&0x1f` —
  same as WBScriptProp `0x1402775c0`): `IfAmbusherProvidedValidateJob`
  bit0=1, `NeverPickAmbusherWithJob` bit1=1, `GlobalVarPredicate`
  bit2=0, `RetryEnabled` bit3=0 share cell 0 (=0b011). Layout is
  **per-record dynamic** — var-arrays expand in place, so trailing
  cell_indices shift with element counts (adjacent record's table
  shows the same keys with slots ±5). The correct accessor is
  therefore `accel[decl].cell_index`, NOT `attr.index`.
  **IMPLEMENTED (2026-09-30o):** `DbEntry::cell_attr`/`cell_addr`/
  `get`/`subentry`/`array` all resolve through `live_slot()`
  (accel `entry+0x08` → `meta>>48` = cell index) with positional
  `attr.index` fallback when the table is absent, the attr unkeyed,
  or the slot collides (ambiguous region — `AI.SpawnAmbientAIType`).
  kind==1 slots are read bit-wise (`get` returns `Int(bit)`); Lua
  `sdk.db.setcell` read-modify-writes the single bit and journals the
  logical 0/1 so reapply replays the same RMW. `sdk.db.cells` reports
  `slot`/`bit` per attr. Var-arrays resolve their accel (slot,count)
  as the inline element run; a pointer at the mapped slot must lead
  a homogeneous element run or the read falls through to the legacy
  container/header probes.
- **Type info** lives in a per-attr descriptor: `desc+0x08` declared
  type string (`"public <List.Type>"` = record ref, `"min=…;max=…"` =
  numeric constraint, bare name = enum/struct), `desc+0x28` packed
  `{class,count}`, `desc+0x30` storage tag.
- **Refs are typed by declaration, not cell bits** — a `public`-declared
  attr's cell is a pointer to the target's entry; `entry+0x18` → owning
  list. Records form a traversable graph.
- **Enums are statically extractable (VERIFIED-STATIC 2026-09-30l,
  G5)** — enum value tables are emitted by per-type registration
  functions (the `0x1410eb300` family): a repeating motif
  `lea rcx,[name]` → `mov slot.name` → `mov slot.ord` (imm) →
  `call namehash` → `mov slot.hash`. Value 0 is typically an
  implicit `and slot,0` zero-init ("None"). Type names register in
  an `.idata` table (`0x1426621xx` region, `{name*}` entries whose
  names live in `.xtext`); attr linkage is the `type=E*` declared-
  type string in the prop descriptor — not the attr's `+0x18` label.
  `tools/enum_tables.py` sweeps the motif across `.arch` →
  `extracted/enums.tsv` (387 groups / 2,458 rows after the
  slot-arithmetic pairing fix: `EAIWalkRun`, `ECommandType`,
  `EQuestDisplayType`, `FactionMember`…). A second corpus is the
  engine-authoritative registry walk (`runtime/recon/enum_bind.lua` →
  `extracted/enums_runtime.tsv`, 582 types / 2,650 rows) — correct
  name↔table binding but ~12% read artifacts (non-enum prop tables
  like `Modifier.*`, foreign-table reads collapsing to one value).
  `tools/merge_enums.py` merges both → `extracted/enums_merged.tsv`
  (948 types / 4,681 rows; per-row `src`: `runtime` |
  `runtime-suspect` | `static`). Runtime-clean rows win on name
  overlap (static table labels are proximity-paired and a few are
  wrong); static fills names runtime lacks; garbage runtime ords are
  never backfilled from static. `nem.py db enum` and
  `tools/gen_enums.py` (`runtime/data/enums.lua`) consume the merged
  corpus; `enum_N` unnamed extraction groups stay queryable but are
  not baked into Lua.

## Safe operations (verified)

| Op | Lua surface | Notes |
|----|-------------|-------|
| enumerate lists | `sdk.db.lists()` | world context ≈2,555 lists / 269k records |
| find/get record | `sdk.db.find(list,name)`, `sdk.db.get(list,i,attr)` | returns scalar / string / `{ref}` / `{array}` / `{obj}` |
| attr metadata | `sdk.db.attrs(list)`, `sdk.db.cells(list,i)` | name/index/tag/packed/default |
| **setcell** | `sdk.db.setcell` | the safe mutation — single atomic cell write |
| DB* accessors | `DBFactionMemberBuilder:get("Captain")` | engine's own record lookup, parse-free |

## Unsafe operations (G41 — gated)

`sdk.db.inject` / `sdk.db.append` splice raw strings into record
structures whose later resolution depends on the intern map. The engine
expects *interned name objects*; unregistered strings resolve to null →
deterministic crash at `ShadowOfWar.exe+0x189b79` on a worker thread.

- Both are behind `NEMESIS_UNSAFE_DB` — fail closed until the engine's
  own intern/register path is located.
- Adding a record correctly = the intern path + name registration, or
  cloning a donor record the engine already registered.
- `nem db` gives the same queries offline from the extracted corpora.

### Intern-map internals (VERIFIED-STATIC + VERIFIED-LIVE, 2026-09-28)

`0x140189898(map, name) -> &bucket->value` is a **lookup-only**
dense direct-index map — CORRECTED 2026-09-28 (earlier notes called
it find-or-insert; it is not):

- `key = 0x1401898e8(name)` = the element's **pool index**
  (`(name - pool_base)/stride`), compared to `map+8` capacity.
- Hit: returns `&bucket[8]` where `bucket = [map] + idx*0x70`,
  verified `bucket[0] == name`. Miss (`idx >= cap` or
  `bucket[0] != name`): returns **0** — no insert anywhere.
- Callers then fill the 0x68-byte value payload at `&bucket[8]`
  (see `0x141872800` filling `+0/+8/+0x10/+0x18/+0x20/+0x28/+0x2c/
  +0x60` relative to the value region).
- **The crash `exe+0x189b79` decoded (2026-09-28):** the
  registration path `0x140189950` calls
  `0x189898(&mgr->+a4e70, name)` → passes the result straight into
  `0x140189b50`, whose second instruction is
  `cmp [rdx+0x10], 0x15`. A miss yields rdx=0 → null-deref at
  `0x189b79`. So uninterned names crash because the *lookup* misses,
  not because any insert fails. For inject to be safe the
  `+a4e70` bucket must already be seeded (`bucket[idx] = elem`)
  before `0x189b50` runs.
- **`0x140189b50` decoded (VERIFIED-STATIC 2026-09-28):** post-lookup
  record resolver. `rcx`=manager → `rbx = manager+0xa1158` (embedded
  list head; `[rbx+8]=[rbx]` rewinds it — a deferred-work queue the
  manager drains). `rdx`=the `&bucket[8]` payload from `0x189898`;
  `[rdx+0x10]` must be `0x15` (record-entry type tag — the crash
  reads exactly this). Then `r8` → `0x140190cf0` → `rax+0x428`
  (attached object, non-null required) and it iterates the
  `[r14+0x48..r14+0x50)` range calling `0x140188c80` per elem;
  anchors `0x141db50d0`, `0x142701d98`. Confirms manager size
  ≥0xa1200 and that `+a4e70` bucket payloads are 0x15-tagged record
  entries with a `+0x48/+0x50` element span.
- Sibling lookups are the same shape: `0x14018f110` (+a4e10 map,
  stride 0xC0 = `idx*3<<6`), `0x14018f164` (+a4dc8, 0x80),
  `0x140d17110` (+a4de0, 0x90), `0x140189860` (an unidentified family
  map, stride 0x40 = `shl,6` — pool-check mismatch → `0x14109dea0`).
- The `elem+0x18 != map+0x10` mismatch path is **advisory**: it logs
  `"Bad record index"` (`0x141f68db0` via `0x140819104`, e.g. at
  `0x14109e256`) then falls through to the index path anyway.
- Payload-fill idiom (VERIFIED-STATIC at `0x141872800`, an
  `obj+0xb0` map): on hit, `[ret+0]=key copy`, `ret+8`/`+0x10`/
  `+0x18`/`+0x20`/`+0x28`/`+0x2c`/`+0x60` filled by per-field
  helpers (`0x14018bdb0`, `0x14186f744`, `0x14186f160`,
  `0x14186f114`, `0x1418701dc`, `0x14187013c`, `0x141870e28`,
  `0x141870dd8`) — the 0x68-byte value region after `&bucket[8]`.
- `0x1426ffe90` is **not** the gamedb manager — its `+a4e70`
  coincidentally has the `{buckets,cap,pool}` shape but buckets are
  stride 0x58 with exe-range code ptrs (`0x1426xxxxx`) — a different
  subsystem. The real gamedb manager remains a heap object reached
  through call-chain args, not a scanned global.
- Manager lifecycle: ctor `0x14181cfxx` zeroes `+a4e60..+a51xx`;
  `0x14181ed58` merges a temp manager (built by `0x14181ccac` /
  `0x141872718`) into the real one via per-map swap during load —
  the mint happens inside `0x141872718`'s call tree against the
  TEMP manager, then swaps in atomically. Destructor
  `0x14178bc50` frees the full map family: `+a4dc8, +a4de0, +a4df8,
  +a4e10, +a4e28, +a4e40, +a4e58, +a4e70, +a4e88, +a4ea0`.
- **Merge-swap fully decoded (VERIFIED-STATIC 2026-09-28)**:
  `0x14181ed58(rdi=live_manager)` — `0x14181ccac` ctors a
  stack-resident temp manager at `rsp+0x20`, `0x141872718` builds
  it, then **every `{buckets,n,pool}` triple is swapped
  field-by-field** (`live+X` ↔ `temp+(X-0xa4d90)`); the temp dtor
  then frees the *old* live maps. Temp↔live offset delta is
  `+0xa4d90`, which assigns each `0x141872718` loader to its live
  map:

  | loader | temp field | live field | map |
  |--------|-----------|-----------|-----|
  | `0x141871244` | +0x08 | +0xa4d98 | ? |
  | `0x1418711dc` | +0x20 | +0xa4db0 | ? |
  | `0x1418710a4` | +0x38 | +0xa4dc8 | descriptor map (0x80) |
  | `0x140d57f30` | +0x50 | +0xa4de0 | **name→holder** (0x90 buckets; seeds `bucket[i].key = pool->+0x38[i*0x28]`, n=`pool->+0x28` — its own pool) |
  | `0x141871174` | +0x68 | +0xa4df8 | ? |
  | `0x14187103c` | +0x80 | +0xa4e10 | typed-entry (0xC0) |
  | `0x1418712ac` | +0x98 | +0xa4ea0 | ? |
  | `0x14187110c` | +0xb0 | **+0xa4e70** | **name→record** (0x70 buckets — matches `0x140d23e75`/`0x189898`) |
  | `0x141871314` | +0xc8 | +0xa4e88 | ? |
  | `0x14187137c` | +0xe0 | +0xa4e28 | 0xE8-stride static-record vec (`{rec_vec@0,count@8,static_tbl@0x10}`) |
  | (unseen) | +0xf8 | +0xa4e40 | ? |
  | (via `0x14181cbac`) | ? | +0xa4e58 | special swap |

- **Bucket write idiom** (VERIFIED-STATIC `0x140d23e75`):
  `[rbp+0xa4e70]` = bucket array base; `index = 0x140d1b9b0(..)`
  → `&buckets + 8 + idx*0x70` passed to `0x140189ec0` (payload
  fill). `0x140d1b9b0` selects the domain through static selector
  object `0x14264c660` (`+0x2b8` count, `+0x2c8` selector byte →
  `[ctx+8]+sel*0x10` slot → `[ctx+0x10]+word[slot+0xe]*8`), then
  tail-calls `0x14181de7c` → `0x140311e8c(domain, sel, 0, &out)`
  under a TLS guard (`gs:[0x58]`, counter `0x142aa1320`) — the
  allocator/index authority.
- **Manager CAPTURED live** (2026-09-28): `0x41487480`, via the new
  `dbwatch` detour on `0x140189b50` (`rcx`=manager on every
  name→record resolution — 1165 hits at title, so the resolver runs
  constantly, not only at load). Verified fields:
  - `+0x30` = 0, `+0x38` = garbage at capture time → *read as a
    separate record manager — CORRECTED 2026-09-28*: `0x1402d0a80`
    calls `0x14114be28(rbx=manager, …)` on the **same** object whose
    `+0xa1100`/`+0xa4de0` fields it uses — one mega-manager owns BOTH
    the type-descriptor array (`+0x30`/`+0x38`) and the record map
    family. The captured `+0x30=0`/`+0x38`=garbage means the type
    array is populated lazily or the live dump raced it; not a
    second manager.
  - `+0xa1158` = `0x484fa8d0` — the deferred-list head `0x189b50`
    rewinds (`[rbx+8]=[rbx]`).
  - `+0xa4e70` = `{buckets=0x403bd180, n=2188, pool=0x4ef93708}` —
    the name→record map, VERIFIED end-to-end. Pool `0x4ef93708`:
    `+0x20`=`0x14264c720` static domain selector, `+0x28`=
    `{n=2188,cap=0x278}` packed, `+0x38`=`0x4f2e9bb0` (the flag-set
    0x50-stride element vec — `0x1898e8` index =
    `(elem-0x4f2e9bb0)/0x50`, and `buckets[i]` keys ARE those
    elements). The `0x1898e8` index math is now verified against a
    real pool, not a probe.
  - The map family is a run of **0x30-stride map records**, each
    holding two `{buckets@0, n@8, pool@0x10}` sub-triples (primary +
    secondary/shard). `+a4de0` sub-triple `{0x44f00010, 29709,
    0x4efb36e0}` = name→holder — the largest count (every name
    interns there). `n` qwords carry a `0x1bf7e00xxxxx` hi-tag on
    allocator-grown maps.
- **Manager remains unlocated** (2026-09-28). Eliminated live:
  `0x1426ffe90` (foreign subsystem), `0x4513ecd0` (the
  list→`+0x18` "domain context" — a 0x70-stride domain-record
  array; its `+0xa4dc0` region holds 0x20-stride
  `{count=4, hash_vec, elem_vec, pool}` units of a *different*,
  hash-accelerated map family sharing pool `0x45141a08`),
  `0x1426c6a60` (static IStringDatabaseMgr), `0x1426ffe00` (a
  ~128-entry table of *list entries* — `0x1426ffe10` → a
  0x58-stride list entry `[+0x28]` = count, same read as
  `0x141b0780c`'s count getter). `manager+0x30` = type-desc count (dword),
  `manager+0x38` = **inline** 0x4f8-stride type array
  (`0x14114be28` does `manager+0x38 + (id-0xF4240)*0x4f8`,
  validity byte at `elem+0x152`). The manager is a ≥0xa5000-byte
  heap mega-object.
- Confirmed live pool `0x45141a08`: flag-clear 0x20-stride element
  vector at `pool+0x48 = 0x451e3010`, every element's `+0x18`
  back-links to `0x45141a08` — the `0x1898e8` index math verified
  end-to-end live.
- **`0x1427019a0` static table** (the `+a4e70` loader arg): a live
  map family keyed on it exists live — objects of the shape
  `{q0, n@8, table@0x10, buckets@0x18, ..}` (e.g.
  `0x11be28448` `{1, 771, 0x1427019a0, 0x11f84cb30}`), where
  `buckets` is a **0x30-stride string-hash map**:
  `{key_ptr@0, packed@8, record_ptr@0x10, inline_cstr8@0x18,
  cstr_tail@0x20, hash/tag@0x28}` — observed keys `"pat"`,
  `"break"`, `"__bnot"`, `"end"`, `"bundles_for"`, `"home"`,
  `"Predicia"`. Keys are canonical string-table pointers +
  inline-copied cstr8; `record_ptr` is the mapped value. This is a
  *content-hash* family, distinct from the `0x189898`
  index-parallel family — which suggests `+a4e70` may be consumed
  by a different accessor than `0x189898` (or the map objects at
  `manager+a4e70` are `buckets` ptrs directly, matching
  `0x140d23e75`'s `[rbp+0xa4e70]` → `+8` → `idx*0x70` usage).
  UNRESOLVED: whether `0x11be28448` really is the `+a4e70` map —
  needs `manager+0xa4e70` ground truth once the manager is located.
- More `0x1427019a0`-keyed maps found live (2026-09-28):
  `0x1231abfe0` = `{buckets=0x11fa68050, n=582, tbl}` whose buckets
  map `"Orc_Defender"`-style names→record ptrs — a real
  name→record map but a standalone per-table auxiliary, not the
  manager field (`0x123107170` at `mgr-cand` is garbage). An 11-table
  cluster scan found ~8 heap objects referencing all 11 static
  tables (`0xd3aee270`, `0x11a45d5f0`, `0x11c3e2990`,
  `0x11c3e6040`, `0x11c712920`, `0x122e9d940`, `0x1231ac000`) —
  none at `mgr+0x30`-count/`+0x38`-desc alignment, so the manager's
  map fields point at per-table auxiliaries, not at the tables.
- **Two element classes confirmed live** (2026-09-28): the
  `+a4e70`-keyed name elements back-link `+0x18` → `0x4513ecd0`
  (domain ctx, NOT a `0x1898e8` pool — its `+0x38`/`+0x48` are
  0/garbage, so `0x189898`'s index math can't apply to them), while
  `0x4514d878` vec elements mix `+0x18`=`0x4513ecd0` with
  `+0x18`=`0x142647960` (a static `.bss` descriptor `{0x1428864b0,
  3, fn, fn, …}` — the static-table pool family). So `0x189898` is
  the `0x15`-tagged-record path; `0x4513ecd0`-backed elements ride a
  different accessor — reinforcing that `+a4e70` is consumed by
  `0x189898` only for *runtime-interned* (pool-indexable) names,
  with built-ins keyed via the static table.
- Static-table record entries (pre-relocation, `.idata`) are
  RVA-packed pairs `{rva24dbxxx, u32}` — the loader resolves them
  into runtime elements; `table+0x38` (populated at load) is the
  element/name array (`0x45143690` → `0x43afxxxx` cstr ptrs).

### Intern-map mechanics — full chain decoded (2026-09-30)

Map shape: `manager+0xa4e70` = `{buckets@0x00, cap@0x08,
pool_tbl@0x10}`. Buckets are 0x70-stride `{key@0 = name elem,
payload@0x08 (0x68 bytes)}`; `0x189898` returns `&bucket[idx]+8`.

Key/index function `0x1401898e8(name_elem)`:

```
if (!elem) return -1
pool = elem->+0x18                        ; MUST == map->+0x10
if (elem->+1 bit1 set):   idx = (elem - [pool+0x38]) / 0x28   ; magic-div
else:                     idx = (elem - [pool+0x48]) >> 5     ; /0x20
```

The pool is dual-vec: flag byte at `elem+0x01` bit1
(`0x140189940` reads `elem`, not the pool) selects the
0x28-stride vec at `pool+0x38` vs the 0x20-stride vec at `pool+0x48`.
For `+a4e70`, `map->+0x10` == `0x1427019a0` — a **static .idata pool
descriptor** `{+0x28 count, +0x38 vecA, +0x48 vecB}` whose vec bases
are populated once at init by the `0x141b09cxx` registration block
(name-keyed `0x14045a0e0` resolves, stored at `0x1427019d8` etc.).

Loader `0x14187110c(map, src_tbl)` (temp-manager side, temp+0xb0 →
live `+a4e70`):

```
map->+0x10 = src (0x1427019a0)
n = 0x140429708(src)                       ; src->+0x28 presumably
0x141873430(map, n)                        ; free old, alloc n*0x70
for i in 0..n:  [map->+0x00 + i*0x70] = 0x1401897fc(src, i)
                ; = src->+0x38[i*0x28]  — bucket key seed
```

Payload fill `0x141872800(obj, elem)` — on hit (`0x189898`):

```
payload->+0x00 = elem                       ; key copy
payload->+0x08 = 0x14018bdb0(elem)          ; flag&2 ? elem->+0x20 : 0
payload->+0x10 = 0x14186f744(elem)          ; 0x140311e8c domain-idx u32
                 (sel = cfg[0x14264c720]+0x40, i=0) — THE 0x15 "tag":
                 not a literal, the elem's domain-table index
payload->+0x18 = 0x14186f160(elem)          ; 0x1401cb3b0 qword fetch
                 (sel = cfg+0x88)
payload->+0x20 = 0x14186f114(elem)          ; same, sel = cfg+0x8c-ish
payload->+0x28/+0x2c = floats (0x1418701dc / 0x14187013c)
payload->+0x60 = byte (0x141870e28)
loop i < 0x141870dd8(elem):               ; sub-elem count
    e = 0x141870d90(elem, i)
    if (!isempty(e)):  push payload->+0x48 vec  ; THE +0x48/+0x50
                       ; span 0x189b50 iterates
    if (payload->+0x10 == 0x74):  payload->+0x30 vec append
```

`0x140311e8c(domain, sel, i)` = domain-index selector:
`sel_entry = domain->+8 + sel*0x10 {count@0xc, base@0xe}`;
returns `u32 domain->+0x10[base+i]`. `0x1401cb3b0` = qword variant.
`0x140189ec0` = secondary tag-dispatched fill (jump table
`0x14018b944`, tags 1..0x9a); tag 0x15 falls to the default tail —
record payloads are complete after `0x141872800`.

`0x140188c80(e)` = is-empty predicate (returns 1 iff `!e || *e==0`).

### Map family census (VERIFIED-LIVE 2026-10-02)

Live in-world capture, resolver object `0x40f6df98` (address varies per
boot — heap). The `+a4xxx` family is **per-schema**: each map interns
the names of ONE list. `pool+0x00` = list path, `pool+0x08` = schema
name, `pool+0x10` = source file, `pool+0x18` = back-ptr to the
**GameDatabase singleton** (`0x4ef85d40` this session — owns every
schema's pool), `pool+0x28` = tagged count, `pool+0x38` = elem vec
(0x28-stride, `elem+0x20` = name cstr).

| off | n | schema |
|-----|------|--------|
| +a4d98 | 18,510 | AudioHook.Key |
| +a4db0 | 29,616 | AudioHook.Hook |
| +a4dc8 | 3,345 | AudioHook.Condition |
| +a4de0 | 29,709 | AudioHook.Action |
| +a4df8 | 3,997 | AudioHook.Conversation |
| +a4e10 | 576 | AudioHook.Character |
| +a4e28 | 17,239 | Dialog.Event |
| +a4e40 | 4 | AudioHook.MixState |
| +a4e70 | 2,188 | **AudioHook.GameCondition** (the G41 map) |
| +a4e88 | 191 | AudioHook.Variable |
| +a4ea0 | 348 | AudioHook.VariableCondition |
| +a4eb8 | 281 | Character.Models |
| +a4ed0 | 2,715 | Model |
| +a4fe8 | 13 | SoundBank.StreamingGroup |

(+a4e58/+a4f00 slots hold non-map members — their `{buckets,n,pool}`
reads are garbage.)

Corrections to earlier claims:

- `+a4de0` is **AudioHook.Action** (`chant_*`, `dx_*` audio names),
  not "every name interns there". Faction.Event/DB record names are
  interned in their OWN pools (same 0xb0-stride pool descriptors, all
  owned by the GameDatabase singleton) but have **no map** on this
  resolver object — they resolve via `0x140278540` name-hash+bsearch.
- The captured object is the **resolver/index object**, not the
  GameDB manager: `0x189950`'s `rcx` arg; the GameDatabase singleton
  (`"GameDatabase"` at +0x00) is a separate object reached via
  `pool+0x18`.
- `dbwatch`'s `0x189b50` detour captures `rcx` on first hit — that IS
  this resolver object (all `+a4xxx` maps live on it), but it is not
  the pool owner.
- Donor names for `+a4e70` probes must be `AudioHook.GameCondition`
  names (`SM`, `RoleCategory_*`, `OrcName_*`/`OlogName_*` — the
  `sdk.names.add` schema). Faction.Event names like
  `Ambush_Presented` are legitimately absent — verified by a full
  census of all 14 maps (0 hits).
- **vecA/vecB are parallel duplicate arrays, both 0x28-stride** —
  `pool+0x38` and `pool+0x48` hold distinct elem objects carrying the
  same name cstrs. **Buckets key the `pool+0x48` array** (vecB):
  `bucket[i].key == vecB[i]` verified. The earlier "flag byte at
  elem+1 selects 0x28 vs 0x20 vec" decode is WRONG — byte+1 is part of
  the tag field (`0x????0a07`); there is no sharding. `dbname` now
  scans buckets directly (authoritative resolver-visible elems).
- Pool elem count exceeds map n at runtime (world-load interns more:
  vecB dense to idx 2363 while map n=2188 — the ~176 extra elems are
  unindexed). Tail slots hold non-elem tags; the mint site is
  `vecB + pool_count*0x28`.
- **Runtime mint is space-blocked in +a4e70's vecB** (VERIFIED-LIVE):
  vecB is packed flush to a foreign 0x28-stride structure at
  `vecB+2364*0x28` — zero headroom. vecA slack exists (zeroed slots
  past idx 2362) but is useless: `callva(0x189898, map, vecA[0])` → 0
  (miss) — **vecA elems are not resolvable keys; only vecB is**.
  Bucket keys are linear `vecB + i*0x28` for all 2188 slots (verified
  i=0..6). Growing vecB via realloc would orphan every live record
  ref holding vecB elem pointers — the engine can't do it either,
  which is why interning is load-time-only (the 176 extras filled
  pre-allocated slack during world load).
  **Consequence for `sdk.db.intern`**: name minting must happen on a
  pool vec with spare capacity, or the intern route is moot — the
  resolver-guard already makes uninterned names resolve "missing",
  and `names.add` records are name-resolvable via `0x278540` bsearch
  without interning. Intern is only needed if an uninterned name must
  resolve as a *reference target* through `0x189950`.
- `0x189950`'s three registration loops are now schema-named:
  `spec+0x10/0x18` refs → `+a4dc8` (AudioHook.Condition) via
  `0x18f164`; `spec+0x40/0x48` refs → `+a4e70` (GameCondition) via
  `0x189898` → `0x189b50`; `spec+0x90/0x98` refs → `+a4ea0`
  (VariableCondition) via `0x638370` → `0x66adec`. The sweep resolves
  a record's embedded audio-system refs — which is why `names.add`
  records (GameCondition-typed) crash when their name misses +a4e70.

### Resolver sweep + type registry (VERIFIED-STATIC 2026-09-28, cont.)

`0x140189950(manager, spec, r8, r9)` — the reference/eventmap
registration sweep — iterates **three** name-element vector pairs
embedded in `spec` (arg2):

| spec field pair | map | lookup | consumer |
|-----------------|-----|--------|----------|
| `+0x10`/`+0x18` | `manager+0xa4dc8` (0x80 buckets) | `0x14018f164` | `0x14018eab4` |
| `+0x40`/`+0x48` | `manager+0xa4e70` (0x70 buckets) | `0x140189898` | `0x140189b50` |
| `+0x90`/`+0x98` | `manager+0xa4ea0` (0x50 buckets) | `0x140638370` | `0x14066adec` |

All three maps are the same dense pool-indexed shape
(`{buckets@0, count@8, pool_desc@0x10}`, `bucket[idx].elem==elem`
verify) with different payload sizes — the spec's three name-vecs are
three *element classes* (descriptors / records / a third class)
registered in one sweep.

Each `elem = *it` is a **name object**, not a cstr: `0x189898` reads
`elem->+0x18` as the pool-descriptor back-link and dereferences it
(`pool->+0x38`/`+0x48` vec bases) — a raw string there crashes on the
pool deref, and a null bucket crashes at `0x189b79`
(`cmp [payload+0x10], 0x15` — the record-entry tag check).

`0x140d23a40(manager, cstr, ?, ?)` = the **public resolve-name→record
path** used at runtime (callers `0x1402d0b2e`, `0x140d20c2f`): resolves
a string through `mgr+0xa4de0` (name→holder via `0x140d17110`), then
`0x140d23df0`/`0x140d23a40`/`0x140d21da0`. Fires `0x189b50` → usable as
a **dbwatch trigger** for live manager capture.

Type-name registry (the `rbx+0x18` ctx consumed by the `0x141b0xxxx`
init blocks): `{+0x48 count, +0x50 elems[0x58], key = u32 elem->+0x50}`
binary-searched by `0x14045a0e0(reg, &hash32)` → returns the 0x58
**type descriptor** or 0. `0x14115b52c(&h, str)` =
`h = 0x140423e70(str)` = our `name_hash` (`h*0x397 + fold[c]`, fold
table `0x141db50d0`) — the registry is keyed by plain namehash.

`0x1427019a0` is a **static slot table**, not the pool itself —
callers do `mov rdx, [slot]` (e.g. `0x14187271e` loads
`*(0x1427019a0)` for `0x14187110c`). Init stores at
`0x1427019b8..0x1427019e8` (slots +0x18..+0x48) are per-type resolved
descriptors (`"Arena/CameraPrerequisite"`, `"Arena/CameraValidation"`,
`"Faction/Events/SimEventTagList"`, …) written by `0x141b09cxx`/
`0x141b0b5xx` blocks. So `map->+0x10` = `*(0x1427019a0)` = the runtime
pool descriptor (live: `0x4ef93708`), and **each type descriptor owns
its vecA (+0x38, 0x28-stride, flag=1) and vecB (+0x48, 0x20-stride,
flag=0) name pools**.

Loader `0x14187110c` seeds buckets from **vecA only** (loop bound =
`pool->+0x28`, elems via `0x1401897fc` = `pool->+0x38[i*0x28]`). VecB
elements are not in the serialized table → vecB is either the
runtime-added/dynamic pool or a variant serialization layout; that
distinction decides whether runtime interning appends to vecB (grow a
live vec) or must reallocate vecA + rebase every bucket key.

Open for the intern recipe (live-only): name-object fields beyond
`+0x00/+0x01/+0x18/+0x20`; vec end/cap fields on the pool descriptor;
whether a native runtime-insert exists (the only insert path seen is
the load-time merge `0x14181ed58`, which lives in Themida-VM'd code —
if none exists, the recipe is: append elem to vecB + append a bucket to
**each** consuming map (`+a4dc8` 0x80-stride and `+a4e70` 0x70-stride,
sized to pool position) under `quiesce_threads`).

**Bucket-array growth idiom** (VERIFIED-STATIC 2026-09-28,
`0x141873430` for the 0x70 map): the map has **no capacity field** —
`{+0 buckets, +8 count}` is exactly-sized; resize = free old
(`0x14178ab84`/`0x14178c598`), `+8 = n`, `+0 =
0x1401b0528(allocator, n*stride, align=8)` where `allocator =
*(0x1426fef88)` (fallback init `0x141113520`; `0x14114cc98` is its
getter thunk). Any replicated insert MUST allocate via the engine
allocator (`callva(0x1401b0528, [0x1426fef88], n*stride, 8)`) — a
plugin-owned buffer would be `free`d by a later engine resize.
Growing the pool vec itself (vecA) additionally requires rebasing
`bucket[i].key` in **every** map on that pool (keys are elem
pointers); appending to vecB avoids that entirely, but whether a vecB
elem can index `+a4e70` (idx computed raw from `pool->+0x48`,
colliding with vecA positions) is the open question. Three models:

1. **Biased base** — `pool->+0x48` = `vecB_begin − vecA_count*0x20`,
   so `(elem − +0x48)/0x20` = `vecA_count + i` continues the dense
   index space and one bucket array covers both vecs. Test: live dump
   whether `(pool+0x48) + (pool+0x28)*0x20` equals the vecB storage
   base.
2. **Separate map** — vecB elems belong to a different map whose
   `map->+0x10` chains to a vecB-shaped desc; `+a4e70` lookups on
   them always miss (bucket-key mismatch), which would make the
   `else` branch dead on this map but correct on another.
3. **Serialization variant** — vecB holds the same logical elems in
   0x20-stride form (e.g. imported-from-child-db names) and only one
   vec is ever populated per descriptor kind.

`dbpool()` prints all relevant fields; the live check is
`vecB_base − pool->+0x48` vs `pool->+0x28 * 0x20` (model 1) or any
nonzero bias (models 2/3).

**Public resolver input is a parsed token, not a cstr** (VERIFIED-
STATIC 2026-09-28f): `0x140d23a40(mgr, token, ctx, out)` first calls
`0x140d1a480(token)` for a kind tag (1/2/3 dispatch). Kind 3 →
`0x140d19e20(token)` returns the name **elem** already bound inside
the token — it's a field fetch through a global arg-descriptor table
(`[0x14264c660]`/`[0x14264c520]`, `token+8` desc array × `token+0x10`
value array, idx from `desc+0xe`), then `0x189898(+a4e70, elem)` +
`0x189ec0` fill. There is no raw-string→elem bridge on this path —
strings were interned by the parser when the token was built (VM'd
loader). Consequence: `sdk.db.intern` must mint the pool elem
directly (template via `sdk.debug.dbname`), not reuse this resolver.

**`0x141872800` = native payload populator** (VERIFIED-STATIC
2026-09-28f): called as `0x141872800(ctx, elem)` where `ctx+0xb0` is
the target map (callers pass `manager+0xa4dc0` so `+0xb0`→`+a4e70`).
It re-runs `0x189898` internally and **early-returns on miss** — the
bucket must already exist. On hit it derives the whole payload from
the elem through the accessor family: `+0x00`=elem,
`+0x08`=`0x18bdb0`, `+0x10`=u32 domain tag `0x14186f744` (the `0x15`
tag `0x189b50`/`0x189ec0` dispatch on is elem-derived, never stored
raw), `+0x18`=`0x14186f160`, `+0x20`=`0x14186f114`,
`+0x28/+0x2c`=f32 `0x1418701dc`/`0x14187013c`, `+0x60`=byte
`0x141870e28`, and a `0x141870dd8`-counted loop pushing
`0x141870d90(elem,i)` sub-elems through `0x141872b4c` into
`payload+0x48` (plus a `payload+0x30` side-list when tag==0x74).
The `0x189950` find-or-fill site (`0x140d23e75`) calls this only
after `0x189898` misses — so **an unregistered elem skips fill and
still hits the `0x189b50` null-payload path**. Interning therefore
needs only: `bucket.key = elem` + zeroed payload, then call
`0x141872800(stack_ctx, elem)` (a fake `ctx` whose `+0xb0` overlays
the real map) to populate every payload field natively — no payload
contract to replicate, and the elem's own accessors produce the tag.

**Runtime intern recipe** (sufficient for the `0x189898`+`0x189b50`
reference-target path):

1. Append a name element to the pool vec — element flag
   `byte[elem+1] bit1` selects vecA (`+0x38`, stride 0x28, set) vs
   vecB (`+0x48`, stride 0x20, clear); fields per live template:
   `{+0x00 {hi,lo} id, +0x01 flag, +0x08/+0x10 links,
   +0x18 = pool back-ptr, +0x20 = .idata payload ptr}` — dump a
   donor with `sdk.debug.dbname` before minting. The vec is
   load-sized — growth needs realloc **plus a bucket-key rebase**
   in every map on the pool (`bucket[i].key = new_base + i*stride`
   for i < count).
2. `idx = (elem − vec_base)/stride`; if `idx >= map->+8`, grow the
   bucket array via the `0x141873430` idiom **through the engine
   allocator** (`callva(0x1401b0528, [0x1426fef88], n*stride, 8)`).
3. `bucket[idx].key = elem`; zero the payload.
4. Populate the payload natively: build `stack_ctx` with
   `ctx+0xb0` = the map and `callva(0x141872800, ctx, elem)` — the
   engine derives `payload+0x10` tag, `+0x48` sub-elem span, and all
   sibling fields from the elem's accessors. Repeat per sibling map
   (`+a4dc8`/`+a4ea0`) if the record needs them. `0x189b50`'s other
   input `r8` is the **sweep's caller-supplied context** (`rsi` at
   `0x189aa0` — an object downcast to vtable `0x14054b100` via
   `0x140190cf0`, `[casted+0x428]` non-null); it is not intern-owned.
   Interning's contract ends at: elem exists in the right pool,
   `bucket.key=elem`, payload natively filled.
   **Staged tooling (2026-09-30):** `sdk.debug.dbintern(name, opts)`
   transcribes this recipe — read-only preflight by default (map/pool
   snapshot, donor template, vec-model discriminators, shard-flag
   check), `{probe="Donor"}` re-runs `0x141872800` on a KNOWN-GOOD
   bucket and diffs the payload (decisive ABI check before minting),
   `{commit=true, donor=…, vec_tail=…}` mints with publish-last
   ordering (elem → grow via engine alloc → key+zeroed payload →
   publish → native fill). Commit refuses on any unresolved field —
   vec headroom is not auto-detected, `vec_tail` comes from the
   `dbpool` dump. Uses `sdk.debug.copymem` (kernel-validated bulk
   copy) since the native resize `0x141873430` frees WITHOUT copying.
5. `+a4de0` name→holder map (0x90 buckets, lookup `0x140d17110`) is
   consumed by a **parallel sweep** at `0x14018ec43` (also iterates
   the spec-vec elems): `+a4de0` payload → `0x140d18880(payload)`
   → `+a4e10` map → ... A `+a4de0` miss is a **second null-deref
   site** — `0x140d18880` derefs `rcx` unconditionally at
   `exe+0x140d18890` (`mov rbp,[rcx]`), same crash class as
   `0x189b79`. So interning a name used by that sweep needs the
   `+a4de0` bucket too — or the dbwatch guard must also arm
   `0x140d18880`'s caller. Confirm at runtime whether the second
   sweep iterates the same spec elems (its loop uses the
   `0x189b40` iterator on `rbx` nodes from `r11-0x48`).

Name objects are **pool elements**, not heap objects
(VERIFIED-LIVE 2026-09-28 on a type-descriptor elem `0x53b82ef8`):

- `elem+0x18` = back-pointer to the owning **intern-pool object**
  (the "domain tag"). Lookup validates `name+0x18 == map+0x10` — a
  name only resolves in the map whose pool it was minted from.
- `elem+0x00/+0x08/+0x10` = three link pointers into a sibling
  container (0x14-spaced nodes — likely intrusive hash/list chains
  used by the owning structures).
- `elem+0x20` = pointer to a static object in `.idata` (payload).
  Elements do **not** embed the cstr inline; string-table elements
  ARE the string (their storage doubles as the cstr — see below).
- Element flag `byte[elem+1] bit1` (`0x140189940`) selects which
  pool array holds it — CORRECTED 2026-09-28 against the live
  `+a4e70` pool `0x4ef93708`: set → `(elem - [pool+0x38]) / 0x28`
  (magic `0x6666..7` ⇒ /40, stride **0x28**); clear →
  `(elem - [pool+0x48]) >> 5` (stride **0x20**). For string-table
  elements the flag is literally the second character of the string
  (a 2-way shard). Verified live: `+a4e70` keys are flag-set
  elements of `pool+0x38 = 0x4f2e9bb0` stepping exactly 0x28, each
  `elem+0x18 = 0x4ef93708` = the pool = `map+0x10` (the `0x189898`
  domain check). `elem+0x00` is an opaque `{hi,lo}` key/id, not a
  cstr — names resolve separately through `+a4de0` name→holder.
- The map "hash" is `(elem - pool_base)/stride` = pool index.
- Caveat: a live type-descriptor elem `0x53b82ef8` (flag set) has
  `pool+0x38 == 0` — its index would be garbage, so descriptor
  elements may resolve through a different pool/map path; the
  `+a4e70` name-map contract (flag-clear, 0x20-stride names in
  `pool+0x48`) is the inject-relevant one and matches the
  `0x42xxxxxx` cstr-blob elements seen live.

List **entry** layout (VERIFIED-LIVE 2026-09-28, stride **0x28**):

| off | field |
|-----|-------|
| +0x00 | key u64 — hi32 = `namehash(record)` |
| +0x08 | dead/freelist link (poisoned `0x6fbb966c`) |
| +0x10 | record data ptr (attr cells) |
| +0x18 | **owning list entry (back-link)** — VERIFIED-LIVE 2026-09-28: for `Faction.NameTable`'s `Orc` entry this reads `0x4516be88`, whose own `+0/+8/+0x10` cstrs are `"Faction/NameTable"`, `"Faction.NameTable"`, `"\Faction\NameTable\gdb.category0"` (the 0x58-stride list layout). An earlier note called this the type-descriptor element — wrong. |
| +0x20 | record name **cstr** |

The record *instance* (the object `0x189950` consumes) carries
**two** name-element arrays: `obj+0x10`/`+0x18` (start/end →
registered into `+a4dc8` descriptor map) and `obj+0x40`/`+0x48`
(→ `+a4e70` record map).

Minting a name therefore requires: append element to the pool's
shard vector, seed `buckets[idx] = elem` in every map that must
resolve it, and bind `name→holder` in `+a4de0` for its string.
The bucket-seed site itself is still TBD (likely inside the pool
element allocator); `0x14054e700` is the holder "create if
missing" wrapper probing `mgr+0xa4f10` first.

**Why `inject` crashes, precisely** (2026-09-28 — the two-path
model): `WBDBRecordLookup`/`0x1404ffcb0` resolves records by
**namehash bsearch over the list entry array** — it never touches
`manager+0xa4e70`. A self-contained injected record is thus
reachable via the Lua accessor WITHOUT interning. The `+0x189b79`
crash only fires when the injected name is resolved through
`0x189950`/`0x189b50` — i.e. when another record references it BY
NAME (cross-record refs, factory/spawn lookups, member builders).
So `inject` is unsafe specifically for names that flow through the
`0x189950` resolution path; the fix is to intern the name.

**Safe-intern contract** (all four writes verified against live
manager `0x41487480`, pool `0x4ef93708`, buckets `0x403bd180`):
1. **Pool-append**: extend `pool+0x38` vec `0x4f2e9bb0` (flag-set,
   0x28 stride) — or `pool+0x48` for flag-clear — with a new
   element `{q0={hi,lo} key, +0x18=pool}`; bump `pool+0x28` count.
   The vec-grow helper is `0x1418737dc`.
2. **Bucket-seed**: `buckets[idx][0] = elem` where
   `idx = pool_index(elem)` — built-ins get this from
   `0x14187110c` (the `+a4e70` loader copying `0x1427019a0`
   static-table records); the runtime equivalent is inside
   `0x140d23e00`'s create family or `0x141872718`'s build.
3. **Payload**: fill `&bucket[8]` (0x68 region) —
   `payload+0x10` = `0x15` record tag, `payload+0x48/+0x50` =
   element-ptr span — via `0x140189ec0` (type-dispatched on tag).
4. **Holder-bind**: bind name→holder in `+a4de0` via `0x140d21430`
   → `0x14054e700` so the string resolves.

   `0x140d23e00` fills the payload on a **pre-seeded** key —
   `0x189898` then always hits. The seeding write `buckets[idx][0]
   = elem` is the one instruction still unlocated (bootstrap-side
   it's `0x14187110c`; runtime-side it is buried in the intern
   allocator).

   **Two intern pools** (VERIFIED-LIVE 2026-09-28): `+a4e70` keys
   are elements of pool `0x4ef93708`; `+a4de0` (name→holder) keys
   are elements of a *different* pool `0x4efb36e0` — verified:
   `+a4de0` `bucket[0]` key `0x4fdefa18`, `key+0x18 = 0x4efb36e0`.
   Interning a name therefore appends an element to EACH pool the
   name must resolve in. `manager+0xa4f10 = 0x2949d4b08` is the
   separate map `0x14054e700` probes before creating a holder.

### Map family + registration chain (VERIFIED-STATIC, 2026-09-26)

The manager carries a family of same-shape maps; each is keyed by
pool-element names and domain-validated by `map+0x10` == the name's
pool:

| offset | bucket stride | role |
|--------|---------------|------|
| `manager+0xa4dc8` | 0x80 | per-descriptor name→? map (`0x14018f164` lookup-only) |
| `manager+0xa4de0` | 0x90 | **name→string-holder map** (`0x140d17110` lookup-only) — binds each name elem to its localized-string handle |
| `manager+0xa4e28` | 0xE8 | name-keyed map probed inside the holder-create path (`0x140d21430`) |
| `manager+0xa4e10` | 0xC0 | string→typed-entry find-or-insert (`0x14018f110`); value tag `[bucket+8]` compared to 4/7 |
| `manager+0xa4e70` | 0x70 | **the name→record map** (lookup-only `0x140189898`, `imul ×0x70`; resolver crashes on miss). VERIFIED-LIVE bucket: `+0x00`=key elem (0x28-stride pool elem, `{hi,lo}` q0, `+0x18`=pool), `+0x08`=payload — `payload+0x10`(=bucket `+0x18`)=type tag `0x15` for records, `payload+0x48/+0x50` = element-ptr span `0x189b50` iterates |

**`+a4e70` is index-parallel (VERIFIED-LIVE 2026-09-28,
`internobj.lua`)**: `buckets[i][0] == [pool+0x38] + i*0x28` for every
i (checked 0..8). The bucket key field holds the *address* of the
flag-set element in the pool's `+0x38` vec — so interning a name is
two writes of the **same** element address: materialise `{q0={hi,
lo=0xa07}, …, +0x18=pool}` at `pool38 + idx*0x28` **and** store that
address into `buckets[idx][0]` (then `&bucket[8]` is the record
payload, `map+8`/`pool+0x28` count bumped). `idx` comes from the
domain index allocator (`0x14181de7c` → `0x140311e8c`). **`q0` is NOT
a namehash** — `namehash("Orc")=0xc18ea0` matches no `q0.hi` in either
`+a4e70` or `+a4de0` (VERIFIED-LIVE). The map is *element-indexed*:
`0x189898` derives the bucket index from the element's own pool
position, so `q0` is element metadata (`lo=0xa07` constant → a
type/tag dword; `hi` varying → a per-element id/serial assigned at
mint, plausibly the record/name id). Name→element resolution happens
upstream (`+a4dc8`/`+a4de0`/`0x140d1adc0` chain); `+a4e70` just binds
`pool-index → record`. Two element classes coexist in
the name system: `0x40cxxxxx` **string chunks** (cstr inline at +0 —
a record's `+0x10` name-array is a ptr list to these; `+0x18` is
string continuation, *not* a pool link) vs `0x4f2exxxx` **opaque key
elements** (`{q0, +0x18=pool}` — the actual `+a4e70`/`+a4de0` map
keys). `map+8` packs `{lo=count, hi=capacity-ish}` (e.g.
`0x1bf7e0000088c` → lo `0x88c`=2188).

### Strings: chunked-blob handles (VERIFIED-STATIC, 2026-09-26; storage layout VERIFIED-LIVE 2026-09-28)

Live dump of the interned-string blob (`0x42dedb9c` region,
VERIFIED-LIVE): names are stored as `{head4, pad4, tail_cstr}`
chunks — e.g. `45 6e 65 6d 00 00 00 00 79 5f 56 65 72 79 48 61 72
64 00 ...` = `"Enem"` + `"y_VeryHard"`. This matches the holder
`{+8: chunk-array, +0x10: ptr-array}` model: `word[chunk_elem+0xE]`
indexes the ptr array for the tail — the head-4 + tail split is the
storage format, not a misread. Names' `byte[elem+1]` (the map flag)
is literally the string's 2nd char, so the two pool vectors shard
names by 2nd-char parity.

Name elements do NOT carry cstrs — `manager+0xa4de0` maps name-elem →
a **holder**: `{+0x8: chunk-array, +0x10: ptr-array}`. The accessor
`0x140d18880(&holder)` picks `chunk[singleton+0x40]` then returns
`ptrarray[word[chunk_elem+0xE]]` — the byte at `0x14264c660+0x40` is a
**language/table selector** (`[mgr+0x30]>=2` required; a second
sub-table uses `+0x5d0`/`+0x5e0`). The returned pointer is itself a
**pool element** (callers pass it to `0x14018f110`, which reads
`[p+0x18]` as the owning pool) — so string-table elements are
`{+0x00: cstr-or-char*, …, +0x18: pool, …}` and the element doubles as
the string. The name system is two-stage: cstr → interned
string-table element (in the blob), name-elem → holder binding in
`+a4de0`, name-elem → record in `+a4e70`.

**Holder create path**: `0x14078ef90(mgr, ?, name, ?)` →
`0x140d17110(+a4de0)` lookup → tail `0x140d21430` — on `+a4e10` miss
calls `0x14054e700` (the actual create; probes yet another map at
`mgr+0xa4f10` via `0x14054ead0` and scans the holder's `+0xc0` entry
vector with comparators `0x140188c80`/`0x141161680`), then probes
`+a4e28` (stride 0xE8) and picks an allocator via
`byte[holder+0x29]` (`0x142702590` vs `0x142702598`). This is the
mint-side machinery — not yet fully decoded; the live `INT|` recon
confirms the read path first (`0x140d17110` + `0x140d18880` are
call-safe: both validate pool tags and return 0 on miss).

**Holder-create internals (VERIFIED-STATIC 2026-09-28)**: inside
`0x14054e700(manager, holder, name?, flg)` the entry-mint is
`0x140d22ee0(manager, holder, elem-text)` → returns the insert index
(`rbx = count - eax` used as the slot). It walks the holder's
`+0xc0`/`+0xc8` entry vector (`{first,last}` span, `sar 3` = count),
case-insensitive string-compares each candidate's `[rcx]` cstr via the
`0x141d935d8` thunk (CRT `strncasecmp`/`strncmp`), and on miss
finds-or-inserts an entry. A separate list `holder+0xa0/+0xa8` is also
traversed. `0x140d1adc0` is a **holder→string-element resolver**
(same `0x140d18880` pattern: `rsi=[holder]`, chunk-array `rsi+8` +
`word[+0xe]` idx → `rsi+0x10` ptr-array → element) plus a bsearch over
the element's `+8` array for `dword==[rbx+0x2a8]`. `0x140d22ee0`
itself is a holder-entry **rebuild/copy**: it iterates `holder+0xa0`
list items and pushes each into the `+0xc0` vector
(`[end]=elem; end+=8`, capacity gate `[r14+0xd0]` via `0x1407a42f4`) —
it moves *existing* element ptrs, not mint new ones.

So the real name→element mint is layered and the raw pool-element
materialisation sits **below** the holder-entry copies — the candidate
sub-calls inside `0x14054e700` that can reach it are `0x140190cf0`
(first call, RTTI cast thunk — not the mint), `0x140d24a40`
(`(rsi,r15,r14)` post-scan), and `0x14054ead0` (the `+a4f10` map
find/grow — a generic hash wrapper: `0x14054ed08` find →
`0x14054ed74`/`0x14018ff20` grow gated by `0x142a25df4` + load-factor
`mulss×0x14202e644`). The `+a4e70` index-parallel seed is downstream
of whichever materialises the element.

**Insert template decoded (VERIFIED-STATIC 2026-09-28)** — the seed
is fully specified:

- `0x189950(manager, desc, names, ctx)` = the post-registration
  **resolver**, read-only: 3 name-list loops — `+a4dc8`→`0x18eab4`,
  `+a4e70`→`0x189898`+`0x189b50`, `+a4ea0`→`0x638370`+`0x66adec`.
  **It never inserts** — `0x189898` is lookup-only, so a missing
  `+a4e70` element is exactly the `0x189b79` null-deref. `+a4e70` is
  populated only at bootstrap/merge, not by `0x189950`.
- `0x14187110c(map, table)` = the bootstrap seeder: `map+0x10`=table,
  `0x141873430(map, count)` grows `map+0` buckets to `count*0x70`,
  then `buckets[i][0] = 0x1401897fc(table, i)` for all i.
- `0x1401897fc(table, i)` = element accessor: `if i<[table+0x28]`
  return `[table+0x38] + i*0x28` — i.e. **the static `0x1427019a0` is
  itself a pool** `{+0x28=count, +0x38=flag-set 0x28-vec}` and its
  elements are the `+a4e70` keys.
  - Same accessor-table shape shared by the `DB*` record accessors in
    `extracted/db_accessors.tsv` (VERIFIED-STATIC 2026-09-29): e.g.
    `DBScenarioSetup` at `0x142702278` is a live object with
    `{+0x28=count, +0x38=0x28-stride record-entry vec}` —
    `0x140429708(obj)` returns `[+0x28]` (count) and list-binding code
    (`0x141aabeb0`, e.g. ambient-subsystem `+0x100e8` bind at
    `0x141aac75b`) snapshots `[+0x38]+i*0x28` entry ptrs. So every
    `DB*` list is walkable read-only in Lua via `read_u32(acc+0x28)` /
    `read_u64(acc+0x38)` — no callva needed.
- `0x141873430(map, n)` = buckets realloc (`map+8`=count → grow
  `map+0` to `n*0x70` via `0x14114cc98`+`0x1401b0528`).

`0x189898`'s lookup reads **only** `elem+0x18` (pool — domain check)
and `byte[elem+1]` (flag → `+0x38`/`+0x48` vec select); it never reads
`q0`. So a valid `+a4e70` element needs only: live at
`pool+0x38 + idx*0x28`, `+0x18`=pool, flag bit set. `q0`/`hi` is
element payload for *other* consumers (the name identity), not the
lookup — its exact mint value still needs a live capture, but a
sibling-consistent value is plausible.

**Minimal inject** (surgical, no full holder-create): (a) grow the
pool's `+0x38` vec + append a 0x28 element (`+0x18`=pool, flag set)
at `idx=count`; (b) `0x141873430`-equivalent bucket grow, write
`buckets[idx][0]=elem` + the `&bucket[8]` `0x15` payload; (c) bump
`map+8`/`pool+0x28`; (d) `+a4de0` holder bind for name-lookup. The
two grows (pool-vec and buckets) are the risky reallocs — safest is a
dedicated `sdk.db.intern` that replicates `0x14187110c`'s writes for a
single new element rather than invoking the whole bootstrap.

**`sdk.db.intern` concrete plan** (gated `NEMESIS_UNSAFE_CALLS`, all
engine calls via `call_va` on the game thread, structural writes under
`quiesce_threads`):

1. `mgr = dbwatch()` (captured `0x189b50` rcx).
2. `map = mgr+0xa4e70`; `buckets=[map]`; `count=[map+8]`; `pool=[map+0x10]`.
3. Name-element mint: `+a4e70` uses pool `0x4ef93708`, `+a4de0` uses a
   **separate** pool `0x4efb36e0` — a name is interned as *two* distinct
   elements (record-name in `+a4e70`'s pool, holder-name in `+a4de0`'s).
   The `+a4e70` element for a runtime name has **no bootstrap caller** —
   it is appended only by `0x14187110c`-style growth or the merge, so we
   replicate that: append a 0x28 element to `pool+0x38` vec at
   `idx=[pool+0x28]`, set `elem+0x18=pool`, flag `byte[elem+1]|=2`
   (selects `+0x38` vec). `q0`/`+0x08`/`+0x10`/`+0x20` = sibling-cloned
   record-name payload (`q0` is not consumed by `0x189898`/`0x189b50`).
4. `call_va(0x141873430, [map, idx+1])` to grow `buckets` if needed
   (engine realloc; frees old, copies, sets `map+8`). Then
   `buckets[idx][0]=elem` under `quiesce_threads`; fill `&bucket[8]`
   `0x15` payload cloned from a sibling bucket (tag `+0x10`=0x15,
   `+0x48/+0x50` = the record's data-elem span).
5. `pool+0x28 = idx+1`. Name→holder in `+a4de0` via `call_va(0x14078ef90,
   [mgr, ?, name, ?])` (holder-create; binds the string for display) —
   arg shape still to be confirmed live.
6. Verify: `call_va(0x189898, [map, elem])` must now return `&bucket[8]`
   (non-null) — the exact hit that previously NULL'd into `0x189b79`.

**callva read-path PROVEN live (2026-09-28, `internobj` IDX block)**:
`callva(0x1898e8, elem)` returns exactly `(elem-pool38)/0x28` (idx
0..7 match `want`), and `callva(0x189898, map, elem)` returns the
non-null `&bucket[8]` payload stepping 0x70 — the engine's own
functions validate the index math + call path on the game thread.
CAUTION: `0x140d17110` (the `+a4de0` accessor) is **not** domain-safe
across pools — feeding it a `+a4e70` element AV'd at `0x819113`
(crashed the game). Only invoke map accessors on their own pool's
elements.

The vector-grow helpers (`0x141873430`, `0x1418737dc`, `0x14114cc98`)
are the only functions whose side effects we can't safely hand-roll —
calling them keeps realloc + free on the engine's allocator so no
plugin-owned pointer enters the maps.

**Create-path index chain (VERIFIED-STATIC 2026-09-28)**: the record
writer `0x140d23e00(name)` does `0x140d1adc0` (holder→string) →
`0x140d1bcf0` gate (eax==1) → `0x140d1b9b0` (holder→element `r9` →
tail `0x14181de7c`) → `eax`=index → `[mgr+0xa4e70]+8+idx*0x70` →
`0x189ec0` `0x15` payload-fill. `0x14181de7c` is TLS-locked
(`0x142aa1320`) → `0x140311e8c(elem, sel, slot, out)` reads
`dword[elem+0x10][(word[elem+8+sel*0x10+0xe]+slot)*4]` — a stored
per-element domain index. LIVE probe (`callva`, sel=0xe0 slot=0)
returns `sidx=0` for elements whose `0x1898e8` address-index is
0..3 — so the stored domain index and the address-derived index are
**different representations**; the element's `+0x10` dword table is a
per-domain index map, not the `+a4e70` slot. The materialiser must
keep `buckets[idx][0]==pool38+idx*0x28` consistent with BOTH — that
coherence is the remaining unmapped primitive.

**Interning is load-time-only — no engine runtime-mint exists**
(VERIFIED-INGAME 2026-09-28): armed a hardware write watchpoint on the
`+a4e70` pool's next slot `pool38+count*0x28` (`0x3b396470`, mgr
`0x36249798`) AND on the `+a4de0` pool's next slot `pool38+count*0x28`
(`0x3bfa8f00`), then drove (a) a full save→world load and (b) a live
`sdk.spawn.summon('Gondorian_Captain')`. **0 hits on both watchpoints**
and both map counts stayed `+a4e70`=2188 / `+a4de0`=29709 throughout.
Two consequences: (1) a spawned entity reuses its existing record — its
generated *display* name is an entity property, NOT a `+a4de0` GameDB
name-element, so summons don't intern; (2) `+a4e70`/`+a4de0` are
**closed sets populated once at bootstrap/merge** — the engine has no
runtime path that mints a fresh GameDB name-element. Therefore
`sdk.db.intern` cannot reuse an engine mint (there is none); it must
either hand-replicate the load-time materialise under `quiesce_threads`
(grow pool-vec + write element + seed `buckets[idx][0]` + `0x15`
payload + `+a4de0` holder), or register the new name as a pending delta
consumed by the next registration/merge hook. The stored-vs-address
index coherence then becomes *our* responsibility too — a fully
hand-materialised element must populate `elem+0x10` (the per-domain
index table `0x140311e8c` reads) as well as the `pool38`/`buckets`
address fields, or `0x140d23e00`-family creates will resolve a wrong
index. `+a4de0` is a hash map (`map8`=`0x58990000740d`: hi=`0x849` hash
state, lo=count) — its element store is NOT `pool38`-index-parallel,
so its insert differs from `+a4e70`'s.

**`0x15` payload contract — fully decoded** (VERIFIED-INGAME
2026-09-28, live `+a4e70` bucket `0x35ed2cc0` for `RoleCategory_194_Branded`):
the `&bucket[8]` payload (0x68 bytes) is
  - `+0x00` = the `+a4e70` **key element** (`pool38+idx*0x28` — a
    back-link equal to `buckets[idx][0]`).
  - `+0x08` = the record's name **string-element** ptr (cstr inline).
  - `+0x10` = tag `0x15`.
  - `+0x18` / `+0x20` = a category/type **cstr ptr** — both point to the
    same string (`0x39939464` → `"Enemy_VryHador"`), a secondary
    record-kind name.
  - `+0x28`..`+0x40` = 0.
  - `+0x48` / `+0x50` = `ptr`/`end` of a heap array of string-element
    ptrs — `n=1`, `[0]` = the same `+0x08` name-element.
  - `+0x58` = `{1,1}` packed dword counts; `+0x60` = 0.
`0x189b50(rdx=payload, r8=ctx)` does `cmp dword[rdx+0x10],0x15; jne
ret` then iterates `+0x48`..`+0x50`: for each element it namehashes
the cstr (`imul 0x397` + fold `0x141db50d0`) and bsearches pool
`0x142701d98` (`0x142701d98` is a `{+0x28:count,+0x38:vec}` pool) →
`0x350e90`/`0x350e44` → compares to `[ctx+0x428]`. **It early-returns on
a non-`0x15` tag — the `0x189b79` AV is ONLY the null-payload deref on
the `+a4e70` miss, never a payload-field read.** So crash-prevention
needs only `buckets[idx][0]=elem` + a `0x15`-tagged payload with a valid
`+0x48`/`+0x50` string-element array; resolution correctness is a
separate concern.

**`pool38` has no spare capacity — append needs the engine vec-grow**
(VERIFIED-INGAME 2026-09-28): `pool38+count*0x28` (`count`=2188) reads
structured live data, not free/uninit space — the element vec is exactly
count-sized, so appending element `count` requires a real grow. Because
`buckets[i][0]` and `payload+0x00` store element *addresses*
(`pool38+i*0x28`), a realloc **moves every element** and all existing
back-links must be re-seeded (`buckets[i][0]=new_pool38+i*0x28` ∀i,
the `0x14187110c` template). The pool (`0x3b02a9e8`) is a rich struct —
`+0x38`/`+0x40`/`+0x48` flag-set/flag-clear element vecs (`0x3b380e90`,
shared base), `+0x58`/`+0x60`/`+0x68` a `{begin,end,cap}` cstr vector of
record-name strings (`"Combat/..."`), `+0x50`/`+0x70` sub-structs.
`0x141873430` grows `map+0` (buckets); the **`pool+0x38` element-vec
grow** is a distinct function still to be located — that's the single
remaining engine call needed before a hand-materialised `sdk.db.intern`
can be written safely. (Alternative: a chunked/reserved-arena append if
the grow proves vec-buffered.)

**Element layout + constructor** (VERIFIED-STATIC 2026-09-28): an
opaque map-key element (`0x4f2exxxx`) is
`{+0x00:q0={id,0xa07}, +0x08:domain-descriptor table, +0x10:index-table,
+0x18:pool, +0x20:?}` — `0x140311e8c(elem,sel,slot,out)` is a PURE read:
`[elem+8]` holds `sel*0x10`-strided descriptors (`+0xc`=count,
`+0xe`=base) and it returns `dword[[elem+0x10]+(base+slot)*8]`, the
stored per-domain index. `0x14181de7c` is only a TLS-guarded wrapper
around it. **Neither appends** — the element materialise (write
`pool38+i*0x28` + build the `+0x08`/`+0x10` tables + set `+0x18`) lives
in an element-object *constructor* in the `0x140d1`/`0x54e`
string-subsystem, still to be located. The stored `+0x10` index and the
address `pool38` index must be built together there.

**Grow/copy/alloc helpers are `call_va`-able** (VERIFIED-STATIC
2026-09-28): **Index-parallel map FAMILY** (VERIFIED-STATIC 2026-09-28): `+a4e70`
is not alone — the manager holds several element-keyed maps all indexed
by the same `(elem-pool38)/0x28` position. `0x140d21430` (holder
create-on-miss, called by `0x14078ef90`) computes `(elem-[pool+0x38])/
0x28` inline via magic-div `0x6666666666666667`, branching on
`byte[elem+1]` flag (set→`+0x48` vec, clear→`+0x38`/0x28 — exactly the
`0x1898e8` math). It then probes `mgr+0xa4e28`, a **0xE8-stride**
index-parallel map (`entry[idx*0xe8][0]==elem` check). On miss it calls
`0x14054e700(mgr, ?, element_source, flag)` — the holder-create entry
that scans the holder's `+0xc0` name-vector (`0x188c80` validity +
`0x141161680` compare) and uses `mgr+0xa4f10` (hash find-or-create).
So `0x14054e700`/`0x140d21430` link an *already-materialised* element
into `+a4e28`/`+a4f10`; the element's `pool38` slot itself is built
during the load-time holder/descriptor construction (upstream), and
`0x140d23e00` later seeds `+a4e70`'s `0x15` payload for it.

**Grow/copy/alloc helpers are `call_va`-able** (VERIFIED-STATIC
2026-09-28): `0x141873430` is a generic `{+0:ptr,+8:count}` vector-grow
(`imul n,0x70` for buckets) built from `0x14114cc98`→`0x140602d20`
(engine malloc), `0x1401b0528` (copy), `0x14178c598`→`0x1401b08c4`
(free), `0x14178ab84` (pre-op). So a hand-materialised `sdk.db.intern`
can grow `pool38` itself: `callva(malloc,(count+1)*0x28)` → copy
`count*0x28` → `pool+0x38`/`+0x48`=`newbuf` → `callva(free,old)` →
re-seed `buckets[i][0]`/`payload[i]+0x00`=`newbuf+i*0x28` ∀i → append
element `count` + `0x15` payload + `map+8`/`pool+0x28` counts — all
under `quiesce_threads`. Engine-owned alloc means no plugin pointer
enters the maps.

**Alloc machinery VALIDATED live (2026-09-28)**: `callva(0x140602d20,
0x1000)`→`0x14277cb40`, `write_u64`/`read_u64` round-trip
`0xDEADBEEF`, `callva(0x1401b08c4,p)`→clean free, no crash. Engine
malloc/free are ordinary callable buffers.

**Rebase risk note**: `+a4e70` is queried ~6.36M×/s in-world, so the
`pool38` grow + re-seed of 2188 `buckets[i][0]` AND `payload[i]+0x00`
back-links must be **atomic under `quiesce_threads`** — a partial rebase
leaves `0x189898` reading a stale/moved element → instant crash. The
per-back-link re-seed is the delicate part; consider doing the whole
grow+rebase in one frozen pass and only then appending the new element.

**Minimal `+a4e70`-only intern may suffice for crash-prevention**
(INFERRED 2026-09-28): `0x189b79` fires inside `0x189950`'s `+a4e70`
sweep — `0x189898` miss → `0x189b50` derefs `null+8`. So the injected
name only *must* resolve in `+a4e70` (the index-parallel pool we can
grow via `0x141873430` + element append + `0x15` payload pointing at
the record's data-elems). `+a4de0` (hash map, harder insert) is a
*separate* name→holder path used for string/display resolution — likely
not required to stop `0x189b79`. `WBDBRecordLookup` itself resolves via
name-hash bsearch over the list-array (no `+a4e70`), so a record whose
only consumer is `sdk.db`/`WBDBRecordLookup` may never even need
interning — the gate exists for the `0x189950` re-sweep case. This
splits `sdk.db.intern` into a small `+a4e70`-seed stage (crash-safe)
and a later `+a4de0`-holder stage (full name resolution).

**Resolver-guard — the chosen near-term crash fix** (IMPLEMENTED
2026-09-28): rather than hand-rebase `pool38` (moves all 2188 elements,
breaks an open-ended back-reference graph — `buckets[i][0]`,
`payload+0x00`, `+a4e28`, `+a4f10`, every `holder+0x10` index table),
`dbwatch`'s `0x189b50` detour now returns `al=false` when `a2==0`
(the `0x189898` miss payload). `0x189950` already handles a false
resolver return gracefully (`je 0x189b15` skip), so an uninterned
injected name degrades to "unresolved" instead of the `+0x189b79`
null-deref — with zero structural mutation. `sdk.db.inject`/`append`
auto-arm it via `dbwatch::install()` after the `NEMESIS_UNSAFE_DB`
gate and refuse if it can't arm (refused inject > dead process).
**Caveat**: this guards only the *miss* path. A wrong-pool element
(`elem+0x18 != map+0x10`) still routes `0x1898b4 → 0x14109deb2 →
0x140819104` = fatal `WBAssert("Bad record index")` — the guard does
NOT cover it, so domain validation stays strict and the guard is not
license to pass foreign elements.

**Injected records ARE natively name-resolvable (VERIFIED-LIVE
2026-09-28d)** — `sdk.db.lookup("Debug.FactionMember","NemesisProbe_G41")`
returned the record after `inject`. `lookup` reads the *live*
`list+0x38` entry array that `inject` repoints, over `0x28`-stride
entries keyed by `entry+0x04`=name_hash — byte-identical inputs to the
game's own `sub_140278540` bsearch (`base+idx*0x28`, `entry+0x04` cmp,
`r8d=count-1`; confirmed in disasm) that every `DB<Name>` accessor
funnels through. So the game's own `DB*` name-lookup resolves an
injected record **without** `+a4e70`/`+a4de0` interning. `0x189950`'s
`+a4e70` sweep is a *separate* subsystem — record→record reference /
eventmap registration, not the `DB*` lookup path. **Consequence for
G41**: `inject`+resolver-guard is sufficient for name-driven `DB*`
consumption (the primary spawn/factory query). `sdk.db.intern` is only
needed when an injected record must be a *reference target* inside
`0x189950`-mediated resolution (another record's attr pointing at it)
or otherwise participate in the intern map — a narrower case than
primary name-lookup.

**The intern is three-layered** (VERIFIED-STATIC 2026-09-28): a name
exists as elements in **two distinct pools** plus a holder binding —
  1. **cstr → name element** in `+a4de0`'s pool `0x4efb36e0` (the
     string-intern — root mint, in the `0x140d1xxxx`/`0x54exxx` string
     subsystem; `0x140d1b9b0`/`0x140d1adc0`/`0x140d1bcf0` are its
     holder→element→index accessors).
  2. **name element → holder** via `0x14078ef90(mgr, ?, name_elem, flg)`:
     `0x140d17110(&mgr+0xa4de0, name_elem)` lookup → `0x140d21430`
     create-on-miss. Takes an *existing* name element — it binds, not
     mints.
  3. **name element → record** in `+a4e70`'s pool `0x4ef93708`: the
     `0x4f2exxxx` opaque `{q0={id,0xa07}, +0x18=pool}` key elements;
     `0x189898` address-indexed lookup. `0x140d23e00` writes the
     `0x15` record payload at `&bucket[8]` via the stored domain index.
A name is fully usable once all three exist; `sdk.db.inject` currently
does step 0 (list-array splice) only — the crash is step 3's missing
`+a4e70` element when `0x189950` re-runs.

**The thunk's caller — the post-load bind pass** (VERIFIED-STATIC,
2026-09-26): `0x14078ef90`'s only caller is `0x14054dfdb` inside
`0x14054df40(manager)` — a big bind function that iterates a
descriptor list at `manager+0xa4178` (count) via accessor
`0x14054d1d4`. Per element `r15` (a name-binding descriptor):

- `[r15+0x0c]` → `0x14114be28(manager, id)` — **type-descriptor
  accessor**: ids `< 0xF4240` (1,000,000) and `== -1`/`0xFFFFFF00`
  rejected; `id - 0xF4240` bounds-checked vs `manager+0x30` count →
  `manager+0x38 + idx*0x4f8`, validity byte at `elem+0x152`. So
  gamedb type-descriptor ids are a 1,000,000-based namespace.
- `[r15+0x08]` → `0x14178c48c(manager, id)` → `manager+0xa4e58`
  map + `0x14078a18c` find/insert — **another family map: id →
  per-type sub-map** (callers `0x14181f360`, `0x141872606`).
- thunk `(manager, descriptor, [r15+0], submap)` → `+a4de0` lookup +
  `0x140d21430` create — binds the name's string holder inside the
  correct per-type sub-map.

`0x14054df40` has **zero static call sites** — dispatched through a
vtable/RTTI slot or called from virtualized code (the post-load
gamedb bind step). So the live mint entry can't be lifted from a
call graph; the per-piece functions are callable though:
`0x14114be28` (id→descriptor), `0x14178c48c` (id→type sub-map),
`0x140d17110` (name→holder lookup), `0x140d18880` (holder→cstr).

**Type-descriptor array**: `manager+0x30` count, `manager+0x38`
elements of stride `0x4f8` — one per gamedb list/category, each
carrying its own maps/pools.

**Record registration `0x140189950(manager, desc?, names, ctx)`** —
the path the loader uses per record:

1. Iterates the record's embedded name array (`rec+0x10` ptrs /
   `rec+0x18` count): `0x14018f164(+a4dc8, name)` → `0x14018eab4`
   links each into the descriptor map. Names must already be pool
   objects — no minting here.
2. Iterates a second list: `0x140189898(+a4e70, name)` →
   `0x140189b50(manager, bucket, ctx, value)` writes the record
   binding into the global map.
3. `0x14018ec36` (the +a4e10 intern/registration sweep) walks a
   name list: `0x140d17110(+a4de0, name)` existing-name check →
   `0x14018f110(+a4e10, key)` find-or-insert → if `[bucket+8]` ∈
   {4,7}, iterate all type descriptors (stride 0x4f8) matching
   `0x14114bdcc` and re-run `0x140189950`/`0x140d1fe10` — i.e. the
   name fans out to every descriptor that claims it.

So a correct `sdk.db.inject` needs: mint/find the name pool element,
bind `name→record` in `+a4e70`, and register into the owning
descriptor's structures (the `0x4f8` element for its list).

## The name system (why it matters for mods)

Captain display names are embedded in record *names*
(`OrcName_018_Ugakuga`); a `Faction.NameTable` record holds the
selection pool as an explicit array (`NameStringIDs` — 80/144 Orc gens
are poolable in patch04; `OrcName_Unique_*` excluded = script-pinned).
Adding a name = mint gen record + display record + append to
`NameStringIDs` — blocked on G41.

## On-disk format (v17, partial)

`GADB` magic, `u32 version=17`, string-table size/count header, strings
at `+0x2c` (u16 len + path + data); `StringPtr` u32s are offsets from
blob base `0x47`. Record refs are stored **by name string**. Extract:
`python tools/nem.py arch extract <archive> game.gamedb <out>`.

## Console commands worth knowing

`Faction.SimEvent.Generate/Simulate`, `Faction.Ambush.Enable`,
`Faction.SetFortAssaultSlot`, and **`faction.load
..\Game\FactionData\DemoFaction.txt`** — declarative roster injection
from a text file. If live-verified it is the strongest native mod
primitive found. `UNVERIFIED`.
