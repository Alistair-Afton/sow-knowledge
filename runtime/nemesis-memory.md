# Nemesis memory — the CharacterEvent system
> **Status:** **partial** — Project Nemesis source doc; per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE / VERIFIED-INGAME / MODEL / UNVERIFIED) preserved inline. Not independently verified by Andúril.

> **Provenance:** Project Nemesis runtime-model documentation (native `nemesis.dll` live-game forensics), mirrored and maintained in the corpus. Their `VERIFIED-STATIC`/`MODEL` tags carry over; unverified entries are marked inside each doc.


How Uruks "remember" encounters (G49). Evidence: extracted gamedb +
exe name-table strings. Status: VERIFIED-STATIC for the read side;
writer path UNVERIFIED.

## Model

Per-character event log — each captain/nemesis carries a native event
log the audio/dialog/behavior layers query by event name:

| Script accessor (exe name table) | Purpose |
|---|---|
| `CharacterEvent(char, type)` | event present? (the GameCondition cell6 `"CharacterEvent"` query) |
| `CharacterEventCount(char, type)` | how many times recorded |
| `CharacterEventLastPlayerEncounter(char)` | timestamp/recency of last player encounter |
| ~~`AddEvent`~~ | ruled out 2026-09-25 — every `AddEvent` in the exe is `CUIHUDEvent<T>::AddEvent` (HUD queue), not this system |

**VERIFIED-INGAME 2026-09-25:** none of the CharacterEvent* names
exist as world-VM globals — `eval` reports `CharacterEvent=nil …
AddEvent=nil` — they're **predicate-evaluator builtins** (the
GameCondition cell6 engine), not script functions. Querying a
captain's log needs a different path: object methods on the member
struct (probe `cap[...]` fields), the predicate-evaluator itself, or
the native log directly. Faction's event pipeline *is* script-
accessible (`Faction.AddFactionEventJobSourceAndOther`,
`DEBUG_AddFactionEvent`, `ApplyFactionEvent`, `GenerateSimEvent*`,
`IsEventOnEventList` — the sim-event side is alive).

Writer path remains unlocated — the accessors sit in a read-only
predicate table (neighbors: `IsAllyToPlayerCharacter`,
`CharacterCombatBuff`, `CharacterOcclusion`,
`CharacterAlertnessRecord`…), so the writer lives in a different
namespace or is native-only. One useful live lever: console command
`Presentation.CharacterEvent` ("applies the specified relay to the
specified character", `PlayerPresentationMgr.cpp`) — fires a character
event on demand for before/after log comparison. Its handler at
`0x1469d47e0` walks `g@0x1426ffa98 → +0x888 → +0x2fc0` then calls
`0x140679480(obj, 0x14)` — the `+0x2fc0` sub-object is a candidate
for the per-character event list itself (VERIFIED-STATIC).

`AudioHook.GameCondition` records bind to it declaratively: cell4 =
event name, cell6 = `"CharacterEvent"`. 57+ condition records exist
(`CharEvent_*`, `CharEventCount_*`, `TimesHumiliatedPlayer*`,
`DLG_*`) — the *query* side is fully shipped and wired into dialog,
music, and behavior.

## Event vocabulary (34 types, harvested from GameCondition records)

Encounter memory: `ActiveFlee`, `ActiveDespawnWhileFleeing`,
`ActiveInitialPlayerThreat`, `ActiveDeathThreatEscaped`,
`ActiveDeathThreatKillPlayer`, `ActiveKillPlayerAssist`,
`ActiveKillPlayer_AlreadyNamed`, `ActiveKilledByPlayer`,
`LastPlayerEncounter` (implicit via the recency accessor),
`Encounter` recency variants.

Terror memory — **the reported regression area**:
`ActiveTerrorTriggered_{Fire, Flies, Goblin, Infighting, Nemesis,
Troll, Warg}` — the captain remembers what scared it.

Shame/domination: `Domination_Shame`, `Domination_Shun`,
`ShamedMultipleTimes`/`ShamedOnlyOnce` (counts).

Resurrection cause memory — `WillResurrect_{Caragor, Fall,
FireNormal, Ghul, OrcSlash, PlayerArrow, PlayerSlash, Troll}` —
how they died determines their scar/undead state and comeback dialog.

Faction sim: `BossKilled`, `BrotherKilledByPlayer`,
`DefendedFortFromPlayer`, `FortDefenseMoveIntoOverlordWarchief`,
`HenchmenKilled`, `IsPlayerFollowerApply/Remove`,
`PromotePlayerKiller_To_WC`, `SimDeathThreatExpired`,
`EventDeathThreat`, `BetrayalTriggerEvents`, `ShamedTriggerEvents`.

Adjacent systems: `Combat.ScarType`/`Combat.ScarSetup` lists,
`NemesisForge.FirebirdModelScarType`, `Trait_Quirk_Humiliator`,
`Faction.SimEvent*` (world-level event generation — separate).

## Hypothesis (user report, UNVERIFIED)

Later game builds stopped *recording* most event types — only
kill/run-away still land. Readers (`CharEvent_*` conditions) then
never fire → encounters feel repetitive; scare-by-caragor/flame etc.
produce no lasting memory. Consistent with a writer path that was
gutted or gated, not missing data: the whole vocabulary + consumers
still ship.

Two possible break sites:

1. **Script side** — flee/terror controllers (`_fleecontroller_SO`,
   terror behaviors) may have had their event-recording calls removed
   in a patch. Check: script diff vs earlier build if obtainable, or
   live-verify the log stays empty after scaring a captain.
2. **Native side** — the combat/flee state machine stopped calling the
   event writer for all but death/flee-end.

## "Broke elsewhere too" — sibling condition families

Same GameCondition census, families that query *history/sim* state
(most likely to share the dead-writer class) vs live state (safe):

| Family | Records | Risk |
|---|---|---|
| `CharacterEvent` | 43 | the reported break |
| `CharacterActiveSimEvent` / `GlobalActiveSimEvent` | 60 | sim-event generation may share the gutted path — worth a live census |
| `Quest`/`QuestComplete` | 156 | quest engine — probably fine, verify |
| `CharacterTraits`/`PersonalityType`/`TraitMarkers` | 271 | live state — reads only, low risk |
| `CharacterHasInventoryItem`/`Weapon`/`CombatState`/… | rest | live state — low risk |

The follow-on sweep once `CharacterEvent` is mapped: `Faction.SimEvent*`
generation (ambushes/duels/betrayals happening without the player) —
if that pipeline also stopped firing, the world loses its autonomous
nemesis activity, which matches the "interactions feel repetitive"
report from the other side.

## Relay bitmask (VERIFIED-STATIC, 2026-09-28)

`Presentation.CharacterEvent` is decoded: the command's handler
(`0x1469d47e0`, zero arg registers — safe for the Nemesis plugin API)
walks `[[0x1426ffa98]+0x888]+0x2fc0` and calls
`0x140679480(obj, 0x14)` → `0x14067949c(target, 0x14)` which does
`target->+0x6c |= 1<<0x14` (bit 0xC excluded). So the command sets a
**presentation-relay bit**, it does NOT append a CharacterEvent log
entry — the relay mask (`target+0x6c`, 32 bits) is a sibling mechanism:
the bit-setter has 27 writer call sites (ids 1,2,3,5,7,0x10,0x11+,0x14
seen), and the "specified character" is the AIInfo-tracked subject
(`AI.InfoTrack.Subject`, cvar-id command `0x101`). The mask consumer
(who reads `+0x6c` to play scar/terror presentation) and the
bit↔event-name map are undecoded — a live probe verifies the
chain live and tests whether the handler call flips bit 0x14.

**Better writer levers (same decode pass)**: `Presentation.QueueEvent`
(`0x1417a5c04`) takes `(event-name, target-name, …)` as argv strings,
resolves them live through `0x1426ff058` vtable+0x68, and queues the
event on the same `+0x2fc0` object — firing
`ActiveTerrorTriggered_Fire` on a named captain is directly testable.
`Presentation.StartEvent` and `Presentation.ScriptRelay`
(`relay + character` args) are siblings. All are invocable via
the Nemesis plugin API (the universal `handler(argc, argv)` console ABI —
docs/model/console-commands.md).

## `Faction.AddEvent` — the strongest writer candidate (VERIFIED-STATIC, 2026-09-25)

`Faction.AddEvent` (second registration, `0x141c5b898`,
`FactionConsolePrograms.cpp:1100` — "Apply a Faction.Events.Definition
record to the current faction target"):

1. `argv[0]` → `0x141b090bc` — hashes the name and looks it up in the
   `[0x142701c48]` record table → **DBFactionReward** entry (corrected
   2026-09-29: this is `Faction/Rewards/Definitions`, NOT the event
   list — `Faction.AddEvent`'s own arg really is a reward-style
   record, or the builtin resolves elsewhere for real events).
   Internals (decoded 2026-09-28): `0x140423e70` namehash →
   `0x140278540(array, &key, count)` bsearch over the accessor's
   `0x28`-stride array (`u32 count @ +0x28`, `array @ +0x38`).
   **VERIFIED-INGAME 2026-09-29**: replicating that pair against the
   real `DBFactionEvent` accessor `[0x142701a20]` resolves
   `ActiveTerrorTriggered_All` → `0x435f2c38` == its
   the Nemesis plugin API("Faction.Event")` `e.addr` — the pending-vector
   element pointer the writer expects. Pure-Lua repair resolve:
   `key = callva(0x140423e70, namep)`;
   `entry = callva(0x140278540, arr, &key, count)` — arg order is
   `(array, &key, count)`; swapped args crash at `exe+0x278560`
   (bsearch bound = the key *pointer*).
2. `[0x1426ffaa8]+0x6c68` → `0x14016a550` — the **current faction
   target** member object (set via `Faction.SetTarget`, the
   interrogation/army-screen selection). Internals: keyed lookup
   `0x14018de04(ctx+0x98, ctx+0x1b8)` — same `+0x98` keyed-container
   family as the member-side script-event state.
3. Optional `argc==3` tail: `argv[1]`/`argv[2]` ints →
   `0x14016e128(member+0x1b8, …)` — sets params on the target.
4. `0x1403511f4(member, element, 0, 1)` → `0x140351af8` (member subobj)
   → `0x140351220(member, event)` — **the event write** (corrected
   2026-09-30: the handler's own tail is `r9b=1` — mark-unviewed; the
   `(0,0)` shape is the commit-drain's replay at `0x1407a61d5`. arg2 =
   the accessor element the pending vector's 8-byte slots point at):
   `member+0x4b8` = per-type event counter (indexed by the event's
   type from `0x14035136c`), `member+0x470` = log base/offset,
   `member+0x474` = `1` (has-new-events flag), `0x14035188c` updates
   member state.

The `member+0x4b8 + event_type` counter layout matches
`CharacterEventCount(char, type)` semantics exactly — this is almost
certainly the same per-member log the predicate builtins read.

**Selecting the target** (`Faction.SetTarget`, `0x1417c0c40`): takes
**3 int args** (hierarchy row/col/slot — NOT a name), resolves via
`0x141b09818`, stores through `0x14016e128` on the SAME
`[0x1426ffaa8]+0x6c68` object. `Faction.Dump <file>` writes all
members (with grid positions) to a file — the offline way to get
real positions. `FactionMember.ProfileDump` dumps the selected
member for before/after verification.

Live test staged in a live probe: `Faction.Dump` →
`Faction.SetTarget <row> <col> <slot>` → `FactionMember.ProfileDump`
→ `Faction.AddEvent ActiveTerrorTriggered_Fire` → re-probe counts.
If counts move, the writer regression is NOT in this path — the
break is upstream (gameplay never calls it for terror), and the fix
is a hook that calls `Faction.AddEvent`'s path (or `0x1403511f4`
directly) on terror/flee state transitions.

**Writer call-site census** (E8-scan, VERIFIED-STATIC): every
`0x1403511f4` (AddEvent wrapper) / `0x140351220` (event write) /
`0x14035188c` (member-update) caller mapped:

- `0x14061a05a` — **bulk replay**: iterates an event list
  (`0x14034f980` = count) writing each to one member — the
  EventReplay/simulation path, not live gameplay.
- `0x1404f932a` — **second bulk path**: iterates a stride-0x38
  descriptor list (`0x1404f8b58` iterator), resolving each element
  via `0x1404f8478` then `AddEvent(rsi, ev, 0, 1)` — same
  import/replay class as `0x14061a05a`.
- `0x1407a61a4` — hierarchy/member-list update (member+0x1c8 list,
  `+0x474` flag set) — simulation bookkeeping.
- `0x140631006`, `0x141bc0b79`, `0x141bf0dae` — faction-console /
  replay-adjacent code.
- `0x140f7cae7`, `0x140f7cb67` — **direct `event-write` calls from
  Themida-virtualized code** — the live gameplay writers (kill/flee
  at minimum). Undecodable statically, but they prove live paths DO
  still reach the writer — consistent with "some events record".
- `0x141c5b927` — the `Faction.AddEvent` console handler itself.
- ~16 `member-update` callers across `0x141c1-0x141c5`
  (FactionConsolePrograms) + gameplay `0x140619fbf`, `0x1406308xx`.

So the write machinery is intact and exercised — matching the
regression hypothesis: specific *callers* for terror/flee events
were lost, not the writer itself.

## Event pipeline — full decode (VERIFIED-STATIC, 2026-09-26)

The member event path is a three-stage queue, not a direct write:

1. **Produce/apply** — `ApplyEvent = 0x140351af8(member, event)`:
   runs the `Faction.Event` record's side-effect fields (each
   `0x140350xxx` getter → conditional member update via
   `0x141bc*`/`0x1407bb8c4`/`0x1403528a0`), then eligibility-checks
   (`0x14035136c`/`0x1403512fc`/`0x140350424` — record-field reads),
   snapshots the viewed index `member+0x4b8` → `member+0x4c0`,
   **pushes the event-record ptr into the pending vector**
   `member+0x1c8` (base@1c8/end@1d0, push helper `0x14115ff60`),
   sets `member+0x51c`, and fires the UI notify `0x1403511bc`
   (writes through UI ctx `0x1426ffb08 +0x7e8`).
   Callers: `0x14034fa40` (applies the member's bound `EventList`s —
   `member+0x1f8` list), `0x140351204` (internal), and the
   **virtualized gameplay producers `0x140f7cadc`/`0x140f7cb5c`**
   (live kill/flee/terror triggers).
2. **Commit** — `0x1407a6118(member)`: collects extra context from
   `member+0x190`, marks `0x14035188c`, sets `member+0x474=1`
   (has-unviewed), drains the `+0x1c8` pending vector into the
   persistent log via `0x1403511f4(member, ev, 0, 0)` per element,
   then vector-clears (`0x1407a6264`) both `+0x1c8` and the second
   vector at `member+0x358`. Called from `0x1403fadec` inside the
   member serialize/state-commit path — **the commit may only run on
   member save/state transitions**, so live terror events could sit
   in `+0x1c8` without reaching `+0x470`.
3. **Persist** — `0x1403511f4(member, event, ctx, markUnviewed)`:
   calls `ApplyEvent` then `0x140351220(member, event)` — the log
   write. The log is a **circular buffer**: `0x14035136c(event)` →
   type id; per-type counters via the `0x141bb*` accessor family
   (`0x141bbdbe8` gate, `0x140278758`/`0x141169594` element writes);
   `0x14114e668(count, cap-1)` wraps the index;
   `0x14035188c(member, wrapped)` updates head/count (`+0x4b8`,
   capacity from `member+0x188` via `0x1403517e0`, flags `+0x508`
   bit 0x1b, `+0x190` suppress check, `+0x474` unviewed flag).
   A second sub-record (`0x1403512fc`) triggers a paired write at
   `+0x470 + count` via `0x140795770`.

## arg2 contract — decoded 2026-09-30 (VERIFIED-STATIC, supersedes
## the "entry vs data" guessing)

The drain (`0x1407a6118`) walks `member+0x1c8` at **8-byte stride**
(`add rbx,8`; predicate `0x14050bfbc` = `iter!=end`; vector-clear
`0x1407a6264` does `sar rax,3`) calling `0x1403511f4(member,*slot,0,0)`
per non-null slot, and the push helper `0x14073b9dc` writes **one
qword** per event. So the pending queue holds **pointers to live
event-record objects**, not entries and not record data.

`arg2`'s required shape (from `0x140351af8` → `0x1407bd518` →
`0x140274ae8`, plus `0x14035136c` → `0x141169594` → `0x14025180c`):

```
event-record object (0x28 bytes, matches live pending records):
  +0x00  u32 id (observed 0xa01 on OutpostBoost_* records)
  +0x04  u32 name-key
  +0x08  items-array ptr — 16-byte groups {count@0xc, base-idx@0xe}
  +0x10  u32 bitmask/array ptr
  +0x18  shared ctx ptr
  +0x20  name cstr
```

`0x14025180c(v)` = the engine's generic value→typedesc resolver
(`byte[v+1]&1` → inline/alt path `0x14025185b`, else payload `[v+8]` →
`0x140251bd0`); `0x141169594` accepts only typedesc tag `+0xa&0x3f==3`.
This is a reusable "is this a valid engine object" primitive for any
native-call argument validation.

## Write contract — VERIFIED-INGAME 2026-09-30f (closes the crash series)

The earlier `exe+0x274afd` crash was **not** a record-subclass problem:
the hand-rolled namehash+bsearch replication landed on an **empty
zeroed slot** (`k0=0, items=0`) because `[0x142701c48]`'s bsearch array
is not the list's element array. The engine's own resolver
`0x141b090bc(name-cstr)` returns exactly the the Nemesis plugin API element
addr (proven live: `ResReward_Terror_Caragor` → `0x45df2160`).
Resolution is now pure the Nemesis plugin API("Faction.Reward")` name lookup —
no native calls needed.

The canonical write is `callva(0x1403511f4, member, element, x, y)` —
args 3/4 are ignored by the callee (only rcx/rdx are forwarded to
`ApplyEvent` + the `0x140351220` log tail). This is byte-identical to
the virtualized gameplay producers (`0x140f7cadc` = `ApplyEvent` +
`0x140351220` per element). Verified live on roster member
`0x129aeb960`:

- `ResReward_Terror_Caragor`: ApplyEvent ran (`member+0x51c` notify
  flag set, rax=0x100), no crash.
- `Reward_FromFlee Infested_RemoveCopy`: consumed `member+0x2e0`
  container — the record's effect applied.

**`member+0x1f8` is NOT the committed log** — it holds bound
EventList elements (`Default_Act1`, `Traits_Defender` …). The
persistent log is the per-type circular buffer written by
`0x140351220` through `0x14035188c(member, wrapped-idx)` with
counters `+0x470`/`+0x4b8` and dirty flag `+0x474`.

**Type gating**: `0x14035136c(el)` (the discriminator) =
`0x140278758(el, cap=global[0x14266bdb0]+0x1000=51)` —
`items[51].count>0` gate → `mask[items[51].base]` u32, nonzero =
loggable type. Only **9** Faction.Reward records qualify
(`Default_OrcRarity_*` ×8 + `Reward_EscapeUltimate`). For all other
records `0x140351220` no-ops — terror/flee/scar memories persist as
ApplyEvent **side effects on member fields**, not log entries.
Similarly `0x1407bd518`'s first gate is cap `global+0xa18` = **30**;
only 22 records carry it (TransformRole/ChangeRole/PinnicleScar +
`Reward_FromFlee_TransformRole_Infested`).

`0x140278758(obj, idx, sub)` semantics (VERIFIED-STATIC): items row
`obj+8 + idx*0x10` → `{count@0xc u16, base@0xe u16}`; requires
`sub < count`; returns `u32 mask[(base+sub)]` where mask=`obj+0x10`
(u32 dwords at 8-byte stride).

**Every crash came from a wrong arg1, not arg2.** `looks_member`
accepted any object with a plausible vector at `+0x1c8`; the feared-
edge resolution was landing on **GameDB record-bearing objects**
(member dossier / applied-boost containers) whose `+0x1c8` holds
inline `0x28` record arrays — `{id,key}` pairs, shared ctx ptrs,
`OutpostBoost_*` name cstrs — not event-ptr queues. `looks_member`
now requires every non-null pending slot to be a readable pointer
(`read_u64` guard), which record arrays fail on slot 0.

**arg2 = the accessor ELEMENT of a `Faction.Reward`-family record**
(VERIFIED-STATIC 2026-09-30, from `Faction.AddEvent` impl
`0x141c5b898`): `0x141b090bc(name)` hashes the name (`0x140423e70`)
and bsearches `DBFactionReward` `[0x142701c48]` (`count@+0x28`,
`array@+0x38`, `0x140278540(array,&key,count)`), returning the
element ptr; the handler calls `0x1403511f4(member, element, 0, 1)`
(arg4=1 = mark-unviewed — the drain's own `(0,0)` tail is the commit
replay shape). The element's `{items@8, mask@0x10}` fields satisfy
validator `0x140274ae8`. A `Faction.Event` element has `+0x08`=small
scalar (observed `0x2702` — crash) — wrong record class entirely.
The pending `OutpostBoost_*` records ARE such elements: pending-queue
slots are 8-byte ptrs to accessor elements.

**The persisted memory records ARE `Faction.Reward` definitions**
(`Faction/Rewards/Definitions`): `Reward_(Fear|Terror|Hate)_*` are the
transition memories ("gained Terror of Caragors" =
`Reward_Terror_Caragor`), `EW_*` the weakness definitions
(`EW_Fear_Burning`, `EW_Terror_Fire`…), `Reward_FromFlee` the flee
memory, `Reward_PinnicleScar_*` shame scars. The `Active*Triggered_*`
`Faction.Event` records are the notification/recent-event names — they
land in `member+0x110` via script `AddFactionEvent`, which the
`CharEvent_*_Recent` predicates (`HaveCharEvent_ActiveFlee_Recent`,
audio hooks `CharEvent_TerrorTriggered_*_Recent`) query. Two memory
surfaces: `+0x110` recent-events (script path) vs `+0x470` persistent
reward log (`0x1403511f4`).

**Producers** (VERIFIED-STATIC 2026-09-30): `0x140f7caa0` drains a
bundle's `+0x20` vec of pre-built event-elements → `ApplyEvent` +
`0x140351220` per element; `0x140f7cb20` mints candidates from
`+0x38` source records via `0x140350500(member, src)` → same two
calls; `0x14034fa3e` iterates `member+0x1f8` bound-EventList buckets
(`edi < [member+0x4b8]`, `0x1403513dc` collects sources,
`0x140350500` picks the best candidate) → `ApplyEvent(member, ev,0,1)`
+ optional `0x14114c42c` cleanup. `0x140350500` walks a candidate
iterator (`0x140350f80`), filters by `0x140105aec` eligibility, and
returns the winning element — the canonical event-factory.

**Real member provenance** (`0x14016a550` → `0x14018de04` →
`0x14018ddd4`, VERIFIED-STATIC + **VERIFIED-INGAME 2026-09-30d**):
`mgr=[0x1426ffaa8]+0x6c68`; roster = keyed open-addr container at
`mgr+0x98` `{arr@0, count@8, td@0x10}`; key = pooled script udata
(`[key+0x18]==td`, hash `0x1401898e8`), current-target key `mgr+0x1b8`.
`record = arr + hash*0x290` (key@0); `body = record+8`;
`subid = u32[body+0x168]`; `inner = [body+0x20]` (a
`{vec@+0x18, end@+0x20}` container); **`member = vec[subid]`** —
and every vec element is a member of that keyed group, so walking all
inner vecs enumerates the full roster. a live probe walks it;
`key==cap.val` would make cap→member O(1)-correct.

Live-pass corrections (2026-09-30d, in-world `roster` run):

- `subid` reads `0xFFFFFFFF` on every record when nothing is selected
  — the no-selection sentinel hitting the `index>=count→0` branch.
- `inner` containers are **shared across records**: 3 distinct inners
  served 38 records (vecs of 511 / 97 / 27 elements; ~635 unique
  member objects) — keyed records are *views* over shared pools.
- **Member pointers span >4GB** (elements at `0x12A…`/`0x10A…`). The
  earlier "members are sub-4GB" claim described the accessor's u32
  handle index (`0x1403513dc` zero-extension), not member pointers.
- A member's queues: `+0x1c8` pending is `(0,0)` when never
  allocated; `+0x1f8` committed reader log is populated from spawn —
  elements `{key@+4, name@+0x20}`, observed `Default`,
  `Traits_Trickster`, `Traits_Spearman`. `looks_member` therefore
  validates BOTH queues as `(0,0)`-or-readable-ptr-vec with ≥1
  populated — the earlier `vb>0x10000` requirement rejected every
  resting member (the `roster char-link miss (0 members)` bug).

## Cross-build diff — build 8302 reference exe (VERIFIED-STATIC, 2026-09-25)

`Helpers/ShadowOfWar.exe` is build **1.0.8302.0** (installed is
1.0.8874.0), a build where nemesis memory reportedly worked. Its
embedded source paths use `C:\P4\KRAKEN_CODE_STREAM`. Both builds are
Themida-packed with shuffled section names (8302's main code section
is `.trace`, 29MB, vs 8874's `.arch`, 30MB).

Anchored via the `Faction.AddEvent` console handler
(`0x141ab1b70` in 8302 vs `0x141c5b898` in 8874 — identical node
layout): the whole pipeline maps one-to-one —

| stage | 8302 | 8874 |
|---|---|---|
| Persist (ApplyEvent + log write) | `0x1405fbb68` | `0x1403511f4` |
| ApplyEvent | `0x140600ae4` | `0x140351af8` |
| log write tail | `0x1405fbb94` | `0x140351220` |
| Commit (pending→log drain) | `0x1405fb8a0` | `0x1407a6118` |

**ApplyEvent is structurally identical between builds**: 126 calls in
the same order, and every member offset shifted by a uniform **+0x10**
(member struct gained one field; consistent with the gamedb schema
+4-fields revision). Persist has 7 direct callers and Commit has
exactly 1 in *both* builds. Live producers are Themida-VM'd in both
(`0x140f9xxxx` stub farm in 8302, `0x140f7xxxx` in 8874) — producer
logic is not textually diffable.

**Conclusion: the write machinery did not change between 8302 and
8874.** The terror-memory regression therefore lives in (a) the VM'd
producer/trigger code, or (b) gamedb bindings (EventList membership,
trigger conditions) — NOT in the writer/apply/commit path. This
eliminates the "writer gutted" hypothesis branch and makes the
live discriminant below the decisive test.

**Regression discriminant** — the two failure modes predict
different live state:

| pending `+0x1c8` | log `+0x470`/`+0x4b8` | diagnosis |
|------------------|---------------------|-----------|
| empty after scare | flat | producers never fire (upstream AI/record gate) |
| nonempty after scare | flat | enqueued but commit never runs — restore flush |

`charevents.lua` §5 now dumps both vectors + counters from the live
captain's member struct (userdata → `+0x08..0x38` member-ptr probe,
validated by vector bounds) — one scare-and-dump settles which stage
broke.

## Event-dispatch + presentation anchors (VERIFIED-STATIC, 2026-09-26)

- `0x1403511c0(argptr)` — the console-command dispatch used by
  `Faction.AddEvent`/`Presentation.CharacterEvent`: arg is
  `{+0: member*, +8: event_id}`; loads the member-event manager from
  UI-ctx `0x1426ffb08 +0x7e8`, calls `0x14034fce4`.
- `0x14034fce4(mgr, &arg, r8)` — **find-or-append in the manager's
  pending vector**: entries stride `0x40`; `0x14034fdb0`+`0x140188c80`
  validate the arg, `0x14016fd3c` is the per-entry match, miss path
  constructs `0x1417ed258`/`0x1417ed554`, appends via `0x1417f00fc`.
- `ApplyEvent` (`0x140351af8`) has exactly **4 call sites**:
  `0x14034fab9` (inside `0x14034fa40`, the `member+0x1f8` EventList
  applier), `0x140351204` (internal pre-Persist wrapper), and the two
  virtualized producers `0x140f7cadc`/`0x140f7cb5c` — confirming the
  only surviving gameplay producers live in the VM'd region.
- `Commit` (`0x1407a6118`) has exactly **1** caller: `0x1403fadec`.
- **Feared-banner anchor**: `CUIHUDCharacterPresentation::
  ShowEnemyFeared` registers handler `0x1419155f0` into the
  presentation manager `uictx+0x70` via `0x14032e3b8` (registration
  site `0x141918306`, stores handle at `obj+0x83a4`, sets state
  `obj+0x8368=5`, flag `+0x83a8`). A watchpoint on the handler's
  invocation = a live terror-transition signal without needing the
  AI morale field first.
- **Cheap live discriminant**: the telemetry-stat enum registers
  `HatesTriggered=0xc1`, `FearsTriggered=0xc2` into the name→id table
  at `0x142a4ab60` (registrar `0x14117e9b4`). If those counters tick
  on a live scare, a stat-producer path still runs and the break is
  purely in the CharacterEvent leg.

## Committed-log anatomy (VERIFIED-STATIC, 2026-09-26, pass 2)

Deeper decode of the write/read internals:

- `0x140351220(member, event)` (the tail of `0x1403511f4`):
  `0x14035136c(event)` → **slot weight** (how many timeline slots the
  event consumes — NOT a type id; bounds-check `0x141bbdbe8` against
  `member+0x4b8 + weight`); script notify `0x140278758`; wrap via
  `0x14114e668`; `0x14035188c(member, wrapped)` advances the write
  cursor `member+0x4b8` and fires the new-event UI notification
  (faction mgr `0x1426ffaa8+0x6c68` → `0x141b5d048`/`0x14030e7b4`
  family, gated by `member+0x508` bits 0x400/0x8000000); sets
  `member+0x474 = 1`. A chained companion event (`0x1403512fc`)
  applies **time-ranged behavior entries** at position
  `member+0x470 + cursor` via `0x140795770 → 0x1407957a0` — which
  resolves the local-player pawn `[[0x1426ffa98]+0x888]+0x26a8+0x318`
  and dispatches `0x140276aec`. So `+0x470` is the member's
  **timeline epoch**, not a byte-buffer base.
- **Property getter `0x14030e7b4(obj, propid:u8, 0, &out)`** — the
  engine's universal attribute accessor (5181 call sites; null-obj →
  returns `*out` unchanged, else tail-thunks into dynamically allocated
  code — Themida/import-protected body, statically unreadable).
  Prop-ids are bytes loaded from version-gated descriptor tables (e.g.
  `[0x14265a930]+0x1e0` count gate → `+0x1f0` id byte). The feared-on
  path `0x1401576cc` chains it: `propget(obj+0x40 char, idA)` → fallback
  `propget(char, idB)` → `propget(res, idC)` → stores the resolved
  feared-source at `obj+0xf0` and notifies the UI via `ctx+0x80`
  (`0x140157038`). A member-shaped result for some propid on a char
  would be the clean char→member link — recon `charprops.lua` scans
  ids `0..0xff` (callva-gated).
- `member+0x1f8` = vector of **polymorphic event-log readers** —
  the committed storage. `0x1403513dc(member, slot, +0x1f8, &out)`
  iterates them; `0x140352dc4(reader, slot, member_ctx, &out)` has
  each reader materialize its entries for that slot (builds a
  `{count@0, entries@8}` view via `0x1403538b4`/`0x141150460`,
  filters `0x1403531e0`/`0x1402774a8`, extracts payload
  `0x1403531c4`/`0x140353154`).
- `0x14034f9f0(member)` = the **new-events sweep**: iterates slots
  `member+0x4c0` (viewed cursor) .. `member+0x4b8` (write cursor),
  reads each via `0x1403513dc`, and for every entry runs
  `0x140350500(member, entry)` → **derived/chain events** re-applied
  through `0x140351af8` (this is how one event spawns follow-ups —
  e.g. terror → terror-bark chains). Finishes with
  `member+0x4c0 = member+0x4b8`.
- `0x1407a6118(member)` drain re-applies every pending `+0x1c8`
  entry through `0x1403511f4` (full eligibility + cursor advance +
  behavior apply), then clears `+0x1c8` and the second queue
  `+0x358`; also flushes `member+0x348` via `0x1407bb8c4`.

Recon implication: `RDR|` dump walks `member+0x1f8` — committed
event counts live inside the reader objects, not in a flat member
counter; `+0x4b8 - +0x4c0` = unviewed-slot backlog.

## Version diff — patch04 vs hotchunk gamedb (VERIFIED-STATIC, 2026-09-26)

`game.gamedb` ships as a full snapshot per archive: Patch_01–04 each
carry ~58.1MB; Patch_05–13 ship none; `hotchunk.arch06` (and the
`Patch_IncreasedCap` mod) carry a different ~56.76MB build — the
current data. String-table diff (267,510 vs 307,730 strings;
2,346 removed / 42,566 added):

- **Terror event vocabulary intact**: all 30
  `ActiveTerrorTriggered_*` `Faction.Event` records exist in both.
- **Morale ranges renamed, not deleted**: patch04's
  `AI/Morale/Ranges` (`Fleeing`, `Panic`, `Cowardly`, `Not_Panic`,
  `CombatRecoilThreshold`, …) reappear as `@Panic`, `@NotFleeing`,
  `ACowardly`, … in current — a naming-scheme change, not removal.
- **Flee/terror presentation content removed wholesale**: the
  `dx_pr_fh_Terror{Alone,Cruelty,DeathBG,GraveWalker}_*` bark/dialogue
  bundles (~170 strings incl. `.bndl` paths) were deleted — the
  fear-bark presentation set is gone from current data.
- **Flee system reworked**: added `Check_IsFleeing_RaiseMorale_
  FalseRetreat`, `Combat.Predicate.MoraleRange{Attacker,Target}`,
  `AI_AOE_MoraleBomb_ForceFlee`, `Curse_LoseAllMorale`,
  `Script_Prevent_MoraleLossOnCaptainDeath`,
  `AI_Trait_FalseRetreat_StopFleeing`; removed
  `CCheck_CBMeter_Flee_Default`, `Replay_Duel_Flee_Cower`,
  `GeneralCombatKey_TerrorKills`, `PickerListRaw_TerrorPoison`.
- Script protos (`AddEvents`, flee methods) identical across
  versions — only hash-renames.

**Record-level binary diff** (VERIFIED-STATIC): extracted both
`game.gamedb` snapshots (`extracted/gamedb_versions/`). Record header
= `{u32 hash@-4, u32 name_sp@0, u16 schema_ver@+4, u32 schema_hash
@+6, …, u16 flags@+0x0e, u32 count@+0x10}`. Every `Faction.Event`
record — terror and non-terror alike — shows the **identical delta**:
schema version +4 fields (0x23→0x27, 0x22→0x26, 0x1c→0x20, …), flags
`0x01→0x04` / `0x0801→0x2004`, count `0x0e→0x0f`. It's a uniform
schema revision (4 fields appended), NOT a terror-specific suppress
flag.

**Conclusion**: the regression is code-side — the reworked flee/terror
producer path (virtualized `0x140f7cxxx` region) no longer calls
`ApplyEvent` for terror transitions. Event records, morale ranges,
and the entire storage pipeline survive intact. Repair design: hook
the terror-state entry (or mirror the missing producer) and call
`0x1403511f4(member, ActiveTerrorTriggered_*, 0, 0)` — the event
records still carry valid side-effect fields. The removed
`dx_pr_fh_Terror*` bark presentations mean native VO/anim responses
may be absent; memory entries will still record correctly.

## Behavior-tree diff — orc `.bvr` patch01 vs patch09 (VERIFIED-STATIC, 2026-09-26)

All five orc archetype trees grew ~8–10KB between `Patch_01` and
`Patch_09` (the same rework that added the morale records — e.g.
`orc_infantry.bvr` 364,929→373,104 B). Full string-set diff of
`orc_infantry.bvr`:

- **Zero `Terror`/`FactionEvent`/`Morale`/`Encounter` string diffs** —
  and neither version contains *any* `FactionEvent` or `Terror` string
  at all. The faction-event write was never in the infantry tree.
- The delta is unrelated content: warg-dismount anims
  (`AIRidingWarg_WyrmDeathDismount_*`), navmesh ledge-fall renames,
  `CombatState_OffMesh_Starting`, `DLC4_NoHeadTracking` predicate, and
  dozens of `*_NEG`/`*_NEW` internal markers.
- No `CCheck_*` strings in either version — combat checks live in
  gamedb (`CCheck_CBMeter_Flee_Default` was a gamedb record removed
  between versions, not a bvr node).

Implication: player-induced terror → faction memory was always
produced **outside the per-archetype behavior trees** — native code
(the virtualized `0x140f7cxxx` producers) or the removed combat-check
record path. `Combat.Action.FactionEventAdd` is registered as a
native behavior action (type descriptor ~`0x142658540`, factory
~`0x14270ee20`) but its impl sits in virtualized `.xtext` — no
decodable static body, and no `.bvr` corpus shows it being invoked
for terror.

**Detection lever for the repair**: script-callable state probes
exist (`IsFleeing`, `ForceFlee` in `call_sigs.tsv`), but the
`IsFleeing` script binding isn't qword-referenced statically —
registration is indirect (same pattern as console nodes). Live
approach: watch the member's flee/terror state field during a scare
(`charevents.lua` member-ptr probe already finds the member), then
fire `0x1403511f4(member, ActiveTerrorTriggered_*, 0, 0)` through
`exec.call` — or equivalently `Faction.SetTarget` + `Faction.AddEvent`
console commands via the Nemesis plugin API.

**Stub-farm caveat** (VERIFIED-STATIC): `0x1407f8xxx` registrations
are `ret 0` stubs (`Faction.AddEvent`#1, `Faction.Save`,
`Debug.Load`, `Faction.SimEvent`, `Faction.Print.Bundles`,
`Faction.Cheat.ForceNextRole`, `Faction.ReplaceProfile`…). The real
`FactionConsolePrograms.cpp` impls live at `0x141c5xxxx` — always
check the SECOND registration when a command lists twice, and
`nem.py disasm <handler>` before wiring a new lever. `Faction.Dump`'s
real impl `0x141c5c048` serializes all members to `argv[0]` verbatim
(file lands relative to game cwd = `x64/`); `Faction.Load`
(`0x141c5b960`) is real too — faction export/edit/reload is a
viable modding primitive.

## Reader-side vocabulary — `CharEvent_*_Recent` (VERIFIED-STATIC, 2026-09-28)

`AudioHooks/GameConditions` records prove the reader queries member
memory by **event-name string**, not record ref — the record's `a[4]`
holds the event name cstr directly:

| Condition record | Reads event(s) |
|---|---|
| `CharEvent_TerrorTriggered_{Fire,Flies,Troll,Warg,Goblin,Infighting,Nemesis}_Recent` | `ActiveTerrorTriggered_<same>` |
| `CharEvent_TerrorTriggered_Any_Recent` | all 30 `ActiveTerrorTriggered_*` |
| `CharEvent_ActiveFlee_Recent` | `ActiveFlee` |

So writing `ActiveTerrorTriggered_Fire` (a *name* into the member's
log) is exactly what restores "you burned me"-class reactions — no
record-ref plumbing needed on the write side.

**Event inventory** (`Faction/Events/Events`, 1933 records, all present
in current data): `ActiveFearTriggered_*` ×14 (`CaragorAttack`,
`Burning`, `Curse`, `Poison`, `Ghuls`, `Graugs`, `Spiders`, `Flies`,
`EnragedEnemies`, `AlliedFear`, `AlliedCaptainDeath`, `Cruelty`, `All`),
`ActiveTerrorTriggered_*` ×30, `ActiveHateTriggered*` ×~40,
`ActiveFlee`, `ActiveKilledByCaragor`, `ActiveDespawnWhileFleeing`,
`WillResurrect_{Fire,FireWraith,Caragor,Poisoned,Balefire}`, plus the
betrayal/shame/humiliation sim-event families.

**Producer-side trigger records still in db** (declarative
predicate→event maps; their dispatcher lived in the removed
combat-check / VM'd producer path):

- `FactionEventKillAI` — `RequiredCharacterType`, `RequiredTargetName`,
  `ExcludedGlobalVarPredicate`, `Event`.
- `FactionEventTargetMountedPlayer` — `RequiredCharacterType`, `Event`.
- `CombatFactionEventPicker` — `RequiredStatPredicate`, `FactionEvent`.
  **Embedded inline** in combat records — `Combat/States/Definitions`
  (e.g. `Radiance`), `Combat/AOE` (`WraithAoE_Fire_Lvl1`,
  `PC_AltFire_*`, `AoE_Poison`…), inventory items (`Mirian_GP_Blade2`)
  each carry a picker field (visible as the truncated
  `…nEventPicker` type string in extracted dumps). This was the
  vanilla source→event map: the combat context that induced terror
  (fire AoE, caragor attack, warg) carried its own picker choosing
  which `Active*TerrorTriggered_*` to record. The evaluation loop was
  in the VM'd producer path — the pickers themselves survive.
- `Combat.Action.FactionEventAdd` — `FactionEvent` @0 (combat-context
  apply; impl in `.xtext`, but the record type registers natively).

**Alternate intact apply surfaces** (beyond `Faction.AddEvent`):

- `Interface.Menu.MenuSelection.Action.Faction` — fields
  `BeginFactionEvent`, `FactionEvent`, `FactionEventSubject`,
  `FactionEventOther`, `EndFactionEvent` — the menu action that applies
  an event with subject/other member refs (army-screen path).
- `Faction.SimEvent` console — "Simulates a FactionEvent for the
  current Faction" (`0x14267b8a0`; check for stub-farm duplicate).
- `Faction.SetTarget` + `Faction.AddEvent` — decoded above; SetTarget
  takes grid ints, clunky for runtime targeting but fine for probes.

## Script-side writer path (VERIFIED-STATIC, 2026-09-28)

`Captain:AddFactionEvent` / `CCharacter:AddFactionEvent` bindings:

- Stub `0x141b73f08` (4 args): resolves member ctx via
  `0x1426ffaa8+0x6c68` (faction manager), then
  `0x141b5a47c(mgr, self, event, other?)`.
- `0x141b5a47c` builds an event struct on the stack — copies the
  subject member's `+0x178`/`+0x174` fields, optional `other` member
  (`rcx+0x178/+0x174`), event payload from `0x141bd2a54` — then
  `0x14040c0ac(member_ctx, &event)`.
- `0x14040c0ac` checks `member+0x1f0` non-empty → `0x1401e2da0`
  enqueues into **`member+0x168`** (list ops `0x141b58504`/
  `0x141b5b420`). Struct teardown via `0x14061ff1c`.
- **`+0x168`/`+0x1f0`/`+0x110` corrected (VERIFIED-STATIC
  2026-09-28)** — earlier "keyed ledger" reading of `+0x168` was
  wrong. The actual `0x14040c0ac(member, &event)` flow:
  - `0x14040c13c(member, event)` is the **event consumer**:
    sets `member+0x208 = 1` (dirty flag — the `0x140185fe0`
    gate), then appends the event to **`member+0x110` vector**
    (`0x1411596d0`) when `0x14040c9d8(event+0x20)` or
    `event+0x50` qualify, and runs a keyed lookup on
    **`member+0x98`** via `0x14018de04(member+0x98, event+0)`
    — `+0x98` is the member's keyed per-event state container.
  - `0x1401e2da0(member)` is a **drain-and-delete** over
    `member+0x168` = a `vector<ptr>` of owned objects: copy out
    (`0x141b58504` vector copy-assign), clear (`0x141b5b420`
    `end=begin`), delete each via `0x141c1dc0c`
    (`0x141b5a020` is just `cur != end`; `0x141b59690` frees
    the local buffer). Runs when `member+0x1f0` tests empty —
    i.e. `+0x168` releases owned objects when nothing staged.
  - So the script `AddFactionEvent` event updates the member's
    `+0x98`-keyed state + `+0x110` event list + `+0x208` dirty
    flag — relation/counter bookkeeping, *separate* from the
    persistent `+0x1c8` → `0x1403511f4` → `+0x470` log pipeline.
    Whether `+0x110` entries later feed the log is the open
    link; `0x140f13fb5`/`0x140cf7bd6` also invoke the +0x168
    drain (member teardown/deserialize).

`Faction.AddEvent` console (`0x1403511c0` → `0x14034fce4`) is the
direct `+0x1c8` append; `Captain:AddFactionEvent` is the script
path into `+0x168`.

## Feared presentation = terror-transition signal (VERIFIED-STATIC, 2026-09-28)

`CUIHUDCharacterPresentation::ShowEnemyFeared` registers handler
`0x1419155f0` for **event id `0x85d`** into the presentation manager
`[0x1426ffb08 + 0x70]` via `0x14032e3b8` (registration site
`0x141918306`, handle stored at `obj+0x83a4`, state `obj+0x8368=5`).
The handler is a thunk → `0x14016a0e4(arg, 0)`. Event `0x85d`
firing = a live feared presentation; hooking the event-post path
with id filter `0x85d` gives a per-frame terror-transition signal
without touching AI/morale internals.

## Live feared capture (2026-09-29b)

First real feared edges captured in-world (5+ in one session):

- fearedwatch detour fires correctly on live feared-on transitions.
- `obj+0x40` = character link CONFIRMED on every edge. char objects
  cluster in the `0x3b1c…`/`0x3bae…` pool; feared presentation objs in
  `0x12x…`; `obj+0x50/+0x58` = constant exe-range table/fn ptrs
  (`0x141dabde0`/`0x141dabdd0`); `obj+0x48` = another `0x129x` pool
  object (per-object).
- char+0x08..0x138 contains NO member-shaped pointer → scan widened
  to `+0x400` + a second-level hop over char fields. One edge did
  resolve `member=0x3e4f7600` (proves reachability; path unclear).
- 'on'-mode edges on untracked members now fall back to
  `write_direct_member` (was: silent drop — seen live losing
  `member=0x3e4f7600`'s event).

## Actor→member native resolver — the full call contract (2026-09-30o)

`0x14018d920(actor)` resolves a spawned actor to its roster member —
decoded completely after it crashed live twice at the universal
assert stub (`exe+0x819113`):

- tail-jumps `0x14018d93c` with `rdx=actor+0x2708`,
  `rcx=[[0x1426ffaa8]+0x6c68]` (roster mgr).
- `[actor+0x2710] == -1` → unregistered → NULL. The dword IS the
  registration index; `actor+0x2708` is the key **object ptr**.
- `0x14018de04(mgr+0x98, key)` — the keyed-map probe — routes to the
  assert (`0x14109e110` → `0x140819104` → `mov [0],0`) when the key's
  **owner-tag** `[[key]+0x18]` differs from the map tag `[mgr+0xa8]`.
  Then hash `0x1401898e8` → 0x290-stride entry probe → `0x14018ddd4`
  returns `[[entry]+0x20]` (the member) or NULL.
- Live confirmation: the owner-tag check discriminates real roster
  keys from lookalikes — member field scans surface key-shaped
  objects (ki=21, ki=2586, even ASCII tag garbage) that ALL mismatch;
  a torn feared-edge char carries a key owned by a different map.
- `member_for_char` mirrors all of it before `callva`: valid
  `[actor+0x2710]`, non-null key object, readable owner-tag equal to
  `[mgr+0xa8]` — else roster-scan fallback (bounded skip log shows
  the failing leg). `member+0x48` actor link was NULL on all 4257
  scanned members (idle roster is all benched).

## Cause-specific terror events (2026-09-29c)

`runtime/data/trait_events.lua` (`NEMESIS_TRAIT_EVENTS`, generated by
the extracted corpus — 86 `Faction/Traits/Definitions` records
carry their memory-event ref inline as `{0x21b,ord}` cells):
`Terror_Caragors`→`ActiveTerrorTriggered_Warg`, `Terror_Fire`→
`ActiveTerrorTriggered_Fire`, `Fear_*`→`ActiveFearTriggered_*`,
`Hate_*`→`ActiveHateTriggered*`, `BetrayPlayer_*`→…

feared_edge event selection, in order:

1. `member_trait_names(member)` — **raw struct read** (SeiKur0 layout,
   same family as the `looks_member` +0x470/+0x4b8 validation):
   `member+0x240` → nested `{+0x10/+0x18}` ptr vector → trait objects
   → name cstr at `+0x20`. Validated+bounded, works on any resolved
   member with no tracked cap. For feared edges, only
   `ActiveTerror*`/`ActiveFear*` mappings are eligible — a `Hate_*`
   mapping is never written for a terror edge.
2. `terror_event(cap)` — pure-read VM probe (`exec.get`/`callm` over
   candidate trait-list field names) when a tracked cap handle exists.
3. `ActiveTerrorTriggered_Any` — honest umbrella fallback.

Dedup (`pending_names`) covers all native terror/fear names — a pending
cause-specific native write suppresses ours, and an unrelated pending
`ActiveFlee`/`Hate_*` never suppresses a terror write.

## Flee stimulus schema (VERIFIED-STATIC 2026-10-01 — native attr corpus)

The *stimulus* side is data-driven too: `AI/Morale/Modifiers` records
(`Environment_Fire`, `AI_Troll_Visible`, `Wraith_OnFire`,
`Player_WargRiding_Howl_Fear`, `Environment_HitByWasps`,
`AI_PrimalRoar`, …) instantiate native class `AI.Morale.Modifier`
(the extracted corpus):

- `Type` (tag 10) → `AI.Morale.ModifierType` — the stimulus taxonomy;
  the gamedb list `AI/Morale/ModifierTypes` holds only `Wraith` and
  `StopFleeing`, so most stimuli share the untyped base.
- `ExcludeFleeing` (tag 1) — default `0x141e72cf8` = interned `"true"`;
  `ExcludeSource`/`ExcludeTarget`/`AllowMultiple` default `"false"`.
- `OnFleeingItem` (tag 10) → `Inventory.Item` — item associated with
  the flee outcome (the gear/trait consequence surface).
- `Adjustment`/`InitialAdjustment` (tag 11) → `CombatStatValueOption[]`
  — stat deltas applied by the modifier.
- `IncludedItem`/`ExcludedItem`/`InitialIncludedItem`/
  `InitialExcludedItem` (tag 10) → `Combat.PredicateItem` — predicate
  gates selecting which stimuli the modifier answers.

Sibling schemas: `AI.Morale.Profile` (`InitialMoraleRange`→
`Morale.Range`, `BeginFleeRunSmartObject`, `FleeFX`→`Effect`,
`Influence`→`MoraleInfluence[]`), `AI.Morale.Range` (`Range` tag 7,
`AllowNoMorale` bool default `"false"`). All record-typed defaults
resolve to the shared empty-record sentinel `0x141db10e0`.

The chain for memory persistence: a modifier's predicates fire →
morale breaks → feared edge (fearedwatch) → trait check picks the
`ActiveTerrorTriggered_*`/`ActiveFearTriggered_*` event →
`callva(0x1403511f4, member, event, 0, 0)` persists it. The modifier's
`Type`/`OnFleeingItem` are the engine's own record of *what* the
captain fled from — a candidate finer-grained event selector than the
trait-name guess when the active modifier is reachable at edge time
(live question: does the feared char expose its active modifier set?).

## Event payload semantics (binary decode, 2026-09-29c)

`Faction/Events/Events` records decoded positionally (see
`docs/evidence/gamedb/binary-format.md`): the event name is the
complete primitive — writing it natively applies alertness levels,
trait markers, gear tag weights, reward lists, ambushes, commands,
replay types, sim tag mods, and the chained-event tail (~73%
populated). `ActiveTerrorTriggered_Fire` chains to
`ActiveBecame_Shun_noDeranged_noManiac` — the native feared→shun
transition the user-visible "forgotten terror" bug loses.

### Native event-source dispatch (VERIFIED-STATIC 2026-09-29)

`0x1403511c0(ctx={ptr,u32})` — the persist path's context resolver:
reads the global faction-event manager at `0x1426ffb08`, uses
`manager+0x7e8` as a container of `0x40`-stride entries keyed by
`{ptr, u32}` (compare predicate `0x14016fd3c`: entry+0 ptr equality +
entry+8 u32 equality). `0x14034fce4` is lookup-or-insert (miss →
`0x1417ed258`/`0x1417ed554`/`0x1417f00fc`/`0x1417ed420` build+insert
chain). This is the engine's event-source map, not the char→member
link.

`0x1403511f4(member, event, a3, a4)` confirmed shape (corrected
2026-09-29): `0x140351af8(member, event, a3, a4)` (ApplyEvent — args
3/4 forward through) then tail-jmp `0x140351220(member, event)`
(event-log write tail). The commit-drain calls it `(0,0)`; the `(0,1)`
tail belongs to direct `0x140351af8` calls (ctx/markUnviewed).

### Member commit loop (VERIFIED-STATIC 2026-09-29)

The function containing `0x14034fab9` is the native **pending-source
drain**: `rbx` = member (`[rbx+0x4b8]` = the event-log cursor our
`looks_member` validates), `rbp+0x130` = a per-context vector of
event-source entries. For each entry it resolves a candidate event
via `0x140350500(member, entry)` — which enumerates the entry's
sub-events (`0x140350f80` → `0x1403505d8` next-iter) and applies the
per-member eligibility check `0x140105aec(member, event)`. Eligible
events go through `0x140351af8(member, event, 0, 1)` — the ApplyEvent
half of our `callva(0x1403511f4, member, event, 0, 0)` (a3/a4 forward
to ApplyEvent; the drain uses 0,0) —
and successful applies feed `0x140351220(member, entry)` (log_write)
per pending entry afterward. Our direct-write path is therefore
structurally identical to the engine's own drain; the only difference
is we pass an already-resolved event record.

Additional member fields surfaced by `0x1403513dc` (called with the
member in `ecx` — `mov ecx,ebx` zero-extension truncates the member
PTR to a u32 handle/index, **not** evidence the pointer itself is
sub-4GB; live roster elements sit at `0x12A…`/`0x10A…`):

- `member+0x280` — **DISPROVEN-INGAME (2026-09-30f)**: reads as a
  pointer into an `id=0xa01` record-element array (`0x28` stride),
  not a script-object handle. No member's `+0x280` equals a tracked
  `cap.val`; the script-object-link theory here is dead. The
  0x20-stride `{scriptobj → value}` registry at `mgr+0xf8` (below)
  exists but holds only **5 entries**, none member-shaped — a small
  auxiliary map, not the member roster.
  - Registry layout (VERIFIED-STATIC 2026-09-29): `mgr = [vmctx+0x6c68]`,
    `base = mgr+0xf8`, `count = mgr+0x100`, `type = mgr+0x108`, entries
    `0x20`-stride `{key@0, val@8}` — directly iterable at runtime
    (recon a live probe).
  - Key hash `0x1401898e8`: `key+0x18` = type descriptor; flag bit
    `([td+1]>>1)&1` set → index = `(key − [td+0x38]) / 40` — i.e. keys
    are **0x28-stride pooled script objects** (`[td+0x38]` = the type's
    object-pool base, `mgr+0x108` gates the type).
- `mgr+0x98` — **the member-identity keyed table (VERIFIED-INGAME)**:
  38 keyed records at `0x290` stride, keys are `id=0xa51` elements
  (`0x28` stride, `+0x18` == `mgr+0xa8` container domain). Each keyed
  record's body (`rec+8`) owns an inner member vector at `body+0x20` =
  `{vec@+0x18, end@+0x20}` — 3 shared vecs hold ~635 unique member
  objects; `body+0x168` is the member's sub-index (`0xFFFFFFFF` =
  no-selection sentinel). **This IS the roster** — the earlier
  "faction-definition table" reading missed the inner-vec indirection.
- `member+0x4f0` — a keyed sub-container object (`0x14034fb7c` lookup,
  result `[+0x18]` consumed).

**Member serializer fields** (VERIFIED-STATIC 2026-09-29 —
`Faction.Dump` → `0x141b5ea0c` iterates `mgr+0x98`/`+0xa0` at `0x290`
stride → `0x141c2749c(member, out)` + tails `0x141c276d0`/
`0x141c277d0`):
- `+0x1eb` byte flag, `+0x1dc`/`+0x1e0` u32s, `+0x260` byte flag —
  serialized scalars.
- `+0x40` serializable sub-object (`0x141bf72b8`), `+0x48` optional
  presence flag.
- `+0x90..+0x98` — vector of ptrs, each serialized via `0x141bf05d8`
  (name/tag strings?).
- `+0xf8..+0x100` — vector of `0x28`-stride records each holding a u32
  sub-vector `[e+8..e+0x10)/4` — grouped id lists (event-type → ids?).
- `+0x188..+0x190` — vector of ptrs serialized via `0x1406f5eb0`
  (event-log entries? — same base the eligibility gate reads as
  class).

**Faction keyed table + current-target chain (2026-09-30g,
VERIFIED-INGAME).** `0x14016a550(obj)` reads the key at `obj+0x1b8`
and the container at `obj+0x98` — i.e. it is a *manager* method that
resolves **mgr's current faction → that faction's currently-selected
member**, NOT char→member:

```
mgr+0x1b8 → current faction key (id=0xa51 element — live names:
            "Faction_Player", "Real_Faction_MtnPass_DLC3",
            "Faction_volcano", … — 38 keyed records ARE the factions)
  → 0x14018de04 keyed lookup over mgr+0x98 (container {array@0,
    count@8, domain@0x10}; key+0x18 domain must match or a FATAL
    WBAssert at 0x14109e110 fires — never call with unverified key)
  → body (rec+8) → member = [[body+0x20]+0x18 vec][u32 body+0x168]
    (0x14018ddd4 vector index; 0x14018dde4 bounds; subid 0xFFFFFFFF
    = no current target — observed on all 38 factions)
```

**Char→member — still OPEN.** Disproven live (2026-09-30g): no
`id=0xa51`/`0xa01` element appears in any feared char's fields
`+0x08..0x400` — chars do NOT carry their faction key inline, and
even a faction key would resolve only the *current target*, not the
char's member. The char holds one record-element ref (`+0x38`,
`id=0x3e8`, unnamed). `member_for_char` is therefore: field-window
member scan → cap reverse-scan → second-hop → bounded member→char
reverse scan over the faction inner vecs. Every feared edge observed
live so far is an ambient NPC (legitimately memberless); the captain
case remains untested.
