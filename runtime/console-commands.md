# Console commands — the engine's `*.ConsolePrograms` system
> **Status:** **partial**** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


Status: `VERIFIED-STATIC` except where noted. The dispatcher entry is
**LOCATED (2026-09-30n)** — the registry object's `vt+0x58` verb —
and program-record commands are invocable through it.

## Command-node table (extracted/native_console_commands.tsv)

~590 static nodes in the exe image, VA range `0x14262a340..0x142683250`
(this build's `.idata` region; non-ASLR base `0x140000000`). Enumerated by
`tools/console_cmds.py` via the `C:\P4\...\*ConsolePrograms.cpp` source-path
pointers. Layout:

| off | field |
|-----|-------|
| +0x00 | back-pointer to a self-linked list head at node-0x10 |
| +0x08 | handler — **code VA**, or **small int** (cvar/id family), or 0 (stub) |
| +0x10 | ctx — the "console program" record (shared per adapter family) |
| +0x18 | name cstr (`"Faction.AddEvent"`) |
| +0x20 | help cstr |
| +0x28 | source-file cstr |
| +0x30 | source line |
| +0x38/+0x40 | self-linked intrusive list head (empty in the image) |

Handler census: ~281 code pointers, ~224 small values (cvar/id family —
e.g. `AI.InfoTrack.Subject` handler=`0x101`), rest stubs/variants.

## Console-program ctx record

`ctx = {+0x00 dtor-fn, +0x08 adapter-thunk, +0x10 name-accessor-fn,
+0x18 flags/line, +0x20 inline program name}` — e.g. `0x141e69180`'s
accessor returns `"ProgramStringLoadedFadeInFadeOutPerfect…"`. Many
commands share one program record: the program is the *marshal family*.

## Handler ABI — `int handler(int argc, char** argv)` (VERIFIED-STATIC)

Every code-handler command takes argc/argv. Reference decodes:

```
Presentation.QueueEvent @0x1417a5c04:
    cmp ecx,1; jb ret          ; argc >= 1
    rsi = rdx                  ; argv
    rcx=[rdx]                  ; argv[0] = event name
    call 0x141b080a4           ; resolve name -> event obj
    if argc>=2: rdx=[rsi+8];  call [0x1426ff058]->vt+0x68  ; resolve target
    if argc>=3: rdx=[rsi+0x10]; call vt+0x68               ; resolve arg3
    call 0x1417a4a74           ; queue(event, target, arg3)
Presentation.ScriptRelay @0x1417a54a4:  cmp ecx,2; jne err (needs 2 args)
Presentation.CharacterEvent @0x1469d47e0: reads zero arg regs (argc-agnostic)
```

So the ctx **adapters are only string→argv tokenizers**. Calling the
handler directly with `(argc, argv)` reproduces dispatch *without* the
unlocated parser — that's the `sdk.console.exec` path.
Name→object resolution for args goes through singleton `0x1426ff058`
(vtable +0x68), so args are cstr names, not pointers.

Two adapter shapes seen (kept for reference — bypassed, not needed):

```
0x141179608 (adapter A):  adapter(a,b,c) -> qword[a+0x38](b, c)
0x14117dd00 (adapter B):  adapter(a,b,c) -> tail(b, c, a->+0x08, &a->+0x38)
```

`a` is the command node — adapter B's tail registers decode as
`tail(argc, argv, node->+0x08 /*varid*/, &node->+0x38 /*storage*/)`,
which pins the cvar family's dispatch (below).

## ctx families (VERIFIED-STATIC census)

Handler-column values appear under multiple ctx programs — the ctx
determines the dispatch, not the handler value:

| ctx | adapter | count | family |
|-----|---------|-------|--------|
| `0x141e69180` | `0x141179608` (A) | 336 code + 82 bound-value | main program — `[a+0x38](argc, argv)`; resolved entry per node |
| `0x141e69fd0` | `0x14117dd00` (B) | 57 small + 5 "code" | cvar — typed tail f(argc,argv,varid,storage) |
| `0x141e69fa0` | `0x14117dd1c` (B) | 47 small + 7 "code" | cvar sibling tail |
| `0x141e69fb8` | `0x14117dd38` (B) | 19 small + 5 "code" | cvar sibling tail |
| `0x141e69f88` | `0x14117dd54` (B) | 9 small + 2 "code" | cvar sibling tail |
| `0x141ffc4f0` | `0x141c8b704` | 6 small + 11 code | render profile family (decoded 2026-09-28 — see below) |
| `0x1420023f0` | `0x141cad994` | 2 | bool-cvar variant — storage `node+0x39` |
| `0x1420038c0` | `0x141cb6554` | 2 | int-cvar variant — storage `node+0x3c` |

**Minor families decoded (VERIFIED-STATIC 2026-09-28):**

- `0x141c8b704` (Render.D3D.*/Render.UI.Profile*/Video.UHD — 17
  cmds): `id = min(argc, 0x40)`; if `0x14114b7f8(&0x14272e8d8)` passes
  (generation check — `obj->+0x3c` vs `(*0x141d92240)()` epoch) →
  `[node+0x38](id, argv)` resolved-program dispatch, same convention
  as adapter-A. Miss path iterates `0x141c8b494` per arg. Small-int
  "handlers" (0/1/2/0x79) = profile-mode selector ids.
- `0x141cad994` (Render.Draw.Overlays, Render.Texture.PrefetchMip0):
  calls the **bool tail `0x14117d4cc` directly** with
  `varid = node+0x08`, `storage = &node+0x39` — a cvar that stores in
  the node itself (no lazy-bound `R`), then mirrors `node+0x38 = byte`
  under the same generation check.
- `0x141cb6554` (Render.Ortho, Vis.AreaCull.Min): same shape via
  sibling tail `0x14117d5c8` (int variant), `storage = &node+0x3c`,
  mirror `node+0x38 = dword`.

So every adapter resolves to one of two conventions:
`[node+0x38]-resolved program` (adapter-A + render profile) or a
`0x14117d4xx` typed-cvar tail (adapter-B + the two in-node variants).
In-node-storage cvars need no `R` binding — `exec` could cover them by
matching the adapter's inner `call 0x14117d4xx` + `lea r9,[rcx+0xNN]`
pattern (4 commands; low value, deferred).

Under adapter-A ctxs, node+0x08 is a **bound value slot**, not
uniformly a selector id (corrected 2026-09-30i): the constant
entries decode as argument constants — `0x3f000000`=0.5f,
`0x442f0000`=700.0f, `0x4461000043340000`={900.0f,180.0f} (a clamp
range on `FactionMember.SetHealthPct`), `0x10101`/`0x10101010101`=
byte masks — so `+0x08` is consumed by the resolved program as a
bound default/arg, with only the small-int entries (`0x1`, `0x8`,
`0x101`) plausibly acting as ids/flags — most likely as
**program-ids within the owning subsystem's dispatch table** (each
source mgr registers numbered programs; e.g. `AI.InfoTrack.Subject`
`0x101` = `AIDebugSubjectMgr` program #0x101). **Narrowed
2026-09-30i**: the adapter-A ctx `0x141e69180` is a **"Program"-group
descriptor** `{dtor=0x141178a04, adapter=0x141179608,
namefn=0x1407e0fc0→"Program", fmt="%d"}` — its `+0x20..` blob is an
inline field-name table `"Program\0String\0Loaded\0FadeIn\0FadeOut\0
Perfected\0\\\0\\?\"` (record-type schema + UNC path prefix), not a
handler table. Nodes never
self-resolve (`+0x38` stays a list link at rest, VERIFIED image-read
`AI.InfoTrack.Subject` `+0x38`=`+0x40`=`0x1426787f8`); the adapter
tail-jumps `[a+0x38]` only after the parser installs the resolved
entry — **live check for next session**: `r64(0x1426787c0+0x38)` —
if it differs from the rest-time list link, it holds the resolved
program entry (follow it to the mgr's program table).
program entry. The id→program table itself is still unlocated.
**VERIFIED-INGAME
2026-09-25:** node+0x38/+0x40 are intrusive *list links* for every
family — `QueueEvent +38=0x14266e0a8 +40=0x142671208`,
`Subject +38=0x142678918` — all point inside the node range, never
resolved code. Small-id dispatch resolves inside the (unlocated)
parser; the nodes are linked at runtime so the live registry is
walkable via the +0x38/+0x40 links. `exec` therefore uses +0x08
directly for code handlers and reports small-id adapter-A commands
as parser-resolved (not callable yet). The "code VAs" under cvar
ctxs are probably large varids — the ctx routes them through the
typed tail regardless.

## cvar family — `f(argc, argv, varid, storage)` (VERIFIED-STATIC)

Under adapter-B ctxs, node+0x08 = varid and adapter B forwards to a
per-type tail — the bool tail `0x14117d4cc` decoded fully:

```
tail(argc, argv, varid, storage):
    argc==0  -> echo fmt "%s = %d" via engine->vt+0x18, then
                vt+0x28(this, varid, storage) re-notify        (GET)
    argc==1, argv[0]=='0'/'1':
        byte [storage] = 0/1; vt+0x28(this, varid, storage)    (SET)
    argc==1, other: warn "Bool was expecting 0 or 1 as input"
```

**VERIFIED-INGAME 2026-09-25:** the adapter's `a` is NOT the node —
for cvar nodes, node+0x38 holds a pointer to the command's resolved
runtime record `R` (e.g. `AI.EnhancedFlee.Enabled` → `0x142a58be0`,
.bss next to the console singleton). The tail receives
`(argc, argv, varid=R+0x08, storage=&R+0x38)` — the storage byte lives
in `R`, not in the node. `exec` therefore resolves `R = *node+0x38`,
rejects node-range pointers (those are list links = unresolved), and
passes `R+0x08`/`R+0x38`.

**BOOL TAIL E2E VERIFIED-INGAME 2026-09-30i:** at title,
`exec("AI.EnhancedFlee.Enabled", "0"/"1")` roundtripped the storage
byte `0xFF→0x01→0x00→0x01` (the `0xFF` initial = non-canonical true;
tail writes byte0 only, upper bytes are adjacent packed fields).
GET (argc==0) dispatches clean. `exec` correctly refuses
unresolved cvars — `Mob.AnimSampleRateMS`'s `R=0x14266f9f8` stays a
list link because **most cvar records bind lazily** (subsystem
init or first typed use); only already-bound records are callable
from the title screen.

Sibling tails for **uint** (`%s = %u`, "expecting unsigned input")
and **float** (`%s = %f`) exist with the same signature — the fmt/warn
strings cluster at `0x1422b57e0..` and each cvar's ctx program points
at its own adapter → own tail. So `sdk.console.exec` doesn't hardcode
a tail: it decodes the `E9 rel32` at the node's adapter+20 and calls
that — typed-tail selection is per-command and exe-update resilient.
The engine singleton `0x1407cbc20()`
(TLS-indexed, returns `&0x142a58bc0`) provides `vt+0x18` = console
echo and `vt+0x28` = `(this, varid, storage, 0)` change notification.
**Decoded 2026-09-30i**: `vt+0x28` = `0x1415cff14`, a trampoline that
forwards `(varid, storage, 0, arg5)` to a registered callback at
`this+0x10` — **`this+0x10 == 0` at title** (verified live), so the
notify is a no-op hook point, not mandatory machinery. Cvar SET is
storage-write-only until a listener registers (likely world init, if
ever).

## Boot cfg chain (decoded)

Startup resolves 4 cfg names (`projectconfig`, `renderconfig`,
`cinematicconfig`, `userconfig`; fallbacks `projectconsole.cfg`,
`\USER\render.cfg`, `autoexec.cfg`) via a vtable FS object at
`0x1426fefe0` (`[obj]->+0x40` = resolve), `FileExists` @ `0x140188c80`,
then `0x14117cf30` — the console-subsystem constructor: singleton mgr at
`0x1426ff050`, sub-objects at `+0xa98`/`+0xab8`, a 0x1188-byte "shell"
object at `+0xac0`. The actual per-line exec lives inside this machinery.

## File-exec path (decoded 2026-09-30r)

`0x14077f614(obj, str, flag)` returns **0 when `str` is an existing
file** (`0x140188c80` FileExists → `xor eax;ret`) — the exec verb
`0x1415cff44` handles names only; file execution is a separate caller
branch. The replay machinery is `0x14077f560`-family: resolve the
name → R, then replay `R+0x10..+0x18` (a vector of per-line
arg-vectors, `{vec begin,end}` × `argc = (q[1]-q[0])>>3`) through a
per-line executor callback `[ctx]+8`. The cfg-file loader that
builds R objects from file text lives in the `0x14117cf30`
console-subsystem init machinery (shell object at `mgr+0xac0`).

## Presentation.CharacterEvent (the G49 lever)

Handler `0x1469d47e0`:

```
rcx=[0x1426ffa98] -> [rcx+0x888] -> [rcx+0x2fc0]  = relay-obj
call 0x140679480(relay-obj, 0x14):
    rcx=[obj+0x100] -> call 0x14067949c(target, 0x14):
        if id != 0xC: target->+0x6c |= 1<<id
```

So the command **sets bit 0x14 in a 32-bit relay bitmask** at
`target+0x6c` — a presentation-relay flag, not a log append. The
"specified character" is the AIInfo-tracked subject (via
`AI.InfoTrack.Subject`, itself a cvar-id command `0x101`).

The bit-setter `0x14067949c` has **16 real call sites** (plus the
`0x140679480` null-guard wrapper which adds 17 relay-routed sites —
`[relay+0x100]` is the target object). id==0xC does NOT set a bit —
it `inc`s the `+0x68` counter. Full id census (2026-09-30):

| id | sites | context |
|----|-------|---------|
| 0 | `0x14037a225` | post flag test |
| 2 | `0x14182d64c` | post `0x14182bec0` predicate |
| 5 | ×5 (`0x140678054`, `0x1406782ac`, `0x14042dada`, `0x14042fe1a`, `0x14042ff73`) | all gated on the `0x14042ffec`-family predicate |
| 7 | `0x14182d299` | post `0x1407b10c0`, clears `rbx+0x1b8` |
| 0xC | `0x14037acb3` (`lea edx,[rax+0xb]` at eax==1) | the `+0x68` counter path |
| 0x10 | `0x1405d21ab` | post `0x14024e678` predicate |
| 0x11 | `0x1405d228d` | gated by `[rbp+0xf4]` byte |
| 0x14 | wrapper `0x1469d4808` | `Presentation.CharacterEvent` |
| 0x15 | `0x14037adbe` | post `0x14024df7c` compare |
| dyn | `0x14042fcd1` (`edx=edi`) | caller-supplied id |

**Reader is not statically locatable (2026-09-30):** every `+0x6c`
access form was swept — `mov` loads +8-insn bit-op use (71 sites, all
other object types — e.g. the `0x140149ef9` site is a count field
with a sign-bit flag, `0x1408c5xxx` is OpenSSL EVP), `test [x+0x6c]`
(F7/F6/85), `bt`/`bts` field ops (0F A3/BA), `mov/cmp/and` clears —
none correlate with the relay target. The AIInfo presentation drain
that consumes the mask is consistent with the producers being
Themida-VM'd (`0x140f7cxxx`/`.bind` sections). The mask object is
`[[[0x1426ffa98]+0x888]+0x2fc0]+0x100` — the AIInfo-tracked subject's
presentation state (`0x1426ffa98` = AIInfo mgr singleton).
Live catch options: detour `0x14067949c` to census (target,id)
writes; a hardware read-bp on `target+0x6c` finds the true reader.

## Registration architecture (VERIFIED-STATIC 2026-09-25)

Registration sites are dense static-init blocks (`0x140055xxx`,
`0x140056xxx`, `0x140058xxx`… — RIP-relative scan found ~220k
node-range refs, one `lea rdx,[node]` per command):

```
if [0x142a58bb0] <= counter:                 # table-full guard
    rax = [0x142a58bc0]                       # registry singleton
    call [rax+0x68]  (rcx=&0x142a58bc0, rdx=node)   # register node
```

`0x142a58bc0` = the **console registry singleton** (runtime-built
.bss object). Two vtable verbs seen: `+0x68` registers a command
node (`0x1410693a0` thunk shape: TLS-guard → `lea rdx,[node]` →
`call [rax+0x68]` — each of the 589 nodes gets one thunk), `+0x10`
registers a bare function (rdx=0x14117f4a8 → thunk → `0x1406aac14`,
a notify/callback wrapper — program-side registration). The
dispatcher's `a` object (adapter-A calls `[a+0x38](argc,argv)`) is a
**runtime record R built here** — R+0x38 holds the resolved code
pointer (node's handler, or the program fn for the small id). R
records live in the registry's containers — `recon/console.lua`
dumps the singleton's fields live to find them. The small-id→program
map is **runtime-built inside register()** — the +0x10 arg is a
builder thunk, not an (id,fn) pair, so it isn't statically
extractable.

`PlayerInput.SetSimInput` (decoded while probing title nav): takes
ONE arg — a named sim-input object (hash → `[0x142700740]` table) —
then requires an **active player**: `[mgr+0xe90]+0x40` = player
index (0xff = none), `[player_arr[idx]+0x38]` must == 3. Frontend
has no player → no-op at title (**VERIFIED-INGAME 2026-09-30m**:
dispatched via gated `console.exec`, `callargv 0x146a6d470` ran,
title unmoved). Useful in-world.

## The canonical exec verb (LOCATED + VERIFIED-INGAME 2026-09-30n)

The console registry object at `0x142a58bc0` (`[obj]` = vtable
`0x141ec6488`) owns the real dispatcher:

```
[obj->vt+0x58] = 0x1415cff44 (obj, str, argc, argv):
    0x14077f614(obj, str, 1)   — FileExists(str)? source-file :
                                 name→R lookup in obj+0x38 map;
                                 returns runtime record R (+0x28
                                 name-match field)
    0x1415cfe88(R, argc, argv) — converts argv[i] cstrs via
                                 0x1404dc114 into a vector, then
                                 invokes R's +0x10 executor — the
                                 program interpreter / handler slot
```

So the `.bss` pointers under adapter-A program nodes (e.g.
`Faction.TriggerAmbush` `+0x08`=`0x146ffdfe0`) are **serialized
program descriptors**, not `{+0x38 entry}` structs — `R+0x38` reads
as packed data at rest (verified live on 5 records). The `a` in
`adapter(a,b,c)→[a+0x38]` is the runtime R the parser resolves.
`vt+0x18`/`vt+0x08` are the console ECHO path (0x240-byte snprintf
frame → `0x14016138c` → emit via `vt+0x08`). `vt+0x38`/`+0x40` are
index-bounded element accessors vs `[obj+0x18]`.

**Verified live**: `callva(0x1415cff44, 0x142a58bc0,
"Faction.Ambush.PrintDeferred", 0, 0)` and
`("Faction.TriggerAmbush", 1, {"MarkedAsTarget_Ambush"})` and
`("Faction.Ambush.AlwaysPassChance", 1, {...})` all returned result
objects with no crash. `sdk.console.exec` routes program-record
nodes through this verb (`registry_exec`); code handlers still use
the direct argv call. Caveat: `0x1407f9xxx` handlers are a `ret 0`
stub farm — check for a second registration (`Faction.Dump`'s real
impl is `0x141c5c048`).

`Faction:TriggerAmbush` is also a script method (colon, ONE explicit
arg — the `Faction/Ambush/Definition` record):
`Faction:TriggerAmbush(DBFactionAmbushDef.MarkedAsTarget_Ambush)` —
VERIFIED-INGAME via `sdk.exec.eval` (pcall returned true). Ambushes
defer on `AmbushDelay`/chance/conditions — `PrintDeferred`'s real
impl is a stub; trigger ≠ immediate spawn.

## Open
- The engine setter's per-varid semantics (`vt+0x28` on the
  `0x1407cbc20()` singleton) — what varids like `0x101`
  (AI.InfoTrack.Subject) actually retarget; the get path itself is
  decoded (echoes `%s = %d/%u/%f` through `vt+0x18`).
- adapter-A small-id resolution — RESOLVED 2026-09-30n via the
  registry `vt+0x58` exec verb (above); it interprets ids/records
  through `R+0x10`, no manual id→fn table needed.
- The ~4 minor marshal families (`0x141ffc4f0`, `0x1420023f0`,
  `0x1420038c0` ctxs — ~14 commands total).
- Relay-bit → event-name map and the `+0x6c` mask consumer.
