# `.skel` — SKEL v17 skeletons

> **Status:** **verified**** — bone hierarchies decoded on real corpus
> **Provenance:** bone hierarchies + rigs decoded on the real corpus.

Named bone hierarchies, animation-controller rigs, LOD groups, and optional
cloth simulation data for every skeletal asset in the game. The layout is the
Monolith `SKEL` container also documented by community hexdump specs; this file
records the parts verified against real game files.

All integers little-endian; sizes **include** any leading `u32` count field.

## Header — 0x40 bytes

| off | field | verified meaning |
|---|---|---|
| 0x00 | char[4] | `"SKEL"` |
| 0x04 | u32 | version (17 on all shipped files) |
| 0x08 | u32 | header size (0x40) |
| 0x0C | u32 | string-table size (bytes, after the BONE section id) |
| 0x10 | u32 | model-strings count (anim paths + lod-group names) |
| 0x14 | u32 | bone-name count |
| 0x18 | u32 | bone-section size (37 × bone count) |
| 0x1C | u32 | animation count (u32 entries in ANIM section) |
| 0x20 | u32 | LOD-group count |
| 0x24 | u32 | socket count |
| 0x28 | u32 | bone count |
| 0x2C–0x40 | i32[6] | unknown (mostly 0 / −1) |
| 0x44 | f32 | unknown scalar (10 / 400 / 500 / 1200 seen) |
| 0x48 | float[6] | AABB min xyz, max xyz |

## LOD groups — at `headerSize` + 0x20

`u32 lodCount`, then per LOD `{u8, u8, f32 threshold, u32 materialIndex,
i32 unknown}` — **14 bytes** each, `lodCount` entries.
Typical thresholds: 0, 300, 600 → `[0, 1]` quality tiers.

## BONE section — id `0xDCDCFF00`

```
u32 id
byte[stringTableSize] stringTable   // \0-separated names
bone bone[boneCount]
```

**`bone`** (37 bytes): `u32 namePtr` (byte offset into the string table —
strtab[0] is usually `""`, so `"Null"` starts at 1) `| u8 active | float[3]
translation | float[4] rotation quat | u32 childCount`.

Bone order is **depth-first** with parents before children (sow-rs:
interpreted parent 0xDEADDEAD is a forward-reference false positive;
accept inert records only when the tail is all `0xFADFADF…`). Bones
live at `headerSize + 0x20 + 8 + 8×lodGroups`. The palette table
after the bindings lists bone indices in **file order** — palette[i]
is the animation-skeleton order a skin matrix uses.

## Rig — immediately after bones

```
u32 controllerCount
controller controller[controllerCount]
u32 socketCount
socket socket[socketCount]
```

**`controller`**: `u32 namePtr | u32 boneCount | u32 controlCount |
u8[boneCount] controls`.

**`socket`** (40 bytes): `u32 boneIndex | u32 namePtr | byte[32]` (padding +
unidentified payload).

**Variant:** some rigs (`camrig`, `param_head`, some shields) carry a
differently shaped table in the controller slot — `{u32 tag, u32
stringTableOffset}` pairs (tags are name hashes such as `0x40F8D776`). The
parser tries the named-controller walk first and, on shape mismatch, keeps the
whole region opaque in `rig_extra` rather than failing the file.

## MODEL section — id `0xDCDCFF01`

`u32[animCount]` path pointers (each an offset into the string table; e.g.
`characters/animations/...`) followed by `u32[lodGroupCount]` lod-group name
pointers.

## ANIM section — id `0xDCDCFF02`

`u32[animCount]` animation-type/index values. Zero-anim files have just the id.

## INDICES section — id `0xDCDCFF03`

`u8[boneCount]` — per-bone index table (usually `0..count` or permutation).

## CLOTH section — id `0xDCDCFF04`

`u32 version`, then payload. `version = 0` → empty, followed by the
`0xDEADDEAD` terminator. Populated → `0xBEEFFEED`-tagged Havok `TAG0` /
`SDKV2016` blob(s), then variable simulation data, terminated by
`0xDEADDEAD`.

## Unknown section — id `0xDCDCFF05`

Same `version` + payload + `0xDEADDEAD` shape. Some files carry a
`0xBEEFFEED`-tagged resource reference — `{u32 ver, 0xBEEFFEED, u16 len, path,
params, 0xDEADDEAD}` where `path` is e.g. a `.GPUCloth.cres` file —
`unk5.tagged_path()` extracts it.

## Footer — id `0xDCDCFF99`

`u32 id | u32 version | "SKPY" magic | u32 field | byte[8] tail`.

## Coordinate frame — sow-rs verified

Game data is **left-handed**: +X right, +Y up, +Z forward, cm.
Bind-pose evidence: Talion `R_Hand` x = +64 cm vs `L_Hand` −64;
`R_Toe` z ahead of `R_Foot`; spine +Y. Quaternions are xyzw,
Hamilton, active — shared by skel/MMSH/anix/PRFB/WDSC. Front faces
wind clockwise (D3D). Right-handed consumers mirror X once at import
and take bitangent handedness per-vertex.

## Coverage

- **All 381 `.skel` files** in the install parse cleanly — bone names,
  transforms, quaternions, rigs, sockets, LOD thresholds, animation paths all
  decoded.
- 3 files (`camrig`, `param_head`, `torvin_desertt1shield`) use the alternate
  controller-table shape → stored in `rig_extra`.
- sow-rs's larger corpus scan (4,117 files incl. bundles) reports
  **8 failures we should expect on a wider scan**: 7 files whose first
  post-string-table bone-name pointer is `0x40f8d776` — outside the
  table (camera rigs, `param_head`, orc bone-armor skels, one Baranor
  crossbow; likely a hash/sentinel the community schema doesn't
  cover — OPEN), plus `torvin_desertt1shield.skel` whose rig records
  don't fit the standard shape. Both are input-true anomalies, not
  parser bugs — a robust reader should surface them without failing
  the file.

## Unknowns

- Rig-tail hash/offset pair semantics (the alt controller table).
- CLOTH payload internals beyond the `TAG0` blob start.
- socket[8..40] payload fields.
- unk1..unk12 / footerField semantics.
