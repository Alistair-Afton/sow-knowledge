# `.scripts` — SCRI script bundles

> **Status:** **working** — constants + opcodes decoded; VM runs; semantics evolving
> **Provenance:** real `.scripts` chunks — constant pools + opcode streams decoded corpus-wide; VM runs scripts.

World/section script state ships as `.scripts` files inside the LTAR
archives. Every file is a 13-byte SCRI header followed by one of two
payload kinds. Implemented in `anduril-scri`; `anduril scri <path>` in
the CLI.

## Header (13 bytes)

```text
+0x00  char[4]  magic "SCRI"
+0x04  u32      version — 1 in all observed files
+0x08  u32      packed: (payload_len << 8) | kind
                 high 24 bits = payload byte size (exact)
                 low 8 bits   = payload kind:
                   0 = LuaQ bytecode, 1 = text prototypes
+0x0c  u8       reserved — 0 in all observed files
+0x0d  payload  exactly payload_len bytes
```

Verified against 460 files across `patch01` and `patch09` corpora:
`field >> 8` equals `file_len - 13` for every file, and the low byte
splits the corpus perfectly (222 bytecode ↔ kind 0, 238 prototypes ↔
kind 1). The Nemesis doc describes this field as a plain u32 size with
a separate flag byte — that reading is wrong; the kind byte is packed
into the low byte of the size field (the byte at +0x0c is padding).

## Payload kind 0 — LuaQ bytecode

One or more compiled HavokScript (modified Lua 5.x) chunks
concatenated back to back. Each chunk begins with `\x1bLuaQ`; a chunk
runs until the next `\x1bLua` or EOF. There is no chunk count or
table — scan for the magic.

Bare `.hks` files are the same chunk format with no SCRI wrapper;
`anduril_scri::chunk_offsets` works on either.

### LuaQ chunk layout (decoded, `anduril_scri::luaq`)

```text
+0x00  char[4]  "\x1bLua";  u8 version = 0x51  — the 'Q' of "LuaQ" is
                really the version byte
+0x05  u8 format (14), u8 endian (1=LE), u8 int size (4),
       u8 size_t size (8), u8 instr size (4), u8 number size (4),
       u8 number type (0=float), u8 compat bits (3=MEMO|STRUCT),
       u8 shared (0)
+0x0e  u32 enum count; entries are {u32 value, u32 len, char[len] name}
       — value FIRST (TNIL=0 … TSTRUCT=12). Note: there is no flag
       word between count and entries; the first entry's value=0
       looks like one.
+…     main function proto, recursively
+…     u32 separator (0), then the structures block
```

Function proto (SoW order — `instructionCount`/`unk` swapped vs stock
hksc):

```text
u32 nupvals, u32 nparams, u8 is_vararg, u32 nslots,
u32 instruction_count, u32 unk, pad to 4,
instruction_count × u32,
u32 const_count, constants (u8 tag = enum index; TSTRING=4 uses a
u64 length-prefixed string, TNUMBER=f32, TUI64=u64),
u32 has_debug → debug info (line/locals/upval counts, path+name
tstrings, u32 line table, {tstring,u32,u32} locals, tstring upvals),
u32 child_count → child protos
```

Instructions are u32: opcode = `raw >> 25`, A = low byte, B at bit 17,
C at bit 8; 92 opcodes with NUMBER/REG/CONST/OFFSET/REG_CONST operand
modes (see `ops.rs`). Verified byte-exact against the Nemesis
`tools/hks_disasm.py` reference on retail files: 914 protos / 2214
instrs / 439 consts on `spawnconditions_fort.scripts`.

CLI: `anduril scri <path> dump` disassembles every LuaQ chunk.

### Structures tail (decoded)

After the proto tree + u32 separator, bytecode chunks carry binary
`hstructure` records — the same ScriptObject interface data the kind-1
text payloads declare, in compiled form:

```text
record:  u64 name_len (incl NUL), name,
         u32 a, u32 b, u64 c, u64 field_count
field:   u64 name_len, name,
         u16 flags (0xffff = regular field), u16 pad,
         u32 unk0, u32 type_tag (enum index), u32 unk1,
         u32 slot_index, u32 unk2
tail ends with a u64 0 name-len terminator
```

Verified on the full corpus: 2–204 structs per file, field types match
the enum table (`scriptId`=TTABLE 5, `lookupId`=TLIGHTUSERDATA 2,
methods=TIFUNCTION 9, `__index`/`__structmeta`=TSTRUCT 12). Exposed as
`Chunk::structs`; `anduril scri <f> dump` prints them.

## Payload kind 1 — hstructure prototypes

UTF-8 text. For every ScriptObject class in the section it declares a
meta structure (methods, `__eventmap`) and an instance structure
(fields + types):

```text
-- SCRIPTOBJECT PROTOTYPES (search for IMPLEMENTATION elsewhere)
hstructure _ambdummy_SO_Script_e72fb96c_meta
    __constructor : ifunction
    __index : _ambdummy_SO_Script_e72fb96c_meta
end
hstructure _ambdummy_SO_Script_e72fb96c_inst
    meta : _ambdummy_SO_Script_e72fb96c_meta
    scriptId : table
    lookupId : luserdata
    MyCooldown : number
    Running : boolean
end
```

- `--` lines are comments.
- `hstructure <name>` opens a block, `end` closes it.
- Field lines are `name : type` where type is an engine/HavokScript
  token (`table`, `number`, `boolean`, `luserdata`, `ifunction`, or
  another structure name for `meta`).

Parser output is field-exact with the reference extractor
(`proto_parse.py`): 29,802/29,802 rows identical on the comparable
corpus files.

## Interpreter (`anduril_scri::vm`)

`vm.rs` is a tree-walking HavokScript VM over the decoded `Proto`s:

- `Value` covers nil/bool/num/str/u64/lightuserdata/table/struct/
  closure/hostfn. Tables are `Rc<RefCell<Table>>` with a mixed hash
  map + array part; structs are slot vectors backed by the decoded
  `StructRec` layouts.
- `Vm::run(&Chunk)` executes `main` with globals pre-populated only by
  what the host registers; `vm.register(name, fn)` installs engine
  bindings (`HostFn` = `Fn(&mut Vm, &[Value]) -> Result<Vec<Value>>`).
- Covered ISA: MOVE/LOADK/LOADNIL/LOADBOOL, GETGLOBAL(_MEM)/SETGLOBAL,
  GET/SETFIELD + GET/SETTABLE(_S/_N/_BK), NEWTABLE, NEWSTRUCT,
  SETSLOT*/GETSLOT*/SELFSLOT*, arithmetic + `_BK` const variants,
  comparisons, NOT/UNM/LEN/CONCAT, TEST/TESTSET/JMP, FOR*/TFORLOOP,
  SETLIST, CLOSURE + GET/SETUPVAL, VARARG, CALL*/TAILCALL*, RETURN,
  CHECKTYPE*. Engine-specific ops (`DATA`, `INTRINSIC_*`,
  `GETGLOBAL_MEM` non-const, `_D`/`_MM` variants) fall back to a
  traceable `VmError::Unimplemented` rather than being silently
  misread — host hooks can intercept via `on_op`.
### `DATA` — instruction extension operands

`DATA` (op 76, Abx) carries no standalone effect: it is *extension
data for its owner instruction*. Verified against retail:

- `NEWSTRUCT` is always emitted as `NEWSTRUCT rA 0 0` + `DATA n` —
  `n` indexes the structures table in **name-sorted order**, not
  file order. Verified on `chase_00`/`sotp_desert_06`: every inst
  record's `__structmeta` field flag equals its meta's sorted index
  (`*_inst` < `*_meta` alphabetically, so classes pair adjacently and
  metas land on odd indices — hence `n` is always odd in practice).
  Decoding `n` as a file-order index silently binds the wrong class
  layout — e.g. `PrintDebug` resolving to `OnSetSpawnComplete`'s slot,
  which self-recurses until the stack dies.
- `*_MEM`/`*_R1`/`*_MM` ops are surrounded by `DATA` instrs carrying
  the engine's memoization/cache-slot indices — semantically
  inert for a tree-walking VM (we resolve through the normal path).
- `RETURN` uses B as the *direct* result count (`RETURN r0 1` hands
  back the module table), unlike Lua's n+1. `CALL*` keeps
  Lua-style `nargs+1`/`nret+1` in B/C.

### Execution notes

- Child protos never declare upvalues in this corpus — CLOSURE binds
  none; script state lives in globals, params, and struct fields.
- Method bodies take `self` in r0 and reach engine state through
  `self.scriptId.<name>` — `scriptId` is the TTABLE field the engine
  populates at instantiation; its `.Instance` member is the live
  object record. Event args (`Sender`, `Approacher`, `Trigger`...)
  arrive as call params.
- `_R1` call ops (`CALL_I_R1`, `TAILCALL_I_R1`) omit the A
  operand: the call register is implicit — the destination of the
  previous producing instruction (`SELF r5 … CALL_I_R1` calls r5).
- `probe` (default) seeds every referenced global as a *traced
  engine object* — a table whose metatable makes each field access
  yield a deeper traced object (`DynamicObjectGroupBrush.Create`)
  and each call append `path(args)` to `vm.out`. `probe --trace`
  prints that per-method call trace — literally what each mission
  handler does engine-side. Instance `scriptId` is itself an engine
  object carrying `.Instance`, and `permissive_fields` makes unknown
  `self` fields resolve to traced objects too.
- `LithThreadSleep(threadId, secs)` parks section threads in retail.
  A tree-walking VM can't suspend, so under `--sim` the call consults
  `call_hook`: the host *advances the world mid-call* (bounded by a
  shared tick budget), ticking timers and queueing emitted events for
  dispatch after the handler returns — sleep-polled conditions see
  real progress. With no hook it degrades to a trace line.
- `nil.__typeof` returns a callable engine object — the engine's
  `obj:__typeof()` type-probe idiom on possibly-nil handles.
- `probe` instances seed their declared `*_inst` fields at the compiled
  slot indices with typed defaults: `boolean`→false, `number`→0,
  `string`→"", `luserdata`→handle 0, untyped/`table` refs→nil, and
  `scriptId` lands on its declared slot. A never-assigned
  `self.Flag : boolean` guard therefore evaluates `false` like retail
  instead of reading a truthy permissive object.
- `probe --stub` replaces every unknown global/instance field with a
  self-perpetuating "any object" (a table whose metatable `__index`
  and `__call` return fresh stubs), and runs each method with stub
  params + a 50k instruction budget. Corpus result (patch09):
  **8,465/11,123 method bodies execute to completion** with full
  engine-call traces emitted; residual
  failures concentrate in `__typeof` introspection on nil event
  args, stringly-typed record refs (`x.RelayRecord` where x is a
  gamedb record name), arithmetic on stub returns, and budget-limited
  loops — i.e. genuine engine-data dependencies, not VM bugs.
- `probe` (engine-object mode) corpus result: **10,569/10,569 method
  bodies clean** over all 287 patch09 `.scripts`. The last residual
  (`sotp_desert_06` teleport-scatter) needed two things: real section
  group data (OBJL — `Group:Create("GTeleportPoints")` now binds the
  8 `SP06_TP_Captain_*` records with real transforms, `At`/`Size`/
  `Remove`/`GetGroup`/`GetDistanceToObject` answer live data) and a
  branch-sense decode fix — the conditional ops below.
- `EQ`/`LT`/`LE` encode the branch sense in the low **A byte** of the
  instruction word (Lua 5.1 style: `pc++` — skip the following `JMP` —
  when `comparison != A`). The opcode table marks A `Unused` for
  these ops, so `decode` discards it — the VM must read `raw & 0xff`
  directly. Treating A as always-0 inverts every `==`-guarded jump
  (a `x ~= None` guard becomes "jump when equal"), which sent the
  teleport re-roll into a permanent remove-free reroll.
- `install_stdlib()` covers the Lua base lib (type/tostring/
  tonumber/print/error/assert/pcall/set+getmetatable/rawget+set/
  next/pairs/ipairs/unpack/select); metatables support `__index`
  (table chains + callables) and `__call`. `install_engine_traces()`
  registers the Channel*/Print*/debugMessage log surface as sinks
  into `vm.out`.
- `anduril scri <f> probe` exploits this: it runs each chunk main,
  then calls every bound method on a synthetic instance, and the
  first failing call names the engine API it needed — an
  auto-inventory of the engine API surface (see
  `target/tmp/api_surface.txt` for the corpus sweep).

- Operand decode note: Abx instrs whose operand table declares
  `BcMode::Const` decode to `Arg::Const(bx)` — `LOADK`, `GETGLOBAL`,
  `GETGLOBAL_MEM`, `SETGLOBAL` all resolve through it, and disasm
  prints the resolved name (`r3 k2 "WBObjectPair"`).
- `TFORLOOP` semantics: the compiler emits the iterator call followed
  by a backwards `JMP`; a *continuing* iteration falls through to that
  JMP (back to the body), while exhaustion must **skip** it. An
  iterator that is itself a traced engine object yields `nil`
  immediately — an unknown collection iterates empty rather than
  fabricating elements forever.

Corpus status: **all 222 bytecode chunks in patch09 execute their
root protos cleanly** — every ScriptObject bootstrap
(`NEWSTRUCT`/`CLOSURE`/`SETSLOTI`/`SETFIELD` class registration) runs
end-to-end. Method bodies still need engine API bindings before they
can run; `scri <f> run` seeds all string constants as empty global
tables so pure-constructor code executes and the first missing host
fn reports itself.

CLI: `anduril scri <f> run` — decode + execute each chunk's main.

## Open questions

- `kind` values other than 0/1 — none observed; parser preserves them
  as `PayloadKind::Unknown` with the raw payload still accessible.
- Whether the low kind byte is exhaustive of the packed field's low
  byte semantics (no other flags seen; all files reserve +0x0c = 0).
- Chunk identity/linkage: how the engine maps a LuaQ chunk back to its
  ScriptObject class. Presumably positional vs. the matching
  `*_meta`/`*_inst` pair in a sibling prototype bundle — unverified.

## Engine event bus (WBEventSystem)

ScriptObjects subscribe to engine events with

    WBEventSystem:Add(descriptor, handler, ctx)

where `ctx` is a positional table `{scriptId, lookupId, eventHash}`.
The third element is a `LightUserdata` u64 — the event **name hash**,
baked into the constant pool at script compile time. The hash→name
vocabulary (2,250 events) is embedded in `src/event_names.rs`
(mined from `__eventmap` registrations across the retail corpus).

`Vm::install_event_bus()` captures every subscription into
`vm.subs: Vec<EventSub>` with resolved event names; probe mode then
fires each handler once (dispatch pass) to prove invocability.

`probe --sim <ticks>` runs the real bridge: builds a seeded nemesis
world (real `Faction/FactionMemberBuilder` captains + `nemesis_tick` +
`mission_tick`), advances N ticks, and routes emitted `nemesis:*`
events through `vm.subs` into subscribed script handlers. SimEvents
surface in scripts as faction events — the `OnFactionEvent`
subscription is the engine's dispatch entry for them. The Lua event
table carries the full `SimEvent.data` Fields bag (mission/kind/
region/participants/simulate/captain state/tick — Refs marshal as
their record names, Lists as array tables), not just `{name}`;
`event.FactionEvent` carries the type name handlers switch on.
Routing is by subscription suffix — `NamedOnFactionEvent`,
`GangOnDeath`, etc. scope the same bus — with death-flavored
`nemesis:*` events additionally routed to `*OnDeath` subs.
Events carrying an `entity` id also get a `Sender` handle: a hybrid
object whose captain accessors (`GetName`/`GetFullName`,
`IsDead`/`IsAlive`/`IsShamed`, `HasScriptTag` over traits+markers,
`GetPower`, `GetTribe`, `GetRegion`, `GetStateRecord`, `__typeof`)
resolve against the real sim `Captain` — `GetStateRecord` returns a
captain-bound record object, so record reads (`record:GetName()`,
`record:IsDead()`) and verbs on the record act on the same live state —
while every other
access/call still degrades to a traced engine object. The handler's
`self.scriptId.Instance` is pointed at the same handle per dispatch.
Mutation verbs on a bound handle (`KillObject`, `Shame`, `SetEnabled`,
`Script_SetState`, `Despawn`, …) don't just trace — they record
effects the host applies to the sim world after dispatch (status →
Dead/Shamed, `orders_disabled`, `custom` fields), re-emitted as
`nemesis:script_fx`. A `call_hook` on the VM lets the sim host
intercept quest/mission lifecycle verbs (`QuestStart`,
`QuestComplete`, `QuestFail`, `QuestCleanup`, `MissionEnd`,
`CompleteMission`, …) on any object: `QuestStart` spawns a real
`Mission` entity (`MissionKind::Custom`, event = the object's trace
path), complete/fail verbs flag `ticks_left=0`, and one extra sim
tick resolves them — `mission_started`/`mission_resolved` feed back
into the bus for a second dispatch round. First live run:
`beasthunt_00`, 60 ticks → 704 sim events → 704 handler dispatches.
