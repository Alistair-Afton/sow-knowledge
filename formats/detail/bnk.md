# `.bnk` — Wwise soundbanks

> **Status:** **proven** — Wwise bank structure verified on corpus
> **Provenance:** all 245,134 media entries extracted from real `.bnk` archives.

Standard Audiokinetic Wwise bank format, version **118** across the
entire install (Wwise 2016.x era).

## Layout

```
"BKHD" u32 headerSize   header: u32 version(118), u32 bankId, [variable tail]
N × chunk { char[4] tag, u32 size, u8[size] data }   → EOF (no padding)
```

- `headerSize` counts only the payload after the size field; chunks start
  at `8 + headerSize`.
- `bankId` is the 32-bit **FNV-1 of the lowercase bank name**
  (offset basis `0x811C9DC5`, prime `0x01000193`, multiply-then-xor) —
  CONFIRMED on `amb_rain`, `amb_rain_canvas`, `amb_rain_stone`
  (e.g. `amb_rain` → `0xC4D128AC`). All Wwise object/event IDs use the
  same hash, so a name can be checked against any ID offline.
- Corpus header sizes: **20, 24, 28, 32** bytes — the tail carries
  project/feedback identifiers (semantics undocumented). Parse `version`
  and `bankId` from the first 8 payload bytes; keep the rest opaque.

## Chunks observed (all 5,547 files walk to exact EOF)

| Tag | Count | Contents |
|-----|-------|----------|
| `DIDX` | 5,472 | media index: raw `{u32 id, u32 dataOffset, u32 length}` array — **no count field**, `size % 12 == 0` |
| `DATA` | 5,472 | media payload pool; `DIDX` offsets index into it |
| `HIRC` | 5,497 | object hierarchy: `u32 count` then `{u8 type, u32 size, u32 id, u8[size-4] payload}` — `size` covers id+payload after the size field |
| `STID` | 234 | string-id table: `u32 unk(1)`, `u32 count`, then `{u32 id, u8 len, char[len] name}` |
| `INIT`/`STMG`/`ENVS`/`PLAT` | 1 each | single global init bank |

~50 banks are **32-byte `BKHD`-only stubs** (no chunks) — valid, parse
to an empty chunk list.

## HIRC object types seen

`1 Settings, 2 Sfx, 3 EventAction, 4 Event, 5 RanSeqCntr, 6 SwitchCntr,
7 ActorMixer, 8 ?, 9 LayerCntr, 10 Segment, 11 Track, 12 Lfo, 13 Env,
14 Attenuation, 16 ?, 18 Bus, 19 AuxBus, 20 FxShareSet, 21 FxCustom,
22 AuxBus2` — names are the conventional Wwise vocabulary; payloads are
kept opaque (per-type record formats not field-decoded).

## Media extraction

`bnk media(&entry)` slices `[offset, offset+length)` out of `DATA`.
Embedded streams are `RIFF/WAVE` containers with a Wwise `fmt` chunk —
playable via standard Wwise `.wem` tools/decoders.
`global_ui.bnk` alone carries 717 entries / 25.4 MiB.

## Verified

- **5,547/5,547** files parse; every HIRC object walk terminates exactly
  on its chunk boundary; every DIDX is 12-aligned.
- `cargo run -p anduril-cli -- bnk <path> [dump <dir>] | bnk scan`.

## Unknowns

- Per-type HIRC payload layouts (the interesting part: event → action →
  sound wiring, switch/state values, RTPCs).
- `BKHD` tail fields beyond `version`/`bankId`.
- `STID` leading `u32` (always 1 in corpus).

## PCM media replacement — VERIFIED in-game (Universal Modder)

The runtime plays **PCM** WEMs with no Wwise authoring tool — this is
how the "Falmer tribe" note swapped captain voices:

- Voice banks are v118 banks with **embedded** 48 kHz mono Vorbis WEMs
  (plugin `0x00040001`). A HIRC type-2 (Sfx) object holds
  `{pluginID, streamType, sourceID, inMemoryMediaSize, bits}`.
- Per media: write a RIFF/WAVE with a 24-byte `fmt ` chunk
  (`0xFFFE` extensible, 16-bit, channel config `0x00004101` =
  mono/front-centre) + `data`, re-lay out `DIDX`/`DATA`, set the
  Sound's plugin to `0x00010001` and its in-memory size. Update the
  size in the `.bndl` listing the bank.
- PCM banks run ~10–25× larger — mind bundle-type budgets (`Sound_5`
  = 5 MB, `Presentation` = 10 MB).
- Per-language presentation archives hold one bank per `.embb`
  (type `Presentation`, Oodle-compressed — extract via QuickBMS).
