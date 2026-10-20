# `.objres` — OBJR v13 object resources

> **Status:** **verified** — sockets/keyframes decoded on real corpus
> **Provenance:** sockets/keyframes decoded on the real corpus.

Per-world/section object packages: sockets (attach points), keyframes,
sound effects, navmesh modifiers, destructibles, audio refs. 680 files,
2.8 KB – 96 MB.

## Header (0x10 B) — CONFIRMED

```text
+0x00  char[4] "OBJR"
+0x04  u32     version = 13
+0x08  u32     hash
+0x0c  u32     declared section count (~14)
```

(Previously documented as 0x18 with `hash2`/`0x10` fields — those
words are actually the **first section's `{name_hash, size}` frame**,
so the real header ends at 0x10.)

## Sections — CONFIRMED

`section_count` entries of `{u32 name_hash, u32 size, size-byte
payload}` starting at 0x10. Verified on `minasithil.objres` (52 MB):
the declared walk tiles to EOF exactly, and every named payload begins
with an ASCII tag + version (tags are **not** 4-aligned in the file —
our old byte-scan survey missed every unaligned tag, which is why it
only found 4 of 14 chunks).

`minasithil.objres` section table (all hashes/sizes read, walk
exhausts the file):

| # | Tag | Content |
|---|---|---|
| 0 | `KEYF` v1 | 9 key-frame tracks |
| 1 | `SFXS` | `.fxp` effect paths |
| 2 | `SWFX` | |
| 3 | `ONLS` | socket names (decoded below) |
| 4 | `NLFE` | ambient-life feature table (PARTIAL — see below) |
| 5 | `PFDG` | nav-surface points + markup records (PARTIAL decode below) |
| 6 | (none) | clutter-prop `.mat` paths |
| 7 | `SCRB` | script bindings (decoded below) |
| 8 | `VOLM` | volumes |
| 9 | `CAWR` | CADB group-key lookup mirror (569 × `{u32 key, u32 0, u32 key}`) |
| 10 | `CADB` v3 | context-action traversal edges — see gameplay/traversal docs |
| 11 | `GADB` | region gamedb (`ObjectDB`); names the `*.ss_*` stamp bundles |
| 12 | (none) | object resources: 740 × `{u16 len, path}` (.inst/.fxp) + `u32 id, f32 draw_dist, vec3 half_extent, vec3 centre` |
| 13 | `OBJL` v5 | object list (71,269 objects on MI — layout below) |

## CADB v3 — context-action edge markup — CONFIRMED (structure)

Decode implemented in `anduril-objr` (`Objres::cadb`) and verified
on `minasithil.objres` — every count matches sow-rs exactly:

```text
"CADB" u32 3,
u32 action_count, u32 edge_count, u32 plane_count,
u32 volume_count, u32 group_count
actions:  u32 name_hash[action_count]    (ContextActions record names)
edges:    52 B = vec3 a, vec3 b, vec3 n, u32 mask[4]   (mask bits over actions)
planes:   16 B = vec3 normal, f32 d
volumes:  32 B = vec3 min, vec3 max, u32 plane_start, u32 plane_count
groups:   52 B = u32 key, vec3 min, vec3 max,
          u32 edge_start, u32 edge_count,
          u32 vol_start, u32 vol_count, u32, u32   (sorted by key)
```

Minas Ithil: 75 actions, 181,578 edges, 829 planes, 85 volumes,
569 groups; largest group `0x6B9D05D0` = the world group (122,245
world-space edges spanning the whole city); most other groups are
prefab-local (origin-centred bboxes — they render stacked at the
origin in a naive plot). `n` points onto the walkable top.
CLI: `objres <path> cadb` (counts + per-action histogram + top
groups), `objres <path> cadb-plot <out.png> [group]`.

**Group-key derivation — CONFIRMED:** every group key is the engine
`name_hash` (`h = h*0x397 + FOLD[c]`) of an **OBJL object name** —
all 569 MI keys resolve. `0x6B9D05D0` = `name_hash("WorldInitializer")`
(the world group). 405 keys are `*.vc` placement objects; the rest
are `*.VC_Alive`/`*.VC_Destroyed` state variants and stamp keys
(`SS_05`). `CAWR` = the lookup mirror
`{u32 count, count × {u32 key, u32 0, u32 key}}` sorted by key.
Local-group edges are in the prefab's local frame — transform by
the OBJL record's pos/quat to world space.

## PFDG — nav-surface points + markup — PARTIAL

Decode implemented in `anduril-objr` (`Objres::pfdg`), verified on
`minasithil.objres` (1,547,607-byte section):

```text
u32 n1, u32 n2                          (33 609, 35 186 on MI)
verts:   (n1 + n2) × 12 B vec3 slots — nav-surface points;
         ~2 % of slots hold non-plausible data (NaN/sentinel) and
         are skipped on the same 12-byte grid
records: 21-byte markup records. Flavour-B shape is the *real* one:
         {u32 seq, rgba u8×4, u32 a, u32 b, u8 flag, u32 -1} — 23 984
         on MI. The apparent flavour-A records ({rgba, a, b, flag,
         -1, -1}) are tolerant-walk false positives in between.
```

- The point cloud renders as Minas Ithil's street layout when plotted
  top-down — the walkable-surface claim is CONFIRMED by shape.
- Real records: `flag ∈ {0,1}` only (binary classification — the
  "flavour A" tail showing dozens of flag values is the misparse
  tell). `a`,`b` are index-like pairs with `b−a` ∈ ~20–42; identical
  (a,b) pairs repeat up to 1 388×, so they identify a *shared region
  window* per record, not a per-record range. What they index is OPEN
  — **not** the vert array: `verts[a..b]` has no spatial compactness
  (0/2000 sampled) and `seq` never falls inside `[a,b)` (1/2141). A
  nav-region/table slot space is the remaining candidate.
- `seq` is `0xFFFFFFFF` on 21 843 of 23 984 records; the 2 141 real
  seqs are **strictly increasing** (2140/2141) spanning
  `25 250..33 607` — topping out right at `n1=33 609`, so `seq`
  indexes the **first vert partition** (`verts[0..n1]`).
- The two vert partitions are **different record kinds**: partition 1
  is clean positional (bounds ±59k/13k/±41k world cm), partition 2 is
  **interleaved 24-byte cells** `{vec3 desc, vec3 pos}` — alternation
  confirmed; small slots sit at even parity (15 376 vs 820), so the
  pair order is `{desc, pos}` on the slot grid. `pos` slots alone
  render as the city's nav-cell centres (street map). `desc` triples
  are mostly distinct per cell (13 951/16 196), quantized-looking
  (low mantissa bytes `00/40/80`), magnitude ~10–300 — per-cell
  extents or packed flags, semantics OPEN. `pos±desc` does not hit
  array1 points (88/820 ≈ noise); mean |desc| (230,142,226) vs mean
  cell spacing (711,135,718) — no edge-vector match. `(a,b)` windows
  are not strictly pair-aligned (~43% even ends). `desc.y` clusters
  at quantized steps (0, 40, 51, 67.5, 100, 155, 200 — ×0.5 buckets),
  consistent with per-cell height/extent data. seq'd array1 verts are
  NOT near their record window's cell centres (nearest-centre
  distance p50 ≈ 19 093 cm, n=2 127 — windows index slot space; the
  dense array is shifted by skipped slots, so treat as approximate).
- `(a,b)` windows (618 distinct on MI) are **fully disjoint** and
  span `100..35 169` ⊂ `[0, n2=35 186)` — they partition *regions of
  the second array*, not the verts. Records sharing a window
  (≤1 388×) are markups grouped under that region; `a` is mostly
  nondecreasing (19 479/23 984) — records are window-grouped.
- `rgba` runs the rainbow spectrum — debug colours per record/group.
- OPEN: the second array's element semantics (nav-cell edge lists?),
  the flag 0/1 dichotomy, what interleaves between seq records.
- CLI: `objres <path> pfdg`, `objres <path> pfdg-plot <out.png>`.

Tags observed corpus-wide: `KEYF` (keyframes: `{u32 hash, u16 f16?}`
pairs seen), `SFXS`, `SWFX`, `ONLS`, `BOLS`, `TEDJ`, `TSDJ`, `BSWA`,
`BEWA`, `BBWA`, `TAOO`, `QSWA`, `QEWA`, `WAOO`, `PGOD`, `QSDJ`, `QEDJ`,
`PELS`, `PMND`, `TAIO`, `WAIO`, `DOMN`, `AOMN`, `MNIA`, `SGOD`, `CFXS`,
`TSED`, `SLOB`, `SCRB`.

## NLFE — ambient-life feature table — PARTIAL

`NLFE u32 1` header then `u32 28, u32 1086, u32 41, u32 16882, u32 3`
(field semantics open). Payload contains a name table of **1,097**
strings — ambient creature/spawn names observed on MI:

- `<none>`
- `con_*` names (`con_HideShrubAllDay`, `con_MeatFlies`)
- `QT_Caragor_*.P.NLM_MF01` (caragor spawn/feature points)
- `HideS_*.Day` hide-spot names, `OakAdult_*.Day` perch names

The string count matches the number of small-int u32 lanes in the
section (1,097). Trailing data is dominated by a repeated `0x1b` (=27)
lane — a per-record flag/stride is the likely reading but element
layout is OPEN. Not nav-surface data; unrelated to PFDG's cells.
CLI: `objres <path> section NLFE` dumps header, hex, lane census and
the string table.

## ONLS (socket names — fully decodable)

`{ONLS, ver, u32 hash, u32 ?, u32 first_table_count}` then:

- Table 1: `count × {u16 len, name}` — SLOC_*/SLOM_*/CE_*/CM_ singles
  (`0x59d`=1437 on ol_sorcerous)
- Table 2+: `{u32 group_size, group_size × u16-str}` runs — named
  groups (`Fort_*`, `CBL_*`, `Cave_*`, `cB_*`, `cM*`, `Shelob_*`,
  `a3`=163-member group seen)
- Then binary socket transform data (f32 records — pos/quat per socket)

`Objres::socket_names` — 681,385 names extracted corpus-wide. These are
attach-point names referenced by prefab/animation systems.

## Verified

680/680 files parse through the VFS. CLI: `objres <path>` (chunk survey),
`objres <path> sockets`, `objres <path> cadb`,
`objres <path> cadb-plot <out.png>`, `objres scan` → 680 ok / 0 failed.

## OBJL (section object list — names + record container decoded)

Present in **section** objres (the `.embb`-embedded member), not the
loose world `.objres` files. Layout:

```text
+0x00  char[4] "OBJL"
+0x04  u32     version = 5     (outside the tag survey's 1..=4 window)
+0x08  u32     flag = 1
+0x0c  u32     record_count    (verified: region/count == 50.0 exactly
                                on sotp_05/_06 and sotp_island_01)
+0x10  u32     ?               (278 / 253 / 551)
+0x14  u32     ?               (2823 / 3628 / 20409)
+0x18  u32     ?               (95 / 193 / 601)
+0x1c  …       cstr name table — terminated by first non-printable byte
after table:  record_count × 50-byte records (OBJL is the last chunk;
              region runs to EOF, small tail of <=8 bytes observed)
```

**Retail read path (Ghidra `FUN_140502b8c`, byte-verified):**
`{OBJL, ver==5}` exact; after the header the loader bulk-reads
**two** blobs (name table + param blob) plus an optional third,
then walks `group` headers `{u32, u32 n}` with `n × 50B` file
records expanding to **0x50-byte runtime records** — the first two
u32s of each file record are **offsets resolved against the two
blobs** (name offset → blob A, aux offset → blob B, matching the
"type name offset" finding below), followed by `u16, u32, …`
fields. A trailing base-list rebuild emits the engine's own error
strings `"The base list off"` and `"Mismatch suggests the object
count logic is incorrect and may be stomping memory."`

Record layout (50 bytes):

```text
+0x00  u32     unknown — zero on every record except record 0
               (0x3bde767d on sotp_06)
+0x04  u32     unknown id — stable per instance-name across sections
               (content-derived), but matches `name_hash` of no
               decoded string (not the record/aux/parent name, nor
               its prefix/suffix; also not crc32/fnv/djb/sdbm/jenkins)
+0x08  u32     type/prototype name offset — lands exactly on a name
               boundary every record; points at a TYPE name for
               instances (`TeleportPoint`, `AINodeMenacePlace`,
               `Group`, `AISpawnerAmbient*`, `ScriptObject`, …) and at
               the previous type name for the type records themselves
               (a prototype chain).
+0x0c  u32     own-name offset — add header +0x10 (`name_bias`):
               `+0x0c + name_bias` lands on a name boundary for EVERY
               record (172/223/1078) and resolves to the record's
               instance name — `SP06_TP_Captain_*`, `GTeleportPoints`,
               `GOrcs`, `SP06_Menace_*`, `Ammo_*`, …
+0x10  u16     link — 0xffff = unbound; else shared by records in a
               placement cluster: the cluster's base index into the
               LOGICAL propdb row space (link + 5 = first member's
               logical row; members occupy consecutive logical rows
               in stream order; the +5 head is the section-level
               block). Each logical row dedups to a physical row via
               `aux_index`. sotp_06: Inf_02's logical rows {18..23}
               dedup to physical {6,12,13,14,10,11} — only 3 new
               rows. Logical table size = 92 (footer word 0x5c);
               max link + cluster tail + 5 matches. Count of
               non-0xffff links == SCRB +0x10 == bound records.
+0x12  u16     aux_index — the record's ObjectPropertyDB row index
               (direct; verified exact on sotp_05/_06, sotp_i01
               301/301, fort_mtl 581/581). Records sharing a value
               share the deduplicated row — e.g. all six
               `SP06_Spwn_Archers_*` carry aux_index 48). Indexing
               the OBJL name table by it yields a record name
               associated with the row — the name table appears
               row-aligned through the propdb range (aux 48 names
               `Ammo_L_02.IIUI`, aux 35 names `Ammo_Inf_02.CP`).
+0x14  u16     flags (mostly 0)
+0x16  f32[3]  position — plausible world coords on every record;
               consecutive records frequently share a transform
               (placement clusters)
+0x22  f32[4]  rotation — a UNIT QUATERNION on 100% of observed
               records (1,473 across sotp_05/_06 and sotp_island_01)
```

The name table is a string intern table: **type names first**
(`WorldInitializer`, `WorldCollision`, `DebugScript`,
`TeleportPoint`, `Group`, `ScriptObject`, `StartupCommand`,
`ScriptRelay`, `CombatProp`, `InventoryItemUI`, `ScriptTimer`,
`SpecialFXSeq`, `MapCanvas`, `QuestRelay`, `AISpawnerAmbient*`,
`AINodeMenacePlace`, `PhysicsBSP`), then instance names —
`SP06_Spwn_*` spawners, `SP06_TP_Captain_*` teleport points, script
instances by class suffix (`Statu_00.SO`, `.QR`, `.TP`), and named
groups (`GTeleportPoints`, `GOrcs`, `GNodes`, `GAmmo`) — the names
scripts bind via `Group:Create(scriptId, "GTeleportPoints")`.

Records are the section's live object list in placement order.
**Group membership is contiguous**: a `Group`-typed record
(`GTeleportPoints`, `GOrcs`, `GNodes`, `GAmmo`) is followed by its
member records until the next `Group`-typed record — verified by
content on sotp_06: `GOrcs` precedes the `SP06_Spwn_*` spawners,
`GTeleportPoints` precedes the 8 `SP06_TP_Captain_*` points,
`GNodes` precedes the `SP06_Menace_*` nav nodes, `GAmmo` precedes
the `Ammo_*` pickups. These are the collections scripts bind via
`Group:Create(scriptId, "GTeleportPoints")`.

`Objres::object_names` + `Objres::object_records` /
`objres <path> objects|records` / `embb <section.embb> objects|records`.

**Undecoded:** the exact `link`→SCRB index arithmetic, header fields
+0x14/+0x18, and the +0x00/+0x04 ids.

## SCRB (section script bindings — partially decoded)

Precedes OBJL in section objres blobs. Layout:

```text
+0x00  char[4] "SCRB"
+0x04  u32     version = 1
+0x08  u32     string-blob size (932 on sotp_06)
+0x0c  u32     field f1 (20 on sotp_06)
+0x10  u32     field f2 = count of OBJL records with link != 0xffff
               (38 on sotp_06 — exact match)
+0x14  …       cstr blob (binding field names + script class names)
then:          {u32 hash, u32 name_off} pair stream — ends at the
               first pair whose name_off >= string-blob size
```

The pair stream alternates two row kinds:

- `{name_hash(field), field name}` — `Script`, `Script1`,
  `StartQuestScript`, `CleanupQuestScript`, `CompleteQuestScript`,
  `FailQuestScript`, `UserCancelQuestScript`,
  `RetryScreenDisplayedScript` — the hash provably equals
  `anduril_strdb::name_hash` of the field name.
- `{0x8ff2f159, class name}` — constant hash tag (the hashed word is
  undiscovered; not any obvious `ScriptFileName`/`ClassName`
  candidate) with the HavokScript class name the binding attaches:
  `_ammopickup_a6dd1295`, `_ammo_infinite_SC_Script_f1f85c6b`, …

So each binding = field/role name + script class name, and OBJL
`link` marks which records consume bindings.

After the pair stream sits the **bound-record table** — `{u32 flag,
u32 name_hash, u32 entry}` rows, sorted by hash, exactly one per
OBJL record with `link != 0xffff` (row count == SCRB +0x10 == bound
count on all three test sections: 38/78/184). `name_hash` is
`name_hash` of the bound record's own instance name (37/38 match on
sotp_06); `entry` indexes the pair stream in 16-byte binding units:
`pairs[entry*2]` = the field-name row (`Script`, `Script1`, …),
`pairs[entry*2+1]` = the bound script class. Verified: `Ammo_Inf_01`
.SO/.SC/.ENABLE/.DISABLE resolve to entries 2-5 =
`_ammopickup`/`_ammo_infinite_SC_Script`/`_ENABLE_Script1`/
`_DISABLE_Script1`, and shared bindings (`Ammo_Inf_02` reuses
entries 2-5) decode identically. flag is 1 on most rows, 6 on the
`Statu_00*.SO` rows — meaning open. So: **record name → bound script
class is fully decodable** via hash lookup; `link` adds a
group-level id whose exact target is still open.

## Trailing chunks (VOLM + GADB)

After SCRB's bound-record table and before OBJL sit:

- `VOLM` v8 — small fixed-size blob (~0x50 B on sotp_desert_06).
  Probably volume/region data; undecoded.
- `GADB` v17 ×N — embedded per-section GameDB images. The larger one
  is the section **ObjectPropertyDB** — **decoded** (below).

## Embedded ObjectPropertyDB (`GADB` v17)

Bvr-family layout (same envelope as `.bvr`, one type block):

```
+0x00 'GADB' +0x04 u32 17  +0x08 u32 strsz  +0x0c u32 type_count(1)
+0x10 u32 record_count    +0x14 u32 0        +0x18 u32 slot_total
+0x1c u32 prelude_count  +0x20 u32 seed     +0x24 u32 0
+0x28 u32 path_off(0x2c)
@path_off: u16 path_len + path bytes (empty on sotp_*)
@strings_base = path_off+2+path_len: strsz-byte cstr blob
             (strptr values index this blob)
@graph = strings_end: prelude_count × 15 B field-schema records:
   {u32 name_hash, u32 link_hash, u8 field_index, u8 f1,
    u8 class, u8 codec, u8 ?, u16 slot}
   — `link_hash` chains each field to the previous field's
   name_hash; `name_hash` resolves against the *engine's* component-
   schema namespace (`name_hash` of the field name, recoverable via
   `anduril_gamedb::attr_name` — resolved examples: `Type`,
   `ScriptTags` (the recurring run-head 0x602c5104), `StartEnabled`,
   `CombatProp`, `Item`, `Timer`, `AIType`, `Radius`,
   `PerceptionOverride`, `DispatchConfig`, `SmartObjectPicker`,
   `Face`). ~30% resolve against the exe schema corpus; the rest are
   per-component-type fields whose names aren't in that dictionary.
@prelude_end: footer {u32 0, u32 type_count}
@type blocks: {u32 type_stroff, u32 aux_stroff, u32 x, u32 0,
   u32 count, u32 type_hash} — unlike .bvr the two stroffs differ
   (here 'ObjectDB' / '').
   count × ObjectRec instances, each:
   {u32 name_off ('ObjectRec'), u16 sc, u16 aux, u32 0x08000000,
    u32 slots[sc]}
   — `aux = {first_field_index << 8 | declared_field_count}` selects
     which slice of the field table this record's slots serialize
     (verified: aux=0x0409 → fields 4..12 = 9 slots; 0x1b0e → fields
     27..40 = 12 slots (2 default-elided); 0x5906 → 89..94 = 6).
   — slot values < strsz are strptrs (component tags '#.CP',
     '#.ENABLE', quest names, anim names 'Bip_Sentry_Looping'…);
     larger values are f32/int cells.
```

sotp_desert_06: 102 field-schema records, 88 ObjectRecs — contents
match the section's objects (quests, `Enable`/`Disable`/`Ammo_Large`
component rows, `SotPCleanup;` tags, archer/captain/grunt entries with
per-instance anim + region + quest slots).

Every ObjectRec's `name_off` resolves to `"ObjectRec"` — a type tag,
not an instance key, so row identity is positional.

A fixed 112-byte chunk follows the record stream and precedes OBJL —
byte-identical across sotp_05/06 (`{0, 0xb9e81cf3, 92, 1, 1,
name_hash("PhysicsBSP"), …}` with `0xfafa…` sentinels and a `1.0f`).
Section-independent constant — probably the `VolumeDB` definition the
catalog GADB's strings name, not propdb data.

This is the value source for `WBScriptPropBool`/`WBScriptPropString`/
`*.Create` calls in bound constructors (`StartsEnabled`, `myCP`,
`myEnableSR`, `IIUI`…). Runtime wiring is in place (see
`simhost::script_prop_call`): `WBScriptProp*` resolves the bound
record's `link`-anchored propdb span, matching cells first by
schema field name (normalized — leading `s`, case) then by `#.X`
component-tag value for slot-style props (`myCP` finds the cell
containing `"#.CP"`); unresolved props return the caller's default
as retail does for unset values. `X.Create(scriptId, tag)` resolves
`#.X` tags to the sibling OBJL record (`Ammo_Inf_01.SO` + `#.CP` →
`Ammo_Inf_01.CP`) via `Objres::resolve_component`.

Mapping OBJL record → ObjectRec: `link` anchors each cluster's propdb
span (see Unknowns). Row *contents* match record semantics exactly
where rows are contiguous:
on sotp_06 rows 6–11 align 1:1 with `Ammo_Inf_01`'s six component
records in OBJL order — the `.SO` row lists `ObjectDB` + `#.FX_GLOW`,
`#.CP`, `#.FX`, `#.NM`, `#.IIUI`, `#.FXS_00`, `#.PFX` component tags,
`.SC` is empty, `.ENABLE`/`.DISABLE` carry `"Enable"`/`"Disable"`
command strings, `.CP` carries `"AmmoRecharge"` (the ScriptTimer
name), `.IIUI` carries `"Ammo_Infinite"` (the item name). Later rows
diverge from OBJL order — `.IIUI` rows elsewhere carry `"Ammo_Large"`,
spawn/NPC rows carry quest names + `SotPCleanup;` tags + `Bip_*`
anim names + 3 f32s (likely region/position). Ordering across the
table may be by registration order or a per-cluster component sort;
the join remains open.

## Retail read path (Ghidra decompile, verified)

Loader `FUN_1407a75c8` + body parser `FUN_14048094c`: `version == 13`
checked by equality; after the 0x10 header, `count` sections are read
as `{u32 name_hash, u32 size, u8[size]}` exactly as documented — and
each payload is dispatched through a **hash→vtable-handler registry**
(`FUN_14115c2c0`), which is why every payload tag (`KEYF`, `SFXS`,
`CADB`, …) maps to a dedicated sub-parser.

Section parsing is **engine-tolerant**: the payload runs on a sub-stream bounded to `size`, the loader force-seeks to `section_start + size` afterwards, and a hash with no registered handler is skipped entirely. Section table entries are 0x20-stride `{hash, pad, handler*, result*}`.

## Unknowns

- OBJL `key0`/`key1` (per-record u32s): `key1` is stable per instance
  name across sections but matches `name_hash` of *no* decoded string
  (not the name, type, bound class, `parent+name`, `#.<suffix>`,
  section-qualified path, gamedb record names, or any printable string
  in the embb); also not crc32/fnv1a/jenkins/djb2 of the name. Likely
  hashes a string we haven't surfaced, or isn't a string hash at all.
- OBJL record → propdb row: **two-level mapping.** `link` gives the
  cluster's base in a *logical* row space (one consecutive row per
  cluster member, +5 section head); `aux_index` gives the *physical*
  deduplicated row directly — records sharing a value share the row
  (six `SP06_Spwn_Archers_*` → 48, both `Statu_00*.Utility` → 37,
  seven ammo `.CP` → 10). Identical property sets dedup across
  prefab instances: `Ammo_Inf_02`'s six logical rows map to physical
  {6,12,13,14,10,11}, reusing three of `Ammo_Inf_01`'s. `#.X` tag
  cells resolve relative to the calling record, so shared rows
  still produce record-relative results. The OBJL name table is
  row-aligned through the physical range so
  `object_names[aux_index]` names a row-associated record.
  Verified exact: sotp_i01 301/301, fort_mtl 581/581, sotp_05/_06.
  `+0x10` header word =
  physical record count (88/132/301/1140), `+0x18` = total slot
  count (Σsc). Row content matches record semantics: `.SO`→
  `ObjectDB` + `#.X` tags, `.ENABLE`/`.DISABLE`→`"Enable"`/
  `"Disable"` strings, `.CP`→timer name, `.IIUI`→item name, quest
  records→quest names, NPC/spawn rows→`Quest_N` + `SotPCleanup;` +
  `Bip_*` anim + 3 f32 region coords, wall/tower rows→alignment
  command + jumpdown/climb strings.
- Per-tag chunk schemas (only ONLS is structurally decoded).
- Declared chunk count vs tag-survey mismatch (nested/untagged chunks).
- Chunk header field meanings (hash/count/size-per-tag varies).
- Socket transform record layout after the name tables.
