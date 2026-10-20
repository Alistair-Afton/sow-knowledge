# `.anix` — ANIX v44 animation libraries

> **Status:** **verified** — full container + all five codecs decoded;
> parse + sampling verified on all 1,666 extracted files; quat decode
> is engine-exact (unclamped, NaN-producing like the retail assert).
> **Provenance:** community spec (LordMonstey/sow-anix-format, MIT —
> vendored `Helper/vendor/sow-anix-format/`) cross-checked against our
> corpus and merged with our independently derived constants; our
> earlier clip-block/sentinel model is superseded.

The animation set for a skeleton/character: a channel table naming
bone channels, a payload of per-segment codec blocks, and metadata
(directory + descriptors + footer) naming and indexing the clips
(`bind`, `Blrg_Cmbt_Attack_Front_Leap`, …).

## Conventions

- Y-up, centimetres; quaternions `x, y, z, w`.
- Times are **ticks = milliseconds**; the 30 fps grid is
  `floor(k * 100 / 3)` (0, 33, 66, 100, …).
- Names are ASCII, NUL-terminated; clip names ≤ 47 chars.
- Root motion rides the `Null` bone (`0x88BAA7E7`) `Pos`/`Rotation`.
- All names/types/tags use the engine `h = h*0x397 + FOLD[c]` hash
  (`anduril-strdb::name_hash`):

| name | hash |
|---|---|
| `Rotation` channel type | `0x27436396` |
| `Pos` channel type | `0x00CE66FC` |
| `LITH` segment time table | `0x2B99DBC1` |
| `KeyString` descriptor extra | `0xB9010BE0` |
| `bind` clip | `0x5CFA71ED` |
| `Null` bone | `0x88BAA7E7` |
| event channel type (STRI) | `0x210C8F5A` |
| float channel type (CAF4) | `0x5F1D8EED` |

## File layout

```
header (48 B)
channel table        @ header.channelTableOff (= 48)
payload: segments    @ header.payloadOff (16-aligned), payloadSize B
metadata             @ header.metadataOff (= payloadOff + payloadSize)
  head (16 B)        {dirOff=16, clipCount, 0, footerOff}
  clip directory     clipCount × (u32 nameHash, u32 descriptorOff)
  clip descriptors   16-aligned, variable size
  footer             runs to EOF — the clip table the loader builds
```

All metadata offsets are relative to the metadata start.

### Header (48 B)

`"ANIX"`, u32 ver=44, u32 headerSize=48, u32 fileSize, u32
channelCount, u32 channelTableOff(48), u32 payloadOff, u32
payloadSize, u32 metadataOff, u32 metadataSize, u32 1, u32 1.
`metadataOff + metadataSize = fileSize`.

### Channel table

`{u32 typeHash, u32 n, u32 boneHash[n]}` groups repeated until
`channelCount` channels are listed. Channel index = position across
all groups; clip tracks reference channels by index. Shared library-
wide by every clip.

## Retail read path (Ghidra decompile, verified)

Loader `FUN_140480aec` (found via the `0x58494E41` immediate) matches
the spec exactly: one 0x30-byte header read, `ver == 0x2c` checked by
equality, then **absolute-offset seeks** — `channelTableOff`,
`payloadOff/payloadSize`, `metadataOff/metadataSize` are file
offsets, not relative. The channel table is consumed as
`{u32 typeHash, u32 n, n×u32 trackId}` groups whose ids sum to
`channelCount` (each `{trackId, typeHash}` pair lands in one 8-byte
runtime record). Loader errors `"Failed to consume the full file"`
unless the stream ends exactly at EOF — there is no unmodeled tail.

### Clip descriptor

| off | type | field |
|---|---|---|
| 0 | char[48] | clip name |
| 48 | u16 | header size = `124 + 16*segments + 8*extras` — does **not** bound the codec/track tables below |
| 50 | u16 | flags; bit 0 picks the bind-fallback buffer |
| 52 | u16 | segment count |
| 54 | u16 | extra count (KeyString etc.) |
| 56 | u16 | codec count |
| 58 | u16 | track count |
| 60 | f32[7] | `Null` TRS at clip start (pos xyz, quat xyzw) |
| 88 | f32[7] | `Null` TRS at clip end (total root motion) |
| 116 | u32 | unknown (0 on ~97% of clips; else a 30 fps tick) |
| 120 | u32 | duration, ticks |
| 124 | char[4]×ncodec | codec FourCCs |
| … | u8×ntrack | codec index per track |
| pad4 | u16×ntrack | channel index per track (unsorted ok) |
| pad4 | 16B×nseg | segment table `{payloadOff, len, firstTick, lastTick}` |
| … | 8B×nextra | extras table `{hash, offset from descriptor start}` |
| … | | extra data (KeyString strings) |
| pad16 | | |

Segments are tick-contiguous (`first[i+1] == last[i]`).

### Segment (payload, 16-aligned)

```
u32  extraBlockCount E        (natives: 1 = LITH, or 0)
u16  codecBlockOff[ncodec]    (segment-relative)
u16  extraBlockOff[E]
pad  to 4 bytes               (required even with even codec count)
u32  extraHash[E]             (0x2B99DBC1 = LITH)
codec blocks in descriptor order
  AQ03/SQ03/SV01: 16-aligned;  UATR/STRI: 4-aligned
LITH block (4-aligned): u32 S, u32 aux1, u32 aux2, u32 tick[S]
pad  to 16
```

A codec block spans to the next offset; the last ends at the LITH
offset. Segments stay under 64 KiB (u16 offsets) — long clips split
into contiguous segments.






### Clip descriptor

| off | type | field |
|---|---|---|
| 0 | char[48] | clip name |
| 48 | u16 | header size = `124 + 16*segments + 8*extras` — does **not** bound the codec/track tables below |
| 50 | u16 | flags; bit 0 picks the bind-fallback buffer |
| 52 | u16 | segment count |
| 54 | u16 | extra count (KeyString etc.) |
| 56 | u16 | codec count |
| 58 | u16 | track count |
| 60 | f32[7] | `Null` TRS at clip start (pos xyz, quat xyzw) |
| 88 | f32[7] | `Null` TRS at clip end (total root motion) |
| 116 | u32 | unknown (0 on ~97% of clips; else a 30 fps tick) |
| 120 | u32 | duration, ticks |
| 124 | char[4]×ncodec | codec FourCCs |
| … | u8×ntrack | codec index per track |
| pad4 | u16×ntrack | channel index per track (unsorted ok) |
| pad4 | 16B×nseg | segment table `{payloadOff, len, firstTick, lastTick}` |
| … | 8B×nextra | extras table `{hash, offset from descriptor start}` |
| … | | extra data (KeyString strings) |
| pad16 | | |

Segments are tick-contiguous (`first[i+1] == last[i]`).

### Segment (payload, 16-aligned)

```
u32  extraBlockCount E        (natives: 1 = LITH, or 0)
u16  codecBlockOff[ncodec]    (segment-relative)
u16  extraBlockOff[E]
pad  to 4 bytes               (required even with even codec count)
u32  extraHash[E]             (0x2B99DBC1 = LITH)
codec blocks in descriptor order
  AQ03/SQ03/SV01: 16-aligned;  UATR/STRI: 4-aligned
LITH block (4-aligned): u32 S, u32 aux1, u32 aux2, u32 tick[S]
pad  to 16
```

A codec block spans to the next offset; the last ends at the LITH
offset. Segments stay under 64 KiB (u16 offsets) — long clips split
into contiguous segments.

## Codecs

| FourCC | kind | notes |
|---|---|---|
| `AQ03` | animated quats | own key times; nlerp eval |
| `SQ03` | static quats | one packed row, constant |
| `UATR` | animated vec3 | keys timed by the segment LITH table |
| `SV01` | static vec3 | one packed u32/track |
| `STRI` | timeline events | strings via clip's KeyString extra |
| `AV01` | animated vec10 | AQ02-style keyed u32 rows; 32 sections carry an extra u32 after key count (mask? spans then start at 16) |
| `UAFT`,`UAVT`,`UAQT`,`UABT`,`UACO` | uncompressed animated f32/vec3/quat/bool/u8×4 | per-track `u32 key_count, u32 time_ms[k], value[k]` |
| `USFT`,`USBT`,`USVO`,`USCO` | uncompressed static f32/bool/vec3/u8×4 | same offset table, one value/track |
| `CAF4` | keyframed f32 (param/facial) | see below |

### `U*` / `S*` codecs (sow-rs verified, not yet in `anduril-anix`)

`u16 blob_off[n]` (padded to 4; bool codecs unpadded), then one blob
per track at `section + blob_off[i]`:

- Animated (`UA**`): `u32 key_count, u32 time_ms[k], value[k]` —
  interpolate linearly between own key times (quats nlerp, bools/bytes
  step).
- Static (`US**`): one value per track, no times.

Value size by tag letter: `F` f32 (4), `V`/`VO` vec3 (12), `Q` quat
xyzw (16), `B` bool u8, `CO` u8×4 (likely RGBA). Census (sow-rs, all
24,728 `.anix`): USFT 36,947 sections (the dominant codec — param and
static values), UAVT 16,620, UAQT 4,086, UAFT 660, UABT/USBT/USVO/
USCO rare. Every section decodes exactly.

### `CAF4` — param/facial float tracks (sow-rs verified)

`u32 1, u32 start_ms, u32 end_ms, u32 key_count, u32 track_count`,
then `u16 span_ms[key_count]` (span[0]=0, key k at start+Σspan),
align 16 from section start, then `key_count` rows × `track_count`
f32 (unpadded). Linear sampling. 19,176 sections; every CAF4 track is
a `param` channel — orc facial pose libraries (smile/jaw/visemes) and
camera-rig parameters. The param → facial-control mapping is OPEN.

### Packed quaternion (AQ03/SQ03)

One 42-bit slot; three slots per LE 16-byte word (bits 0, 42, 84):

```
bits 0-12   component a (13b)   13-25 b   26-38 c
bits 39-40  m = 3 - omitted index (0 omits w … 3 omits x)
bit  41     sign of omitted component

value  = u / 8190 * sqrt(2) - sqrt(0.5)         (f32 constants)
omitted = ±sqrt(1 - a² - b² - c²)
```

Engine decodes **all** slots incl. padding, **no clamping** — a
negative radicand yields NaN and the game asserts. Native unused-slot
filler: `a=b=c=4095, meta=0` (decodes to identity).

- `SQ03`: `ceil(T/3)*16` bytes — one packed row, constant.
- `AQ03`: `{u32 start, u32 end, u32 K, u16 delta[K]}` (delta[0]=0,
  cumsum → end) then K packed rows at `align16(12+2K)`.
  Evaluation: clamp t, bracketing keys, flip upper if dot<0,
  **nlerp** (lerp + normalize).

### SV01 — static vectors

u32/track: `x=bits2-11, y=12-21, z=22-31`;
`v = u/1022*512 - 256` cm. Then pad to 16.

### UATR — animated vectors

No own times — uses the segment's LITH ticks (S samples):

```
u16 trackStart[T]   (4-aligned offsets into the block)
per track:  S × (u8 ctrlIdx, u8 frac)   pad4
            N × f32[3] control points   (N = rest/12)
```

At time t: bracket LITH samples i,i+1 → fraction f;
`lo = frac[i]/255`, `hi = idx[i+1]==idx[i] ? frac[i+1]/255 : 1`;
`w = lo + (hi-lo)·f`; result `lerp(ctrl[idx[i]], ctrl[idx[i]+1], w)`.
Missing/misaligned LITH → UATR silently writes nothing.

### STRI — timeline events

`u16 2, u16 totalBytes, u32 count, u16 0`, then count ×
`{u32 tick, u16 off, u16 len}` into the clip's `KeyString` extra.
Bound to one event-type channel on `Null`. Examples from natives:
`BCOL`/`APPLYDAMAGE`/`ECOL` (hit windows), `APPLYBUFF`, `vx_*`,
`fts_*`, `HeelStrike`, `audio_hook_key …`, `MeleeFX …`.

## Footer — the load-time clip table

Starts at metadata `head.footerOff`, runs to EOF:

```
u32 h[6]:  h0 streamOff, h1 totalFrames, h2 totalEventMarkers,
           h3 totalMarkers, h4 eventStringTableOff, h5 streamSize
body:  [0,h4)     NUL names: library path, then clip names
       [h4,h0)    deduplicated event strings
       [h0,h0+h5) stream: 41B header (first u32 = h0-5) then one
                  record per DIRECTORY clip, in order:
                  f32[3] extents, u32 nameOff, u32 unk, u32 unk,
                  u32 unk, u32 flag, u32 frames, u32 N,
                  u32 mark[N], u32 marker[N]   (tag<<24 | tick;
                  tag = event-string index, 0xFF = none)
       12B zeros
```

The loader builds its clip table from the footer — records must be
1:1 with directory entries, `h[]` totals exact, or the game crashes
at load. Time grid: 30 fps ticks merged with marker ticks; first
marker at 0, last ≈ duration. Tag limit 255 event strings/library.

## Runtime semantics

- A character's `.skel` lists ANIX paths; clips play **by name** and
  must be unique across all its libraries.
- Every library has a `bind` clip evaluated at t=0 into the fallback
  pose; unanimated channels keep bind values.
- Evaluation: per codec, gather that codec's tracks, map channel →
  bone, assert all outputs finite.

## Loader/evaluator reference (retail exe, Oct-2026 image base)

| function | address |
|---|---|
| AQ03 evaluator | `0x1402D26E0` |
| SQ03 evaluator | `0x1402D21B0` |
| SV01 evaluator | `0x1405D4114` → `0x140266FC0` |
| UATR evaluator | `0x140183810` |
| clip evaluation | `0x140261360` |
| LITH lookup | `0x14030269C` |
| library loader (footer) | `0x1404B9E58` |
| footer arena-fit assert | `0x14024A4FC` |
| NaN assert | `0x140819104` |

## Encoder notes (for later write support)

The spec's encoder rules are proven by in-game write-back: keep
payload/segments 16-aligned, pad the segment header to 4 before the
extra-hash table, always include LITH with UATR, fill spare quat
slots with the identity filler, descriptor header size =
`124 + 16*segs + 8*extras`, contiguous segment ticks, footer 1:1
with exact `h[]` totals. `anduril-anix` is decode-only today.
