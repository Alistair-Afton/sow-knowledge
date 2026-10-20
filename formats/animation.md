# Animation: `.skel` + `.anix`

> **Status:** **verified**
> **Provenance:** ANIX/SKEL decode on real corpus — Andúril + sow-rs.

Skeletons, bind poses, and every animation codec. Source: sow-rs
docs (all-codecs decode verified on the full install) cross-checked
against Andúril's own working anix/skel pipeline, which skins
animated captains correctly in the live viewer.

## SKEL — `.skel` — CONFIRMED

- Bone records in **depth-first order with child counts**;
  translations are **model-space bind positions in centimetres**
  (checked against head-mesh bounds).
- Carries: node hierarchy, bind transforms, LOD groups (name +
  per-level distance threshold + material index), sockets
  (attachment points for weapons/props), controller records.
- Bone order used for **skin palettes** is the **Havok sibling-block
  order**, not file order — convert to source bone indices before
  indexing mesh weight palettes (VERIFIED via Monstey's notes).
- Full scan (sow-rs): 4,117 files, 4,109 clean, **8 fail**: seven
  share `string offset 0x40f8d776 out of range` (camera rigs,
  param_head, orc bone-armor, a Baranor crossbow — the value is a
  hash/sentinel the community format doesn't cover; OPEN), and
  `torvin_desertt1shield.skel` has a rig record shape no pattern
  describes. Both failures are input-true, not reader bugs.
- Andúril: `crates/anduril-skel`.

## ANIX — `.anix` — VERIFIED (every codec decoded)

24,728 files (loose + bundled) parse end-to-end. Structure:

- Header 0x30: `"ANIX"`, constants, file size, total node-hash count,
  offsets. Block A at 0x30: track-name groups
  `{u32 type_hash, u32 count, u32 node_hash[count]}` — types `pos`,
  `rotation`, `param` (facial/misc scalar tracks), `0x210C8F5A`
  (1-track, unknown); node hashes = bone names.
- Clip data: per clip `u32 named_count`, `u16 offset[codec_count +
  named_count]` (codec sections in tag order, then named blocks),
  align 4, `u32 name_hash[named_count]` — `LITH` (0x2B99DBC1) is the
  only named block seen.
- Clip directory at file end: `u32 0x10, u32 clip_count, u32 0, u32 ?`
  + `{name_hash, offset}` table + records carrying the ASCII clip
  name, duration ms, root start/end transforms, and codec tag bytes.
- Clip names are real (`PL_Walk`, `bind`, `smile_l`, visemes);
  orc head anix = facial pose libraries (~230 clips); camera rigs use
  `param` tracks too.

### Codecs (all decoded — VERIFIED)

Registered in the exe by FourCC with SSE/scalar variants:

| Codec | What | Layout |
|---|---|---|
| `SV01` | static vector | 4 B/track: three 10-bit fields → `c/1022*512−256` cm |
| `SQ03` | static quaternion | 42 bits/track, 3 per 16 B: 13-bit components + 3-bit index; `a = c/8190·√2 − 1/√2`, missing component reconstructed, sign by `idx&4` |
| `SQ01/SQ02`, `PQ03`, `Q101` | quaternion variants | registered; layouts like SQ03 family |
| `AQ03` | keyed quaternion | `start_ms, end_ms, key_count, u16 span[key]` (key k at start+Σspan), pad 16, rows of 42-bit quats (`ceil(n/3)` blocks) |
| `AQ02`, `AV01` | keyed quat32/vec10 | same header; `(n+3)&!3` u32 rows |
| `UATR` | uncompressed translation | `u16 blob_off[n]`; per track `frames×{u8 key,u8 frac}` then vec3 keys; sample by LITH frames + `frac/255` interpolation (exact algorithm in sow-rs anix.md) |
| `UAFT/UAVT/UAQT/UABT/UACO` | uncompressed animated f32/vec3/quat/bool/u8x4 | `u16 blob_off[n]`; per blob `u32 key_count, u32 time_ms[k], value[k]` |
| `USFT/USBT/USVO/USCO` | uncompressed static f32/bool/vec3/u8x4 | same offset table, one value per track |
| `CAF4` | keyframed float (param/facial) | `u32 1, start, end, key_count, track_count`, u16 spans, align 16, `key_count` rows × `track_count` f32 — linear sampling |
| `STRI` | timed event track | `u16 blob_off[n]`; per track `u16 string_bytes, u16 count, u32 string_base` then `{u32 time_ms, u16 off, u16 len}` into clip event strings at `string_base` |

Census totals (VERIFIED): CAF4 19,176 sections; UATR 55,186; USFT
36,947; UAVT 16,620; UAQT 4,086; STRI 25,599; AQ03/AQ02/AV01 keyed;
UABT/USBT/USVO/USCO/UACO rare. Every section of every codec decodes
exactly on the install.

### LITH — frame time table — VERIFIED

`u32 frame_count, u32 b, u32 c, u32 time_ms[frame_count]`, align 4,
then `b` 6-byte records (unknown, usually 0). Frame times are **not**
fixed 30 fps and **not** the AQ03 key times — always use the LITH
table for UATR sampling.

### Timed events — VERIFIED

STRI + the clip's event strings give `timed_events: [(ms, name)]`:
`HeelStrike R`, `APPLYDAMAGE`, `EHITWIN`, `AEE`, `EPW`, `STOP_ALIGN`,
`RAGDOLL`, `CMSADD/CMSREM QuickStun`, ... These drive hit timing,
counter windows, and FX sync — the game reads them, not fixed clip
fractions.

### Root motion — CONFIRMED

Clips carry root start/end transforms; the root bone track (node
hash `0x88BAA7E7` in these clips) is removed from the pose and applied
to the entity as a per-frame delta (sow-rs: `Spr_Combat_Run` travels
318.7 cm in 733 ms matching the header root end). Blending: linear
for vectors, shorter-arc nlerp for quats.

## What animates what — CONFIRMED

- Track node hashes = bone name hashes in the paired skel; per-vertex
  skin weights (MMSH u8 weights + palette) pose via each bone's delta
  from bind. Andúril's `skin()` + `skin_normals()` implement it —
  captains animate correctly in `mesh view`/`sim view`.
- Param/CAF4 tracks drive facial pose library curves and camera
  properties (PARTIAL: which param maps to which control is OPEN).
