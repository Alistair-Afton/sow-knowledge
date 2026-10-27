# GADB — the game database format

> **Status:** **verified**
> **Provenance:** GADB record model probed on real game.gamedb — Andúril + sow-rs.

`database/game/game.gamedb` (56.8 MB), every `.bvr` behavior DB, and
the `GADB` section inside each region `.objres` share this format.
It is the game's answer to "where does the behavior live": nearly all
gameplay data is typed GADB records linked by index.

## Binary layout — CONFIRMED (both projects parse it)

- Header: `GADB`, version, counts and offsets; **string-table offset
  at 0x28** (VERIFIED).
- Strings: `u16` byte length + bytes; a heap follows.
- **Layout records are 15 bytes each**; category and record data
  follow the layout table; file ends with a `u32` zero terminator.
- **Links are `category_index << 20 | record_index`**;
  `0xFFFFFFFF` = null. Links are indices, not hashes — to resolve by
  name you hash the target name and find the record carrying it.
- Categories are paths (`Combat/Attack`, `Faction/Roles/Definitions`,
  `Interface/HUD/Widget`, ...); records are named or unnamed
  (`_Structures...` struct records are typically unnamed).
- Scale (VERIFIED, installed `game.gamedb`): 2,742 categories,
  361,939 layouts, 278,865 named + 415,767 unnamed records,
  ~8.5 M value words.
- Embedded GADBs differ: the `.objres` object DB is GADB version
  0x11 with one category `ObjectDB` of `ObjectRec` records.
- Readers exist in Andúril and sow-rs.

**Header (0x2C bytes; CONFIRMED on the installed `game.gamedb`, word
meanings cross-checked with THaych's Player Skins Overhaul notes):**

| Offset | Meaning | Installed value |
|---|---|---|
| `+0x00` | `GADB` | |
| `+0x04` | version `0x11` (17) | |
| `+0x08` | string-pool size (bytes) | `0x9EE380` |
| `+0x0C` | categories + struct arrays | 2,742 |
| `+0x10` | named records (incl. default records) | 278,865 |
| `+0x14` | unnamed struct records | 415,767 |
| `+0x18` | total value words | 8,544,670 |
| `+0x1C` | layout descriptors (15-byte entries) | 361,939 |
| `+0x20` | DB identity stamp (checked by `.gamedb_patch`) | `0xD186F247` |
| `+0x24` | 0 | |
| `+0x28` | offset of the self-path (`0x2C`) | |

`+0x20` is **not recomputed from content**: the installed
`Patch_IncreasedCap` mod's `game.gamedb` differs from retail
`hotchunk` in 114 bytes yet carries the same `0xD186F247` and loads
fine — it's an identity stamp a patch is matched against, not an
integrity check on the base (CONFIRMED by byte diff).

At `0x2C` a `u16`-length-prefixed **self path**
(`database\game\game.gamedb`) follows; the string pool begins right
after it (`0x47` on retail) and runs exactly `+0x08` bytes — the
layout table starts at `0x47 + pool_size` (CONFIRMED: the first
15-byte layout entry sits exactly there). String fields are offsets
relative to the pool start.

**Retail read path (Ghidra `FUN_140503694`, byte-verified):** the
loader reads the 0x2C header in one shot, checks `GADB` +
`version==0x11`, reads the self path through the stream's
string-reader slot into a **0x100-byte buffer**, then reads
`+0x08` bytes of pool. If the self path's **extension** is not
`gamedb_patch` (and the path is non-empty) and a global enable byte
(`DAT_14266cd70`, statically `1` in retail with no code writers) is
set, it accepts an optional second stream — the patch — and drops it
unless the loaded patch's `+0x2c` field equals this file's header
`+0x20` identity stamp; an accepted patch's extra table/record counts
are added to the allocation and its tables merge into the base by
name hash (see body layer below).

`FUN_141c3ff20` (the `database\game\game.gamedb` open — reached from
the DB manager global `0x142a86a30`, vtable slot 1, via
`0x141c3fefc`) first asks the asset system for a type-`0x1d` (gamedb)
resource; failing that it builds the patch path by optionally
appending `_<suffix>` and then **replacing the extension with
`gamedb_patch`**
(`FUN_14017aa78` — appends `.gamedb_patch` when the path has no
extension) → `database\game\game.gamedb_patch`, and opens base +
patch as a stream pair. `FUN_1417662b0` is the **whole-file blob
loader** used at startup: it peeks the header + self path, and for a
non-patch file opens the `.gamedb_patch` sibling and calls itself on
it, linking the result as a second layer, then rewinds and reads
the entire file into memory. Because a file whose self path already
ends in `gamedb_patch` never looks for its own patch, a well-formed
chain is **one layer deep (base → patch)** — an earlier revision of
this doc claimed patch-of-patch chains; the extension guard rules
that out. No `.gamedb_patch` ships with retail; the practical
how-to (patch requirements, merge-by-table-name semantics, tooling
routes) is in [`gamedb-patch.md`](gamedb-patch.md). (A community note
words the stamp check as "patch `+0x20` == base `+0x2c`"; in the
decompile the base's own header `+0x20` is compared against `+0x2c`
of the *loaded patch object* — the reading above.)

The blob loader is what retail uses for `game.gamedb` (reached from a
resource-type handler at `0x141d292ec`); the stream parser
additionally runs ~4,500 times per launch on small GADB-like streams
that are not the main database (THaych, measured via hook logs).

The body layer (decoded via `FUN_1404b9b50` → `FUN_1404ba07c`):
`{u32 schema_off, u32 n_tables}`, then per-table 0x18 headers
`{name_off, 2×data_off, n_fields, n_records}` → 0x58-byte runtime
descriptors. **Patch tables merge into base tables by name_hash** —
matching names get extra records appended. Fields are
`{u16 n_idx, u8 n_descs, u32 field_pool_idx, u8 flags}` + `n_idx×u32`
index slots over a shared 0x10-stride field pool (`+0xA u8
type&0x3f, +0xC u16 count, +0xE u16 start`); pointer-typed field
slots are **rebased by the blob base at load**. Records are the same
shape plus a leading `u32 name_off` (rebased + name-hash'd,
flags `|2`), producing 0x28-byte runtime records.

Field `type&0x3f` selects pointer-typed kinds (`FUN_1404ba364`):
**4–9 and 0xC–0x10** are rebased by the blob base. Links
(`0x0A`/`0x0B`) are not rebased (record indices, not offsets);
`0x07`/`0x08` are — confirming those cells are heap offsets to
constant aggregates (FloatRange/vec3), not scalars.



Byte-check vs `patch04/game.gamedb` (58 MB, an older build): the
same header word meanings hold (`+0x0c` 2,435 tables, `+0x10`
233,110 named records, `+0x14` 354,979 struct records, `+0x18`
9,353,656 value words, `+0x1c` 395,395 layout descriptors) —
allocation ≈100 MB runtime. Correction: the pool does **not** start
at `0x2c` — the self path occupies `0x2c..0x47` and the pool begins
after it (an earlier revision placed the pool at `0x2c`, which ends
mid-string on the installed DB).

## Writer-level layout (Universal Modder, verified in-game)

Field-note detail from the "Goblins of War" / "Falmer tribe" projects
(Universal Modder kb, MIT), whose C# writer round-trips `game.gamedb`
byte-for-byte and whose rebuilt DBs load and persist in-game. Where it
overlaps our own parse the two agree.

- Heap base is file offset `0x47`; every text/blob reference is a u32
  offset from `0x47` (4-aligned). Offset `0x4C` is the empty string.
- **Layout table**: `layoutCount` × **15-byte** entries, one per
  attribute of a record shape:
  `u32 nameHash, u32 hashOfSortedSlot, u8 attrIndex, u8 sortedIndex,
  u8 kind, u16 valueCount, u16 firstValue`.
  `valueCount` is a true **u16** — CONFIRMED: 38 retail entries
  exceed 255 values, the largest 20,418 on
  `Presentation/Bundles/Facialanims:facialanims` (exactly its cell
  count). A `u8 count, u8 pad` reading of those bytes (seen in
  another community note) only works below 256.
  Kind codes observed: `01` **packed bool** (see below), `02` float,
  `03` int, `04`/`07`/`44` string/path, `0A` link, `0B` struct link —
  matching the `k=` tags our own record dumps show.
- **Kind `01` attributes are bit-addressed booleans, not value
  cells** — VERIFIED-STATIC (the engine's bool reader extracts
  `word >> (firstValue & 31) & 1`) and CONFIRMED corpus-wide:
  `firstValue` is a *bit index*; the value lives in
  `cells[firstValue >> 5]`, bit `firstValue & 31`; `valueCount` is
  always 1; the bool words are a record's leading cells and no other
  attribute claims them. 307,914 records carry `01` attributes and
  38,053 have bit indices beyond their cell count — the tell that a
  cell-indexed reading is wrong. Writers must read-modify-write the
  containing word (a bit-indexed "cell write" corrupts unrelated
  flags, and `catIdx<<20|recIdx`-shaped values there are bits, not
  links).
  **Identical value runs are shared between attributes** — rewrite a
  record by re-laying it out (append a new layout), never by poking
  one slot in place.
- After the layout table: `u32 0, u32 categoryCount`, then categories
  to EOF. Category header: `{path str, type str, file str,
  unnamedCount, namedCount, hash}` (folder categories look like
  `Path/Name, Dotted.Type, \Path\gdb.category06, 0, N`; struct
  categories `_StructuresX, X, "", instances, 1`).
- Unnamed struct instances first (`{hdr, type, values}`), then named
  records (`{nameOff, hdr, type, values}`).
  `hdr = u16 valueCount | u8 attrCount << 16 | u8 layoutLo << 24`;
  layout index = `(type & 0xFFFFFF) << 8 | layoutLo`.
  `type`'s high byte is a **constant 0x08 tag** across all 278,022
  records (full-emit census, VERIFIED) — a fixed marker, not a
  per-record field; only the low 24 bits carry layout meaning.
- **`hdr` hi16 = `{attrCount, layoutLo}`** (VERIFIED census, 5,599
  distinct values). `hi16 == 0` marks **schema-less marker records**
  (8,223 of them — tag lists, `FortAssault/Stage` names,
  `TypeOfAttack` predicates, `Combat/CharacterType`): zero cells,
  pure named markers that exist only as reference targets.
- **Named records in every category are sorted by `name_hash(name)`**
  and the game **binary-searches** them when resolving references.
  A record appended unsorted is silently invisible to lookups —
  the "new records come back as another record or vanish after
  reload" symptom. Rewriting must re-sort touched categories and
  re-point every `catIdx<<20|recIdx` link file-wide.
- Header named/unnamed totals are **off by 2** from a record walk;
  preserve the original delta when rewriting.
- Block taxonomy after the layout table (THaych's Player Skins
  Overhaul notes, in-memory patcher shipped and tested in-game):
  struct-array blocks (24-byte header, entries `8 + 4·nf`), **default
  records** (`DEFAULT_RECORD_NAME…`, `12 + 4·nf`, counted inside the
  header's named total), and categories (24-byte header; records
  `{u32 name, u16 nf, u8 ndesc, u24 layout id, u8 0x08}` + `4·nf`).
  Links address non-default blocks only. Data after the layout table
  must stay **4-byte aligned** — pad pool growth. The block count
  after the table equals header `+0x0C`; the file ends 4 bytes after
  the last record. Blocks classify by pool strings: struct array =
  valid name ∧ `w2 == 0x4C` (empty string) ∧ `w1` a valid string ∧
  `w4 ∈ {0,1}`; default record = name starts `DEFAULT_RECORD_NAME`;
  category = string at `w2` ends with `gdb.category06`. A category's
  record count sits at header `+16`.
- **Field typing** from layout entries (THaych, cross-tabulated over
  decoded values): an entry covers fields `firstValue …
  firstValue+valueCount−1` (count 0 covers nothing); kind `1` is a
  flags/bool layer overlapping other kinds — ignore it when another
  kind covers the field. Kinds: `1/2/3` numeric (`2` float), `4`
  string, `5` **enum string** (`Normal`, `CharacterCombatBuff`),
  `0x44` (68) file path, `7` vec2 pool constant, `0x0A` record link,
  `0x0B` struct-array link. Value-shape heuristics without the kinds
  misfire (`0x100` looks like a pool string).
- The record header's layout id is an **absolute index into the
  layout table** (records with the same shape share it), so a patch
  names a layout by "a record that already has it". Records of the
  same type may have different field counts; lists are inline.
- **Growing a DB without renumbering** (LordMonstey's compiler,
  1,208 weapons installed in one batch with existing ordinals
  preserved): keep the string pool as a byte-exact prefix and append
  aligned atoms; keep existing layout blocks; when a list changes
  length, clone its layout block, update that field's count/offset
  and relocate later fields in the record; recompute value counts,
  layout indices, category counts and DB totals; reparse and prove
  unrelated records and prefixes unchanged. Raw byte offsets are not
  a stable interface — tables move even when ordinals don't.
- **Naming appended records:** reordering to sort names would change
  ordinals, so give new records names whose hashes **extend the
  existing sorted tail** (LordMonstey and THaych independently use
  this). Propagate the final names to exact string consumers such as
  scripts.
- Category order = ascending unsigned name hash (THaych verified on
  Skin, Models, Buffs, Hooks); the largest categories end near
  `0xFFFxxxxx`, so new names there rarely sort last.
- **Appending without renumbering:** a new record can be appended to
  the end of a category without re-pointing any link *only if its
  name hash sorts last* in that category (THaych picked a menu-node
  name whose hash sorts last for exactly this reason). Otherwise the
  full re-sort + link rewrite above applies.
- Growing a record's field count is safe when the new shape reuses a
  layout the DB already contains (THaych grew two `Combat/Tree`
  profiles 311 → 321 fields by reusing an existing 321-field
  layout).
- The same `name_hash` (`h = h*0x397 + FOLD[c]`) keys both attribute
  names and `string.strdb` entries — new localisation keys are added
  by inserting `{hash, char_index}` pairs sorted by hash (verified:
  `TribeDef_Goblin` → `0xE2D0F0F6`, `TribeDef_Machine` →
  `0xEBB642C8`; our `name_hash` reproduces both exactly).

## Attribute names — PARTIAL, structural limitation

Attribute *names are not serialized* — each attribute slot is a hash.
Recover them by hashing candidate strings with the engine name hash
(dictionary + brute matching). sow-rs recovered ~34% of nemesis
attribute occurrences; gameplay-critical names (`DamageFormula`,
`InitialValue`, `MaterialSets`, `ModelFile`, ...) are known because
the same names appear in record categories, heap strings, or were
brute-matched. When documenting a field, cite the candidate name and
whether the hash match is unique.

## Field kind tags (CONFIRMED across our work + sow-rs)

- Ranges are kind `0x07`: a heap offset to two f32 `[min, max]`
  (both zero = unset).
- `Predicate.Command.ActionId`-style fields (kind `0x05`) are heap
  offsets in `.bvr` files to the **command name string**, which names
  a `Profile/Commands` record in `game.gamedb` (VERIFIED — used for
  the input-bindings map).
- String fields hold heap offsets to `str16` values; typed links hold
  the packed `cat<<20|rec` index; struct links nest records inline.

## Content map (record categories that matter)

VERIFIED record paths (sow-rs, read from installed DB):

- **Combat:** `Combat/Attack` (16,976 records), `Combat/Sync`,
  `Combat/Stat/Formula/{Float,Damage,FloatOperation}`,
  `Combat/Stat/Variable/{Float,Int,Bool,FloatIndex}`,
  `Combat/Stat/{Charge,Energy,WraithMeter,Ammo}MeterSetup`,
  `Combat/FightCircle/{Participant,PlayRole}`, `Combat/Predicates/*`,
  `Combat/Buffs/*`, `Combat/CounterType`, `Combat/Hint`.
- **Inventory/skills:** `Combat/InventorySetup:<name>` (character
  loadouts incl. stats), `Inventory/Items`, `Inventory/Skill/Skills`,
  `Inventory/{Weapon,RangedWeapon}Data`, `Difficulty/Settings`.
- **Models/materials:** `Model:<name>` (`ModelFile`, `MaterialSets`,
  `TranslucentMaterial`), `Model/SimpleModel(Pieces)`,
  `Character/Models` (capsule radii, `PlayerMovement`,
  `LocomotionIK`, `ContextActions` = 124 traversal edge tags).
- **Nemesis:** `Faction/{Tribes,Roles,Traits}/Definitions`,
  `Faction/NameTable`, `Faction/ClassMemberSetup` (see nemesis.md).
- **UI:** `Interface/HUD/Widget*`, `Interface/Elements`,
  `Interface/NavMarkerTypes`, `Interface/FloatingText`,
  `Interface/GlyphReplacement` (see ui.md).
- **Weather/world:** `Weather/Climates`, `Weather/Patterns`,
  `Weather/Lighting/Base`, `Weather/MaterialOverride/*`.
- **Input:** `Profile/Commands` (382 records: `ActionId`, `PCKeyboard`,
  `PCGamepad`).
- **Scripts/quests:** structural notes in the Andúril repo's
  `docs/formats/scri.md` (CONFIRMED there).

## Semantics that are proven (not just structural)

- Stat formulas evaluate as a node pipeline: running value starts at
  `InitialValueRange.min`, each node's `Operation` applies when its
  predicates pass, `ReturnImmediateIfPassPredicates` short-circuits,
  `ClampRange`/`ClampOutput` clamps. `FloatOperation` sums each of
  ADD/SUB/MUL/DIV: type 0 `(prev + A − S) × M / D`, type 1
  `prev × M / D + A − S` (VERIFIED — checked against named formulas
  like `Override_Return_10` and `Gem_GreenRing_DefScalar`).
- Side 0 = source character, side 1 = target; a formula reached from
  side 1 swaps them.
- Mesh→material binding: `Model:<n>` record `MaterialSets[0]` indexed
  by the skeleton LOD-group material index (VERIFIED — see
  materials.md).

## Editing implications

A mod that edits `game.gamedb` can retune any formula, stat, meter,
input binding, HUD widget linkage, or perception record without
touching the executable — provided the tool preserves link indices,
layout records, and the string heap. This is the single most
powerful modding surface in the game. A working GADB writer exists —
the Universal Modder project's C# reader-writer round-trips
byte-for-byte and its rebuilt databases ship working mods in-game
(new tribes, records, roles) — proving the format is fully writable
given the sort-by-hash and link-repointing rules above. A second,
non-file route is proven too: THaych's Player Skins Overhaul plugin
hooks the GADB loaders and patches the database **in memory as it
loads** (symbolic patch keyed by category + record name, refusing to
apply when an expected record/layout is missing) — no game file is
replaced, so it coexists with archive mods. The native
`.gamedb_patch` sibling path (header section above) is a third
route — loader and merge semantics fully decoded in
[`gamedb-patch.md`](gamedb-patch.md); a crafted patch has not been
booted yet.
