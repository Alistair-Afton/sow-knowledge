# Spawning, summon templates, and NPC asset bundles
> **Status:** **partial** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


How a spawn request becomes a materialized NPC — record classes, the
bundle pipeline that streams assets, and what `sdk.spawn` does with it.

## The one-native-factory fact (design axiom)

The game spawns every object through one internal factory: a dynamic
object group is created, the object's **model and collision are linked
to it by name**, and the engine initializes the rest natively. SDK spawn
work should reach for that path — not hand-assemble entities.

## Template classes (what `SummonNPCs` can actually materialize)

`sdk.spawn.resolve(name)` -> `DBAISpawnTemplate[name]`. Resolution
success does NOT imply summonability — the record's class decides:

| Family | Example | Via `SummonNPCs` |
|--------|---------|------------------|
| `Summon_*` / `GangType_SummonGroup_*` | `Summon_OrcDefenderTough`, `Summon_OrcBerserkerTough`, `Summon_OrcDefender` | **Visible entities** — summon-ability family, bodyguard-class assets treated as resident |
| `AI.SpawnAmbientAIType` | `Gondorian_Captain`, `Gondorian_Soldier`, `GruntOrc`, `Olog_Tower` | Dispatches cleanly (pcall ok) but **materializes nothing** — ambient AI *types* placed by the ambient scheduler, not the summon path |

Proven live: `summon("Summon_OrcDefenderTough", 2, {faction="PlayerFollower"})`
-> two visible orc defenders beside Talion (screenshot-verified), SummonData
clone consumed through `DBAISummonData`. Stock-vs-injected `sd` behaved
identically — `db.inject` is not the gate for visibility.

Do not use `GetPlayerFollowerCount()` as a spawn signal — it stayed 4
throughout summon sessions (counts Nemesis-system followers, not
summon-followers). `SpawnedActors` is similarly undependable.

## `Server.NPCBundle` — name -> asset paths (NOT record refs)

Each record (~146 live) is a thin wrapper: attrs = `{name}` + one
`Bundle` attr of type `AssetBundle`. The `Bundle` cell is a pointer to
a **path struct** with inline filesystem strings — peek the cell
pointer to decode:

    bundles\goldenpath\gondor.bndl
    bundles\modelcharacters\player\base\playerskin_shadowwarbaranor.bndl
    Playerskin_DarkBaranor            (bare name, adjacent)

`bundle_keys(cellptr)` (`runtime/sdk/spawn/bundles.lua`) peeks 256B,
splits printable runs, takes `.bndl` basenames + normalized tokens.
Bundle **record names are character/family names**: `Zog`, `Gondor`,
`Easterlings_Base`, `Shelob3_Orc_DefenderWarchief`, `BruzTheChopper`.

## `Server.NPCBundleList` — named lists of bundle refs

~135 live records; attrs `Bundle` + `RemoveBundle` are **array cells
whose elements are a container array of `Server.NPCBundle` entry
pointers**. Raw q64 fingerprinting misses them — decode the container
(`entry.array` / `refset` descent). Example lists:
`PlayerBase_Barrows`, `MonsterHunter_Orc_Beastmaster`,
`Tower_Golden_Uruk`, `Oasis_Uruk`.

`sdk.spawn.bundles(listname)` -> `LoadNPCBundleList` pre-streams the
bundle assets; `sdk.spawn.clearbundles(listname)` -> `ClearNPCBundles`.

## The match problem (why `bundles_for` can't fingerprint)

Template -> bundle linkage is **name similarity**, not a record graph:
the template's transitive refs terminate in AI/arch attributes, while
bundle records only hold asset paths. `bundles_for` therefore matches:

1. **Direct**: template's raw cell addrs vs bundle entry addrs (rare).
2. **Canon name**: resolved record name vs bundle record name —
   exact / containment / shared-prefix>=6 / token-prefix for short
   names (`Zog_Fighter` -> `Zog`; `Zogged` correctly rejected).
3. **Path keys**: `.bndl` basenames from `bundle_keys` vs the same
   canon + ref names.
4. **List membership**: list cells + container-array descent vs the
   matched bundle entry addrs.

Normalization keeps `_` (Lua `%w` excludes it — the token-boundary
rule needs it). Scan memoized per entries-generation; `resolveentry`
enriches ref names when the DLL exposes it, degrades to canon-only.

## Summon flow (`runtime/sdk/spawn.lua`)

`summon(template, n, opts)`:
`bundles_for` -> `LoadNPCBundleList` per matched list
(`opts.bundle` overrides with an explicit list name; `bundle=nil` in
the report = no list matched -> `LoadDefaultNPCBundleList` fallback) ->
`SummonData` base pick -> `CloneSummonData` -> optional `Faction` set
via `db.inject` clone (faction forces the clone path; `sd=K_SD_*` in
the report means an injected record was consumed) -> `SummonNPCs`.

`summon.report` fields: `ok`, `sig`, `sd`, `bundle` (list name),
`bundles`/`lists` (matched counts), `faction`, `tmpl`, `tmpl_addr`.

## Open threads

- Live-verify the name/path-key matching finds lists for real
  templates (e.g. `Summon_OrcDefenderTough`, `Gondorian_Soldier` ->
  `Gondor` bundle) once a new-DLL session is up.
- Ambient-class placement: `AISpawnAmbientAIType` records need the
  ambient scheduler path (or a different engine entry) — not
  `SummonNPCs`. Candidate: `AISpawnManager` other methods, or the
  factory group call from the design axiom.
- Ambient-subsystem bind site (VERIFIED-STATIC 2026-09-29): the only
  non-registration consumer of the `DBScenarioSetup` accessor
  (`0x142702278`) is `0x141aac75b` inside the subsystem init fn
  `0x141aac6b4` — `0x141aabeb0(field, accessor)` resolves the setup
  list into a `{entries@0, count@8, accessor@0x10}` view at
  `this+0x100e8`; `DBScenarioCategory` (`0x142702468`) binds the same
  view shape at `this+0x48` (`0x141aabde0`). **Singleton decoded
  2026-09-30i (VERIFIED-STATIC): `this = [0x1426ffc68]`** — the ctor
  stores `rbx`→`[0x1426ffc68]` at `0x141aaa00d` immediately before
  `call 0x141aac6b4`, and the sibling reader `0x141aac623` loads the
  same global for `+0x98`/`+0x100d8` fields. No watchpoint needed —
  live probe is one `read_u64(0x1426ffc68)`: expect
  `[this+0x100f8]==0x142702278` and `this+0x100e8`=`{base,count}`.
  **+0x100e8 consumer cluster mapped**: 30 `[reg+0x100e8]` sites total;
  the same-module readers `0x141aa9f5b`, `0x141aaa387`, `0x141aab2de`,
  `0x141aac185` (all in the `0x141aa*` ambient module alongside the
  bind site) sit inside real scheduler code.
  **Scheduler anatomy (VERIFIED-STATIC 2026-09-30i)** — the ambient
  module's spawn pipeline, top-down:
  - Category loop `0x141aacf49`: iterates `ctx+0x30` (8-stride array of
    scenario-category objs), gates each via `0x141aaa9dc`, scores via
    `0x141aab9f0`, keeps the max-`comiss` winner.
  - `SelectSpawnSet = 0x141aabaf8(container)`: walks the container's
    `{base@+0x28, end@+0x30}` 0x28-stride spawn-set array; per-set
    gates (`0x1405ee98c`, `0x141abc2d8`, `set+0x259` flag) then
    `CanSpawn(this, set)`; valid sets accumulate weight
    (`set-view+0x14` via `this+0x60`) into a weighted-random pick
    (`0x140164918` reservoir sample); returns `rbp` winner or null.
  - `CanSpawn(this, set) = 0x141aab110`: reject-chain predicate.
    Rejects (each reporting via `0x141aac464` when verbose flag
    `0x142a354c7` set): position `set+0xac` within `xmm3` of another
    set's spawn point — "too close to another set" (`0x1422fe8b0`);
    "on cooldown" (`0x1422fe910`); "disabled by other running
    scenarios" (`0x1422fe950`); `0x1422fe998` family. Spawn-point
    iteration = `{x,y,z}` float triples (stride `0xc`) through
    `0x141aaa4d4`. Returns 1 at `0x141aab46d`.
  - Lifecycle: tick (`~0x141aaccf0(this)`) deactivates current via
    `0x140168110(this+0x10108)` (unlink + clear `+0x20`), re-validates
    via `CanSpawn`, then `0x141ab9a04(set)` = auto-cooldown roll
    ("Set '%s' put on auto cooldown (chance = %f)" `0x1422fecd8`) →
    `0x1405eb3c8(set+0x238, duration)` writes `{code@0, seconds@4}`
    cooldown and sets `set+0x25e`; `set+0x1cc` = cooldown timer.
    `set+0x259` = done flag; `0x142a35358`/`0x142a35806` gates.
  - Winner → `0x141a8f898(this+0x10108, set)` = activate: relinks the
    node's `+8` entry and `0x1405ee8f0` marks `node+0x20 = set` —
    `this+0x10108` is the active-sets list node, so `node+0x20` IS
    `this+0x10128` (the current set). The tick
    (`~0x141aaccf0(this)`) first re-validates the current set via
    `CanSpawn(this, [this+0x10128])` and maintains the list via
    `0x140168110(this+0x10108)` before picking new winners.
    `set+0x248` (word) indexes the usage counters at
    `[[0x1426ffab0]+0x778]+0x30` `{used@0x180, cap@0x190}`
    (decremented via `0x140713ed0` on cooldown reporting, flag
    `0x142a354c6`). `set+0x1b8`/`+0x1c0`/`+0x1f0` = per-spawn
    bookkeeping args — `0x141a91280(rbx,…)` forwards them to
    `0x141aaa648(ambient,…)` which increments usage counters in the
    `ambient+0x60` weight map (`0x1405ee9d8` slot → `inc [+0xc]`),
    keeping the weighted pick fair. Candidate categories
    live at `this+0x30`/`+0x38` (8-stride ptr array).
  - **Spawn-request manager (VERIFIED-STATIC via RoamingCaptains
    v1.1.0, 2026-09-30j)**: `[0x1426ffbe8]` = the native spawn/request
    manager — queue head `mgr+0x98` (intrusive list, node links
    `+0x08/+0x10`), request-node vtable `0x141fb3f10` with **`[vt+8] =
    0x141a927f8` = the physical-spawn routine** (`+0` = `0x141a9118c`
    cancel). Node layout: `+0x38` member, `+0x40` position,
    `+0x5c` spawn-profile name (0x20-byte bounded string —
    `"AmbushNPC"`, `"AutoInfightNPC1"`), `+0xa0` completion callback,
    `+0xa8` its arg, `+0xac` mount byte; ctor `0x141a90560`,
    enqueue `0x141a91360`, unlink `0x1401b3678`.
  - **Enqueue ABI (VERIFIED-STATIC 2026-09-30k)**: `0x141a91360(mgr,
    member, pos*, profname*, cbobj*, cbarg32, mount8)` — allocs a
    `0xb0`-byte node (`0x1403de238`), calls ctor
    `0x141a90560(node, member, pos, profname, cb, arg, mount)` which
    sets vt `0x141fb3f10`, self-links `+0x08/0x10`, ctor's the
    `+0x18` sub-object from `member` (`0x140597e84` → `0x140597f00`:
    `node+0x38 = member` AND splices `node+0x20` into
    `member+0x08`'s list — the member tracks its pending requests),
    copies pos → `+0x40`
    (`0x1407fd96c`), profname → `+0x5c` (`0x1402ff060` →
    `0x1402ff0b0` = bounded strncpy, 0x20 cap, **NULL-safe** →
    empty name = engine default profile), cb → `+0x80`
    (`0x141a9028c`), `+0xa8` = arg32, `+0xac` = mount; then
    tail-pushes `node+8` into `mgr+0x98` via `0x141161ed4`. Arg
    formats: `pos*` = vec3(0xc) + extension tail (`0x1407fd96c` →
    `0x1407fe6ec`); `profname*` = cstr of the DOG spawn profile.
  - **Queue drain (VERIFIED-STATIC 2026-09-30k — the missing
    dispatch)**: mgr tick `0x1401646a8` → `0x1401647cc(mgr)`
    walks the queue: `mgr+0x98` self-linked → empty; else
    `node = [mgr+0xa0]-8` (head.next) → `0x140606bf0(mgr, node)`
    dispatches: gate `0x14115aeb4([mgr+0x1d8])` (AI-registry
    enabled-check on the mgr's registry sub-object `+0x1d8`) →
    `rsi = 0x141aaad58([mgr+0x1d0])` (ambient spawn-slot alloc —
    iterates the ~200-slot pool `ambient+0x98..+0x100d8`, stride
    `0x148`, free = `{+0xe8, +0xcc, +0xcd, +0x128} == 0`; null →
    `[node.vt+0x10]` couldn't-spawn handler, ret 0) →
    **`call [node.vt+8]` = `0x141a927f8` THE SPAWN** → ent →
    `rsi+0xcd=1` + `0x14115d1b4(rsi, ent)` ambient-slot
    registration → `0x140607030(mgr+0x1d8, ent, 0x14114a48c(ent))`
    registry insert → `0x140167c58` bookkeeping → `mgr+0x60`
    track + `0x14037eaa4([mgr+0x68]-0x40)` AI registry. On
    return, `[node.vt+0x18]` cleanup-pred → `[node.vt+0]` =
    `0x141a9118c` frees the node. Node vtable `0x141fb3f10` slot
    map: `+0` cancel/free, `+8` spawn, `+0x10` reject handler,
    `+0x18` cleanup pred, `+0x20` bookkeeping tag.
    Engine enqueue sites (VERIFIED-STATIC): Ambush requester
    `0x141c7e63f` ("Ambush = %s; Requesting to spawn faction
    member %s" → `[FACTION|AMBUSH]`, prof `"AmbushNPC"`) and the
    infight-pair spawner `0x141a8586e`/`0x141a858a6` (two members,
    profs `"AutoInfightNPC1"`/`"AutoInfightNPC2"`, mount=1).
    This is the natural placed-captain drive:
    `callargv(0x141a91360, mgr, member, pos, "AmbushNPC", cb=0,
    arg, mount)` enqueues a fully native request the engine
    drains itself on the next mgr tick.
  - **Physical-spawn call graph (VERIFIED-STATIC 2026-09-30k —
    `0x141a927f8(node)` disassembled)**:

    ```
    member  = [node+0x38];            null → return 0
    service = [[0x1426ffab0]+0xf58]   ← world spawn service (the DOG
                                      factory endpoint — same
                                      0x1426ffab0 global the ambient
                                      scheduler uses for counters)
    kind    = 0x14115f788(member)     member's kind descriptor
    ent     = 0x1401036d0(service, &profname@+0x5c, &pos@+0x40, kind,
                          member, 1, 1, 0)   ← THE spawn call — DOG
                          entry: builds req@rbp-0x60, resolves
                          profname → descriptor@req+0x28 via
                          0x14043daa0/0x1401045d4, materializes via
                          [[0x1426ff058]]->vt+0x40 (DOG service
                          singleton) at 0x1401038f0
    0x140274a04(ent)                  liveness gate (sibling of the
                                      ambient post-spawn 0x1402749b0)
    ent+0x26a8 → comp+0x692 = (node+0xac != 0)   mount flag
    ent+0x26f0 = 0x140168da8(ent,0,0) nav/attach
    0x14035dc38(ent, 0x140607610(ent))
    0x141aa20f4(member, ent)          finalize:
        0x14040ed18(ent, member, 1)   member adoption — entity takes
                                      member+0x174/0x178 list ctx,
                                      links member's +0x210 sub-object
                                      array into ent+0x2d28, writes
                                      ent+0x2338, ent+0x2738
        0x140766f78(fm=[world+0x6c68], member)   roster notify
        ent+0x26a8 → 0x140351580(comp, member+0x4b8 + member+0x470)
                                      rider accounting
    cb = [node+0xa0] → cb->vt+8(ent, [node+0xa8])
    return ent
    ```

  - **Node `+0x80` = task/callback sub-object (VERIFIED-STATIC
    2026-09-30m/q)**: `0x141a9028c` constructs it in place — vtable
    `0x141fb3ed8`, self-link `+0x88/+0x90`, back-ptr `+0x98`,
    `node+0xa0` starts 0; the enqueue's `cb` arg forwards into
    `0x1417a4390(cb, emitter)` which **splices the cb's `+0x08` list
    node into `emitter+0x08`'s child list** and stores
    `cb+0x20 = emitter` — i.e. `node+0xa0` IS the emitter back-ptr
    (the "completion callback" `cb = [node+0xa0] → cb->vt+8(ent,
    [node+0xa8])` in the spawn routine fires the emitter's vt+8).
    **Null emitter is legal** (VERIFIED-STATIC 2026-09-30q): a third
    enqueue site `0x14079337e` guards `[obj+0x2fc0]` with
    `neg/sbb/and` → passes cb=0 when the owner is absent — and uses
    a different queue (`[[0x1426ffab0]+0x778]+0x30`), so
    `0x141a91360` is a generic request enqueue across managers.
    The Ambush requester's post-enqueue `0x14018bdb0(rdi+0x90,
    member+0x498)` is gated behind debug flag `[0x142a34db4]` and
    feeds only the dead `[FACTION|AMBUSH]` print — **not** required
    registration. The requester also pre-builds a nav/position
    descriptor via `0x140573324` + `0x1402b2a8c` (pos vec AND-masked
    `0x141e618b0` lane-clear), sets `task+0xa4=1`.
    **Deferred-assert hypothesis (revised)**: emitter-less nodes are
    legal, so the ~60 s post-drain assert (`exe+0x1a6f33` →
    `0x1402ffe08` float-range comiss vs `0x1426c6570`) more likely
    tracks the spec/ctx (`rdx` = canonical `[emitter+0x38]`) or the
    +0x18 member-registration sub-object rather than the missing
    emitter link.
    **Canonical drives (VERIFIED-INGAME 2026-09-30n)**:
    (a) `Faction:TriggerAmbush(DBFactionAmbushDef.<def>)` — script
        method (colon → 1 explicit arg, a `Faction/Ambush/Definition`
        record). Gate-free via `sdk.exec.eval`.
    (b) console `Faction.TriggerAmbush <def-name>` — program-record
        node; `sdk.console.exec` routes it through the registry
        `vt+0x58` exec verb (`0x1415cff44`, see
        docs/model/console-commands.md). `Faction.Ambush.AlwaysPassChance
        <def>` forces the roll.
    Both fire the engine's own task → canonical enqueue → drain →
    materialize. Ambushes defer on `AmbushDelay`/chance/conditions —
    trigger ≠ immediate spawn (no slot claim within 60 s for
    `MarkedAsTarget_Ambush`/`PC_SummonBodyguard_Ambush`; the def's
    context predicates gate it).

    `member+0x48` = **actor link** (back-key `actor+0x2708`) — the
    finalize path's member-adoption writes it (RC live-verified the
    field). Actor→member resolver `0x14018d920(actor)` → member
    (fully decoded): `reg=[actor+0x2708]` = registration record
    `{+0x08: roster index (-1 → NULL = unregistered), +0x18:
    map-owner tag}`. `0x14018de04` validates `reg+0x18 ==
    [fm+0x98+0x10]` (owner tag; mismatch → fatal `0x14109e110`),
    derives `idx = (reg − poolbase)/0x20` (`0x1401898e8` — reg
    records live in a 0x20-stride pool parallel to the roster;
    poolbase `[owner+0x38]`/`[+0x48]` by `0x140189940` branch),
    direct-indexes the 0x290-stride roster `[fm+0x98]`, checks
    `[slot]==reg`, returns `slot+8`; `0x14018ddd4` then yields
    `member=[slot+8+0x20]` (NULL → unplaced). O(1) — `member_for_char`
    uses it as the fastest path under `NEMESIS_UNSAFE_CALLS`,
    falling back to the roster `+0x48` scan when the gate is closed.
    Nav placement: `entity+0x2528` nav comp →
    `vt+0x130` navobj → `0x140f2bc80(navobj,pos,out)` project.
    Position fetch `0x140180458`. RC's hook C on `0x141a927f8`
    confirms the node semantics live — RC schedules via
    `0x141aa2414(arg3=-1)` then `0x141aa20f4`.
  - **Materialization (VERIFIED-STATIC 2026-09-30i, closes the
    set→NPC link)**: the spawn-execution object `rbx` **IS the
    request manager `[0x1426ffbe8]`** (one object: queue `+0x98`,
    budget `+0x180` cap / `+0x190` spawned — `0x141aab4dd` reads
    those exact fields on `[0x1426ffbe8]` for the "already at max
    AI" report; plus `+0x1a8` active list, `+0x1c8` pending set,
    `+0x1d0` → ambient `this`, `+0x60` tracking map, `[+0x68]-0x40`
    AI registry). `0x141a93302(mgr)` = "spawn one AI from current
    set": gate `0x1402f9f90` + `0x141b50a68(mgr+0x198)` →
    `0x141aab904(mgr+0x1d0)` = current-set accessor (indirection
    over `+0x10128`) → `0x140104970(set)` = per-set materialize =
    `call [[set+0x38].vt+0x120](handler, set)` → spawned obj →
    `call [set.vt+8](set, obj)` post-spawn notify →
    bookkeeping/report/track/register/`+0x190++`. (Two spawn
    styles coexist: the `+0x38` handler's `vt+0x120`, and the
    descriptor-array elements' `vt+0x128` below.) The sync at
    `0x141a93475`: when `rbx+0x1c8` is empty it pulls
    `[this+0x10128]` into its list via `0x141a8f898(rbx+0x1a8, set)`.
    Then `0x141abc5f4(set, &it)` iterates the set's descriptor array
    — `{begin,end}` at `set+0x130`/`+0x138` (NOT `+0x1b8`; `+0x1b8`/
    `+0x1c0`/`+0x1f0` are bookkeeping args to `0x141a91280`), element
    name at `[elem]+0x7d`, output at `set+0x2f8`, presence flag
    `set+0x256`. Per descriptor **`call [rdi.vt+0x128]` IS the NPC
    spawn** (vtable call — descriptor polymorphism = member-builders
    / AI-type records). Gate `0x141a91c44(rbx)` = remaining AI
    budget `rbx+0x180 − rbx+0x190` (cap − spawned count; 0 →
    "Spawner set '%s' unable to spawn - already at max AI"
    `0x141fb3bf0`); the tick itself is gated by `ambient+0x10100`
    enable flag (`0x141164248`). Success reports "Spawned AI %s
    from Spawner %s (%s)" (`0x141fb3dd8`) and registers the object
    (`0x14037eaa4([rbx+0x68]-0x40, ai)`, `inc [rbx+0x190]`).
    `0x1402749b0(obj)` = liveness gate — `obj+0x78` membership in a
    `+0x8`-chained list bounded by sentinel `0x1426785f0`
    (`0x140274a04` walks it; returns obj only if still registered —
    i.e. `vt+0x128` hands back a handle that becomes live on
    registration, consistent with a factory-created DOG object).
    `rbx+0x1a8` = its own currently-spawning list node,
    `rbx+0x1d0` = ambient `this` (written via non-mov init — SSE or
    block copy; direct-store AOB finds only unrelated classes).
    Spawn tick `0x141a93302(rbx)` — head `0x141a932f8` — called from
    `rbx`'s tick `0x1401646a8` (caller `0x140164746`, predicate
    `0x141164248` on the ambient ptr), itself reached via a parent
    subsystem at `parent+0x30` (`0x1401f73e0`) inside the world
    object sweep (`rsi+0x331`/`+0x348`/`+0x438`/`+0x540` cluster).
    Completion: when the set is done, `0x141aac0c8(ambient, set)`
    reports "Finished spawning from %s" (`0x1422fe648`,
    `set+0x40` = set name) and tail-jumps
    `0x140168110(ambient+0x10108)` to unlink the set from the
    active list.
    - Diagnostic: **`FactionMember.WTF <FortUpgradeGroup>`** (`0x141c5be28`,
      arg resolves via DBFortUpgradeGroup `[0x142701c38]` — a
      `Faction/FortUpgrade/Group` name, not a member name; corpus
      names: `Assault_Lvl_01..06`, `Defense_Lvl_01..05`,
      `Defense6_Battlefield1_PoisonMines`, …),
    code-handler — `sdk.console.exec`-able) = the engine's own
    "why isn't this member spawning" reporter: iterates a
    `0x1402774a8` candidate collection, per element resolves the
    current target via `0x14018de04` and runs gate evals
    (`0x141bf7088` list-membership on `body+0x40`,
    `0x141c23e68`/`0x141c1dd44`/`0x14062eb14` member-state,
    `0x1419c3f3c` on `body+0xd8`) — output via `[FACTION|BUNDLE]`
    channel (`0x1419c83ac`, verbose flag `0x142a34db6`).
  - Live probes now one-liners: `read_u64(0x1426ffc68)` = this;
    `[this+0x100f8]==0x142702278` confirms; `this+0x10128` = current
    set, `[set+0x40]` = its name cstr. **NOTE**: `[0x1426ffc68]==0`
    at the title screen — the singleton is world-scoped, constructed
    on session load (probe only in-world).
- Member-builder load path (VERIFIED-STATIC 2026-09-29): non-registration
  `DBFactionMemberBuilder` (`0x142701be8`) consumers are field-loader
  fns around `0x141bc1565`/`0x141bc2181` — named-field reads
  (`0x141879468`/`0x141879554`) populate a record object (`rdi`) and
  store the builder link at `record+0x3a0` (+ flags `+0x51b`, `+0x520`).
  The placed-spawn trigger would consume that `+0x3a0` link — pairs
  with `Faction.Load` (`0x141c5b960`) as the closest native
  member-placement entry.
- `RemoveBundle` semantics unprobed (presumably unload-on-exit).
