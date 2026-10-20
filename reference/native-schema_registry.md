# Native schema registry — registration record + attr-entry layouts
> **Status:** **partial** — Project Nemesis source doc; per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE / VERIFIED-INGAME / MODEL / UNVERIFIED) preserved inline. Not independently verified by Andúril.

> **Provenance:** Project Nemesis runtime-model documentation (native `nemesis.dll` live-game forensics), mirrored and maintained in the corpus. Their `VERIFIED-STATIC`/`MODEL` tags carry over; unverified entries are marked inside each doc.


**Tag:** VERIFIED-STATIC (entry writer `0x140c6da00` + init-thunk drivers
fully decoded; live `schemadump()` cross-check pending)
**Evidence:** a corpus tool; the extracted corpus
(2,157 classes); the extracted corpus (17,931 attrs);
a live probe.

## Registration record (`.idata`, ~0x30 bytes)

The class descriptor table is a linked list of records:

| Off | Type | Meaning |
|-----|------|---------|
| +0x00 | ptr | attr table VA (`.idata` BSS tail, populated once at boot) |
| +0x08 | u32 | attr count |
| +0x10 | ptr | C++ class name cstr (`MenuLayerOptions`) |
| +0x18 | ptr | description cstr |
| +0x20 | ptr | schema name cstr (`Interface.Menu.MenuLayer.Options`) |
| +0x28 | ptr | parent registration record (inheritance chain; null at root) |
| +0x30 | ptr | optional evaluator hook — only `Parameterization.*` leaves set it (the per-leaf compute fn, e.g. `DeltaX`→`0x1416ce67c`); zero elsewhere |

Registration records sit in a **dense 0x40-stride run** in `.idata`
(interleaved with other tables — most gaps between adjacent records
are 0x40, some 0x80+ = unrelated data between runs).

**Duplicate schema names (18, all `Parameterization.*`):** the same
leaf schema registers once per family — under `Parameterization`
(14 attrs) and `StateParameterization` (13 attrs) parents. Identity
is `(schema, parent)` not schema alone.

Extraction: `native_schema.py` walks `.idata` for this shape — 2,157
records, parent links resolve the full hierarchy. TSV `reg` column
holds the record VA; `hook` column holds +0x30 when nonzero.

**Live registry array (VERIFIED-STATIC 2026-10-01):** CRT-init
publishes every reg-record ptr into the contiguous slot array
`0x142a58cf0..0x142a5f290` (3,253 slots — 2,164 source-verified reg
ptrs via `lea rax,[reg]; mov [slot],rax`; gaps hold other per-class
statics via different forms). `schemadump()` walks it directly —
self-describing, no TSV needed — and cross-validates the static
corpus.

## Attr entry `{0x48 bytes}`, `table[i]` (`0x140c6da00` = entry writer)

Writer signature (decoded 2026-10-01 — MenuManager `OptionsLayer` thunk
`0x141701f98` and `DefaultLayer` thunk `0x1416e4304` traced end-to-end;
stack deltas fully resolved):

```c
0x140c6da00(entry /*rcx*/, name /*rdx*/, tag /*r8d*/, ctx16 /*r9*/,
            type_str /*stack+0x20*/, aux1 /*+0x28*/, aux2 /*+0x30*/,
            flag_fn /*+0x38*/, aux3 /*+0x40*/)
```

| Off | Type | Source | Meaning |
|-----|------|--------|---------|
| +0x00 | cstr ptr | rdx | attr name (`OptionsLayer`) |
| +0x08 | cstr/obj | stack+0x20 | constraint/type annotation — `"public X"` for refs (`"public None"` = self/poly), `"min=0.0"`-style range strings for scalars, or the shared `0x141db10e0` descriptor object for unconstrained types (bools observed) |
| +0x10 | * | ctx16[0] | **attr default value** — float attrs carry a literal (`f:1`, `f:0.8`); bools carry the **interned constant elem** (`0x141e72cf8`="true", `0x141e72d00`="false"); colors carry packed ARGB (`0xFF0000FF`); record-refs and arrays share the empty-record sentinel `0x141db10e0` (`{0,'LITH',0,…}`) |
| +0x18 | u32 | ctx16[1] | tag echo — only tag-3 thunks write it (`[r11-0x10]=r8d`, 4 sites); pad/uninit elsewhere |
| +0x20 | u32 | computed | name hash (`h*0x397+fold`, LUT `0x141db50d0`) |
| +0x24 | u32 | r8d | **codec tag** (10 = record-ref for `OptionsLayer` ✓) |
| +0x28 | u32 | stack+0x28 | aux (1 observed) |
| +0x2c | u32 | stack+0x30 | aux (6 observed) |
| +0x30 | u32 | — | **entry class** (VERIFIED-LIVE): `2` = leaf attr, `0` = group marker (`BeginBaseFX`/`EndBaseFX`/`BeginEffectIntensity`…) |
| +0x38 | fn ptr | stack+0x38 | writer/accessor thunk VA — families split leaf vs marker (`0x1403e*` leaf, `0x14162*`/`0x14078*` marker in CameraShake) |
| +0x40 | u32 | — | leaf ordinal within the marker-delimited segment — equals `i` until the first marker, lags per marker after (CameraShake: leaf 5→4 post-`BeginBaseFX`; VERIFIED-LIVE) |
| +0x44..0x47 | — | — | tail/pad |

**No instance field offsets live in the schema entries** — the table
maps name↔type↔codec↔default only; C++ layout carries offsets. Attr→
storage binding is therefore name-keyed (record cell bags), consistent
with GameDB's name→record model.

**Record-constructor consequence:** since every attr carries its
default in `+0x10` (11,373/17,931 attrs extracted to
`native_attrs.tsv` col 6 — `f:<float>` literal, `i:<int>`,
`0x<va>` pooled-elem/sentinel ptr), a record can be fully
instantiated from the class table alone — for each attr, copy
`ctx16[0]` into the new record's name-keyed cell. Cells store
**interned elems, not raw scalars** (bool defaults are the pooled
`"true"`/`"false"` elements themselves). The engine almost certainly
has a native schema→record ctor doing exactly this; finding it is the
clean path for record creation (G41 follow-on).

Codec tag vocabulary (from 17,931 extracted attrs, join vs
`schema_dump.tsv` confirmed the schema-side tag column is NOT the codec):

| Tag | n | Meaning |
|-----|---|---------|
| 10 | 6,322 | record-ref (`+0x08` = target schema, e.g. `AI.Morale.ModifierType`) |
| 4 | 2,856 | sentinel/struct markers (`BeginActions`/`EndLayers`…) + scalar-ish |
| 1 | 1,839 | numeric/bool scalar — bool defaults = interned `"true"`/`"false"` elems (`0x141e72cf8`/`0x141e72d00`) |
| 2 | 1,672 | float scalar — `movss` literal defaults, `min=`/`max=` constraints |
| 11 | 1,110 | array/container (`AutoWidgets`, `MoraleInfluence[]`) |
| 3 | 874 | packed color/int (`MapRenderStartColor` default `0xFF0000FF`) |
| 7 | 325 | ranged/time-ish scalar (`Duration`, `Range`, `FollowDelay`; `min=` constraints, f32 defaults) |
| 5 | 252 | **enum** — annotation `type=E<EnumName>` (109 distinct enum types; 17 registered name-only — see G5) |
| 13 | 166 | **CurveFloat** — `CurveFloat;curve=Linear` annotations |
| 8 | 148 | vector/mask-ish (`HitDimMultiplier`, `LockNodePosMask`, `PenShapeOffset`; `i:0x3F800000` = f32 1.0 via imm) |
| 16 | 43 | **CurveVector** — `CurveVector;curve=Linear` |
| 12 | 39 | **CurveColor** — `CurveColor;curve=Linear` |
| 15 | 18 | **CurveInt** — `CurveInt;curve=Linear` |
| 9 | 14 | matrix/rect component (`ColorMatrix.*Channel`, `MouseWheelArea`) |
| 14 | 8 | **Texture** asset ref (annotation `Texture`) |
| 6 | 2 | rare (`XOneAchievementID` ptr-default, `PlayerName`) |

Curve tags carry a `;curve=<Interpolation>` annotation suffix; enum
tags carry `type=E*`. 2,243 rows have empty tag (thunk-variants whose
tag write wasn't matched — inheritance fills most names).

## Init mechanism

Each class's `.arch` init driver does `lea rcx,[tbl+i*0x48]; call thunk`
per attr. Two thunk forms:

- **Standard** — `lea rax,[type_str]; lea rdx,[name]; lea r8,[callback];
  mov r9d,tag; jmp FILL_FN` (or `call`; FILL_FN = `0x140c6da00`).
- **Variant-B** — inline fill; name on `r8`, type on `rax`.
- **Jump-stub** `0x1408c4ee0` — Themida-VM'd shared self-registration
  used at idx 0 by ~1,600 classes; names unrecoverable statically.

Child tables carry the **parent registration prefix**: the child driver
runs the parent's init helpers on the same table region, so inherited
attrs occupy the same indices (`native_schema.py` fills unnamed child
entries from ancestors).

## Live capture (VERIFIED-LIVE 2026-10-01)

`! _G.schemadump()` walked the live registry array — 2,156 class
records self-described from `0x142a58cf0..f290`, full dump landed in
the extracted corpus (22,866 ATTR rows; each ATTR row
carries its entry VA so rows bucket to their owning table —
`e − idx·0x48` — which is what separates the dual-family
`Parameterization.*` registrations). Live vs static
(a corpus tool → the extracted corpus):

- **+0x20/+0x24 confirmed**: qword `0x0b'766ea39b` for
  `InitialAICount` = tag 11 over the canonical name-hash
  (`h*0x397+fold[c]`, a corpus tool::name_hash`) — all sampled
  rows match.
- **+0x30/+0x40/+0x38 revised** — see table: +0x30 is a leaf/marker
  class flag (2/0), +0x40 a per-segment leaf ordinal, +0x38 the
  writer thunk (families distinguish marker entries).
- **Coverage**: 2,138 live schemas vs 1,973 static — every live class
  record matches a static one by table VA (zero live-only classes;
  the 166 "gaps" were classes whose *attr rows* the static scan
  missed via the Themida `0x1408c4ee0` jump-stub motif — all now
  recovered through the merge); 1 static-only.
- **Count divergence** (1,670 rows in the diff TSV): 476 exact, 877
  differ only by phantom empty-name rows (static emits unnamed slots
  the live table never populates — parent-prefix spill), 515 static
  *short* (multi-init paths write more attrs than the first-seen
  thunk declared — e.g. `AnimatedExtra.MovementPosition` 2→7), 104
  over for other reasons.
- **The live dump is authoritative for count/name/order**; static
  adds tag/type/default/thunk the dump can't see (name-hash + tag now
  verifiable both ways via +0x20/+0x24).
- **Merged corpus** (22,866 attrs, VERIFIED-LIVE): live rows keyed
  per registration VA, including the 166 jump-stub classes and both
  `Parameterization.*` families.
