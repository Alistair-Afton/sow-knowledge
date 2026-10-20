# `.bvr` — behavior shards (GADB v17)

> **Status:** **verified** — behavior graphs walk; node/edge semantics decoded
> **Provenance:** real GADB `.bvr` object graphs walked; node/edge semantics decoded.

Per-character AI/animation **behavior graphs** under `behaviors/`
(29 files: `ai/characters/**` for NPCs, `player/player_*.bvr` for the
four player rigs). Each file is a self-contained "BlendDB" — the
blend-tree + predicate graph that drives a character's animations.

## Envelope

```text
+0x00  char[4] "GADB"
+0x04  u32     version = 17
+0x08  u32     strsz — size of the string table (after the path)
+0x0c  u32     type_count — registered types (= block count on corpus)
+0x10  u32     0
+0x14  u32     0
+0x18  u32     record_count — scales with total instance count
+0x1c  u32     prelude_count — number of 15-byte prelude records
+0x20  u32     seed-like word
+0x24  u32     0
+0x28  u32     source-path offset (= 0x2c)
+0x2c  u16     path byte length
+0x2e  bytes   source path (`behaviors\...\x.bvrBlendDB` — NOT NUL-term)
 …     u8[]    string table (strsz bytes, offset-addressed)
 …     prelude (prelude_count × 15-byte field records + 8-byte footer)
 …     type blocks, sorted ascending by type_hash → EOF
```

Unlike the master `.gamedb` (same `GADB` v17 magic), a `.bvr` carries
**no decl/record table** — it is a serialized object graph.

## String table

Starts immediately after the `u16`-length source path at
`path_end = 0x2e + pathlen`, spanning `strsz` bytes. All references
(string-offset fields, block headers, instance names) are **u32
offsets into this table**, each landing on a NUL-terminated printable
string. The table embeds binary runs between strings, so extraction
scans the declared span for printable NUL-terminated runs rather than
walking back-to-back cstrs.

Contents: node names (`SD_ScrambleFinder_FAT`, `RideWarg_JumpVaultHigher`),
schema tags (`Animation.SimpleAnimation`, `BlendNamedNode`,
`Predicate.IsPlayingAnimation`), field names (`Value`, `DestX`), asset
refs (`characters\dlc\werewyrm_base.skel`).

## Prelude (`table_end .. first block`)

`prelude_count` (header `+0x1c`) × 15-byte field-schema records,
then an 8-byte footer `{u32 0, u32 type_count}`:

```text
+0x00  u32  name_hash    field name hash (0xe4316e1f = shared `<base>`
                        field, sits at field_index 5 of every run)
+0x04  u32  link_hash    namehash of the next field in the chain
+0x08  u8   field_index  flattened index in the owning schema —
                         contiguous ascending runs delimit schemas
+0x09  u8   f1           chain cursor / scalar class
+0x0a  u8   class        field class (1 scalar, 7 struct, 10 ref, …)
+0x0b  u8   codec        codec / element-type selector
+0x0c  u16  serialized_slot << 8 (low byte always 0)
+0x0e  u8   0
```

Runs split where `field_index` stops increasing — ~1,066 runs on the
player shards. Nearly all runs are copies of a ~50-field **embedded
base-node template** (`Mirrored`, `Weightset`, `StartPCT`, `Rate`,
`YawOffset`, `EnableFootIK`, `EnableGroundIK`, `Gravity`, `Null`,
`Parameters`, `Collision`, `Movement`, `Rotation`, `NativeState`,
`Threshold`, `LockToNavMesh`, `lockMovement`) — one per node-type
context, differing only in `class`/`codec` bytes. `serialized_slot`
indexes the **runtime object**, not the instance record (most fields
carry slot 0 = not serialized); an instance's `aux >> 2` indexes the
flat record table directly and the containing run is that node's
embedded schema (`Bvr::embedded_schema`).

Instance-record slots are the type's *own* serialized fields and their
names are NOT derivable from the file alone — that binding lives in
the executable's reflection data (Nemesis `blend_fields.tsv`).

## Type blocks

Every registered type serializes one block; blocks are stored
**strictly ascending by `type_hash`**, back-to-back at byte-granular
alignment (NOT 4-aligned).

```text
+0x00  u32  type_name table offset
+0x04  u32  same offset (repeated discriminator)
+0x08  u32  field_x — 0x14 (Nemesis census), 0x24 observed on this
            install's shards (version drift); informational
+0x0c  u32  0
+0x10  u32  instance_count
+0x14  u32  type_hash = name_hash(type_name)
+0x18  …    payload: count instance records, then optional
            embedded/extra data — runs to the next block header
            (last block → EOF)
```

Blocks are located by forward-scanning for headers that self-validate
(`off1 == off2`, offset resolves to a printable table string, and
`name_hash(string) == type_hash`). The instance walk alone does NOT
reach the next header — a block's payload may end in a trailing blob
(embedded sub-object data / transition tables, opaque pending RE);
`BvrBlock::extra` carries that range.

## Instance records

```text
+0x00  u32   name — string-table offset
+0x04  u16   slot_count
+0x06  u16   aux — aux >> 2 selects an embedded sub-object schema
+0x08  u32   marker — 0x180000XX family; low byte ≈ serialization class
+0x0c  u32[] slots — string offsets | node refs | scalars | 0xffffffff null
```

Node-reference slots pack `(type_index << 4) | flags` in the high u16
and the target instance index in the low u16 (Nemesis `bvr.py`).
`type_index` indexes `blocks` in file order — verified: type names are
interned in the string table in exactly block-hash order, so registry
index == block index (interned order and hash order coincide on all
29 files). Classification is bound-checked — a word only counts as a
ref when `type_index < blocks.len()` and `instance < block.count`;
f32 literals and hashes alias the shape constantly (`0x3f800000` =
1.0f reads as type_index 1016). 875,055 refs resolve corpus-wide.

## Sizes / verified

5.4 KB (`dlc_werewyrm`, 28 types / 28 instances) – 10.4 MB
(`player_bar`, 284 types / 96,659 instances). All four player files
parse fully: `player_elf`/`_el2`/`_elt` 270/270 types,
92–94k instances each; `player_bar` 284/284.

## API

`anduril_gamedb::Bvr::parse(&data)` → `Bvr { path, strsz, type_count,
strings, strings_map, fields, blocks }`; `BvrBlock { offset, type_off,
name, field_x, type_hash, instances, extra }`. `decode_slot` classifies
slot words (Str / Ref / Null / Raw). CLI: `anduril bvr <path> [nodes]`,
`anduril bvr scan [filter]` — 29/29 corpus files.

## Open questions

- Header `+0x18` (`record_count`) semantics — counts something other
  than instances on player shards.
- Block `extra` payloads (36 B – 720 KB; e.g. `AnchorData` 720 KB,
  `ValueModifier` 306 KB): serialized embedded sub-object data in the
  same tagged-cell family as master-GADB records — observed
  `{u32 tag, u32 0x10000001-class marker, u32, u32}` 16-byte records
  (`tag` varies: `0x4a020002`, `0xbd020002`, `0xe5020002`…) and
  flat f32 runs (`LeanAndTwistRoot` extras are raw floats). Bound by
  the aux-selected embedded schema — the next decode target.
- Field-class (`class`/`codec`) value semantics.
- Instance own-field names (exe reflection; Nemesis `blend_fields.tsv`
  recovered them — regenerate from `ShadowOfWar.exe` if needed).
