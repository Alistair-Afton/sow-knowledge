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

**Retail read path (Ghidra `FUN_140503694`, byte-verified):** the
loader reads the 0x2C header in one shot, checks `GADB` +
`version==0x11`, then reads a **0x100-byte path buffer** before the
blob data and compares it against the literal `"gamedb_patch"` —
patch gamedbs are recognized by *file path*, merged in-place by the
same loader, and rejected when the patch's `+0x2c` field doesn't
match the base DB's `+0x20` content hash. A second consumer
(`FUN_1417662b0`) is the **recursive patch-chain loader**: given any
`.gamedb` path it synthesizes a `"gamedb_patch"` variant, opens it
through the VFS, and calls *itself* — the result is a **linked list
of DB layers**, so patches can themselves be patched and the chain
resolves base→patch→patch-of-patch.

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



Byte-check vs real `patch04/game.gamedb` (58 MB): the blob spans
`[0x2c, 0x2c+strsz)` exactly (ends on a NUL-terminated string), and
its first bytes are the u16-length-prefixed self path
`database\game\game.gamedb` + `GameDatabase`. Header counts map
`+0x0c → n_tables(2,435)`, `+0x10 → n_records(233,110)`,
`+0x14 → paired pool (354,979)`, `+0x18 → value-word slots
(9,353,656)`, `+0x1c → field descriptors (395,395)` — allocation
≈100 MB runtime, consistent with the documented section sizes.

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
  Kind codes observed: `01` any 32-bit (float/bool), `02` float,
  `03` int, `04`/`07`/`44` string/path, `0A` link, `0B` struct link —
  matching the `k=` tags our own record dumps show.
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
- **Named records in every category are sorted by `name_hash(name)`**
  and the game **binary-searches** them when resolving references.
  A record appended unsorted is silently invisible to lookups —
  the "new records come back as another record or vanish after
  reload" symptom. Rewriting must re-sort touched categories and
  re-point every `catIdx<<20|recIdx` link file-wide.
- Header named/unnamed totals are **off by 2** from a record walk;
  preserve the original delta when rewriting.
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
given the sort-by-hash and link-repointing rules above. Andúril's
own writer is still OPEN tooling.
