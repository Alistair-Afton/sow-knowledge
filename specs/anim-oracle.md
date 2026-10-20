# Animation oracle — posed-bone ground truth from the live game

> **Status:** working — procedure for capturing posed-bone ground truth
> **Provenance:** live-game posed-bone captures via Project Nemesis `nemesis.dll` memory reads,
> correlated with `.anix` files from real `.arch06` archives. Method doc; rounds cited inline (r52-r104).

> **Mostly superseded (commit 13b1779):** the rewritten `anduril-anix`
> decoder evaluates clips directly — `Anix::clip_rotations` /
> `clip_translations` give engine-exact posed-bone TRS at any tick for
> every bone with an AQ03/SQ03/UATR/SV01 track. Live capture is now
> needed only for blended/IK'd poses, crowd-mob instances, and runtime
> validation of our evaluator.

How to obtain per-frame posed bone TRS (the ground truth needed to decode
the variant-B `.anix` record grid) from the running game via Project
Nemesis. Evidence tags follow the Nemesis convention
(`VERIFIED-INGAME` / `VERIFIED-STATIC` / `INFERRED` / `UNVERIFIED`).
Nemesis docs/extracted are evidence, not authority — items below were
re-checked this session.

## Status

- **Pose-buffer layout DECODED (2026-10-12, VERIFIED-STATIC).** The
  NonInteractiveMob CPU interpolate fn `0x14180deb8` reads/writes a
  multipose array of 32-B/bone `{f32 pos4, f32 quat4}` records —
  see the dated section below. Remaining RE step: locate a live
  mob-subsystem object (vtable `0x141dcdcc8`; registry globals
  `0x142a25ca8/0x142a25cc0` were empty at world idle).
- Live session 2026-… (this session): game booted via `nem.py world`,
  pipe alive, minimal read-only probes run, process killed afterwards.

## Verified this session (VERIFIED-INGAME)

- `python tools/nem.py world` booted cold → pipe alive, `=1+1` → `2`.
- Entity anchor `[[0x1426ffa98]+0x888]` read via
  `sdk.debug.read_u64` → entity `0x09b731b0`; `+0xac` = pos
  `(-10329.27, 7845.90, 9705.00)`, `+0xb8` = unit quat
  `(0, 0.7071, 0, 0.7071)` — the documented root-TRS layout holds.
- `sdk.console.exec('AnimLink.Dump.History')` fired (log shows
  `callargv: 0x141bdedc8(argc=0, argv=0x8)`) but `sdk.console.tap` +
  `tap_drain()` captured **0 bytes** — consistent with the documented
  compiled-out channel-print (`0x1419c83ac` = 3-insn no-op). Console
  dumps will NOT yield pose data through the tap: dead end for output.
- `sdk.debug` API surface (checked live): `read_u32`, `read_u64`,
  `read_str`, `write_u32/64`, `write_str`, `alloc`, `callva`, `callargv`.
  No f32 reader — decode via `string.unpack('f', string.pack('I4', u32))`.
- `sdk.exec.call('System:GetPlayer')` returned `true` (boolean), not the
  documented `.val` handle — the recon-doc call shape may have changed;
  prefer the `[[0x1426ffa98]+0x888]` anchor.

## The ground-truth targets (ranked)

1. **NonInteractiveMob multipose buffer** — VERIFIED-STATIC.
   Cvar strings in exe: `Mob.AnimSampleRateMS` (`0x141f71cf8`),
   `Mob.GPUMemoryOnly` (`0x141f71df8`) whose desc
   (`0x141f71d90`, referenced from cvar node `0x142670780` at `+0x20`)
   reads "…once the mob animation compute buffer is created we ditch the
   non-compute multipose buffer" — i.e. crowd mobs keep a CPU-side array
   of per-bone posed transforms, and `Mob.EnableCPUInterpolate`
   (`0x142670740`) makes the engine interpolate poses on CPU (readable,
   no GPU readback). One buffer = all bones × all mob members playing
   ambient anims. Best target.
   *Next RE step*: derive the cvar node layout (value-field offset) from
   the cvar registration code, then xref the value address to find the
   update fn that owns the multipose buffer; also `nem.py disasm` around
   `NonInteractiveMob.cpp`-attributed code. The two xrefs found so far
   (`0x1410711d2/0x141071272`) are only the cvar linked-list splice at
   `0x142670778`.
2. **Tracker/bone-name evaluators** — VERIFIED-STATIC (schema).
   `Character.Models.Skeletons` (`0x14264b720`) attrs `Nodes`,
   `LookAtTrackerNodes`, `AimAtTrackerNodes`, … with
   `TrackNodeName`/`TrackNodeWeight` — the engine resolves named bones →
   live transforms every frame (aim/eye tracking). Disasm the tracker
   evaluator to find (a) the per-character bone-table address and (b)
   the name→index map — exactly the interface needed. `Modifier.SyncBone`
   and `Modifier.StoreTransformInfo` (WorldPosition/WorldRotation
   ValueNode getters) are smaller walk-in targets.
3. **AnimNodeLink / ThreadAnimNodeLink** — INFERRED. Per-character anim
   instance objects (`AnimLink.PlayAnim`, `AnimThreadLink.Break.AnimComplete`
   commands; handlers `0x141bdedc8`/`0x141bdee0c`; sources
   `AnimNodeLink.cpp`, `ThreadAnimNodeLink.cpp`, `CharacterBehaviorInterface.cpp`).
   Disasm `0x141bdedc8` (PlayAnim handler) upward to find the link
   object and its pose output.
4. **`ClientFX.AnimatedSkeleton`** — VERIFIED-STATIC schema. Spawnable
   FX that plays a named anim on a skeleton — a *controlled* oracle if
   its instance is findable (spawn `BipInf_SentryLowLoop` on demand).
5. **`Debug.AnimFrameError`** — VERIFIED-STATIC schema. Engine-side
   per-frame error node; exists precisely to compare generated poses.

## What the script VM cannot do

`=<expr>` eval reaches *handles* only (`System:GetPlayer`,
`System:DEBUG_GetNPC(i)`, `NPC:GetAIWithinDistance`) — the NPC class has
~180 methods, none exposing bone transforms. All bone reads go through
`sdk.debug` memory reads or a `detour`-crate watchpoint
(dbwatch/fearedwatch pattern). All game-VM calls via `gamethread`;
per-frame capture should drain to `sdk.state`/file, not pipe replies.

## Correlation plan (once the pose buffer is found)

- Target clip: `characters/animationcharacters/global/bip_ambient/
  bip_amb_sentry.anix` — `BipInf_SentryLowLoop`, a near-static 77-bone
  biped idle: decoded slot values should correlate ~1:1 with an almost
  constant pose. Corpus copy: `target/tmp/anix_all/...` (gitignored).
- Capture per-frame local (or world) TRS for named bones; correlate the
  (A,B) sign-coherent pair against quat components (which components,
  sign convention, 2^31 scale), and C/D/E/F against remaining channels.
- Verify quantization: A/B saturate at ±1.000 at s32 scale 2^31.

## Session 2026-10-12 — multipose buffer DECODED (VERIFIED-STATIC)

The cvar node layout is resolved and the `Mob.GPUMemoryOnly` consumer —
the NonInteractiveMob per-frame CPU pose interpolator — is fully decoded.

### Cvar node layout (VERIFIED-STATIC, image read)

Records are 0x40-byte strided, base == the TSV `node` column:

| off | field |
|---|---|
| +0x00 | back-link (points at previous record's +0x38 cell) |
| +0x08 | **live value / default** — the field consumers actually read |
| +0x10 | ctx program record |
| +0x18 | name cstr · +0x20 help cstr · +0x28 file cstr · +0x30 line |
| +0x38 | forward link cell (intrusive list; later repurposed as resolved-R ptr by the console program — but see below) |

Per-node registration thunk (`[console_mgr 0x142a58bc0]->vt+0x68(&node->ctx)`
then link splice): `Mob.AnimSampleRateMS` thunk @ `0x141071040`,
`Mob.GPUMemoryOnly` thunk @ `0x141071190`. Adapter `0x14117dd00` →
`tail(argc,argv, varid=rec+0x08, storage=&rec+0x38)` → bool tail
`0x14117d4cc`.

**VERIFIED-INGAME (2026-10-12, in-world):** all three Mob cvar nodes
still carry intact list links at +0x38 and untouched defaults at +0x08 —
they are NEVER resolved through the console program. The engine reads
`node+0x08` directly:

- `Mob.AnimSampleRateMS` value @ `0x142670708` — `movss xmm1,[rip]` @
  `0x141809c7f` (f32 5000.0).
- `Mob.EnableCPUInterpolate` value @ `0x142670748` — `mov eax,[rip]` @
  `0x140d13087` and `0x140d142f7` (dword; default 0x3c — read as int
  gate/counter, inside the mob tick fn spanning ~`0x140d12b20..0x140d14400`).
- `Mob.GPUMemoryOnly` value @ `0x142670788` — `cmp byte,[rip]` @
  `0x14180cab6` (creation path) and `0x14180df99` (per-frame update).

### The multipose update fn — `0x14180deb8` (VERIFIED-STATIC)

Signature `fn(this)` where `this` = a mob-subsystem object:

```
member array:  base = this+0x138, count = this+0x140, stride 0x88
per member rdi:
  +0x50 render/gpu ctx obj          +0x60 dst pose-buffer wrapper
  +0x68 anim ctx: numBones = [ctx+0x158];
        src multipose = *((*([member+0x68]+0x140))->vt[0]())
  +0x78 anim source {u32 @4 = frame divisor, @8, @0xc}
  +0x80/+0x84 sample cursor / sub-frame
GPUMemoryOnly set → if (ctx+0x140 != 0) lazily creates the +0x60 compute
buffer via engine alloc [0x1426fefd8]->vt+0x58 (member+0x50 vt+0x1b8/0x1d0).
Interp loop (unrolled ×8, 0x14180e0d5..0x14180e2c8):
  elem stride 0x20 (SHL 5); srcA = base+(frameA*nBones+bone)*32,
  srcB likewise for frameA+1, dst = *vt0(*[member+0x60]) same indexing.
  Per record:  quat = slerp(A.quat,B.quat,t) via 0x140210078 (xmm0=A,
  xmm1=B, xmm2=t broadcast) → dst+0x10 ; pos = lerp → dst+0x00.
```

**Multipose record = 32 B/bone: `{f32 pos[4] @+0x00, f32 quat[4] @+0x10}`**
— plain uncompressed TRS. This is the pose ground-truth format to
correlate the ANIX B-grid against.

- Vtable containing `0x14180deb8` (slot 42): base `0x141dcdcc8`,
  installed by ctor ~`0x1407cd2a0` (object ≥0x68 B, inits +0x30..+0x60).
- Registration globals `{begin,end}` @ `0x142a25ca8/0x142a25cb0` (8-stride
  obj list) and `0x142a25cc0/0x142a25cc8` (0x10-stride {obj,refcount}) —
  **VERIFIED-INGAME both empty** at world idle (mob subsystem object not
  live in that area/state; likely populated where mobs actually run —
  crowd-heavy areas — or on-demand).
- Associated: element vector grow fn `0x14180dc04` (0x88-stride),
  slerp helper `0x140210078`, non-interp path at `0x14180e3dc`,
  second GPUMemoryOnly reader fn near `0x14180ca60` (+0xd0 flag word,
  +0x150 flag byte, embedded list node at +0x60).

### Remaining step for capture

Locate a live mob-subsystem object (vtable `0x141dcdcc8`) — either wait
for a world area where mobs tick (registry globals populate) or find the
owner that holds `this` (ctor caller / member-vector owner). Then per
member: `*vt0(*[member+0x68]+0x140)` → src multipose base;
`numBones=[ctx+0x158]`; dst pose = `*vt0(*[member+0x60])`. Bones are
32-B `{pos4,quat4}`; sample cursors `member+0x80/+0x84` give frameA/frameB.

## Capture round 2 — live pose buffers (VERIFIED-INGAME)

`target/tmp/capture_round2/` — benchmark world (`BM_Island`), 412 active
mob members, 14 frames, `Mob.GPUMemoryOnly=1`. See `NOTES.md` there.

- MobController vtable is `0x141dcdd88` (not `0x141dcdcc8` — earlier
  identification was a sibling/base); update fn `0x14180deb8` at `+0x90`
  (slot 18). Registries: `0x142a25ca8/cb0` (8-B list), `0x142a25cc0/cc8`
  (0x10-B {obj,refcount}).
- Members stride `0x88`: `+0x60` dst pose wrapper, `+0x68` anim ctx
  (`+0x140` src multipose wrapper, `+0x158` numBones=20, `+0x38` model
  path), `+0x80/+0x84` frame cursors.
- Dst pose = `0x48`-B header + numBones × 32-B `{f32 pos4, f32 quat4}`,
  4-slot ring (write idx at wrapper `+0x50`, slots `+0x10..+0x28`); pick
  slots whose quats are unit-norm.
- Src multipose = `0x48`-B header + array of 8-B `{u32 payload, u32 tag}`
  entries (tags 0,1,2,11 common) — an indirection structure, NOT poses.

## Clip-identity + trajectory reconstruction (VERIFIED-INGAME)

- `asrc` (manifest, member's anim source ptr) groups members playing the
  same clip — 10 distinct sources in the capture, largest group
  `0x15890c47c` with 142 members spanning `cur84` 0–181.
- `cur84` = per-instance clip cursor. Members at the same cursor hold the
  same pose modulo per-quat sign convention: after sign-canonicalizing
  (dot with previous quat), pairs agree to ~0.01 (interp timing). Raw
  diffs of ~2.0 are q↔−q flips, not divergence — the engine stores quats
  with arbitrary sign per bone/frame.
- 30 valid f0 members → per-bone sign-continuous quat trajectories over
  the full clip cursor range (samples in `oracle_match.py`).

## Whole-corpus oracle match — NEGATIVE (REFUTED)

Scanned all 1667 corpus `.anix` files for 16-B grid records (marker
`fe {1f|5f|9f|df} ff 03`, uniform row pitch), decoded the `rol(8)`/`rol(24)`
s32 fields ÷2³¹ per slot, and absolute-matched against the captured
sign-canonicalized quat component trajectories (cursor→row linear map,
err < 0.02–0.03 over ≥10 samples):

- Raw values: **0 hits**.
- Cumulative-sum (delta) interpretation: **0 hits**.

⇒ The smooth decoded `d0/d1` fields are NOT direct quat components in
any linear sense, and the 16-B grid rows are probably not per-frame
rotation keys at all — the fields are an intermediate/compressed
representation (coefficients, angles, or channel data needing another
transform), or the rotation keys live in a different sub-block than the
grid. The q↔−q sign-coherence between `d0/d1` remains real but is
consistent with several non-linear interpretations (e.g. stored sin/cos
or axis-angle pairs).

### Next avenues

- Try non-linear maps on the decoded fields (sin/cos half-angle, atan2
  pairs → axis-angle) before matching again.
- Decode the src-multipose 8-B `{payload,tag}` entries — they may be the
  runtime-side channel/record indirection that names which anix sub-block
  feeds each bone.
- Identify `asrc` → clip by reading the anim resource name out of the
  live ctx (`ctx+0x38` model path is captured; clip name likely nearby).

## Capture round 3 — clip identity + descriptor graph (VERIFIED-INGAME)

`target/tmp/capture_round3/` (NOTES.md there). Key deltas:

- **`asrc` object embeds the literal `.anix` path** — clip↔file identity
  is now directly readable (`orc_sync_stealthkill_resist.anix`,
  `orc_sync_infightfatalkill_vicdlc.anix` observed).
- `Mob.GPUMemoryOnly` must be set BEFORE member init (set cvar, then
  LoadWorld) — otherwise wrappers never spawn.
- Src buffer is a channel-descriptor graph (ptr pairs + in-buffer
  relocated pointers + param f32s + embedded strings); all external
  pointers chased and dumped (`*_ptr*.bin`, 149-241 regions/member).
- **DEFINITIVE REFUTAL**: matched the exact playing clip's grid (all
  slots/fields, affine fit) against its captured quat/pos trajectories —
  zero non-trivial hits. The 16-B grid records are not per-frame TRS.
- Runtime track store is now on disk: the descriptor blocks' ptr pairs
  point into the chased regions; next step is mapping block ↔ region ↔
  channel, then diffing runtime arrays vs .anix sub-blocks.

## Correlation campaign — final status (REFUTED, comprehensive)

Matching attempts against grid `rol8`/`rol24` (+u16/le variants) fields:

| Oracle data | Files | Fit | Result |
|---|---|---|---|
| interpolated dst quats (sign-canon) | 1667 | abs err | 0 hits |
| same, cumulative-sum | 1667 | abs err | 0 hits |
| exact playing clip (stealthkill_resist) | 10 grids | affine m·v+c | only m≈0 trivial |
| real src-block keys (6 frames) | 1077 | abs err <0.02 | 107k boundary hits = noise |
| src keys byte-search in file | corpus | f32/s16/s32 | 0 hits |

Conclusion: **the 16-B grid records do not store per-frame TRS in any
linear/affine encoding**, and decoded src pose values do not appear as
bytes anywhere in .anix — the codec is a true transform (encoding),
not repacked floats. The src multipose itself is the decoded product
(verified {quat4,pos4} local-pose blocks).

## What IS nailed down

- `.anix` file ↔ runtime clip via `asrc` embedded path.
- src multipose = `{quat4, pos4}` keyed-frame blocks (20-bone low-LOD
  subset, parent-relative pos under a fixed basis permutation; r2/r3 =
  L_Legu/R_Legu anchor).
- dst = `{pos4, quat4}` interpolated pose; frame cursors at member
  `+0x80` (running) / `+0x84` (derived index).
- Runtime pipeline: `.anix` → decode → keyed src blocks → slerp → dst.
  The compressed→decoded boundary is *load/apply-time*, not a memory
  dump artifact.

## Remaining unknowns (ranked)

1. Which anix sub-block produces each src keyframe — the `(ch,t)` index
   blocks + vec3 blobs remain undecoded suspects for rotation keys.
2. `cur80/cur84` → src block-index mapping (which keyed frame each
   cursor value selects).
3. c0m0's exact clip path (asrc path at obj+0x3f4 — dump 0x800+ next
   capture; also dump ALL 4 src ring slots, not just sidx).
4. Whether grid records are a separate mechanism entirely (LOD weights,
   morph targets, event lattices) — their per-frame smooth fields are
   real but don't encode TRS.

## Round 5 — the real frame array + cursor math (VERIFIED-INGAME + VERIFIED-STATIC)

Disasm of the multipose update fn `0x14180deb8` (mob-subsystem vtable
`0x141dcdd88` slot 18) revealed the actual addressing:

- Src pose wrapper `sw = member+0x68 ctx → +0x140` shares the dst
  wrapper vtable `0x141da9718`; `vt[0]` (`0x140251f5c`) returns
  `sw[0x10 + idx*8]` where `idx = u8 sw[0x50]`.
- The returned slot pointer's **first qword is the frame array base**:
  `fb = *(u64*)slot`. The update fn then reads
  `frameA = fb + (cur84 + asrc[0xc])*nb*32` and
  `frameB = fb + ((cur84+1) % asrc[8] + asrc[0xc])*nb*32`.
- Frame records are `{f32 pos[4], f32 quat[4]}` — **same layout as
  dst** (pos at +0, quat at +0x10). Verified live: fb block0 = identity
  root + 19 unit-quat bone records.
- Cursor math: `cur80 += elapsed_subframes; cur80 %= asrc[4]`;
  `cur84 = trunc(cur80/asrc[4] * asrc[8])` (i.e. asrc[4] = clip length
  in subframes, asrc[8] = frame count). Subframe fraction
  `t = (cur80 − cur84·spf)/spf`, `spf = asrc[4]/asrc[8]`.
  Live check: cur80=3179, a4=14466, a8=232 → cur84=50, t≈0.99. The
  earlier "cur84 = rem/cur84*a8" reading was wrong — the divisor is
  `[asrc+4]` reloaded from the stack slot.

Corrections to earlier notes:

- Round-3/4 "src slot" dumps (`*_srcs*.bin`) captured the slot buffer
  *contents* — a per-clip serialized resource whose first qword is the
  frame pointer. All four `sw` slot pointers can alias the same
  buffer (round 4: sb0==sb1==sb2==sb3).
- The round-3 c0m0 `src.bin` did contain six inline keyed-pose blocks
  (`{quat,pos}` order, swapped vs the frame array) — a different
  resource layout within the same buffer family. Both are decoded pose
  data.
- dst pose buffer header varies: `0x48` and `0x50` both observed
  (quat records at hdr+0x10+k*32).

### Oracle pair (offline, complete)

423 members shared `asrc → orc_sync_stealthkill_resist.anix` and an
**identical** 0x1448 src buffer containing 6 keyed `{quat,pos}` blocks
(bone order anchored to orc_base.skel: r1=Torso_Base, r2=L_Legu).
So: `(file, clip family) → decoded keys` — the codec fit is now a pure
offline problem over one known bank (153 clips, all variant-B).

### What round-5 dst correlation showed

Interpolating `fb` frames at the computed index does NOT reproduce the
member's dst pose (~0.3 quat error, worst in high-index bones) — dst
is a composite (crossfade/layered) output, not the raw lerp. Not
blocking: the frame array itself is the decode product and the codec
oracle.

### Round-5 capture artifacts

`target/tmp/capture_round5/`: `manifest.jsonl` (163 rows: cursors,
sw/slot/fb/asrc), `c*m*_frames.bin` (0x8000 at fb), `c*m*_dst.bin`,
`c*m*_asrc.bin` (0x800), plus `probe_*` and `orc_*` sets. Note: this
world state populated **arrow-bolt FX attachments** only
(`orc_bolt.skel`, `arrow_head.fxp`) — no orc mob members; round-3's orc
crowd needs the earlier world/layer state to recapture.

### Next attack vector

1. Locate the anix→frame-array decoder. The exe is protected for
   static xref (vtables absent from disk) — find it live: resource-load
   path for `.anix`, or trace who allocates/fills `fb`.
2. Offline: `orc_sync_stealthkill_resist.anix` clip index identification
   via the 6 decoded keys (asrc[0]=0x52e8cd85 is probably the clip name
   hash — no match among resolved names so far).
3. Recapture orc members with the corrected fb chain (dump4.lua needs
   a world state where mobs exist, e.g. revisit round-3 world).

### Round-6 correlation results

- Clip identity via translation keys: `f32` vec3 keys in per-bone
  streams decode correctly; key sequences match the captured `c2m0`
  position trajectories under axis permutation. Strongest candidates:
  clips 16/18/21 (clip21: 167 anix frames ≈ 211 runtime at ~1.26×
  resample).
- Biped rotation encoding: NOT s16 quads in the bulk block, NOT s16
  triples with reconstructed w (no sequential or monotonic-frame runs),
  NOT f32, NOT affine-linear on the 16-B grid fields. The bola-verified
  `u32` pool layout does not appear in biped banks.
- `orc_bolt.anix` minimal clip = 2×16-B records; terminator signature
  `?ffe|0x03ff` confirmed at record end.

### Round-7 (decoder hunt — VERIFIED-STATIC + LIVE field map)

**ANIX bank object layout** (`0x140480af0` loader, assert strings confirm
roles — "AllocateAnimBlockChunk"/"AllocateAnimChunk"/"AllocateNodeAndTrackName"):

```
bank (0x28B): +0x00 blob1 = "AnimBlock chunk" ptr (+0x18 size)
              +0x08 blob2 = "Anim chunk"  ptr   (+0x1c size)
              +0x10 index = {u32 off, u32 size} node/track-name table (+0x20 count)
```

The loader only reads raw blobs — **no decode at load**. Decoding happens
when a clip instance binds to a member.

**Update/blend fn `0x14180deb8` fully mapped** — confirmed it *only*
interpolates already-decoded frames: per layer (0x88-stride array at
`r13+0x138`), `t=(cur80−cur84·spf)/spf`, per-bone `nlerp` at
`0x140210078` + pos `subps/mulps/addps`. No codec call inside — frames
are pre-decoded into the shared multipose.

**Layer field map (live, current build)** — `m+0x50` = channel/decoder
object (primary vtable `0x141ffe3e8`, ≥80 slots; `m+0x58` = same object's
secondary-vtable base −0x28); `m+0x68` = ctx/multipose owner
(vt `0x141dcc088`, `+0x140` → sw → `[slot]` frame array);
`m+0x78` = asrc `{a4,a8,a12}` record. `m+0x60` = 0 (unused this build).

Decoder call sites in update fn: `channel.vt+0x1c0(edx=frameElemIdx)`
→ `0x140650744` (engine reflect glue → `0x141166e70`/`0x140251e00`);
`vt+0x1c8(&multipose)` → `0x140159760` (bind; deep call `0x1407a9a94`
is plumbing, not a bitstream reader — verified score 2, no shr/movzx
loops). All `0x141c9xxxx` vtable methods = engine dispatch glue, not
the codec.

**Havok evidence**: exe embeds full Havok Animation type set
(`hkaSpline/Predictive/Quantized/Delta/WaveletCompressedAnimation`) —
the rotation mass is plausibly a Havok compressed-animation block
wrapped in Monolith's clip container; the `{33,34}` per-frame table
fits spline codec's fixed-bit-budget frames.

**Remaining**: the actual bitstream decoder fn is still unidentified —
the channel object's methods dispatch through the reflect layer so the
codec doesn't appear in its vtable. Next leads: (a) callers of
`channel.vt+0x1c8` bind — whoever fills the multipose after bind;
(b) xref on `[bank+8]` Anim-chunk reads at clip-instance creation;
(c) live read-breakpoint on the 47 KB rotation region (needs a real
debugger — none installed).

### Round-7b (job system + bind path — VERIFIED-STATIC)

**Job names decoded** (scheduler `0x1413ecde0` type→name table):
`TtAnimation Mapping`, `TtAnimation Sample and Blend`,
`TtAnimation Sample and Combine` are distinct anim job types —
the channel-sample/decode runs inside one of these job workers
(scheduler dispatches via `call [job+0x30]` at `0x1413ecfc9` —
a *direct* fn ptr in the job record, not a vtable).

**Channel obj vtable is script glue** — every `0x141ffe3e8` slot
traced (`+0x1c0`→`0x140650744`, `+0x1c8`→`0x140159760`, method
cluster `0x141c9xxxx`) funnels through the reflect/dispatch layer
(`0x140251e00`, `0x141166e70`, `0x140307f90` refcount-release).
The codec does NOT surface through this vtable — the channel object
is a script-facing component that submits anim jobs.

**Refuted leads this pass**: `256.0f`/`1/256f` immediate hits are
unrelated (vertex/color constants); `0x14180a200` window = vertex
writer; job-scheduler region = generic dependency walker;
'Decompression failure.' = file-stream zlib; resource-table
neighbor `0x140480898` = anix type unload (frees `+0x30` clip
holder, `+0x40`, `+0x50` list — confirming clip-instance objects
hang off the resource record at `+0x30`/`+0x40`).

**Two live options to name the decoder** (pick one):
- **A. Job-record capture**: dump a `TtAnimation Sample*` job's
  `+0x30` worker fn ptr live → disasm → codec. Needs a world state
  with active anim jobs (same capture path as round 6/7 works).
- **B. Hardware read-breakpoint** on the 47 KB rotation region in
  the `.anix` bank blob — needs x64dbg/CE installed; lands directly
  on the decoder with zero guessing.

## Round 8 — streaming decode VM mapped via exec breakpoints

**New instrument**: `sdk.debug.watchx(addr)` — Nemesis-side hardware
*execute* breakpoint (DR0-DR3, 4 slots, regs + stack census per hit).
Verified live: `watchx` on the blend fn gives callee-entry census —
no external debugger needed. Caution: arming hot fns throws millions
of exceptions/min and kills the game — arm cold/bind paths only.

**Decode pipeline (VERIFIED-INGAME + STATIC)**:

```
channel obj (vt 0x141ffe3e8)
  +0x680 = anim-source object ──bind──► static manager 0x14272e8d8
                                        (registrar 0x141154ac8)
worker thread drains continuation queue (manager+0x108):
  puller thunks 0x140258708/0x140177d60/0x14076b37e
    items = {cursor, payload_obj, next_thunk, f32 time}
    jmp [item+0x10] — chained stage handlers
cache-guard 0x140177d7c (key: edx&0xfffffff|0x50000000, time compare)
  └── evaluator 0x140177e24 = tagged-record stream scan
        per-record dispatch 0x140180c5c
        tag handlers 0x140178050 / 0x140698f58 / 0x140702b48
```

- The tagged-record VM is ENGINE-GENERIC (8.3M evals/min across all
  subsystems) — high-nibble tag dispatch (`shr ecx,0x1c`), paged
  records (`[rdi+4]` byte stride per page).
- Decoded pose buffers confirmed live (quat f32 blocks at e.g.
  `0xa0b4a88`), touched by both the game thread (samplers
  `0x14114fxxx`) and the decode worker.
- Member-layout this build: `m+0x50` channel obj (vt `0x141ffe3e8`),
  `m+0x68` ctx (vt `0x141dcc088`), `m+0x70` track-record list,
  `m+0x78` small record. `m+0x60`/`m+0x78-as-asrc` are dead.

**Still unisolated**: the rotation-codec fn — it is a record-type
handler below the stream VM (or the payload obj's vtable eval).
Next probe: brief `watchx` on `0x140180c5c` *during a fresh spawn
only* (bind window) → dump the record tags + handler table entry for
anim records → that handler is the codec entry. Or data-watch the
payload object's compressed-ptr field during the same window.

**Refuted**: resident compressed-blob copies (12 in heap) are never
re-read at idle — decode is bind-time only; data-watch on them yields
allocator noise.

## Round 9 — item-arena correction + spawn-window census

**Correction**: `0xa0b4a88` (previously "decoded quat buffer") is the
worker's **continuation-item arena** — items `{cursor, payload,
next_thunk, f32}` live there and every captured read/write RIP on it
was dispatch glue (puller thunks `0x1401ee088`/`0x140177d60`,
item-field copier `0x1401777a3`, fill `0x14025874b`). The "quat
floats" were item payloads. Real pose output remains the `fb` slot
arrays from rounds 5-6.

**Spawn-window census** (watchx on bind `0x140159760` → 68 hits):
- `rcx` = channel obj (vt `0x141ffe3e8`), `rdx` = `0x15d738`-class ID
- `[channel+0x680]` reads 0 at rest — anim-source ptr is only live
  inside the bind call, or cleared after registration into the
  manager. Capture it *at* the bind hit (register dump: rdx path) or
  read the manager's map payloads.
- Second decode worker tid observed (38720) — decode is
  multi-threaded; tid=34332 was one of ≥2.

**Bind-hit stack** `+0x159722 +0x1b05e4 +0x1594f3 +0x159495
+0x159431 +0x158e39 +0x3fe7fb...` — bind is driven from the
`0x14015xxxx` channel-method cluster via `0x141b05e4`.

**Refined next probe** (now cheapest-correct): `watchx` on bind
`0x140159760` gives `rcx` at entry → **immediately** read
`[rcx+0x680]` *in the same Lua command as the spawn* (the field is
live only during bind — poll in a tight loop during spawn, or set a
second `watchx` a few instructions later where rdx=[rbx+0x680] is
loaded → the hit's rdx IS the anim-source ptr). Then data-watch the
anim-source's data region during the next spawn → the codec reader
RIP, directly.

## Round 10 — registrar anatomy + blob-ptr dead ends

**Registrar anatomy** (watchx `0x141154ac8`, full-reg logging):
- Anim registrations identified by `r9=0x140733c2c` (the anim cb).
- `rbx` = channel obj (vt `0x141ffe3e8`), `r8 = rbx+0x10` — matches
  bind's `lea r8,[rbx+0x10]` — rdx=0: `[channel+0x680]` is null even
  at registration; the anim-source is created/held elsewhere.
- Registrar is engine-wide HOT — killed the game ~90s armed.
  watchx is cold-path only (2nd confirmed death).

**Dead ends this round**: no raw ptr within ±2MB of a resident
bip_mov_action blob copy in — manager map elements, queue payloads,
or live anim channels. The compressed blob is reached via the anix
resource record (+0x30/+0x40 clip holders), not runtime objects.

**Bind timing**: binds mass-fire at world-entry only; summons of
already-bound types trigger none. To census: arm `0x14015977e`
before in-world completes.

**Remaining probe** (pinned): arm `0x14015977e` at pipe-up on a
fresh boot → rdx = anim-source per bind → dump sources → find the
blob-ptr field → data-watch it during a second boot's bind phase →
codec reader RIP.

## Round 11 (2026-10-12) - CODEC PRIMITIVE CRACKED: Havok QT_32bit spherical quats

**The rotation mass is a packed stream of Havok `QT_32bit` spherical quaternions** (`hkaSplineCompressedAnimation`-family encoding, implementation in PredatorCZ/HavokLib `hka_spline.cpp`): each quat = 4-byte LE u32 - 10-bit w-magnitude `R = 1-(bits/1023)^2`, 18-bit spherical `phi/theta` payload, 4 sign bits (bits 28..31). VERIFIED against the live oracle: decoding `u32`s from the `.anix` blob yields quaternions matching captured runtime frame quats at |dot| up to 0.9999 (e.g. captured bone-15 quat vs stream quat at 0.9954 - the residual is neighboring-key resampling, not decode error). Refutes every earlier custom-codec / entropy-coded theory for the rotation data.

Structural status:
- The ~47KB rotation mass decodes cleanly as u32 quats throughout - real anim quats (exact identity matches, plausible non-identity).
- The per-clip `u16` table of ~150-170 entries valued {33,34} is confirmed **per-clip** (~numItems), but is NOT row-lengths and NOT track-key-counts as tested (column/track smoothness failed). Exact role still unresolved - candidates: knot counts, mask bits, spline degree metadata.
- Stream ordering is NOT simple frame-major (34/row) or track-major at the offsets tested - consecutive u32 quats map to scattered (frame,bone) in the oracle. Havok's real block layout (TransformMask headers + per-track spline blocks + knot lists + alignment) is the next thing to fit.
- ~20k near-exact u32 matches spread across the file - many are near-identity codes; the distinctive-quat clusters remain to be mapped.

Tooling: offline `r32` decoder matches HavokLib `Read32Quat` exactly; runtime capture `capture_round6/c2m0_*` (20 bones x 211 frames) is the scoring oracle.

Next: fit the Havok block grammar - scan for TransformMask records (quantization-type bytes incl. QT_32bit) preceding quat runs, parse `numItems`/degree/knot lists per `hka_spline.cpp`, then assign tracks via the node-name index. The codec fn in-engine is presumably a Havok decompressor invoked per-record under the tagged VM (round-8 map).

## Round 11b - correction: QT_32bit "confirmation" was a false positive; real structure found

**Refutation of the r11 claim**: a control test showed decoded-u32 quats match the captured-quat oracle at LOWER rates inside the rotation region (0.41) than in unrelated file data (0.50-0.66). The captured quat value-space is too small/low-diversity - most decoded values hit *some* captured quat by chance. QT_32bit oracle agreement is therefore NOT evidence; the codec primitive is back to UNKNOWN.

**What IS established about the rotation mass (clip-1 region, file offsets 0x2600-0xd600):**
- ~150 records of ~288-304B each (stride alternates ~288/304), followed by a u32 prefix-sum table at 0xd7f0 (151 entries, 33..5033, diffs {33,34}) and a u16 count table at 0x2480 (150 entries {33,34}).
- Consecutive records are ~90% byte-identical with localized deltas = sequential keys of the same skeleton (frame-major most consistent; track-major can't be ruled out by similarity alone).
- High per-element entropy: no fixed-stride decode found for s16-quat4 (norm fails), q3 (s16 xyz + recomputed w ~52% expected by chance, ~83% observed = weak signal only), QT_32bit, or 6/8/9B strides; no per-element index/flag field found. Elements look bit-packed (~9B each if 33-34/record).
- `fe df ff`-family byte patterns recur at paired offsets (16B apart) in many records - probably near-constant packed values for static bones, i.e. fixed bit-field positions rather than arithmetic coding (identical bytes would not survive range coding).

**Interpretation**: per-record bit-packed element array; ~33-34 elements/record at ~8-9B each; long identical byte runs between records = static bones' packed fields unchanged. Suggests fixed-width bit-fields (Havok QT_40/48-bit or a custom fixed-point quat) rather than variable-length entropy coding - earlier "7.7 bits/byte entropy" conclusion is consistent with dense fixed-width packing too.

**Next**: determine element bit-width via record-diff bit positions (identical elements produce identical bytes; find the fixed element bit stride that maximizes cross-record run length), then decode one element's fields. The u32 prefix-sum table maps record i's element span directly.

## Round 12 - live blob-watch experiment: resident copies are NOT the decode source

Setup: in-world, `sdk.debug.scanmem` located 8 live copies of `bip_mov_action`'s compressed mass (needle at file+0x4000) and 12 live `{33,34}` count-table instances (`21 00 21 00 22 00 21 00 22 00` pattern) across heap regions `0x2ec3exxxx`, `0x30086xxxx`, `0x30774xxxx`, `0x31c16xxxx`, `0xd3e9xxxx`, `0x9cf8xx`.

- Armed data-watchpoints on 4 blob-copy record regions: all hits were ntdll/`0x7ffd...` heap-scan noise (Rtl-walk RIPs `0x7ffd0e005588` etc., tid 44756) - not decoder reads.
- Re-armed on 4 count-table buffers directly: zero exe-range hits during ~2 min ambient play.
- `Enemy_Infantry` summon dispatched mid-run; game exited shortly after (keydebug shows user input; final VEH event was a benign KERNELBASE 0x71a; process gone at session end).

**Conclusion**: the resident blob copies and count-table buffers are *not* read during ambient per-frame sampling - the decoder reads a runtime-only buffer populated at bind time (consistent with the round-5/6 `sent` object and `fb` frame-buffer model; Havok-style decompress-into-stage then sample). To catch the codec: watch the runtime staging buffer (not the resource blob) during a fresh bind, or watchx a codec-candidate fn in the bind window. The `sent` object layout from capture_round6 dumps is the pointer map for finding it live: member `m+0x50` = channel obj (vt 0x141ffe3e8), `m+0x68` = multipose ctx (vt 0x141dcc088).

## Round 13 - staging-buffer hunt: landed in BENCHMARK mode, not live play

- Menu driving landed the game in the built-in benchmark sequence (screenshot verified: Benchmark Results overlay, fixed scene). `in_world()` and the member enumeration still worked (6 controllers / 412 members), but no member's ctx fields (+0x148..0x158) or pose buffers changed over time - benchmark anims don't flow through the normal member update fields we sampled.
- Found a live pose buffer by pointer-walking member ctx (m+0x68 vt 0x141dcc088, then +0x148 -> obj vt 0x141da9778, +0x10 -> slot array -> packed f32 quat/pos pairs, e.g. 0x2bbf6b3e0: `3f80000000000000` = identity pair). Watchpoints on it: 0 hits - buffer was static (dead-pose member or write-once snapshot).
- `Enemy_Infantry` summon DID dispatch in benchmark state (log: `spawn: Enemy_Infantry x1`) but produced no field deltas - again consistent with benchmark-mode anim routing.
- LESSON: verify live gameplay state with a screenshot BEFORE sampling member liveness; benchmark/demo states populate members but don't tick them through the same path. Also note the earlier 'armed2' count-table watches persisted across `watchclear` in this session's first arm attempt - always confirm arm state.

## Round 14 - live benchmark capture: per-frame evaluate loop mapped

Environment note: session.py lands in RUN BENCHMARK when save/continue isn't offered; NEMESIS START menu lacks Continue when profile unsigned. Benchmark scene = continuous anim decode on worker tid; usable for capture. ModMenu overlay (SoW_ModMenu.dll) auto-opens and eats keys; F6 toggle via GetAsyncKeyState only - had to move DLL to .off before launch.

New verified structure (live, benchmark scene):
- `watchx(0x14180deb8)` (blend fn) fires continuously on anim worker tid 14976: `rbx`=0xdf60f4d8 = controller object (vt 0x141dcdd88), `rcx` walks controller array at ~0x97baXXX (stride 0x168), `rsi`/`rdi` = 0xc3ba038/0xdf60f5a8 ctx objects.
- Track-record evaluate loop at `0x1402fa118`: `eax=[rcx]` current key node; `movss xmm0,[rax+0x14]` = key time; intrusive-list unlink of consumed key at `+0x60`; `xmm1=[rcx+0xd0]` current anim time; `subss` remaining; `jb` continue; else `jmp [vt+0x90]` -> per-track sampler dispatch (for these channels = blend fn 0x14180deb8).
- `0x14059ab3c` = intrusive-list relink helper (not codec).
- Key node layout: `+0x14` = f32 time; node = pooled object. Controllers double as track/channel records.
- Data-watches on the 0x97baXXX region hit the list/time ops above - still not the compressed-data reader; the key-node payload read is the next watch target: watch `[key_node+0x18]` region contents during benchmark to catch the payload fetch, or data-watch a key node's payload bytes to identify the decompressing reader.

## Round 15 - record data buffer = HavokScript VM stream; codec is script-invoked

Session ran in windowed mode (render.cfg Windowed=1, restored after) - avoids
fullscreen focus-stealing during automation. session.py in-world drive lands on
the benchmark scene (FPS overlay flythrough) which runs continuous anim decode.

Verified live (benchmark scene, anim worker tid 15252):

- Track record (vt 0x141dcdd88, 0x168 stride, 0x9ae4XXX pool) fields:
  - +0x18/+0x20/+0x28/+0x30 = intrusive node-list heads (nodes link back at +8)
  - +0x58 = parent controller back-ptr (0x27d244ed8 family)
  - +0x60/+0x68 = sibling list links (the region walked by blend fn rcx)
  - +0x88 = data buffer ptr (e.g. 0xa004888)
  - +0xb0 = f32 elapsed ms (~32122), +0xc0..0xc8 small f32s
- Data-watching record+0x88 (0xa004888) caught the reader immediately:
  rip=0x140c38043 = `call qword ptr [rax+0x10]` inside a loop
  `rbx=[rbx+8]` - a linked-list command dispatch. Element +0 = descriptor
  ptr into exe .xtext (0x1422c2570): {0x1411b4a74, 0x1411b4b74 (dtor/free
  thunks), handler@+0x10 (.bss slot, runtime-patched), cstr "Stack index out
  of bound"}. This is the HavokScript/Lua VM opcode descriptor format -
  the anim evaluation loop executes script on the worker thread.
- IMPLICATION: per-frame anim sampling is driven BY HavokScript. The ANIX
  codec is a native function invoked from the script VM (script ops call
  it), not a tight native loop on the record array.
- CAUTION: exec-bp / data-bp on the opcode dispatch is fatal in ~seconds
  (4 hits then 0x80000004 single-step death + watchdog). Use only on cold
  paths.

Next pinned probe: find the native-side call from the VM into the decode -
either (a) data-watch the POSE output buffer (member+0x68 ctx -> +0x148 ->
+0x10 slot array -> packed f32 quat/pos buffer, round-13 chain) during
benchmark - writer = codec output, or (b) enumerate the VM's native-function
table for anim-entry natives (the desc handler slots, runtime-patched) and
exec-bp candidates on the decode-shaped ones (cold enough not to kill it).

## Round 16 - REAL gameplay reached; registry records are persistent rigs

- **Menu path to real gameplay**: NEMESIS START menu -> ENTER on first row
  (WBPlay/START) -> save-select screen -> SPACE on "Shadow of War" save.
  session.py 'world' drive lands on benchmark because the extra ENTER
  hits RUN BENCHMARK; real path needs the save-select hop.
- Confirmed in-world with animated ghuls on screen (perf overlay).
- Registry globals {0x142a25ca8,0x142a25cb0} hold only 6 records
  (0x168-stride, vt 0x141dcdd88) - persistent/menu rigs, NOT the live
  animated NPCs. Both distinct pose buffers (ctx+0x148 -> +0x10 slot
  array) were byte-identical over 4s even in real gameplay.
- record+0x138 -> member-like obj (vt 0x141f71e58): +0x68 ctx
  (vt 0x141dcc088), +0xd8 channel (vt 0x141ffe3e8), +0x60 dst wrapper
  = NULL for these (CPU-path members; GPU wrapper lazy).
- member+0x60 dst-pose wrapper was 0 - docs' `*vt0(*[member+0x60])`
  path is GPU-only (GPUMemoryOnly); CPU-path output lands elsewhere.
- Static multipose = consistent with whole-clip decode at bind (grid
  decoded once, sampling only interpolates) OR idle rigs.
- COST NOTE: exec-bp on blend fn 0x14180deb8 killed the game again even
  armed for ~3s (0x80000004 after ~256 hits). Never arm hot paths.

Next pinned probe (needs real gameplay, path now known):
1. in-world via save-select; find a visibly animated NPC's member -
   member pool unknown, but spawn.lua 'spawn Enemy_Infantry' in real
   gameplay creates a fresh animating member whose ctx/src-multipose
   can be read post-spawn; if src multipose WRITES continuously ->
   write-wp on it lands the decoder writer.
2. If src is whole-clip-static: the decode runs once per bind; catch it
   by watching the NEW member's multipose region the instant after
   spawn (poll member list for a new entry -> immediately watch its
   src grid for the writer that fills remaining keys), or accept
   bind-time decode and instead disasm the bind-path callee that fills
   the multipose grid (channel+0x680 anim-source -> fill fn).

## Round 17 - ctx enumeration solved; decoded-grid ORACLE extracted

New live-RE capabilities (all committed tooling-level knowledge):

- **scanmemptr(vt_addr)** enumerates every heap qword equal to a vtable
  VA -> every live object of a class. `scanmemptr(0x141dcc088)` listed
  all 24-34 anim-ctx objects in one shot (no breakpoints needed).
  False positives exist (ASCII/double bit patterns) - filter by
  plausible +0x158 bone-count field and +0x148 sw pointer.
- **scanmemptr(obj_addr)** finds all refs to an object - a member's
  +0x68 slot points at its ctx, so scanmemptr(ctx) returns the member
  array region (refs stride 0x88 = member stride).
- **io works in the plugin VM** - chunked r.dump()+string.pack writes
  multi-100KB memory snapshots straight to disk.

Extracted `target/tmp/grid_e9fb20a8.bin` (139264 B): a live decoded
multipose grid - **frame-major {f32 pos[4], f32 quat[4]} records,
20 bones/frame, ~217 frames, all quats unit-norm, adjacent-frame
|dot|=0.9967** (other strides ~0.3-0.5 random). This IS the decode
ground truth: grid[frame][bone] = plaintext corresponding to blob
elements - plaintext-ciphertext pairs for codec verification WITHOUT
catching the writer.

- Member layout reconfirmed: member+0x68=ctx, +0x78=anim source,
  +0xd8=channel (vt 0x141ffe3e8). Member refs to one shared ctx.
- Spawn-diff method works: sdk.spawn.summon('Enemy_Infantry',1)
  dispatched from benchmark state; new ctxs appear (need re-scan +
  filtering - most early hits are mid-init garbage, settle ~10s).
- CONFUSION NOTED: r16/r17 'real gameplay' screenshots were actually
  benchmark flythrough (Benchmark Results overlay followed). Real
  gameplay = HUD not FPS overlay. Save-select path is still the
  correct real-world entry.
- Static grids confirmed in gameplay too: whole clip decodes at bind;
  the codec writer only fires at bind/clip-change. To catch it live:
  spawn while watching a brand-new ctx's sa region (poll scanmemptr
  every ~2s post-spawn, watch the NEW ctx's grid for the writer).

## Round 17b - offline corpus pairing attempt + refined target

- Extracted grid (217f x 20b, upward-chained positions 24->57 - a
  tail/spine/fellbeast-style rig) did NOT shape-match any clip in the
  1666-file corpus: no u32 prefix-sum table with 180+ rows of ~20 elems,
  no exact f32 position match (positions are packed too), no u16
  ~20-count table. The 20-bone rig's clip is either outside the
  extracted subset or uses a different table layout.
- Refined pairing hypothesis for bipeds: clip21 = 150 rows x {33,34}
  elems -> if rows=frames and elems=bones, a biped member's grid is
  ~150 frames x ~34 bones (5033 total elems ~= 5100=150*34 minus
  non-animated bones/rows).
- LIVE CAPTURE TARGET: enumerate ctxs (scanmemptr 0x141dcc088), pick
  one with numBones ~33-40 (orc biped), dump its grid, verify
  ~150f x ~34b dims vs clip21's table. Then codec-fit: each candidate
  9B decode must reproduce grid[f][b] quats exactly - unambiguous
  proof. The biped ctx candidates seen post-spawn: nb=68 entries
  (0xba0a0c0, 0x3205e3950) - possibly {bones=34, *2} or LOD-doubled
  counts; check nb field semantics live.

## Round 18 - grid enumeration at scale; biped ctx class still unidentified

- scanmemptr(0x141da9778) (the sw vtable) returns ~1000 hits, mostly
  false positives. Filter pipeline: sa=[cand+0x10] heap-ptr -> sampled
  f32 quat-component plausibility -> 218 candidates -> bulk-dump 8KB
  each via io -> offline unit-quat fraction >=70% -> only ONE real grid
  (0x7f4a9040 region, sw 0x7f4a8348): 99% unit quats, ~27-bone stride.
  Full 256KB dump at target/tmp/grid_7f4a8348.bin (positions like
  (0,-104,40),(0,39.7,202) - another rig specimen).
- ctx census (vt 0x141dcc088) in benchmark: 24 objects, bone-count lo
  u32 at +0x158 = 3,5,7,20 only - no biped ctxs. Biped orc ctxs use a
  DIFFERENT vtable - class unknown (mob-vs-character path or a sibling
  ctx subclass).
- All decoded grids confirmed static in every state sampled - decode
  is strictly bind-time; sampling interpolates the grid.
- Extraction now routine: scanmemptr class census -> walk -> io dump.
  Specimens in hand: 20bx217f (grid_e9fb20a8.bin), ~27b
  (grid_7f4a8348.bin).
- OPEN: biped rig's ctx vtable + a biped grid (need ~34 bones to pair
  with clip21 150x{33,34}). Options: (a) scanmemptr for member+0x68
  pointers into ANY ctx-class object (enumerate member objects via
  channel vtable 0x141ffe3e8 census instead - channels ARE enumerable,
  member+0xd8=channel so scanmemptr(channel) -> member array ->
  member+0x68 -> ctx vt), (b) spawn-diff with bigger settle window.

## Round 19 - entity census; 20bx320f animated-grid specimens

- ENTITY census solved: entities embed the anim member at entity-0x88
  (member+0x88 = entity vt 0x141f71e58); ctx ptr at entity-0x20.
  scanmemptr(0x141f71e58) -> 406 entities -> ALL ctx vts are
  0x141dcc088 (one ctx class for all characters, shared per-clip).
- So the biped ctx IS the 0x141dcc088 class but the benchmark census
  only yields small rigs (lo u32 of +0x158 = 3,5,7,20 = bone count,
  verified against the 20-bone grid). The benchmark's fighting orcs
  either use the mob crowd path or a state we haven't reached.
- Mob-subsystem objects (vt 0x141dcdcc8) found (3) but the member
  array offsets from the r14 notes don't match this build state
  (+0x140 cnt=1; candidate arrays at +0x1e8..+0x200 gave garbage
  members) - needs the fn-prologue disasm to re-pin offsets.
- SPECIMENS banked (target/tmp/grids/ctx*.bin, 200KB each):
  ctx240f7ad68 + ctx2c38bf0d8 share a 20bx320f grid (adj-frame
  |dot|=0.997), ctx2c2f1ddc8 5bx713f, ctx2c3999960 7bx873f,
  ctx2c3d585c0 5bx996f - all score ~1.0. Real decoded keyframe
  plaintext, several distinct clips.
- Nested scans (scanmemptr inside loop) killed the game once -
  O(channels x heap) is too heavy; keep scans single-level.

To pair a grid with its .anix blob we still need either (a) the
member's clip name (channel/resource -> name hash -> strdb), or (b) a
biped grid (~34b) - likely requires real gameplay where faction orcs
are full entities, or the mob path mapped.

## Round 20 - clip-identity hunt; asrc decoded as binding table

- ctx (vt 0x141dcc088) embeds the SKELETON mesh path at ctx+0x38:
  the 20-bone specimen ctx carried `characters\model\characters\
  ologhai\base\olog_low.mesh` -> the 20bx217f grid is the OLOG rig.
  These ctxs are per-skeleton objects, not per-clip.
- member+0x78 "anim source" is NOT a single-clip ptr - it is an
  ARRAY of binding records, stride ~0x4c-0x50, each:
  `+0x00 u32 id/hash, +0x04 u16, +0x08 u16, +0x10 flags 0x80000000,
  +0x28/0x40 f32 weight=1.0` - looks like a Havok-style
  animation-binding (track->bone map). IDs observed:
  0xb2bd82ce/cf/d0 (3 consecutive), 0xb4bd7c24/25/26 (3
  consecutive) - consecutiveness suggests slot IDs or open-
  addressed hash slots, NOT namehashes.
- NEGATIVE results: 0xb2bd82ce is not namehash() of any extracted
  .anix path/basename/stem; not found as raw bytes in any of the
  1666 extracted anix files; not in the extracted corpora
  (strdb/gamedb). All 8 live-memory hits for its byte pattern were
  coincidental string bytes.
- members do NOT embed path strings (441-entity ascii scan, 0 hits).
- `sdk.debug.dump` returns a TABLE of "+0x00: 0x..." strings -
  hex only, no ASCII column (gmatch on it found nothing).
- scanmemptr returns a plain array (use #ms, not ms.n).
- dofile() exists in the plugin VM - multi-line Lua probe scripts
  under target/tmp/ work fine.

Clip identity is still unresolved. Remaining ideas: (a) the
channel's per-track buffers (channel+0xb8 -> 0xNNNdfX0 array,
0x20 stride) may embed the clip name or resource link; (b) the
binding-record u16 fields (0xd3,0x1a2,0x356,0x540) look like
offsets/indices - map one to the blob; (c) find the clip
registry keyed by the consecutive IDs.

## Round 21 - channel track registry mapped; pairing attempts negative

- channel+0xb8 points into a GLOBAL per-channel track registry at
  ~0x9c21000, 0x20-stride records; each channel owns ~3 slots.
  Record types: type-A {u32 hash 0xe3c8292c, u32 seq#, 0, seq<<32}
  and type-B {vt 0x141db4508, 0x100000001, bufptr, seq<<32}.
- type-B buffers = tagged serialized resources ('pref' 4CC +
  embedded path). In the benchmark's loaded set they resolve to
  prefabs\ui\mordormap\* - UI/prefab resources, not anix. The
  0x141db4508 class is a generic serialized-blob wrapper; anix
  clips presumably wrap the same way when loaded.
- Track-registry entry stride = 0x20, base aligned at 0x9c212b0
  (not 0x9c20000 - grid offset by 0x10).
- NEGATIVE: verbatim f32 pos-key match of grid bone-11 (high
  variance, 16 frames, 256B needle) against all 1666 anix files =
  zero hits -> position keys are also packed/transformed in the
  bank; the '~6.5KB f32 vec3 section' in bip_mov_action is not a
  raw track dump (or not in grid order/layout).
- NEGATIVE: corpus-wide u16-table scans find only index ramps
  (150,151,... / 201,202,... = sequential track/bone index
  tables) and key-count plates (~256-281) - no ~20x217 shape.
  The '20bx217f' Olog grid's clip remains unpaired.
- dofile() + chunked io.write = a workable live-extract pipeline;
  keep probe scripts under target/tmp/*.lua.

## Round 22 - ANIX format structure decoded (benchmark4.anix)

Header (0x00): 'ANIX', u32 0x2c, u32 0x30, u32 filesize,
u32 secB(0x190), u32 0x30, u32 secC(0x740), u32 secD(0x53f0),
u32 secE(0x5b30), u32 0xdfc, u32 1, u32 1, then section data.

- **Clip names stored in PLAINTEXT**: section E holds a
  {namehash -> offset} directory (keyed by engine namehash(),
  verified: namehash('DeadWargKill_Inf')=0x24c77454 matches) plus
  0x80-stride records at E+0x600: {name cstr, pad, f32 1.0 x2}.
  benchmark4.anix = 16 named clips: DeadWargKill_Inf,
  OrcInf_LeatherHead/BeastChest/BeastHelmet/BeastLegs/Null/
  HunterChest/Apron, OrcArc_Rand/DruidLegs/Grog, OrcDef_Ancient/
  Rand, Olog_LongTooth, DrakeSit, DrakeFlyBy.
- Section A@0x30: 0x48-stride records {u32 hash, u32 count=16,
  u32 clipNamehashes[16]} - the bank's clip-hash table.
- Section C@0x740: u16 index ramps + clip records
  {u32 60000?, u32 0x040000ff, u32 0, u32 decodedSize, u32 off}.
- Section D@0x53f0 + similar tables inside C-region mass:
  **u32 element-start offsets, consecutive diffs = 33 or 34** -
  the same {33,34} element-count pattern found in bip_mov_action!
  Element indices into a shared element pool, NOT byte offsets
  (values exceed filesize -> decoded-buffer-relative).
- Compressed key mass: 0x1f40..0x2f40 and 0x3740..0x3f40
  (high-entropy f32-looking pairs); u16 offset tables at
  0x1340..0x1b40 (per-key offsets, ~0x11-0x30 step) - consistent
  with 8-9B elements at variable key counts.
- Constant-fill regions 0xffc0/0xffd3 = sentinel values
  (probably static-track defaults).
- live binding-record u32s (0xb2bd82ce..) are NOT namehash() of
  any clip name in the corpus - different id space (possibly
  bone names or runtime-allocated ids).

Format hypothesis to test next: bank = {clip dir (E),
per-clip track tables (C/D), shared element pool}. Element = the
~9B packed rotation/pos key - decoding it vs a dumped grid is
now a concrete offline problem: benchmark4's 16 clips are small
and its names are known.

r22b (cont): key-mass = u32 element-offset table @0x3f40, 1324
monotonic entries (0x1b1..0xadf5, diffs ~33-34 ELEMENTS), u16
sub-tables @0x1340..0x1b40 (per-key byte-offsets, first byte ~
slowly-increasing = likely frame#/track# index, second = offset),
f32-looking quantized mass at 0x1f40..0x2f40 + 0x3740..0x3f40.
benchmark4 is a 16-clip micro-bank (26KB) - ideal format lab;
our big grids come from larger banks but the format is shared.

r22c: deeper layout in C region - 12 identical 19B 'AN' records
@0xd18 {'AN',0,60000,0x030000ff,zeros[7]} then a variant
0x040000ff record with {0x9790 size, 0xc350 offset} -> f32 mass
@0xe44+ (pairs like 20071/730/6677 - quantized?). Corpus-wide
namehash dict (45604 names -> target/tmp/clipnames.json);
binding ids still unmatched (not clip names).
Element tables: 2 chunks (1324 + 464 entries) covering element
indices 433..60000 = one logical table split per group;
60000 sentinel = cap. 1788 entries ~= frames total (16 clips
share pool).

## Round 22d - table semantics cracked (benchmark4)

- u32 table @0x3f40 (1324 entries) + @0x53f0 (464): values are
  KEY TIMES in ms - diffs 33/34 = 33.3ms = 30fps frame grid.
  Clip A ~44.1s, clip B ~15.5s. 'AN' 19B records' u32 60000 =
  60s duration field.
- Second index family (u16 pairs {flag,val} / BE-u32s at
  0x4d2a+, 0x1340+): values up to ~0x75xx = ELEMENT-POOL
  indices (cap 60000). Per-key diffs 33-34 elements ->
  ~33 elements decoded per key = per-key bone elements,
  frame-major. Matches live grids ({pos4,quat4} frame-major).
- Element pool is the decoded output buffer (~60000 elements
  ~= 240KB of f32 data) - the file stores a compressed stream
  that expands to it; per-key the table says where that key's
  33-34 elements land in the pool.
- Compressed mass regions: 0x1f40..0x2f40 + 0x3740..0x3f40
  (entropy), with 0xffc0/0xffd3 constant-fill = static tracks.
- benchmark4's clips are small prop/UI anims - the captured
  5b/7b grids are strong pairing candidates (need live verify).

r22e: decompression model confirmed - clip records carry
{compressedSize?, decodedSize=0x9790, decodedPoolOff=0xc350}
(pool-relative, exceeds file size). Position tracks = RAW f32
vec3 in MILLIMETERS, u16 ramp index @0xe3c steps 0x0c=12B ->
keys at 0xe44+0x1c. benchmark4's pos keys decode to a smooth
camera path ((5.09,1.02,23.29)m etc - benchmark flythrough rig).
So ANIX stores: plaintext clip names (namehash dir) + per-track
f32 vec3 pos keys + packed rotation elements + decompress-block
map -> decoded pool. Rotation elements remain packed.

## Round 23 - plaintext-ciphertext PAIR secured; 16B record grid mapped

### The pair (VERIFIED-INGAME, capture_round6)

- `scan_anix.lua` (target/tmp/capture_round6/) found mob members whose
  `member+0x78` anim-source embeds the bank path cstr:
  `characters\animationcharacters\orcs\orc_sync_chord\
   orc_sync_infightkill_victim.anix` (19 members on ctl2 + 3 on ctl5).
- For each member it dumped `fb` = ctx slot-array blob
  (`sw+0x10+sidx*8 -> slot -> buf`) => `c2mN_frames.bin` (256KB each)
  + `c2mN_slot.bin` + `c2mN_dst.bin` + manifest.jsonl.
- `c2m0_frames.bin` decodes CLEAN: 8192 records of
  `{pos4, quat4}` f32, FRAME-MAJOR, bone 0 = {0,0,0 / 0,0,0,1}
  identity every 20 records => **20 bones x 409 frames** (tail is
  truncated mid-frame). Every quat unit-norm; adjacent-frame
  |dot| ~0.99. This is the decoded multipose grid = the ciphertext
  half of the pair; the .anix file (1.41MB, 27 CLIP_SENTINEL
  blocks) is the plaintext side.

### Clip block layout (VERIFIED-STATIC, this bank = doc's variant B)

Per sentinel: 16B header u16s `{2, X, nrec, 0, 0, Y, 0, 0}` where
X = total key count incl terminator (e.g. clip21: 257 = 243+7),
Y = flag byte echoed (0x21->33, 0x63->99). Then nrec x 8B records
`{u16 n_keys, u16 data_off, u16 flags, u16 cum}` (cum chains;
terminator has flags=0x10, doff=0). Then a small u16 sub-table
(5 entries: {2156,2524,4596,5036,5500} on clip21), then the
SHARED KEY-TIME INDEX: u16 entries {lo=frame, hi=sub-frame/256}
sorted ascending, ~178 entries for clip21 (resets mid-array where
a second table begins).

### Per-frame record grid = the rotation codec carrier

Each clip's data region contains rows of 19 x 16B records
separated by 8B markers `fe df ff 03 ff ef ff fd` (stride
0x130 = 304 = marker + 19x15.6B? - records at +8 then 19x16B
lands the NEXT marker inside rec18's tail => exact row =
8B marker + 18x16B recs + partial? NOT fully pinned).
Record fields (doc r8 semantics hold): {s32 d0=rol8 comp A,
s32 d1=rol24 comp B, u16 C, u16 D, u32 E, u16 F, u16 sel}
- all A,B in [-1,1], sign-coherent pair (previously verified).
- E low/high halves and F look f16-like in places (0x39f4->1.27,
  0xcfc5...) but NO 4-field combination of {A,B,C,D,E-halves,F}
  as f16/u16-normalized gives unit-norm quats (tested 1566 recs
  on clip1) -> remaining components are NOT plain f16/u16 in
  those fields, or A/B aren't literal quat comps.

### Negative results this round

- c2m0 grid pos keys do NOT appear verbatim (nonzero) anywhere in
  the 1.4MB bank; quats neither - grid is a resampled surface,
  pos keys are quantized/packed in-file (interpolation also
  mutates values at fractional key times).
- No contiguous s16x4 or f16x4 unit-quat runs exist in the file
  (whole-file scan, both phases): rotation keys are NOT stored
  as naive 8B quat elements in this bank.
- Whole-quat / smallest-three / 2-comp s16 verbatim searches: 0.
- rol8/rol24 pair verbatim search vs grid quats: 0 (expected -
  resample interpolation).

### What pins the codec next

The 16B record is now THE decode target: we have ~19 recs/frame
row matching ~20 grid bones, grid ground truth quats, and two
verified components (A,B). Missing: which fields carry the other
two components and the bit semantics of C,D,E,F. Strongest route:
live disasm of the record->quat decode fn (watch the grid buffer
writer during bind) OR enumerate field-wise correlation: for a
record whose grid quat is known, brute which byte-fields
reconstruct (x,y,z,w) - the pair (A,B) covers 2; test whether
C,D map linearly to comps 3,4 under scaling.

### r23b - record-row layout corrected; grid frame-count anomaly

- Marker rows: `fe df ff 03 ff ef ff fd` recur at EXACT 304B
  stride inside clip data regions (86 consecutive on clip1).
  Row = {8B marker, 18 x 16B records, 8B trailer} - the "next
  marker inside rec18" artefact resolves to trailer+marker.
- All decoded (A,B)=rol8/rol24 values are trivially in [-1,1]
  (s32/2^31 range) - plausibility scoring on that basis is
  VACUOUS; don't reuse it.
- GRID ANOMALY: c2m0's grid = 409 frames of one smooth clip, but
  every clip in the paired bank is 93-186 frames (ms tables end
  3.1-6.2s). Either grid frames != 33ms (e.g. resample ~72Hz =>
  5666ms), or c2m0's fb slot holds a different bank's clip than
  its asrc path implies. RESOLUTION NEEDED: member cursors
  (cur80/cur84) or dst-pose xform should pin the actual clip
  + time; don't assume c2m0 == clip21.
- exe search: marker constants NOT immediates in code
  (0 hits for fe df ff 03); the `fd ff ef ff` hits are LUT
  data inside ".bss a" blob (file 0x55ea212 = u16 cluster
  ~0xffxx - possible quantization/residual table; also second
  hit at 0x5eac58d inside xor-packed-looking block).

### r23c - dst=grid EXACT match; live state blockers

- **Ground truth locked**: `c2m0_dst.bin` (712B) = 20-bone current
  pose at 32B stride (quat at +48). All 20 quats match grid
  frame 70 EXACTLY (q<->-q) => `fb` buffer IS the member's active
  pose grid, and dst is its sampler output. cur80=6122ms cursor >
  every clip in the paired bank (max 6233 fits clip12; still
  unresolved which clip's 409 frames map).
- Live probe this session (windowed, benchmark mob battle):
  foreground stealing needed AttachThreadInput trick (AddToSet
  `refg()` helper works); ModMenu eats keys but PostMessage does
  NOT reach it - must have real foreground for keybd_event.
  - Members present (ctl census OK) but this state has:
    asrc with NO embedded .anix path, member+0x60 (dst wrapper) = 0
    (GPUMemoryOnly), ctx+0x140 sw chain empty - capture_round6's
    conditions (paths in asrc, dst wrappers live) were a different
    world/anim state. Moral: the pair capture was state-sensitive;
    reproduce the same state (BM_Island world load, mobs settled)
    before re-running scan_anix.
- Codec-field brute force concluded negative: no combination
  {A,B}+{C,D,E,F} fields under s16/u16/f16 gives unit quats on
  1566 records - the 16B record is NOT a plain 4-field quat.
  Next decode lead: watchpoint on a fresh member's grid at bind
  (spawn) OR find decoder via 'ANIX'/strdb-name xref in exe.

### r24 - ANIX stream loader located in exe; codec fit still open

- **Loader found**: `ShadowOfWar.exe` VA `0x140480aeb` (file
  0x4800eb; PE `.arch` section => rva = fileoff + 0xa00). Code:
  reads 0x30 hdr via stream vtbl+8, checks `'ANIX'` + ver 0x2c,
  allocs 0x28 descriptor, then loops reading the
  decompression-block table: per record reads `{u32 outer, u32
  count, count x u32}` and stores pairs `{inner_u32, outer}` into
  a table indexed cumulatively until hdr count exhausted. Then
  skip/alloc-copy two sections verbatim (sizes from hdr fields).
  => the .anix stream reader only SECTION-COPIES; track/rot decode
  happens later at clip bind, elsewhere.
- Loader has ZERO static xrefs (no imm64 ptr, no rip-rel lea in
  whole image) => reached via runtime-built registration; upward
  tracing from it is a dead end statically.
- Reader stream-object vtable `0x141db5dd0` instantiated at
  0x140480cc8 and 0x1411b5da4.
- Negatives added: s16x4/u16x4 quats at offsets 0..8 in the 16B
  record (unit frac <=2.2%); knot-frame quats tolerant-s16x4 file
  search: 0 hits; marker-row->grid-frame correlation of (A,B)
  vs |quat| comps across phase/stride/bone-map sweeps: ~1% =
  noise. c2m0_slot.bin shares only coincidental constant blocks
  with the file; its 428 s16-unit-quat hits are scattered noise.
- DECISION: offline byte-fitting is closed until row->(key time,
  bone) mapping exists. Decisive route = exec-watchpoint on
  `0x140480aeb`?? (fires on LOAD not decode) - better: write-watch
  a live dst/grid buffer during a forced rebind in BM_Island
  world state (LoadWorld(DBWorldInfo['BM_Island']) reproduced
  round6 conditions), then disasm the writer.

### r25 - pose sampler fully disassembled (MobController.Update)

Live state: BM_Island loaded via pipe `=System:LoadWorld(DBWorldInfo['BM_Island'])`
(game-VM eval; works at title, no menu input needed). MobController
registry 0x142a25ca8/60 holds 6 controllers; cnt=20 ctl members at
0x237e3b360 all share ctx 0xe59af810 (vt 0x141dcc088, nb=20) with
independent cursors.

**MobController.Update = 0x14180deb8** (vtable 0x141dcdd88 slot 0x90):
- iterates members [ctl+0x138], count [ctl+0x140], stride 0x88.
- member+0x78 = asrc record (stride ~0x98): `+4` duration ms,
  `+8` frame count, `+0xc` ELEMENT BASE OFFSET into shared grid
  (verified: m0 13166ms/211f base0; m1,m3 share clip 26066/418 base
  436; m2 18633/299 base 2078).
- member+0x80 = ms cursor, wraps mod duration; +0x84 = frame index
  = floor(cursor/duration*frames) (verified 9063->145).
- member+0x60 = refcounted dst-pose wrapper (allocated lazily via
  global-obj vtbl+0x58 when grid path active).
- GRID PATH (ctx+0x140 != 0): base = [ctx+0x140]->vtbl[0]();
  cur = base + (frame*nb + base_off)*32, nxt = +32*nb;
  per bone: quat nlerp fn 0x140210078 (sign-aware), pos lerp;
  writes to dst wrapper target. Grid = frame-major 32B
  {pos4,quat4} rows. => dst pose = interpolation between two
  ALREADY-DECODED grid frames. Codec runs earlier, at bind.
- NO-GRID PATH (ctx+0x140 == 0, current live state):
  member+0x58 = anim player obj (vt 0x141dc47b8):
    vtbl+0x58 = request element-range job {frame0,frame1},
    vtbl+0x18 = sample -> 0x1403094fc thunk -> 0x141154b34
      builds async job {exec 0x140177d60, cb 0x140177d7c, arg,
      frac, id}; 0x140177d7c pushes a tag-5 command into a
      lock-free stream (0x140177e24 walker, tag nibble = cmd>>28,
      size LUT 0x140178050: {10,c,10,4,4,4,...} bytes).
  => on-demand sampling is ASYNC through the job/stream system;
     the actual element decoder is a stream consumer.
- member+0x50 = channel obj vt 0x141ffe3e8; ctx+0x160 -> binding
  id array ({u32 id, u32 0x10} entries e.g. 0x10b2bd82ce..);
  ctx+0x168 -> f32 block; ctx+0x68 region embeds mesh path.
- bm clip frames are ~62.4 ms apart (16 fps) - NOT 30fps. The
  round6 409-frame grid at 32.2ms remains unexplained (either a
  different bank clip, or grid stride isn't frames there).

### r25b - live watchpoint trial; game down

- `sdk.debug.watchx(0x140177d60)` works and hits carry full regs
  (rax=stream buf, rdx=job ctx 0x14272e9e0, rcx=global
  0x14272e8d8). BUT the fn is a hot generic job pump: ~2.6M hits
  in seconds and the game died (watchpoint storm; exception
  0x80000004 logged at the site). LESSON: never exec-watch hot
  dispatchers; pick leaf fns or gate by args.
- Job ctx: rax = ptr to stream cursor (buffers 0xae/0xaf/0xb0
  pools), so the tag-5 sample commands land in a shared stream
  that a WORKER drains - the element decoder is the stream
  consumer, not yet located (walker code at 0x140177e24 is the
  WRITE side; reader dispatches per-tag-nibble handlers incl.
  0x140702b48 for tags e/f, 0x140698f58 tag1).
- Next live: instead of exec on hot fns, WRITE-watch a member's
  asrc-adjacent fields? Better: catch the grid FILL at bind -
  force a fresh MobController bind (spawn) with a write-watch on
  the ctx grid region, or exec-watch the stream drain fn once
  identified by disasm.

### r26 - anix bank verbatim in memory; parsed clip region found

- `sdk.debug.scanmem` works on raw-byte patterns. The
  `orc_sync_infightkill_victim.anix` bank is RESIDENT VERBATIM:
  needle records at `0x2b84e7094`+ (marker-prefixed, contiguous =
  file image) and marker sigs at `0x15d10xxx`/`0x15d28xxx`.
  pool_c.bin (dumped) confirms: mem +0x1d50 == file 0x324 etc.,
  1596 sequential record matches.
- Two other copies at `0x12d8dc050` / `0x2b64c9478` are REORDERED
  (needles at non-file deltas) - candidate element pools or
  per-clip copies; pool_a/b.bin dumped for offline diff.
- Parsed clip region `0x15d28xxx` (mark_a.bin): contains
  CLIP_SENTINEL 0x2b99dbc1 inside records like
  `{u32 id, sentinel, u16 2, u16 8, u32 1, u32..., u16 tab}` -
  the bank's clip blocks re-materialized as parsed objects with
  id 0xebf4d910-style hashes preceding each sentinel.
- Read-watchpoints on all three record copies during live
  benchmark playback: ZERO hits => active clips are not re-read
  per frame (elements already decoded to pools at bind), or
  those copies belong to non-playing clips.
- Dumped artifacts: target/tmp/pool_[abc].bin (64KB each),
  mark_[ab].bin (32KB), scan61.txt.

### r26b - pool dump offline diff

- pool_c (0x2b84e2f84): byte-exact verbatim bank image (8KB
  window identical). Dead end for decode; it is the raw copy.
- pool_b (0x2b64c5478): REORDERED. Needle record present
  verbatim but neighbors diverge. Bulk of region = 0x40-stride
  struct array embedding a repeated constant that verbatim-matches
  file 0xb6d+ region (a shared sub-blob, likely a hash table);
  a few true record copies at irregular offsets
  (m3c80=f2704, m4000=f26e4, m4800=f26f4).
- pool_a (0x12d8d8050): VOLATILE - matched the file needle at
  scanmem time but contents changed before dump; now contains no
  record bytes. Prime suspect for the live decoded element
  pool / decode scratch buffer. Next live session: re-scan
  needles, then dump pool_a twice a few seconds apart - if it
  cycles, it is the decode output buffer.

### r26c - file 0xb60 = per-track {count,offset} table; pool_b parsed twin

- File region just before CLIP_SENTINEL at 0xb75 is a table of
  6-byte records {u16 count, u16 lo?, u32 off}: offs climb
  ~+0x100 monotonically = per-track key-stream offsets; counts
  0x0c-0x85 = per-track key counts. Clip block thus =
  {track table, sentinel, prologue, key data}.
- pool_b 0x40-stride records = the PARSED form of these
  entries: +0x10/+0x18 vary (offsets into key blob), +0x20
  carries a 0x52-tagged field (pointer into resident bank?),
  +0x28 = {0x20, 0x1} const. The repeated file-0xb6d window in
  pool_b is this same table copied into each record.
- pool_a remains the volatile suspect (contents changed between
  scanmem and dump ~seconds apart) = live decode scratch or the
  resident decoded element pool. Double-dump A/B test queued.

### r28 - ANIX header fully decoded; bone-name hash table found

File layout for orc_sync_infightkill_victim.anix (and generic):
- 0x00-0x30: ANIX header (magic, ver 0x2c, section offsets).
- 0x30-0x2b0: TWO consecutive copies of an ~82-entry u32
  NAMEHASH TABLE (copy at 0x170 is the same list shifted by 2
  entries - really one table + variants). Entries are engine
  namehash() (h=h*0x397+fold[c], case-folded) of BONE NAMES.
  Resolved via tools/bvr.py:namehash against a name dictionary:
  pelvis, head, neck, attach, l_/r_ thigh, calf-ish, foot, toe,
  shoulder, forearm, hand, wrist, index1/2, middle1/2, ring1/2,
  pinky1/2, thumb1/2, weapon, shield_l ... - a ~40-bone
  humanoid rig. THIS is the track->bone binding table.
- 0x2b0-0x2bc: clip-block prologue, CLIP_SENTINEL at 0x2bc.
- Right after sentinel: dense packed-s16 rotation data
  (repeating f7fc/7fdf = s16 -2052/+32735 ~ (-0.063,+0.999)
  unit-quat-like pairs) then marker-delimited 16B record rows.
- Region 0xa80-0xb60 = 28B records {u16 0,u16 0xff00,
  12B payload, 12B payload dup} - duplicated vec3-ish data
  (f32 triples like 5.84,-0.15,-0.07) inside the clip block.

### r28b - record internals; bind-pose table; negatives

- Records' u32 lanes show marker-like constants only in slots
  0 (lane2 = 0x?ffdffe) and 10 (lane3 = 0x?ffefff) - the rest
  scatter => the 16B "records" are windows into a BIT-PACKED
  stream, not fixed-field structs. That explains why every
  aligned-field decode attempt fails. Row stride stays 304B =
  8B sync + 288B packed data + 8B tail.
- Marker bytes do NOT appear as code immediates in the exe
  (0 hits in .arch/.xtext) - decoder tests masked/bitwise forms.
- Region 0x2c0..0xb60 = 57 records {0000 ff00, vec3 f32, dup}:
  symmetric L/R pairs (+-10.64 etc.) = likely the clip's
  BIND-POSE translation table (~40 bones -> ~57 entries with
  some multi-offset bones). Magnitudes match grid pos but no
  exact frame match (rest pose != animated pose).
- Rotated-3-component quat fit on records: best ~57% pass
  (noise). Codec decode still requires the decoder fn.

### r29 - LIVE: bound track->bone tables found in player objects

- Region 0x2f52e8000 (this boot) is a VOLATILE 0x540-stride
  struct array - written ~6.7KB/8s even at idle: the live
  animation-player/member state blocks (self-pointers inside).
- member+0x58 (player, vt 0x141dc47b8) embeds THE BINDING:
  at pl+0x50.. an array of {u32 trackHash, u32 0x00ce66fc}
  pairs - 0x00ce66fc = the bank's table entry[0], so each pair
  binds a bank-table hash to a track slot. Some players instead
  hold {hash, u32 index} = hash->track-index map (m107 layout).
- Captured to target/tmp/bindings.txt: e.g. ctl a974e60 m23/24/
  59/96 show ~80-entry lists in FILE-TABLE ORDER (937a36,37,38,
  2ad7ba8, fef266e, 182e18cf...) interleaved with extra hashes
  (Cam01/Cam02 resolved; others = numbered series - other
  tracks or attachment/helper bones).
- Net: pl+0x50 binding order == the order record-rows/table
  entries are consumed => record slot i in a marker row maps to
  bone = bindings[i]. This is the row->bone key needed to
  correlate the bit-packed records with named grid bones.

### r30 - CLIP BLOCK LAYOUT FULLY DECODED

Clip block (after CLIP_SENTINEL 0x2b99dbc1):
  {sentinel, 12B prologue {02 00, u16 ntrk?, u16 ?, 0,0,
   u16 0x63?, 0,0}, TRACK TABLE, key-time blob, marker rows}

Track table = 8B records {u16 nKeys, u16 blobOff, u16 rsv,
u16 elemStart}; VERIFIED elemStart = cumulative sum of prior
nKeys (0,77,89,159,184,220,244,256,328,343,417,432,444...).
Block1 (0xb90) has 12+ tracks; blobOff runs 0x85->0x11d6 into
a packed key blob (~8B/key effective => bit-packed).

UNIFIED MODEL: the marker-delimited rows ARE the shared
element pool - each row = {8B sync 0x03ffdffe 0xfdffefff,
18 x 16B PACKED ELEMENTS, 8B tail}, element index E =
row[E/18].rec[E%18]. Track elemStart picks the first element;
nKeys elements follow; the key blob at blobOff holds per-key
times (u16 {frame,subframe} idx - matches prior finding).
Block0 (0x2bc) = BIND POSE block (s16 consts + vec3 table +
1 marker row). Remaining unknown = the 16B element packing
(bit-packed; rot+pos per element).

### r30b - element correlation negatives

- Marker-row records show NO byte/nibble/s16-lane smoothness
  between consecutive rows or slots (XOR popcount ~63/128,
  s16 lane |d| ~9-22k): elements are fully bit-packed per key
  (no byte-aligned fields), possibly delta/reference-coded.
- Element-index model: track {elemStart,nKeys} selects from
  the shared element index space; whether elements are the
  marker-row 16B records or the rows are the key schedule is
  still open (87 rows vs 451 elems in block1 - neither
  elements-per-row nor keys-per-row matches trivially).
- blobOff anchor = block start (0xb90): key blob entries look
  like {u16 frame, u16 extra} ramps (e.g. at 0x1d66: idx 9..16
  paired w/ 0x0956,0x0b28,0x0b81,0x0c99 = element offsets?).

### r31 - LOAD-TIME BANK READERS MAPPED (write/watch on file image)

Watchpoint on a verbatim bank needle (0x11f901590) during
`System:LoadWorld(DBWorldInfo['BM_Island'])` produced real
hits. Census of in-engine readers at bind/load time:

- 0x1405bde8e / 0x1405bdea1 / 0x1405bdd52 - bulk copy loop
  (rbx=src bank ptr, rcx=1 stride, rdx=byte count ~0x11cc0);
  identical loop at 0x7ffd9xxx = ntdll memcpy-ish. These are
  the loader's verbatim section copies, not decode.
- 0x140183559 (stream cluster) - index-remap pass over
  16B {ptr,u32count} records: reads u32 index arrays,
  remaps each through a -1-sentinel u32 LUT
  (`cmp [rsi+r8*4],-1; cmovne`), writes remapped u32s.
  = track/binding index fixup at instantiate.
- 0x14015fa40 - key-time lower_bound: `comiss xmm0,
  [f32arr + idx*4]` scanning u32 index array vs time in
  [r15+0x50]. Stack chain +0x15f607 -> +0x114af67 ->
  +0x22dc840 -> +0x114aef0 -> +0x797e0d. Candidate:
  per-track key-time bracketing (sample path).

Also resolved: the loop found earlier at 0x140180a40 calls
per-item fns 0x140180a90 (= stream cursor advance over
{tagword,size} records, tag=top4bits, stride LUT at
0x14202e4d0) and 0x140180c90 - still plumbing, not the
element decoder.

Method note: `sdk.debug.watch` is a READ+WRITE watch (8B
aligned qword). arming fails (false) if the page isn't
committed - volatile regions move across world loads, so
re-scan needles each session. Volatile block this session:
needle cluster 0x1683a1xxx (0x540-stride live state);
verbatim image 0x11f901xxx; stable copy 0x168037xxx.
BM_Island reload = `=System:LoadWorld(DBWorldInfo
['BM_Island'])` from results screen - rebinds clips.

NEXT: the decoder is still uncaught. Elements decode at
bind via the async job path (0x141154b34 submitter).
Remaining good moves: (a) exec-watch the cursor-advance
neighborhood 0x140180a90/0x140180c90 during LoadWorld to
see if it processes tag-5 element commands; (b) watch the
player-object binding array write (pl+0x50) - the bind fn
itself writes those pairs; (c) disasm callers of stack
frame +0x15f607 (key-bracketing caller chain up =
+0x114af67 -> +0x22dc840 -> +0x114aef0 -> +0x797e0d).

### r32 - STREAM DISPATCHER MAPPED (exec-watch negatives + captures)

0x140180c90 = hot generic u16-tag dispatcher over stream
command objects (~4.3M hits during boot benchmark; exec-watch
on it crashed the game AGAIN - hard rule confirmed: never
exec-watch anything on the stream path).

Captured dispatch table (u16 tag at [rcx], rcx=cmd obj,
rdx=rdi=0x14273c628 = THE STREAM OBJECT (sibling field of
0x14272e8d8 global)):
  tag1 -> 0x141cada24 (rdx=[rdx] deref first)
  tag2 -> 0x14033d814 (r8=[rcx+8])
  tag3 -> 0x1407489d0 (edx=u16 [rcx+2] payload)
  tag4 -> 0x14017c514
rbx in hits = tag value (observed 2..6?? - likely size/tag
field). Caller: enclosing loop 0x140180965 (the r30 per-item
loop), above it +0x55b64d, +0x3058d8, +0x39ef68.

Enclosing fn 0x1401808xx-0x180axx: iterates an array of
{lo,hi} bound pairs; per item calls 0x140180a90 (cursor
advance over {tagword,size} records via tag LUT
0x14202e4d0) then this dispatcher - i.e. this is the
stream FLUSH/index routine, not the decoder.

Also: watchpoint crash artifact = VEH exception 0x80000004
(single-step) at the watched VA -> the DR exec bp fires
faster than the handler drains under load. Same failure
class as the 0x140177d60 pump watch (2.6M hits, crash).

NEXT viable moves (ordered, cheapest first):
  a) static: disasm the 4 dispatch targets (0x1407489d0,
     0x141cada24, 0x14033d814, 0x14017c514) - one is the
     element decoder; tag3 carries a u16 payload arg which
     smells like an element index.
  b) live: write-watch the player-object binding pair array
     (pl+0x50) - bind-time single write, narrow.
  c) live: read-watch a bank needle during LoadWorld ALREADY
     WORKS - the readers were copy/remap only; but watching a
     needle that is INSIDE a specific clip's element range
     during a forced rebind of that clip may land the decoder.

### r32b - sample-path hop mapped

Caller of key-bracket fn (ret +0x15f607): wrapper at
0x14015f5e0 calls bracket 0x14015f718, then on success
0x1407d6aa0 = element-fetch thunk chain:
  0x1407d6aa0: rcx=[rcx+0x10]; call [0x141d92208] (IAT slot)
              -> then jmp 0x140160ae8: jmp [0x141d924a0]
Both targets are runtime-resolved function-pointer slots -
RESOLVE LIVE: read_u64(0x141d92208), read_u64(0x141d924a0)
next session to land the real fetch/decode VAs. Likely the
per-clip vtable sampler (grid-nlerp path or async path).

tag4 handler resolved: 0x14017c514 -> 0x14114b9cc =
slot-getter `return obj->slots[obj->n]` (helper, dead end).

STATUS: every layer of the async anim pipeline is mapped
except the element bit-unpacker itself. The fetch thunk
chain above is the sharpest unresolvable-statically edge;
one live read_u64 resolves it.

### r33 - sample-cmd layout decoded; spawn-watch negative; entropy hypothesis

Sample stream command layout (from 0x141154b34 + trampolines):
  cmd = 0x20B { +0x00 fn=trampoline 0x140177d60,
              +0x08 channel arg,
              +0x10 fn2=completer 0x140177d7c,
              +0x18 u32 element index,
              +0x1c f32 frac }
Worker trampoline 0x140177d60: cursor += 0x20, then
calls cmd[+0x10](rcx=channel, edx=index, xmm2=frac).
Completer 0x140177d7c: re-enqueues `index|0x50000000` (tag5)
into the channel's OWN embedded ring at channel+0x00 via
0x140177e24 (generic ring insert; tags 1/0xe/0xf clear a
slot via 0x140702b48), then writes frac into the returned
slot + bumps a TLS counter (gs:[0x58]+0x140) - a per-channel
"latest requested frac" dedup table. So the channel ring
carries completion/dedup records; the anim-side drainer
(ring at channel+0) still unidentified.

NEGATIVE: read-watch on a bank needle inside element rows
(0x2d8d889d4) + sdk.spawn.summon('Summon_OrcDefenderTough')
=> ZERO hits. Element records aren't re-read on spawn-time
bind either. Combined with r26 (zero reads during playback):
the bank bytes are touched ONLY at LoadWorld parse (copy,
index remap, key bracket) - the element payload is either
decoded eagerly at load into an intermediate form, or is
entropy-coded data consumed by a reader we haven't caught.

STRONG HYPOTHESIS: each marker row's 288B payload is a
block-entropy-coded chunk (rANS/range-coder style), not 18
independent 16B records. Evidence: no byte/s16/nibble lane
smoothness, sync constants only at slots 0/10 of each row,
~128 bits per element average. If true, offline decoding is
impossible without the coder tables -> the decode function
is MANDATORY. Where it runs: at load/parse time (reads of
the bank are confined to the LoadWorld window).

### r33b - registration table + iterator mapped (static ceiling)

Caller chain decode: 0x1422dc840 (.xtext) = a pointer table
(entries 0x1413e8d08, 0x1406003xx, 0x140130xxx, 0x1407exxxx)
+ "Stream is not ok" log text = a STREAM-READER vtable,
registered with global factory 0x142a86808 (vtable slots
+0x18 register / +0x20 finalize). Entries are bump-allocator
cursor ops (align16 advance over {cursor+0x18, limit+0x10,
end+0x20}) - reader plumbing.

Iterator fn at ~0x14114aed0..0x14114afa2: registers the
reader vtable (tag 3), then loops: 0x1401a9f7c(entry+8) +
bracket wrapper 0x14015f5d4 per record. Generic
record-deserialization walker.

The ANIX descriptor ctor (0x140480f5c) just zeroes 4 qwords -
descriptor is passive {ptr,ptr,ptr,ptr} data; consumers are
runtime-registered. STATIC TRACE ENDED: the element decoder
is behind runtime registration; no further xrefs exist.

Remaining live moves (all cold-path, no flooding):
  a) exec-watch 0x14015f5d4 (bracket wrapper, ~once per
     track per clip) during LoadWorld -> captures which
     records get processed and the f32 key-array base.
  b) exec-watch the bracket inner 0x14015f718 same window.
  c) read-watch a needle inside the KEY BLOB (not element
     rows) during LoadWorld - if the blob is read at parse,
     the blob decoder lands; elements may decode lazily.

### r34 - bracket fns are the per-frame STREAM SAMPLER

Exec-watch on 0x14015f5d4 + 0x14015f718 during live
benchmark (BM_Island, 102fps): paired hits ~per frame on ONE
object 0x551dd8e8 = a SEEKABLE-STREAM wrapper (first field
0x1422c39a0 = descriptor block containing "Need a seekable
stream here. Not having a seekable stream will cause memory
corruption" guard text).

Object layout: +0x08/0x10/0x18 = counts (0x12a8,0x15a0,
0x1598), +0x20..0x40 = buffer ptrs incl +0x40 -> 0x4dc05280
(f32 array - vec4-ish values, NOT monotonic times).
Bracket call args: rcx=stream+0x30, rdx=r8=0x10280,
r9=0x28e000 (offset/limit pair?). This is the generic
seekable-stream keyed-sampler the flythrough camera uses -
the SAME machinery family as ANIX sampling.

Class vtable-ish block at 0x1422c39a0: +0x08 = 0x1411e1e74
(dtor), +0x68 = 0x1413c2a78 (type-init), +0x70 =
0x1411e2228 (flags/size fn), +0x78 = 0x14012f0f4 (release).
Method slots are refcount/type plumbing - the real read ops
must be in other slots or the descriptor.

NEXT: exec-watch the "read at position" entry - the
sampler's element fetch goes through the stream read fn;
alternatively the stream object +0x20/+0x30/+0x40 ptr chain
maps which buffer holds current decoded data.

### r35 - ELEMENT FETCH + OUTPUT STAGE FOUND (live watch hit)

Write-watch on the volatile region (0x3547fbb00) during live
benchmark landed the ACCESSORS of that buffer - it is the
LIVE ELEMENT ARRAY (holder+0x10, 8B slots):

  0x1401cb3b0 / 0x140278758  getElement(obj, u8 sel, u32 i):
    rec = obj[+8] + sel*16   (16B track records:
        u16 cap @ +0xc == nKeys, u16 base @ +0xe == elemStart)
    if i < cap: return qword/u32 of elemArr[base+i]

CONFIRMED LIVE: the parsed in-memory track table is 16B
records ending {u16 nKeys, u16 elemStart} - matches the
file's 8B {pad,elemStart,nKeys,blobOff} table (parsed form
reorders fields). Elements are 8-BYTE slots (ptr or packed).

Consumer chain (stack +0x223cb9 +0x67f182 +0x220d70 +
0x67f06d +0x221555 +0x220d30 +0x262c703 +0x28fcd8
+0x220c88 +0x2240d9): lands in vec4 interpolation at
0x140223b00-0x140223b59: reads prev output [rdx+0..0xc],
blends xmm7/xmm8*deltas scaled by frac, writes vec4 back -
the classic `out = base + delta*t` application AFTER unpack.

THE DECODER sits between getElement's return (r9) and the
0x140223bxx vec4 writes - look for shift/mask ops on an 8B
element in the enclosing fn (~0x140223a14 helper and the
frame at +0x220d70 callers: 0x140223xxx body above 0x223b00).

Also: earlier hit rip=0x140189944 (rax=0xa) and
0x1447e6f7a (0x1447xxxx = different module region?) during
the same watch - secondary accessors worth a look.

### r36 - LIVE ELEMENT ARRAY CONTAINS s16x4 QUATS (verified vs grid)

Analysis of the volatile dump xB_3547f7c90 (the buffer the
getElement accessors were caught reading):

- 257 aligned s16x4 unit-norm vectors; 5 match captured
  round-6 grid quats within s16 quantization (min 0.0099).
  => the element array holds DECODED rotation keys as
  4xs16 quats (not f32 - compact runtime form).
- Region shows 0x20-STRIDE periodicity: a near-constant
  s16x4 (~(-0.40,-0.75,0.34,-1.0) normalized) every 32B -
  consistent with 32B-per-element {half1, quat_s16x4}
  records where the second half is the rotation key.
- Source NOT in this bank verbatim: the same s16x4 quats do
  not appear in orc_sync_infightkill_victim.anix (any
  alignment/perm, tol 12) - either different bank or values
  are renormalized on decode.
- The FILE also stores dense s16 data: 36k unit s16x4
  sequences at any alignment, 20k at 8B alignment - the
  file's rotation payload is likely s16-quat-based too,
  just inside the bit-packed rows.

MODEL UPDATE: element = 8B slot; the array mixes rotation
elements (s16x4 quat) and probably position/other types.
The 16B file record -> 8B runtime element decode is the
remaining unknown (2:1 unpack: probably 16B {pos s16x3 +
quat packed} -> split into pos elem + rot elem?).

NEXT: find WHICH bank/clip the matched elements come from
- scan ALL .anix in target/tmp/anix_all for the matched s16
  values; when found, that file's 16B record containing
  them reveals the exact file->elem transform.

### r36b - live quats absent from ENTIRE corpus

Scanned all 1666 .anix in target/tmp/anix_all for the 5
live-matched s16x4 quats (any component permutation): ZERO
hits. The runtime rotation elements are TRANSFORMED on
decode - not verbatim s16 copies. Possibilities:
  - file stores quats slightly non-unit; decode renormalizes
  - file stores 3 components + sign/index (smallest-three)
  - live values are post-interp results stored back in a
    compact s16 pose cache (not raw keys)
  - per-clip normalization/dequantization scale

The 0x20-stride near-constant vec4 (-0.40,-0.75,0.34,-1.0)
recurring every 32B in the same buffer suggests a
{position half, quat s16x4} or {data, fixed-mode tag}
layout - the -32768 saturation could be a type marker.

STATUS: runtime element = 8B slot; rotation elements hold
s16x4 unit quats numerically matching live pose grid
values. File-side 16B records remain bit-packed; the decode
transform is now confirmed non-trivial (no verbatim copy).
The unpacker fn (between getElement ret and 0x140223b00
vec4 writes) is the last missing piece.

### r37 - dequant region mapped; stack chains walked

- The lerp site (0x140223b00) scales by 0.5 (const
  0x14202e8f8 = 0.5f) - midpoint/half-angle math, not the
  s16 dequant scale.
- The whole file region .arch 0x2230c0-0x224500 (= VA
  0x140223ac0-0x140224f00) is saturated with cvtsi2ss
  (int->float) - the packed-element dequantization code
  lives in this fn family. The lerp fn itself does
  cvtsi2ss xmm1,rax at 0x140223b11.
- Stack-walked callers of getElement: +0x195aaab lands in
  0x14195aaxx (string/serialization helpers - record
  naming, not sampling); +0x223cb9 = shared-ptr store;
  +0x67f0xx = tagged-handle dispatch. The sample-time
  consumer path is NOT the stack the watchpoint showed -
  the getter is shared by serialize AND sample paths.

NEXT: read a few qwords at the live elemArr to get real
element byte values, then correlate against the 0x223xxx
dequant fn's arithmetic (find which fn converts an 8B
element to vec4). Or exec-watch 0x140223a14's CALLER
during playback (cold enough? it is in the sample path).

### r38 - element expander decoded: type-tagged endpoint writer

The function containing the lerp site (entry ~0x1402238xx,
body 0x140223aa0-0x223c67+) is the per-element EXPANDER.
rcx = element/key descriptor, rdx = 0x20B output pair
{key0,key1} - matching the live array's 0x20-stride records.

Decoded semantics (verified constants):

    tag = u32 [rcx+0x10]
    K   = (tag==3 || tag==5) ? 2.0 : 1.0   (via 0x1410acee3
          thunk that only swaps the const 0x14202e5a4=2.0)
    n   = (f32)(u32 [rcx+0x1c])            cvtsi2ss
    d_y = (f32 [rcx+0x24] - 20.0) * 0.5*K*n
    d_x = (f32 [rcx+0x20] - 20.0) * 0.5*K*n
    out[0].x -= d_x ; out[1].x += d_x    ([rdx], [rdx+8])
    out[0].y -= d_y ; out[1].y += d_y    ([rdx+4],[rdx+0xc])

i.e. element descriptor = {type tag, u32 packed index/count,
f32 base_x, f32 base_y biased by +20.0}; output = symmetric
endpoint pair base +/- halfwidth. This is min/max range
dequantization for translation (bias 20 matches the bind
offsets range +/-12-20). Other branches (0x223bd0+) apply
the same pattern per component with cvtps2pd round-trips
and a helper 0x141040272; tag!={1,3,4,5} additionally calls
0x14066f5f0 with arg 0.5 - likely the rotation pair.

The track descriptor (+0x10/+0x1c/+0x20/+0x24 fields) is the
object the getElement accessor's 8B slot points into, or a
larger record the element selects.

NEXT: find where rcx comes from (fn entry ~0x2238xx region
has data gaps; search backwards for prologue saving xmm6-13
at [rsp-...]), and disasm 0x14066f5f0 - the probable
rotation decode (half-angle 0.5 arg suggests quaternion
slerp/nlerp).

### r38b - the expander's second stage: 0x14066f5f0

    fn(vec4* p /*rcx*/, f32 d /*xmm1*/, u8 flag /*r8b*/):
        if flag: d *= 20.0            // const 0x14202e738
        p.x = H1(p.x - d)
        p.y = H1(p.y - d)
        p.z = H2(p.z + d)
        p.w = H2(p.w + d)

H1 = 0x14104028a, H2 = 0x141040272 - both jump through the
runtime-resolved helper table at 0x141d93xxx (.xtext slots,
not PE imports). Suspected angle wrap/round helpers; the
x,y vs z,w split suggests {xy = one quantity, zw = another}
e.g. {pos.xy vs pos.zw} or {quat.xy vs quat.zw} stored as
separate dequant groups.

So the full element decode for one 0x20B record is:

    expander(0x140223xxx): reads descriptor
        {tag@+0x10, u32 n@+0x1c, f32 fx@+0x20, f32 fy@+0x24},
        writes out = base +/- (f-20)*0.5*K*n per component
        (K=2 for tag 3/5) into a {key0,key1} pair, then
    0x14066f5f0 finishes the vec4 by shifting pairs
        through wrap helpers (x,y via H1 minus; z,w via H2 plus)

INTERPRETATION: the "element" is a quantized DELTA packet -
descriptor stores a float center (biased +20 for positions)
and an integer n that scales a shared delta; key0/key1 are
center -/+ halfstep. Rotation tracks likely carry a parallel
descriptor decoded through a sibling path (tag 3/5 get the
2.0 scale = half-angle quat range).

NEXT: resolve helpers live (read_u64(0x141d93218/0x141d93248)
mid-session names them), or exec-watch 0x14066f5f0 during
playback to see live (descriptor, out) pairs.

### r39 - helpers resolved live: floorf / ceilf

Live read of the .xtext helper slots:
  0x141d93218 -> ucrtbase!0xf8030 = floorf  (H2)
  0x141d93248 -> ucrtbase!0xf8050 = ceilf   (H1)

So 0x14066f5f0(vec4* p, f32 d, flag) is:

    if flag: d *= 20.0
    p = { floorf(p.x - d), floorf(p.y - d),
          ceilf (p.z + d), ceilf (p.w + d) }

i.e. it snaps the stage-1 shifted values to the surrounding
integer lattice - the decoder computes the TWO bracketing
quantized keys a fractional sample falls between. Elements
are quantized integers on a lattice with spacing
d = (f-20)*0.5*K*n / something; floor/ceil give key0/key1.

Confirmed model end-to-end:
  file blob -> parse -> descriptor {tag,n,fx,fy}
    -> stage1: endpoint pair via symmetric delta
    -> stage2: floorf/ceilf lattice snap
    -> lerp(0x140223b00): blend key0/key1 by frac
    -> vec4 out

NEXT: exec-watch 0x14066f5f0 (cold enough? fires once per
sampled element) during playback to capture real descriptor
fields, or find the fn entry of the expander (~0x140223800s)
to see where rcx descriptor comes from.

### r39b - live probing of the expander

- Helper slots resolved live via sdk.debug.read_u64:
  0x141d93218->ucrtbase!floorf, 0x141d93248->ucrtbase!ceilf.
- Fn entry found: 0x140223a88 = expandKeyPair(rcx=desc,
  rdx=outPair). xmm6-13 spill prologue at +0x08..+0x30.
- The "call 0x14066f5f0" at 0x223b82 is actually
  `call [rax+0x10]` (ff 50 10) - an indirect virtual; the
  floorf/ceilf fn is dispatched through a runtime table.
- Neither 0x140223a88 nor 0x14066f5f0 appears as a direct
  call target or vtable entry in the image - reached only
  through runtime-built tables.
- Exec-watch on both during a full benchmark world-load +
  playback: 0-1 hits total. The element decode for the
  benchmark's clips does NOT route through this function.
  The 0x223xxx family is a different variant (perhaps a
  dequant path for a different element/clip class).

REVISED MODEL: the per-frame path reads pre-decoded
{key0,key1} 0x20B records from the live array and lerps;
the actual file->runtime decode runs during world load in a
still-unidentified sibling fn (probably in the same
0x22xxxx/0x23xxxx family - the cvtsi2ss-dense region).

NEXT: during a LoadWorld, exec-watch candidates: the fn
that WRITES the 0x20-stride records (write-watch on a live
elemArr 0x20 record during bind is the sharpest - the
writer's RIP is the decoder).

### r40 - the record deserializer found via needle read-watch

Read-watch on marker-row needle 0x2ce0a6518 during
LoadWorld(BM_Island) fired in-engine at:
  0x14050165d - inside clip reader loop (this fn)
  0x14041e5d4, 0x14041f940, 0x14041ffd3 - sibling record
      walkers (imul rax,0x38 stride)
  plus ucrtbase/ntdll copies

Deserializer 0x1405014xx-0x5018xx (stream r14, obj r15):
  arr0 = alloc(count[rbp-0x3d]) at r15+0x20
  for i: rec = arr[i] (0x140501890, stride 0x38)
    stream.read3xU32(rec+0x10)   ; {tag,?,?}
    stream.read3xU32(rec+0x1c)   ; {n, fx, fy}
    stream.read3xU32(rec+0x28)   ; {?,?,?}
    if flag==2: read_u32
    rec[0],rec[8] = 2 x read_u64
  then arrays at r15+0x30, r15+0x40 ... same pattern

Element record = 0x38B {u64,u64,u32 tag, u32, u32,
  u32 n, f32 fx, f32 fy, u32,u32,u32 [,u32]}.
Field layout matches expander exactly: tag@+0x10,
n@+0x1c, f32 endpoints @+0x20/+0x24.

The primitive reads go through `stream->vtbl[+8](dst,len)` -
a VIRTUAL stream read; the bit-level unpack of the file
format lives inside that stream class (same "seekable
stream" wrapper family as 0x551dd8e8 from r34).

NEXT: exec-watch the deserializer entry (~0x140501440)
during LoadWorld to grab r14's vtable, then read
vtbl[+8] -> the actual bit reader fn.

### r41 - regs captured live: clip object + vtable confirmed

KEY DISCOVERY: watchpoint hits already log full regs +
exe-range stack census to Nemesis.log (count<=4 or %256).
No DLL rebuild needed - the data was there all along.

Deserializer entry hits (0x140501478) during LoadWorld:
  rcx = 0x620af4e0 = clip object; [rcx] = 0x141ff6b98 =
    the .xtext registration table containing 0x140501478
    itself (vt[3]) + 0x14041e564 (vt[4], the 0x38 walker) +
    embedded "Bad code"/"out of bounds" strings - a type
    ops table.
  rdx = 0x1ae6fb58/0x1ae6fd88 = stream-ish objects, but
    their [0] fields (0x1405035c2 code ptr, 0xfcc5dcc8 heap)
    are not classic vtables - the fn resolves the inner
    stream deeper (r14=rdx is not the object that gets
    [rcx]->vtbl[+8]'d in the read helpers; indirection in
    between).

Also confirmed from hit regs: 0x14041e5d4 walker iterates
rcx=0..3 over 0x2ce0a64f0 records (rbx=0xb7c3b178 reader
ctx); 0x14041ffd3 compares [rax+rcx*8] vs r8=0x2ce0a6518
(our needle) - a lookup/probe fn.

NEXT: exec-watch 0x14115dfcc (the u32 stream read) during
LoadWorld; regs will show rcx=real stream -> [rcx]=vtable
-> vtbl[+8]=the bit reader. That closes the last gap.

### r42 - instability; stream capture deferred

- Exec-watch on 0x14115dfcc (u32 stream read) FROZE the
  game - hot primitive, VEH flood. Cleared; do not
  exec-watch leaf readers.
- Title screen ignores PostMessage keys entirely - only
  ctypes SendInput with the game window foregrounded works
  (SetForegroundWindow(0xe7068a) + VK_SPACE). WM_KEY works
  for menus but not the press-any-key gate.
- Armed watchx(0x1405015c9) (the read3xU32 call site -
  rcx there = resolved stream), got past title, but the
  game died in the transition: VEH 0x0000071a at
  KERNELBASE+0xc41ca, stack +0x6d8efa - same signature as
  the earlier boot crash. Both crashes followed a SLOW lua
  'load'/'ui' handler stall (63s watchdog) - plugin-side
  instability, independent of watchpoints.

NEXT (unchanged): on a healthy boot, watchx 0x1405015c9
during LoadWorld; hit regs give rcx=real stream ->
[rcx]=vtable -> vtbl[+8] = bit reader.

### r43 - boot instability = the real blocker

- Third consecutive 0x0000071a crash (KERNELBASE+0xc41ca,
  stack +0x272ea38) during autoload. Pattern: all three
  crashes happened with an exec-watch armed on a function
  inside the clip-parse path during the boot load - the
  per-record hit flood (thousands of single-steps in
  seconds) trips an engine watchdog/abort. Exec-watch on
  parse-path code during load = crash. Confirmed unsafe.
- What did work without crashing: exec-watch on a COLD
  entry point (0x140501478 fired ~4x total). Read
  watchpoints on file needles are safe (r31, r40).
- Stream object first-field points into code (0x1405035c2
  = mid-fn in a sibling reader family 0x1405034xx-0x5037xx
  writing fields at +0x180..0x19c) - so rdx on entry is a
  WRAPPER/iterator, not the leaf stream. 0x140503628 is a
  shared default method used by 13+ tables (generic reader).

REVISED SAFE PLAN: arm the read-watch on a needle (region
b = parsed rows) AND a single exec-watch on a cold fn
deeper in -OR- prefer: statically find the leaf by
following r14 (rdx) through 0x140501478's prologue: the fn
calls 0x1407cb194(rbp-0x49) which builds the real stream.
Disassembling that constructor statically will name the
stream class without another crash-prone live run.

### r44 - the dispatch layer decoded; stream leaf = buffered

Caller anatomy (fn containing 0x140480a3d return site):
- rdi = stream object; [rdi]->vtbl[+0x28] = position/tell
- per record r15 (0x20-stride array at [r12]): r15[0]=u32
  count read from stream, r15[8]=handler obj
- builds ctx on stack: ctx[0]=r9 handler, ctx[8]=rdi stream,
  ctx[0x10]=0; then handler->vtbl[+0x18](handler, &ctx,...)
  = our deserializer 0x140501478 (confirmed: same fn sits at
  vt[3] of the clip's ops table 0x141ff6b98)
- so inside the deserializer, r14=ctx, and stream reads go
  rcx=ctx -> rax=[ctx]=handler -> call handler.vtbl[+8]:
  the byte reader is the HANDLER's vtbl[1], not a stream
  method. Handlers are per-record objects.

- 0x1407cb194 = CADB chunk header reader: magic 'CADB'
  (0x42444143) + version 2|3 + 5 u32s, read via the same
  u32 stream read - ANIX clips are serialized through the
  generic chunked stream layer.

- Leaf reads land on ucrtbase memcpy (the earlier
  read-watch hits) => the stream read is a BUFFERED copy.
  The file->record transform (bit unpack) therefore happens
  either inside the handler vtbl[1] read OR at buffer fill.

- Negative: no 0x38-aligned {tag,n,fx,fy} record runs found
  verbatim in bola.anix -> records are NOT stored raw in
  file order; the handler's read does real work (probably
  delta/bit unpack) OR storage order differs.

NEXT (safe): find the handler class - the ops table
0x141ff6b98's sibling entries (vt[1]=0x1403dcdc0 etc.) and
the per-record handler array ctor. Handler vtbl[1] = the
bit reader = the last unknown.

### r45 - full read chain mapped end-to-end

COMPLETE STREAM STACK for clip deserialization:

  deserializer 0x140501478 (clip vtbl slot 3)
    rcx = clip obj (vtable 0x141ff6b98)
    rdx = ctx {+0: adapter vtbl 0x141db5dd0, +8: inner stream}
  read_u32/read_u64 -> [ctx]->vtbl[+8] = 0x1404ba4fc
    = bounds-checked adapter: cursor+len <= limit, delegates
    to ctx.inner->vtbl[+8], bumps cursor
  inner stream vtbl[+8] = leaf read = buffered memcpy
    (ucrtbase hits) - bytes served from a decoded window

  So the file->record transform is inside the INNER stream's
  refill - CADB chunk decode. ANIX is chunked: 'CADB' magic +
  ver 2|3 + 5xu32 header per block (0x1407cb194).

  Verification (offline): bola.anix starts "ANIX"; clip
  payload ~entropy 6.7 = still-encoded inside the
  decompressed member - consistent with per-chunk CADB
  coding, not raw records.

- Session instability: crash #4 = 0xc00000fd stack overflow
  in gameoverlayrenderer64.dll (Steam overlay) + stray
  Alt-Tab keys - environment-level, unrelated to watches.

NEXT: identify the inner stream class statically (it
services CADB decompression) OR exec-watch the CADB reader
0x1407cb194 (cold, once per chunk) on a clean boot to read
rdx=[inner stream] then vtbl[+8] -> the decoder.

### r46 - boot wedge: f8-check stall, CADB watch never fired

Plan: exec-watch 0x1407cb194 (CADB header reader - cold, once
per chunk) on a clean windowed boot, then LoadWorld to fire
it; rdx=ctx -> [ctx+8]=inner stream -> vtbl[+8]=decoder.

What happened instead:
  - Boot reached a black screen; main loop went stale 64s+
    inside a lua 'load' handler (handler 2/2) during autoload,
    then again 62s during the F8 reload - the SAME class of
    stall that preceded the three 0x0000071a crashes.
  - Game VM threw HksError C++ exception mid-stall;
    `System` global was nil -> world never loaded -> the
    CADB reader never fired (0 hits). sdk stayed usable in
    the plugin VM throughout.
  - Root wedge candidate: Mods/orc_names registers the only
    sdk.on("load") handler and every inject is G41-gated
    (fails by design) - pure waste + stall risk. Disabled it
    via `Mods/orc_names.disabled` marker so future boots skip
    the load handler entirely.

Console gotcha (relearned): `! <chunk>` echoes the chunk's
RETURN value - `! tostring(x)` prints nil because the
statement returns nothing; use `! return tostring(x)` for
values. `=expr` evaluates in the GAME VM (System etc. only
exist in-world).

NEXT (unchanged): on a clean boot, arm watchx(0x1407cb194)
BEFORE triggering System:LoadWorld - it is cold enough to be
safe even during parse. rdx -> ctx -> [ctx+8] -> inner
stream vtbl[+8] is the last unknown.

### r47 - RESOLUTION: cooked clips live in .objres CAWR/CADB chunks

The live probe (cold watchx on 0x1407cb194) fired 4 clean hits
during System:LoadWorld(BM_Island) - rdx = ctx = 0x1ae6fb58 /
0x1ae6fd88, DETERMINISTIC across boots (same as r41). Game then
died to an nvwgf2umx.dll 0xc00000fd during render - unrelated to
the watch.

Static chain completed from the deserializer's caller
(0x14048094c = "OBJR" v0xd chunk dispatcher):

  per record (0x20B stride): {u32 id, u32 len, handler*}
    ctx = {+0 adapter vtbl 0x141db5dd0, +8 stream, +0x10 limit,
           +0x18 cursor, +0x20 flag}
    handler->vtbl[+0x18](handler, ctx) = 0x140501478

  stream read = rdi->vtbl[+8] = plain buffered member read.
  NO in-stream codec exists - 'CADB' appears exactly once in
  the whole exe (the compare) and 'CDAB'/'BADC' never; the
  cooked bytes are ON DISK.

WHERE: .objres files! 553/680 .objres members contain 'CADB'
(bm_island.objres has 3 - the benchmark world). Structure:

  "OBJR" v13 file -> chunks -> "CAWR" v1 wrapper
    {+0c hash, +0x10 0, +0x14 hash2, +0x18 hash3, +0x1c size}
  -> "CADB" v3 {u32: 2, 0x3c, 0, 0, 1} (0x1c hdr)
  -> payload (0xc68B on baraddur, entropy ~5.9) = cooked
     element records consumed field-wise by 0x140501478.

IMPLICATION: the shipped .anix (ANIX v44) is a SOURCE-format
asset; the runtime bank is the CAWR/CADB cooked form embedded
in per-world .objres. The earlier "entropy 6.7 = encoded"
observation on .anix sections is unrelated - the live path
never reads .anix clip payloads through this chain.
'ANIX' header check is at 0x140480b00 (same fn family) - the
.anix reader exists too but the sampled path is OBJR.

NEXT: correlate CAWR/CADB payload bytes against the live
element-array dumps (target/tmp/capture_round2) to pin the
exact cooked record stride/fields; the deserializer reads
{u64,u64} then 3x read3xU32 into +0x10/+0x1c/+0x28 per record
- verify that stride pattern in the CADB payload directly.

### r48 - TWO parallel anim paths identified; .anix reader decoded

The deserializer chain (OBJR dispatcher -> 0x140501478) is the
COOKED path for clips embedded in world .objres (CAWR/CADB
chunks). Character .anix files take a DIFFERENT reader:

0x140480b00 ANIX reader (validates "ANIX" v0x2c/44):
  - reads the 0x30B header directly from a member stream
    (vtbl[+8] = plain buffered read)
  - count = hdr[+0x10]; skips to hdr[+0x14]
  - ENTRY LOOP: per outer {u32 tag, u32 n} then n x u32 ->
    flattened into 8-byte records {value=u32, tag=u32}.
    VERIFIED on blrg_combat.anix: 299 records, tag values
    include 0x210c8f5a and 0x27436396 = the known bone-name
    hash-table tags. Section A tag tables become a flat
    (value, tag) index array at runtime.
  - skips to hdr[+0x18], bulk-reads hdr[+0x1c] bytes  = sec B
  - skips to hdr[+0x20], bulk-reads hdr[+0x24] bytes  = sec C
  - asserts stream pos == stream size.

  => The .anix runtime image = flat 8B (value,tag) index +
     RAW section-B/C blobs. The 8-byte "element slots" seen
     live are these index pairs. Clip blocks (sentinel-
     delimited, inside section A after the tag tables) are
     presumably parsed lazily per clip.

CORRECTION to r45-r47 framing: there is no decode/codec on the
.anix path either; raw blobs are memcpy'd wholesale. The two
representations (.anix source vs CAWR/CADB cooked) converge on
the same runtime clip/track/element structures - the
0x140501478 deserializer serves the cooked path.

OPEN: where the cooked CAWR/CADB payload and the raw .anix
sections meet in element-record semantics; clip-block
consumption path for .anix (the per-clip reader that walks
sentinel blocks); how the flat index maps tag->record
sub-arrays.

### r49 - reconciliation: what remains for .anix is the per-bone stream format

Existing anduril-anix status verified this round:
  - clip/track/sentinel layout parses on all banks;
  - translation channel-0 keys decode + pose correctly
    (anix pose produces a plausible skeleton);
  - rotation_pool s16x4 -> quat decode verified;
  - BROKEN: BoneStream.indices are NOT (channel<<8|key) entries -
    real data shows monotone 0x00FF/0x01FF/0x03FF runs = a
    bit-packed stream misread as BE-u16s. rot=0 on pose output
    because rotation_schedule's pool-slot hypothesis fails.

  The last unknown for .anix is therefore the PER-BONE STREAM
  BIT FORMAT (inside clip blocks, in the region bulk-copied
  into runtime blob1), plus how stream keys map to the
  rotation pool. On the cooked CADB path the same semantics
  are expressed as {tag,n,fx,fy} descriptors -> expander
  endpoint pairs - the two paths must converge on identical
  element semantics: elements = quantized key endpoints.

  Hypothesis for next round: a bone stream's index data is a
  bit-packed list of ELEMENT indices (into the flat
  (value,tag) index / element array), where each element's
  u32 value is itself packed endpoint data (e.g. s16x2 or
  s16x4). The 255/511/1023 pattern is consistent with
  small monotone indices in a narrow bitfield.

### r50 - clip block internals: s16 stream + u16 offset ramp, no bitpacking

Direct byte inspection (bola bank) overturns the "BE-u16 index
list" parse:

CLIP LAYOUT (variant A bind + variant B animated agree):
  [20B prologue][sentinel 0x2b99dbc1]
  [sent+4 .. sub0)      s16 key stream - raw quat data
                        (bind: fcf7df7f unit-quat run then
                         per-bone s16 quats; ~0x11cB for 2 bones)
  [offset ramp sub]     u16s, monotone +stride (0x1c bind,
                        0x98 anim clip4) = per-key offsets into
                        a record pool / per-channel block
  [tail sub]            small u32 header then next clip

  The streams are raw s16 key arrays - the earlier indices=
  parse was reading quat data as BE u16s. The real decode is:
  per-bone s16 quat runs delimited by the sub-offsets, plus a
  u16 offset table whose constant stride selects the record
  size (0x1c = 28B records; 0x98 = 152B - probably per-channel
  or per-frame groupings).

NEXT: walk the sub-offset boundaries on several clips to pin
which sub-block is streams vs ramp vs pool; map the u16 ramp
targets to figure out the record size semantics; then keys x
bone assignment falls out.

### r51 - 28B record = 2x14B endpoint pair, f16 translations confirmed

Bind clip0 (bola) fully walks:
  sent+4 .. +0x40:   s16 stream head (bind = fcf7df7f run)
  .. +0x120:         s16 quat stream region
  +0x120 ramp:       20 x u16 monotone +0x1c (0x118..0x32c),
                     0x0000 terminator
  then 28B records = 2 x 14B {u16 meta, 3 x f16 xyz, u16, u16}
    - f16 decodes to plausible bone translations:
      (-2.83, 9.60, -0.37) stable across records 1-4
    - both halves share xyz; differ only in the meta u16
      (0xff00/0xfffe/0xfffc/0x0000/0x3330...) = the
      {key0,key1} endpoint pair - SAME semantics as the
      cooked-path expander output {base-d, base+d}.

So the .anix file-side record of a keyed channel is a
precomputed endpoint pair around a center value; the expander
math on the cooked path (d = (f-20)*0.5*K*n) is just the
runtime regeneration of these pairs for CADB elements.

clip4 subs verified: [0x40]=u16 ramp stride 0x98,
[0x1140]=u16 records {0x21,0x22} pattern (per-bone heads?),
[0x1430]=s16 quat pool, [0x14a0]=second ramp stride 0x22.

### r52 - clip sub-blocks resolved: bone offset table + key-count records

clip2 (bola, 31 bones) measured precisely:

  [sent+4 .. +0x40):   31 x u16 bone-stream offsets
                       (0x40,0x90,0xf8..0xd0c, +pad) - first
                       entry 0x40 == table byte size (offsets
                       relative to table start); variable
                       deltas = variable stream lengths.
  [+0x40 .. +0xD90):   per-bone s16 key streams
  [+0xD90 .. +0x1310): per-bone key-count pairs {u16,u16}
                       (~33/34 - nTranslation/nRotation keys?)
  [+0x1310 .. ):       s16 quat pool + f16 data
  [+0x1390):           second u16 ramp, stride 0x22 - indexes
                       the 28B endpoint-pair records? (0x22 = 34
                       matches the ~34 rotation key count)

  Same shape on bind clip0: 20 x +0x1c ramp -> 28B
  {meta,3xf16 xyz}x2 endpoint pairs (translation channel).

  Remaining: exact stream->key decoding within a bone stream
  (the s16s are probably per-channel endpoint data needing
  the expander semantics: center +/- delta), and which of
  the {count,count} pair is which channel.

### r53 - bone stream format: {u16 keytimes.., 0x0000, s16 keys[]}

Measured on clip2 bone0-2 streams:

  stream = u16 ramp of key FRAME TIMES (monotone, ends
  0x0000 terminator... actually last entry = clip duration
  e.g. 0x4ff, then 0x0000), then s16 key payload.
  bone0 (static): times 0x42,0x100,0x170,0x1ff x3 - the
  repeated identical times explain the old parser's
  "255/511" garbage. bone2 (animated): 22 distinct times.

  clip sub-block map (corrected):
    block0 = u16 offset table (per-bone) + per-bone streams
    block1 (+0xD90) = per-bone {u16,u16} key counts
    block2 (+0x1310) = s16 quat pool + f16 tail
    block3 (+0x1390) = ramp -> 28B endpoint records

  TODO: which channel each stream's keys belong to, whether
  s16s are quat(4x) or vec3(3x) endpoints or center+delta,
  and the exact terminator semantics.

### r54 - bone stream tail = f32 vec3 translation keys

bone1 (clip2) tail decoded as f32 triples:
  (-16.63,-0.17,0.72) (-17.34,0.70,1.19) (-19.02,1.56,0.50)
  (-20.41,1.64,-0.61) - smooth monotone translation, clearly a
  moving bone's position keys.

  bone stream = [zero head] [u16 time ramp, 0x0000 term]
                [f32 vec3 keys, 12B/key]
  bone2 (21 times) tail 70B -> NOT 12B/key: longer streams
  likely carry a second channel (s16 quats?) or the stream is
  per-channel within a multi-sub-stream block. Zero-head size
  varies (0x1e..0x28) - probably channel-enable padding or
  an alignment/header field, TBD.

  Open: exact head semantics, why animated bones have shorter
  f32 tails than 12B*keys (maybe keys counted differently -
  times may cover both rot+trans keys sharing one clock).

### r55 - key times are BE u16; ramp covers all channels

bone2 clip2 precise layout:
  head 2B zeros | 20 x BE-u16 times (0x88..0x4ff monotone)
  | 0x0000 term | 6 x f32 vec3 translation keys (72B)
  Times outnumber translation keys ~3:1 - the ramp is the
  shared key clock (rotation keys at finer cadence); sparse
  f32 vec3s are the translation channel.

  The old parser's "(channel<<8)|key-index" BE u16s were
  actually these time values - right endianness guess,
  wrong semantics.

  Stream = [variable zero head][u16 header?][BE-u16 times]
  [0x0000][f32 vec3 trans keys]. Head field before times
  on some bones (e.g. 0x0142) may be duration or track idx.

### r56 - rotation side: s16 quat pool + quat endpoint pairs + u32 element ramp

clip2 layout refined:
  block1 (+0xD90): 17 x u16 counts (33/34 = rot key counts)
                   then s16 quat key pool (efff/dfff patterns)
  block3 (+0x1330): s16 quat ENDPOINT-PAIR records (16B ea:
                   {4xs16 q0, 4xs16 q1}) - f4176080/fcf7df7f pairs
  block4 (+0x1390): u32 offset ramp (delta ~0x21) into a
                   0x33ac record region - the element index

  Full .anix clip model:
    flat index -> clip block -> per-bone {BE times, f32 vec3
    trans keys} + rot-key counts + s16 quat pool +
    endpoint-pair records (trans f16 28B / quat s16 16B) +
    u32 element ramp.
  Cooked path equivalent: {tag,n,fx,fy} descriptors expand to
  the same endpoint-pair elements at runtime.

### r57 - rotation mapping: pool located, per-bone split unresolved

clip2 measurements (bola bank):
  Σ times = 397, Σ vec3 keys = 174, s16 quat pool ~= 170 keys
  (block1 region +0xD90: 17 x u16 counts {33,34} header, then
  interleaved quat-quad data - not cleanly 8B-aligned runs).

  block3 (+0x1330): quat endpoint PAIRS {4xs16}x2 - e.g.
  {0x17f4,0x8060}x2 + {0xf7fc,0x7fdf}x2 = static-bone quats
  stored as degenerate pairs (endpoint == key).

  block4 (+0x1390): u32 ramp (~0x21 stride) -> big record region.

  Leading hypothesis: quat count per bone tracks the vec3 count
  (174 ~= 170), i.e. rot keys parallel trans keys per bone -
  NOT the times count (397). The 17-entry {33,34} table may
  instead be per-segment frame counts. UNVERIFIED - test by
  sampling quats with each split and checking pose sanity.

### r58 - rot split falsified; per-FRAME element model emerges

  Split tests on clip2 s16 pool (~170 keys @block1+0x2c):
    bone-major (slice = vec3 count):   mean |dot| 0.06-0.66 FAIL
    frame-major (stride 31):           mean |dot| 0.27-0.52 FAIL
    -> pool is indexed per-keyframe, not per-bone.

  New model (cooked-path consistent):
    block1 {33,34} x 17 = per-FRAME key counts (17 keyframes)
    block4 u32 ramp deltas = 0x21/0x22 = 33/34 = THE SAME counts
      -> the ramp indexes per-frame element records; each element
      = {bone/channel ref, pool key idx} - matching the runtime's
      flat {value, tag} element array.
    -> bone->quat resolution goes through the ELEMENT INDEX, not a
       contiguous pool slice. Next: decode a frame block in block4:
       expect ~33 entries of {bone_idx, pool_offset} pairs.

### r59 - block4 = ms keyframe times; flat index is the element table

  clip2 block4 (+0x1390): 18 x u32 keyframe times, deltas 33/34ms,
  last = 0x29a = 666ms clip duration. Ends right before clip3's
  sentinel. (Earlier "bone-hash hits" were coincidental - the ramp
  values landed inside the hash list region, not real records.)

  Quat pool (block1, ~170 keys) is indexed per-keyframe via the
  flat {value,tag} element index (299 records ~= 17 frames x ~17.6
  elements) - same structure the runtime builds (r48). Element
  records likely encode {pool_key_idx, frame|bone_ref}.

  Next: dump the flat index records for clip2's frame range and
  check whether values point into the quat pool / bones' streams.

### r60 - quat pool marker-segmented but not slot-consistent

  block1 pool @clip2+0xD90+0x2c: 170 s16-quads split by marker
  quads {.., 0xdffe, 0x03ff} into 22 groups x 7. Tests:
    fixed slot across groups: slot3 smooth (0.97), others 0.3-0.8
    permuted match across groups: best ~0.8 max - not smooth
    within-group consecutive: 0.15-0.55 - not tracks either

  Interpretation: quads are probably NOT raw quat keys - likely
  endpoint/delta encoded (cooked path expands descriptors into
  endpoint pairs). OR pool grouping isn't 1-frame=1-group.

  Byte-fishing has plateaued. NEXT: static decompile of the clip
  consumer - the bank object from 0x140480b00 has a vtable whose
  clip/sample methods read this exact layout. Finding that fn is
  now the reliable path (it IS the ground truth for the format).

### r61 - ANIX reader disasm: instruction-level confirmation

  0x140480b6c..0xbd0: outer loop reads {u32 outer_tag, u32 n},
  inner loop writes flat [r15 + i*8] = {inner_val, outer_tag}
  records. Then vtbl+0x28 (pos) -> skip -> 0x140480ec4 (alloc +
  bulk read) for blob B, 0x140480e6c for blob C, final
  pos==size check via vtbl+0x30.

  Runtime bank = {flat (value,tag) index, blobB, blobC}; clip
  blocks (sentinel-delimited) live verbatim inside blobB.
  The clip SAMPLER is a separate consumer fn not yet located
  (bank has no vtable). The (value,tag) records are the element
  table - values index streams/pools inside blobB; the 299-count
  flat index IS the clip lookup structure.

  Next: find the consumer via who reads bank+index (xref on the
  handler-registration record, or trace the runtime anim eval).

### r62 - GROUND TRUTH: the anim-eval sampler at 0x14180deb8

  Track loop: imul rbx,rax,0x88 -> 0x88B track records.
  Per track: r15d & 7 = CHANNEL TYPE switch (8 cases) -> each
  case indexes a 0x20-stride element array twice (r9, r10 =
  bracketing keys), then:

    out[+0x10] = 0x140210078(k1.hi, k0.hi, t)   (nlerp/slerp)
    out[+0]    = lerp(k0.lo, k1.lo, t)

  => runtime element = 32B {vec4 lo, vec4 hi} = endpoint pair.
     File's 28B {meta,3xf16}x2 records expand to this 32B form.
     Flat index {value,tag}: value = element slot (x32B), tag&7
     = channel decoder variant.
  => per-bone stream times + {start,count} select elements.

  NEXT: which case = rotation (the nlerp'd half is the quat);
  decode tag bits fully (bone idx = tag>>3?); map stream time
  ramp -> element index so sample_rotation works.

### r63 - element = {lerp vec4, slerp vec4} pose fragment

  0x140210078 disasm: dot = horizontal sum of q0*q1, sqrt(1-dot^2),
  sin-based scale -> SLERP. So the 32B runtime element is:
    [+0x00] vec4 - lerped  (translation / scale)
    [+0x10] vec4 - slerped (rotation quat)
  Each key element is a full pose fragment; the 8 channel-type
  cases (tag&7) presumably differ in WHICH half/component is
  written or the endpoint source (stored vs shared).

  File-side: 28B {meta,3xf16}x2 endpoint pairs -> 32B elements.
  Quat w likely reconstructed (store xyz, w=sqrt(1-x2-y2-z2)) -
  explains s16 quats in the pool lacking consistent norms.

  TODO: extract tag bitfields (bone/channel/encoding) and the
  track-record 0x88B layout to wire pool->bone fully.

### r64 - block1 = per-frame quat groups

  [17 x u16 frame counts {33,34}] then 17 x ~33-u16 groups:
  each group = [zero pad][small hdr u16s][~8 x s16 quats]
  delimited between frames by fe df ff 03 markers.
  First quat of frame0 normalizes to ~0.94 unit -> real rot data.
  ~8 keyed bones per frame (subset of 31) - static bones use
  the block3 endpoint-pair records instead.

  Runtime eval model now complete conceptually:
    flat index {value,tag} = bone-hash orderings
    blobB clip blocks: per-bone trans streams + per-frame quat
    groups + endpoint records; 32B runtime elements expand to
    {lerp vec4, slerp vec4} pose fragments.

  STILL OPEN: the bone<->frame-quat assignment table (which bone
  each quat in a frame group drives) - candidate: element index
  in another sub-block or implicit order of "animated" bones.

### r65 - frame group = 3 endpoint-pair records {meta, quat0, quat1}

  Each ~33-u16 frame group splits into 3 records of ~11 u16:
    {3 x u16 meta}{4 x s16 quat0}{4 x s16 quat1}
  Both quats normalize to ~unit (0.94 / 1.03 verified).
  Meta is CONSTANT per record slot across frames
  (rec A = efff f9ff ??, rec B = 41 20f9 407 ...) -> meta
  identifies the bone/channel; the varying 3rd u16 = key/time ref.
  So rotations: ~3 bones keyed per frame via bracketed quat-pair
  records; all other bones use static endpoint pairs (block3).

  Runtime mapping confirmed consistent: these are the file-side
  form of the 32B {lerp,slerp} elements - interp between the two
  endpoint quats via slerp.

  TODO: crack the 6B meta (bone id + key idx bit-packing) -
  try byte-level reads vs the 31 track-hash order.

### r66 - quat pool = 3xs16 compressed (w reconstructed)

  Norm test on block1 tail: ~78% of 3-s16 groups have |v|<=1
  (vs ~25% at 4-s16 stride) -> quats stored as {x,y,z} s16
  with w = +-sqrt(1-|v|^2). Matches runtime element hi vec4
  being slerped (and explains 28B records' 3-component xyz).

  Per-frame groups (17 x {33,34} u16) = ~11 compressed quats
  each (33 u16 = 11x3 s16 exactly; 34 adds a pad/hdr word).
  Consecutive quats within a group are NOT smooth (~0.15 mean
  |xyz dot|) -> each group is a per-FRAME key set across
  multiple bones, not a single-bone track.

  ~11 keyed bones per frame for 17 frames ~= 187 pool quats,
  matching the ~170-190 s16 data volume.

  OPEN: the bone id for each slot (either implicit slot order
  shared by all frames, or an index vector elsewhere). Static
  trace of the clip-sample consumer is the reliable finish -
  byte structure alone has reached its limit here.

### r67 - falsified: fixed slot->bone order across frames

  Slot-wise |dot| across 17 frames ~0.2-0.35 = random; the ~11
  compressed quats per frame group do NOT reuse a fixed slot
  order. Likely each quat carries (or is preceded by) a bone
  index - but u16-pair records ({idx,3xs16}=8B) don't divide
  the 33/34-u16 group sizes evenly, so there is a header/
  packing detail unaccounted for.

  Reader has NO call sites (registered indirectly) -> next
  probe: trace blobB consumers via field access, or disasm the
  eval-loop's element-base pointer provenance ([r8+0x158]
  chain) to reach the per-clip element builder.

### r68 - eval-loop field map (0x14180deb8 head)

  track rec (0x88B):  +0x68 = clip/bank obj
                      +0x78 = per-track keytable {?, dur@4, cnt@8, keyBase@0xc}
                      +0x80/+0x84 = playback cursor/accum
  clip obj +0x158 = r15d: shared element-per-frame multiplier
  element index = keyBase * r15 (+ key idx) ; *0x20 -> elem ptr
  => elements are FRAME-MAJOR blocks; r15 = elems per frame.

  Consistent w/ file: 17 frame groups x ~11 quat keys = the
  frame-major element payload; per-track keyBase selects which
  frame block / slot. Bone assignment likely = slot order in
  the frame block (the tag in flat {value,tag} index?).

### r69 - plateau note

  Consumer trace status: eval fn + reader have no direct call/
  lea sites (vtable/handoff-indirect). clip+0x158 write sites
  too generic (~40 hits). block3 in clip2 = s16 quat pair
  records (bind-pattern visible), not 28B trans records - the
  28B {meta,3xf16}x2 records belong to the translation path
  in the ramp-indexed region.

  RESUME HINT for next session: rather than more scans, find
  the clip-builder via the bank object's consumer - dump the
  bank struct fields after reader returns, then search for
  functions reading [bank+8] (flat index ptr) - that is the
  small set of anim consumers. Or instrument: nemesis pipe
  read of a live bank object while a clip plays.

### r70 - ANIX bank runtime struct (0x28B)

  +0x00 ptr  blobB (clip blocks, verbatim file bytes)
  +0x08 ptr  blobC (C-section / name-record table)
  +0x10 ptr  flat {u32 value, u32 tag} index array
  +0x18 u32  blobB size   +0x1c u32 blobC size
  +0x20 u32  index count
  (built by 0x140480f5c/f04/ec4/e6c from reader 0x140480b00)

  Consumer hunt: reader + eval fn reached only via indirect
  tables (no call/lea xrefs). Next: find code reading
  [obj+0x10]/[obj+0x00] on anim bank objects, or live-dump a
  bank through the nemesis pipe while a clip plays and walk
  its pointers into the clip runtime (track recs, elements).

### r71 - registration hunt exhausted (static)

  No call/lea/ptr/RVA refs to the reader; eval fn appears only
  in its own switch table. The ANIX dispatch is wired through
  a mechanism our scans can't see statically (likely a runtime
  hash->fn map built from a ctor, or member-function table).

  DECISION: the cheap finish is a LIVE dump - boot game, use
  nemesis pipe to locate an ANIX bank object (scan heap for
  the 0x28B struct signature: idx-count + blobB size), follow
  +0x00 blobB into a playing clip's runtime state, and dump
  track records + element array. Correlate element slots to
  bones once, empirically - ends the byte-guessing entirely.
  Requires a booted game; deferred to next session.

### r72 - LIVE bank dump via nemesis pipe (game booted, title screen)

  Bank object FOUND live: 0x21a489950 (bl_movaction.anix -
  Blade movement, loaded at title). Struct verified:
    +0x00 blobB=0x21afe8e20 (0xe170B, verbatim clip bytes -
         sentinel 0x2b99dbc1 + bind-pattern quats visible)
    +0x08 blobC=0x1390a6a30 (0xffcB, starts u32 0x10, has
         "bind" string + {key,off} records)
    +0x10 flat index=0x35189e50, count 0xab=171
  Index = {u32 value,u32 tag}: 85x{0x00ce66fc} + 1x
    {0x210c8f5a} + 85x{0x27436396}. BOTH tables hold the SAME
    85 bone hashes in the SAME order (bone-name list, not an
    element permutation).
  A live clip obj found pointing into blobB: 0x31701f9f0 with
    +0x158 = 16 = elements-per-frame (matches eval r15d field)
    and +0x148/+0x150 = ptrs (element/keytable chains).
  A live track rec (0x1239a9d08): +0x68=clip, +0x78=keytable
    chain node (linked nodes {next/back ptrs, f32 ~786 dur}),
    +0x70 = packed u16s {0103,0002,0001,0002}.
  Eval: elem base = [[clip+0x140]->vtbl[0]()]; elem idx =
    keyBase*r15 + keyIdx, *0x20.

  NEXT: dump keytable chain of a live track fully + element
  array; correlate element slots to bone hashes. Game closed
  after capture per house rules.

### r73 - live in-world capture (session 2)

  Booted into gameplay (save loaded). Multiple blobB buffers
  resident (sentinel hits across 0x4437/0x4560/0x461b/0x635e
  regions). Identified bank 0xd63f00f0: blobB=0x45600040
  (0x17b60B), blobC=0x62dc5d80 (0xaecB), index=0x7365f8f0
  (177 recs: 89x{00ce66fc} + 1x{210c8f5a} + 88x{27436396}).
  blobB byte-prefix matches the player-movement bank family
  (pl_/bl_ jump/vault/zipline share the skeleton+prefix).

  Confirmed in-world: the flat index is JUST the bone-hash
  table (two copies), ~88-bone rig. Clip objs + tracks live
  alongside; several blobB-pointing objs found but none with
  a clean elemsPerFrame at +0x158 this time (different clip
  state class?).

  Game closed per house rules after capture.

  NEXT (live): rather than scanning for clip objs, arm a
  hardware READ watchpoint (sdk.debug.watch) on a blobB quat
  data addr INSIDE a playing clip block -> the hit RIP is the
  clip-parser/consumer instruction = ground truth. Pick a
  blobB addr past the bind clip (only touched when its clip
  activates), then force an anim change.

### r74 - elem index = frame*elemsPerFrame + track's fixed slot

  Fallback path (0x14180e2d8, taken when clip+0x140==0) shows:
    elemIdx = keyIdx*r15 + keyBase     (r15=elemsPerFrame)
  Both branches agree -> elements are FRAME-MAJOR blocks of
  elemsPerFrame x 32B; each track owns slot keyBase inside
  every block. keyBase comes from the clip builder.

  FILE-SIDE CONSEQUENCE: a frame group in block1 = the
  elemsPerFrame element payload for that frame; slot j always
  belongs to the same track. The per-frame quat groups being
  "non-smooth across frames" means the file stores only a
  KEYED SUBSET per frame (delta of which slots changed), or
  groups aren't slot-aligned - consistent w/ the {meta,quat}
  record pairs carrying the slot/bone id in their meta bytes.

  Session notes: user asked to stop driving the game with
  input - live work will use read-only pipe memory dumps only
  (no key injection, no foreground forcing), or stay offline.

### r75 - eval fn located in a vtable; anim-player class identified

  tools/nem.py xref @<va> does RIP+absolute xref scanning (the
  manual scans missed .xtext abs pointers). Eval loop
  0x14180deb8 is vtable slot in anim-player vtable
  @0x141dcdd60 (.xtext). Class: 0x160B object, dtor
  0x14180b824 frees sub-objects at +0x120/+0x138; sibling slot
  0x14180dd44 = registry lookup. The clip-builder (parses
  blobB -> track recs + element blocks) is another slot or a
  free fn called from activation - not yet pinned.

  NEXT: identify ctor via xref to vtable base+8 (0x141dcdd68)
  sites 0x1407cdda1 (was dtor-reset) / scan ctor that sets
  +0x158 and allocates nTracks*0x88 + frame-major elements.

### r76 - hunt status + remaining leads (checkpoint)

  CONFIRMED reachable leads for the clip builder (blobB ->
  track recs + elements):
  1. anim-player vtable @0x141dcdd60 (26 slots; eval=slot23);
     ctor writes vtable -> find via xref to vtable slots.
  2. Clip activation likely calls a free fn that walks the
     bank: reads bank+0x10 flat index (bone hashes) to bind
     tracks by name, then parses blobB clip block.
  3. Watchpoint route stays available (read-only live dump).

  Nemesis docs do NOT cover the anim codec (anim-picker.md is
  the picker/sever layer, not clip data).

  DONE AND USABLE NOW (independent of rotation mapping):
  - full bank/container decode; per-bone stream decode;
    translation keys; anix pose animates all 31 bones
    positionally. Rotations are the only missing channel.

### r77 - VolumeDB object vtable + reader decompiled (Ghidra)

  Big static wins this round (headless Ghidra decomp working):

  - VolumeDB *instance* vtable = 0x1422c2c70. Slot1 =
    FUN_140480aec = the bank reader itself (explains why the
    reader had zero direct code refs - reached via vtable).
  - Reader decomp CONFIRMS the file format from binary truth:
    magic 0x58494e41 ("ANIX"), version check ==0x2c (44).
    Index = grouped records {value u32, count u32, tag u32
    x count} flattened into 8B {tag,value} records - matches
    the live-dumped index (bone_hash + group tag). Then two
    size-prefixed blobs read verbatim: blobB (clip blocks)
    and blobC (name table). End check: "Failed to consume
    the full file" if stream short/over.
  - Reader does NOT cook clips - the bank keeps blobB raw.
    Clip cooking (blobB -> 0x88B tracks + 32B elements) must
    run at bind time, i.e. a consumer of the returned bank.
  - 0x142addc10 earlier "data ref" was .pdata unwind records
    ({begin,end,unwind} RVA triples) - red herring, noted.
  - Loader chain for the resource: vtbl slot10 0x140481318
    -> 1404b61c4 -> 14115da64 = generic async work-queue
    push with completion cb 140616318 (refcounted vector
    insert). All plumbing; parse entry is slot1 reader.

  Next leads, in order:
  1. Consumer of bank object: find callers holding the
     bank* that read +0x00 blobB (clip cook on bind).
  2. anim-player vtable 0x141dcdd60 slots - a bind method
     should take (bank, clip-name hash) and build tracks.
  3. Live watchpoint on blobB read during clip start
     (read-only; ask user before launching game).

### r78 - Havok discovery + anim-graph update fn + src.bin corpus

  STATIC (Ghidra):
  - Exe embeds full Havok reflection type names:
    hkaInterleavedUncompressedAnimation, hkaDeltaCompressedAnimation,
    hkaQuantizedAnimation, hkaSplineCompressedAnimation, etc.
    ANIX is NOT a packfile (no Havok magics) - proprietary cooked
    format; hka* are the runtime/authoring-side representations.
  - Assert strings name the bank fields officially:
    NodeAndTrackName (index @+0x10), AnimBlockChunk (blobB @+0x00),
    AnimChunk (blobC @+0x08). "allocate... only during anim bank load".
  - FUN_14180b8d8 = anim-graph update: iterates anim nodes,
    computes blend weights, writes track recs (i*0x88 stride at
    player+0x138 array), track+0x68 = clip obj, clip+0x158 =
    elemsPerFrame, player+0x150 flag, player+0x158 arg.
    It CONSUMES cooked clips - still not the blobB parser.
  - First loop registers FUN_14180d550 per slot (up to 10) ->
    FUN_14180cd50 walks anim-graph node tree (0x28-stride nodes).
  - VolumeDB reader vtable 0x1422c2c70 slot1 = FUN_140480aec.

  OFFLINE ORACLE (already on disk, no live session needed):
  - capture_round2/*.bin: live-dumped src multipose buffers
    (cooked keyed input) + dst pose buffers (verified {pos4,quat4}
    32B/bone records). Src layout unknown (mix of ptrs + packed
    u16 pairs + counts) - decoding src.bin against known dst poses
    + file 16B grid records is the offline crack path.
  - manifest.jsonl has all addresses/numBones/slot indices.

### r79 - src.bin multipose structure decoded (partial)

  From capture_round2/c0m0f10_src.bin (olog anim src, nb=20):

  - Header ~0x20B: {tag u64, u32 1, u32 nb=0x14, u32? 0x19230,
    u32 stride=0x20, ...}
  - Then a ~0x80-strided descriptor array (one per track):
    {f32 ~27.4 duration, u32 flag=1, ptr into 0x0994xxxx pool
    (per-track KEY DATA), f32 ~0.0005 rate?, 2 self-relative
    links (prev/next into this array), u32 key_count}
    - observed counts 99, 35, 67, 43 - per-track key counts.
  - The 0x0994xxxx pool (NOT captured) likely == this session's
    blobB verbatim -> ptr - blobB_base = FILE OFFSET of each
    track's key table. Verifying that is THE decisive check:
    it would map every runtime track directly to file bytes.
  - asrc (member+0x78) chain: 0x15890cxxx / 0x2346cfxxx ranges.

  NEXT SESSION, fastest finish (ranked):
  1. Live read-only dump: bank blobB base + one src.bin
     descriptor array + the keydata pool it points into.
     Then ptr-base arithmetic gives file offsets of runtime
     key tables -> close the format completely. (~1 session)
  2. Offline: match src.bin descriptor ORDER/durations/
     counts against per-clip structures in the olog .anix
     file (find which clip has ~20 tracks + key counts
     99/35/67/43 pattern) - may pin the mapping w/o live.

## r80 — vtable slots resolved; hash cross-validated

  - Instance vtable 0x1422c2c70 has 24 slots (earlier 13-slot read
    was a different table, 0x1422f2d18).
  - slot 8 (FUN_140480898) = bank TEARDOWN: zeroes obj+0x38/+0x68,
    frees the param_2[6] buffer via FUN_140480900, cleans the
    param_2+0x50 region (FUN_1404bbc78), frees param_2[8], clears
    param_2[9]. Confirms bank layout: [6]/[8] = owned buffers
    (AnimBlockChunk/AnimChunk), [9] = count field.
  - slot 0 (FUN_1407b833c) = refcount release w/ jumptable delete;
    slot 2 = trivial getter. No cook/parse among the other slots —
    all generic resource plumbing.
  - CONCLUSION: the clip-cook path is NOT on the bank vtable; it
    lives behind whatever consumes the bank. Static search for the
    cook fn is exhausted for now — the live-dump route (r79 #1)
    remains the fastest finish.
  - CROSS-VALIDATION: Universal Modder "Goblins of War" note gives
    independent gamedb name-hash vectors. Verified
    name_hash("TribeDef_Goblin")==0xE2D0F0F6 and
    ("TribeDef_Machine")==0xEBB642C8 against our FOLD LUT —
    added as permanent test vectors in anduril-strdb.

## r81 — frames.bin = cooked interleaved pose grid; track order is the open bit

  capture_round5 (dump4.lua) captured per member:
  - `*_frames.bin` (0x8000) from `fb = *slot` of the src ring
    wrapper -> THE COOKED POSE GRID: all 51*20 records are valid
    {f32 pos4, f32 quat4}, quats unit, frame-to-frame smooth.
    == hkaInterleavedUncompressedAnimation-style output.
    So the ANIX cook product is a per-frame LOCAL/MODEL pose
    buffer in TRACK order; game interpolates between frames.
  - `*_dst.bin` captured the slot WRAPPER struct (deref level
    bug: dumped dsp not *dsp) - not pose data, ignore.
  - `*_asrc.bin` (0x800) = anim-src object: 0x50-stride records
    with sequential u32 ids (NOT name_hash values) + embedded
    {pos4,quat4} bind-pose-ish table at ~0x510.
  - Grid is NOT bone-order model-space (implied local lengths
    non-rigid under skel hierarchy at same index) => track order
    != skeleton order. name_hash-sorted bone order partially
    predicts measured rigid pairs (slots 12<->13 = R_Legl/R_Legu
    edge; 14<-15 = R_Arml<-R_Armu) but not all => CONFIRMED-ISH
    but incomplete.
  - Best-parent sd matrix + simulated annealing over bone->slot
    bijection (tools/anix_grid_map.py) gives a candidate map at
    cost 233 - unverified; annealer needs a ground-truth anchor.
  - NEXT: (a) pin ONE slot->bone anchor by correlating a decoded
    ANIX translation curve vs grid pos curves once the clip is
    identified; (b) or live-dump the bank NodeAndTrackName index
    (hash list gives exact slot order). The rotation decode itself
    is moot if grid == cooked output: we need file->grid, i.e.
    the per-frame decompression, not per-key interp.

## r82 — index = sorted name-hashes (VERIFIED); grid-space narrowed

  - CONFIRMED via olo_body_presentations.anix clips -v: the
    0x00ce66fc index block IS bone/track name-hash sorted.
    Track 8=0x32c64873="Torso", 13=0x3fb5b890="L_Foot",
    21=0x4d5b5cb4="Torso_Base", 26/27=L_Legl/L_Legu,
    28/29=L_Arml/L_Armu, 34=R_Shoulder, 37=R_Foot,
    44/45=R_Legl/R_Legu - all ascending in track order.
    Small-hash helper tracks lead (0x00937a36..38 sequential).
    => Track order = global name-hash sort, clips pick subsets.
  - Grid (frames.bin) hypothesis test: hash-sorted olog map
    scored WORSE than random-min on rigid-link sd (394 vs 338)
    for spine edges though limb-chain edges were low-sd
    (L_Arml<-L_Armu 2.78, L_Legl<-L_Legu 1.97, L_Foot<-L_Legl
    2.59 - all correct under that map). Mixed => either grid
    isn't pure model-space or spine links are genuinely
    non-rigid in this clip (stretchy rigs exist).
  - Bind-pose assignment test FAILED: grid pos don't match
    bind model pos (per-frame greedy unstable) - grid is NOT
    near-bind model space.
  - Neither local-offset space (slot mean pos ~(-15,0,15)
    cluster, not spread like bind local offsets).
  - Rendered grid as scatter: f45 side view = elevated
    12/13 pair + low main mass - creature-like, consistent
    with model-space in unknown frame.
  - Captured clip NOT olo_body_presentations (max 240 fr <
    observed cur84=311). Clip id still unknown.

  NEXT (revised ranking):
  1. LIVE DUMP (needs user OK): bank base + NodeAndTrackName
     blob + one clip's track map = definitive slot->bone.
  2. Offline: correlate translation-curve SHAPES: resample
     decoded BoneStream translations per frame, compare speed
     profiles |p(f+1)-p(f)| vs grid slots across candidate
     banks (frame-invariant, space-invariant).

## r83 — spine chain maps identically: slots 0-7 = bones 0-7

  - Rigid-link analysis (slot-local-length sd under identity map):
    the spine chain is rigid and SEQUENTIAL —
    slot1<-0 sd=12.1, 2<-1 sd=0.40, 3<-2 sd=3.2, 4<-3 sd=2.0,
    5<-4 sd=2.0, 6<-5 sd=1.5, 7<-6 sd=2.8 — matches the skel
    chain Null->Pelvis->Torso_Base->Torso->Upper_Torso->Spine4
    ->Neck->Head exactly. Grid IS model-space, slots 0-7 =
    skeleton bones 0-7.
  - Limb slots (8-19) are NOT rigid under identity map —
    implied local lengths oscillate 5->130 (too big for any
    stretchy rig). Limbs are permuted to other slots.
  - L/R x-sign doesn't hold either — the grid frame is rotated
    vs the skel bind frame (whole-character yaw/offset), so
    position-based side tests are invalid; only rigid links
    (rotation-invariant) discriminate.
  - pos.w = tiny 1e-6 residue: records confirmed {vec3 pos,
    unit quat} 32B, plain dequantized pose data.
  - Why spine rigid + limbs not: with model-space data the
    rigid-link test is unambiguous, so limb slots must map to
    different bones OR the clip remaps limbs through helper
    tracks. The 12<->13 pair (sd 10.7, symmetric) is the
    strongest limb link — likely R_Legl/R_Legu or a pair of
    same-limb bones.

  NEXT: two-anchor finish — either live NodeAndTrackName dump,
  or brute-force limb permutation under fixed spine (12! ->
  constrain by sd matrix, pick consistent chain).

## r84 — SOLVED: cooked grid is skeleton bone order +1 (slot s = bone s-1)

  - dst.bin records re-read: layout is {quat16, pos16} (NOT {pos,quat}
    like frames.bin); header 0x60; b0 = identity quat. hkQsTransform
    convention (rotation, translation).
  - DECISIVE MATCH (c2m10, cur84=5): every dst bone b's position
    matches grid slot b+1 at frames 5/6, worst err 2.7 — 19/19 bones.
    Cooked grid = [extra root track] + skeleton bones 0..18 in order.
    (c2m13/c2m12 windows stale/out-of-range — c2m10 alone is clean.)
  - Explains r83: the "rigid spine slots 0-7" was the extra root (slot0)
    + bones 0-7 spine chain — all rigid links were real; limbs were
    "non-rigid" under identity map because they were off-by-one.
  - pos matching works because dst pose = grid sample at cursor frame:
    BOTH are model-space... or both local — same space either way.
    Grid slots are model-space track outputs (positions 10-60 units,
    creature-shaped scatter) and dst = same space (interpolation
    explains residual <3).

  IMPLICATIONS:
  - The cooked interleaved buffer = skeleton order (for direct
    consumption), not name-hash order. Name-hash order is the FILE
    index layout; the cook reorders (or the clip's track table
    already encodes bone order).
  - frames.bin slot0 = extra node — likely the animation root/
    attachment transform (not a skeleton bone).
  - Track map for the 20-bone olog grid: slot s -> bone s-1
    (slot 0 = root/helper).

  NEXT: file->grid decompression. With track order known, compare
  decoded BoneStream keys per track against the grid's slot b+1
  curves on a candidate clip; residual is the dequantization.

## r85 — correlation attempts after slot-map solve

  - Resolved track->bone via name_hash on skel names: clip track 29
    hash 0x588cb073 = L_Armu (bone 9 -> grid slot 10). Its 16 keys
    ((120,48,+-36)) are real decoded translation data.
  - `pose` cmd can't validate clip identity: rotation_schedule still
    empty -> FK model positions wrong by construction. Verified
    empirically (mean err ~90 vs grid).
  - Single-curve shape correlation saturates (~0.99 on many slots):
    gross body sway dominates every track; can't discriminate clip.
  - Clip id still unknown; fortdefense_intro clip1 (330fr) is the only
    >=300fr candidate in the olog corpus; cur84=354 may count across
    loop wraps (frames.bin only captured the first 51 frames).
  - Variant-B files DO embed a cnt x K x 16B record grid
    (Clip::grid_off/seg_record_count) — K=14 for these clips vs 20
    runtime slots, so file records are NOT 1:1 runtime tracks
    (likely channel groups per the track_records table).

  NEXT: (1) correlate the file's OWN grid segment bytes (K*16B rows)
  against runtime 32B records — the 16B->32B dequantization, now that
  slot order is known; (2) or live dump of blobB base to pin keydata.

## r86 — capture_round6 = file<->runtime GOLD PAIR

  - manifest.jsonl records the CLIP PATH per member:
    all c2 members play
    characters\animationcharacters\orcs\orc_sync_chord\
      orc_sync_infightkill_victim.anix (extracted at
    target/tmp/anix_all/...). Clip identity: SOLVED for r6.
  - frames.bin here = 262144B = 409 frames x 20 slots x 32B,
    ALL frames valid {pos,unit quat}; all members share
    fb=0x2c8580040 -> ONE shared cooked buffer per clip set.
  - slot0 = constant identity record every frame — fixed root
    node; slots 1..19 = skeleton bones in order.
  - c5m0 dst ({quat,pos} @0x60) matches grid frame 74:
    19/19 bones, most err ~1.0, quats match (qdot_err <0.05).
    Slot=bone+1 CONFIRMED on a second skeleton. Limb bones
    11-14 err ~10-14 = mid-lerp between frames.
  - c2m0/c2m1 dst buffers have DIFFERENT content layout —
    those members' dst dumps are stale/other-state; c5m0 and
    c2m5 have the clean {quat,pos} form.
  - cur84 range 33..202 vs clip lens 145..165 -> cur84 is NOT
    the raw frame index into a 165-frame clip (likely a global
    or unwrapped clock).

  - BLOCKER found: `anix clips -v` decode is UNRELIABLE on this
    bank — most tracks emit garbage/empty keys (track70's "keys"
    are clearly the 8.8 time ramp misread as f32). The biped
    track-table layout verified on clip21 does NOT carry over:
    this bank's clip table/pool offsets differ. Next decode
    target: locate the shared 8.8 time index + per-track
    {n_keys,data_off} table in this bank's clip blocks, then
    the vec3 key pool -> resample -> compare to grid slot1
    (=first bone local, slot0 root identity makes it direct).

## r87 — round6 clip decode probing

  - Bank orc_sync_infightkill_victim.anix: 26 clips, <=187 frames,
    <=6.3s each. Runtime shared buffer = 409 frames -> NOT one clip;
    an arena of active clip grids (members share fb, each has own
    asrc/clip instance and own frame window). c5m0's dst matched at
    grid frame 74 -> its clip's cooked segment lives there.
  - Clip prologue subs[1]=6384 region holds a couple f32 vec3s then
    zero pad — sparse, not the dense key pool. Bulk data = the
    16B-record region (markers fe df ff 03) — still compressed.
  - Track table confirmed again: 8B {count, data_pos, 0, cum} at
    sent+0x44 area; data_pos = start ms (<=duration, 33ms deltas),
    counts 12..77 = key-event counts per channel GROUP (12 groups
    for clip1, cum chain to 444; terminal record flags&4|8).
  - The `anix clips -v` BoneStream decode is wrong for this bank —
    garbage keys; needs the group->stream resolution fixed against
    this layout before pose extraction works.

  NEXT: map each group to its channel data inside the 16B-record
  region (grid slot count 20 vs 12 groups -> groups are per-channel
  not per-slot; probably {pos,quat,scale} channel groups across
  tracks). The time index at sent+136 continues past the track
  table — u16 ramp of 8.8 times presumably shared.

## r88 — the 16B frame-record region decoded structurally

  - Clip1 region contains a dense record array: 152 rows x 19 recs
    x 16B at 304B/row (clip1=150 frames; rows = frames). Region
    rel 0x1b7c..0xcc7c (~45KB) = the BULK of the clip.
  - Row rec0 always opens with const bytes `fe df ff 03 ff ef ff fd`
    (4 const s16) + 3 varying s16 + const -5 -> a bone with constant
    local translation and rotating quat. NOT a tag — a real bone
    record; earlier "fe df ff 03 marker" = record0's const head.
  - All other 18 records vary in all 16 bytes.
  - Rec tail = s16x4 quat (norm~0.95 for rec0) BUT does NOT match
    any runtime grid quat directly (mean|qdot| ~0.25 over all
    windows) -> delta-compressed (needs per-track scale+offset, cf.
    embedded hkaDeltaCompressedAnimation) OR local-vs-model space.
  - Grid is model-space; file recs are almost certainly LOCAL
    transforms. Rec0 const pos fits a rigid local bone whose model
    pos still moves. To verify packing need local runtime values —
    derivable via inv(qparent)*qchild once hierarchy per slot is
    solved; local-pos variance solve on grid gives candidate parents
    (s1<->s2 sd0.5, s2<-5, s5<-2, s6<-5, s7<-6, s8<-7...).

  NEXT: (a) solve slot hierarchy from grid (constant-local-pos
  constraint), compute ground-truth local quats, then match file
  rec s16s allowing per-track linear dequantization; or (b) find
  the decompressor in Ghidra — search for the const signature
  bytes/structure reads.

## r89 — file rec quat != runtime local quat; arena offset unsolved

  - Recovered context: ../formats/detail/anix.md already established the
    record decode (d0=rol8 s32, d1=rol24 s32, u16@8/@10/@14 smooth
    channels; marker = LAST record's u16@12/@14 tag). Records plausibly
    = quat packed at mixed precision; translations in separate f32
    vec3 pools ("Variant-B key blobs: N x f32 vec3 pure keys").
  - Greedy parent solve on grid (sd-min): tree {0->17->6->5->{2,3,4},
    6->7->{8,9},8->11,2->1->18,4->{10,19},10->15->14,9->12->13,3->16}
    — low-sd edges plausible, high-sd edges dubious (animated trans).
  - Correlation test FAILS as posed: local quats computed under that
    tree do NOT match file record channel curves under slot=rec+1 at
    any arena offset (0 hits @|corr|>0.8). Per-pair z-corr hits
    (slot6~rec6 ch0 -1.05, slot7~rec2 -1.0) may be coincidental
    smooth-curve matches or right-pair/wrong-window.
  - Interpretation: too many unknowns at once (hierarchy, arena
    offset, per-track dequant, rec<->slot). Direct file->runtime
    diff needs the DECOMPRESSOR code path.

  NEXT (static): find the 16B-record reader in Ghidra — it writes
  the 32B {pos,quat} grid; look near anim-player code for loops
  with stride 0x10 and the 0x03ffdffe/0x1f tag-bit constants, or
  the hkaDeltaCompressedAnimation construct/decompress helpers
  (hkaQuantizedAnimation::decompress style).

## r90 — REFLECTION GOLDMINE: cooked format is stock Havok 2016

  - Exe embeds the full hk2016_2_0_r1 hkReflect database
    (src paths E:\proj\Kraken\code_streams\External\hk2016_2_0_r1\...).
    Type names + member names + versioning strings all present.
  - Type-name registry table found at VA ~0x1426f28xx:
    {name_ptr, record/cluster ptrs} chains; nearby string tables
    at 0x1422dfXXX/0x142209XXX hold MEMBER names.
  - Animation member vocabulary recovered (file 0x2207700 region):
    referencePose, numStaticTransformDOFs, numDynamicTransformDOFs,
    maxBitWidth, qFormat, offsetIdx, blockIndexSize, blockIndexIdx,
    dataBuffer, scaleIdx, preserved, staticDOFsIdx, numberOfPoses,
    staticMaskIdx, blockSize, quantizedDataSize, quantizedDataIdx,
    extractedMotionMapping, twistAxis, memOffsetToAngleOffset,
    mTrackToBoneIndices, floatTrackToFloatSlotIndices,
    referenceFrameSamples, relativeToEndOfClip...
  - => cooked ANIX data = hkaDeltaCompressedAnimation/hkaQuantizedAnim
    BIT-PACKED DOF STREAM: static DOFs = fixed values, dynamic DOFs
    packed at per-track bit widths inside blocks w/ per-block
    offset+scale (qFormat.maxBitWidth). The "16B records" are just
    16-byte windows of that bitstream — explains the rol8/rol24
    field rotations (bit-aligned, not byte-aligned data) and the
    `0x1f` tag bits (block headers, not record types).
  - Field names confirm structure, not layout order yet — but this
    IS stock Havok: format semantics can be taken from the Havok
    2016 SDK docs/source conventions rather than re-derived.

  NEXT: implement hka delta-decompression per Havok semantics:
  read qFormat (maxBitWidth, per-DOF bit width table), per block:
  offset+scale + bitpacked DOF values -> local TRS -> FK to model
  grid. Verify against round6 frames.bin ground truth.

## r91 — quantized-anim hypothesis + reference research + negative results

  - hkaQuantizedAnimation spec (public Havok docs/community):
    m_data = header {headerSize, numBones, numFloats, numFrames,
    duration, numStatic{Trans,Rot,Scale,Float},
    numDynamic{Trans,Rot,Scale,Float}, frameSize,
    staticElementsOffset, dynamicElementsOffset,
    staticValuesOffset, dynamicRangeMinimumsOffset,
    dynamicRangeSpansOffset} then frame-major quantized data.
    Static rot = 48-bit quat; dynamic elements = 16-bit values,
    value = rangeMin + q*rangeSpan/65535; element-index tables map
    each element to (track, component).
  - Prior art found: Mercenaries-Fan-Build/mercs2-wad-simulator —
    Rust wavelet decompressor re-ported FROM the retail exe's own
    decomp (their python oracle was wrong). NOTE: even they left
    delta compressed unimplemented — "not publicly documented".
    Their quantization-descriptor fields match our member list
    (maxBitWidth u8, preserved u8, numD u32, offsetIdx, scaleIdx,
    bitWidthIdx at qFormat+0/1/4/8/12/16).
  - NEGATIVE: lane-level correlation of the orc bank's dense grid
    vs round-6 ground truth found ZERO |r|>0.99 matches (all 19
    records x 8 u16 lanes x s16/u16 x raw/cumsum x 8 gt comps x
    all windows). File values are NOT affine-mappable per u16 lane
    onto output comps => records are bit-packed DOF streams, bits
    cross byte lanes (matches delta format, not u16-lane quantized).
  - NEGATIVE: no Animation/*.cpp assert strings in exe (Havok anim
    asserts stripped; only Base/Cloth/Physics/Ai unit tests carry
    assert paths). The type-name string xref lands in a unit test
    (FUN_141406b48 = stringformat test) — string-pool dedup means
    the registrar ref is elsewhere.
  - Reflection registry confirmed: type-name->record table at VA
    0x1426f28xx; member-name strings at 0x1422dfXXX / 0x142209XXX.

  NEXT best anchors:
  a) Walk the type record for hkaDeltaCompressedAnimation: its
     hkReflect member table (name+offset pairs) gives the runtime
     object layout (m_data ptr, m_blockSize, ...). Useful for
     reading a LIVE object's dataBuffer base = the file bytes.
  b) The decompressor itself is reachable only via vtable of the
     cooked animation object — needs one live read of the anim
     object (vtable ptr -> VT slot -> decompress fn). Short
     read-only dump remains the decisive shortcut.
  c) Offline: model the bitstream as delta-blocks {offset f32,
     scale f32, bitpacked values}; solve per-DOF bitwidth from the
     block structure and validate vs c5m0 slot1 (root, local==model).

## r92 — hkaDeltaCompressedAnimation member vocabulary recovered

  - PE section map fixed: file off 0x1e843d0 -> VA 0x141e85dd0
    (.xtext, not the earlier mis-mapped 0x141e843d0).
  - Reflection name table entry for hkaDeltaCompressedAnimation
    found at VA 0x1426f2748 (file 0x26f0748). The table lists
    type names + member names in registration order:
      hkaDeltaCompressedAnimation, endian, totalBlockSize,
      staticDOFsSize, maxFramesPerBlock, staticMaskSize,
      blockDuration, transformOffsets,
      hkaDeltaCompressedAnimationQuantizationFormat, blockOffsets,
      numBlocks, lastBlockSize, extractedMotion, duration,
      skeleton, boneFromAttachment, annotationTracks, attachment,
      mappings, ... numberOfFloatTracks, trackName, blendHint,
      hkaAnimatedReferenceFrame, referencePose,
      floatTrackToFloatSlotIndices, parentIndices,
      transformTrackToBoneIndices, numberOfTransformTracks ...
  - Field semantics: delta anim object = {endian, totalBlockSize,
    numBlocks, blockDuration, maxFramesPerBlock, lastBlockSize,
    staticMaskSize, staticDOFsSize, transformOffsets[],
    blockOffsets[], qFormat, dataBuffer...} + hkaAnimation base
    (type, duration, numberOfTransformTracks, numberOfFloatTracks,
    extractedMotion, annotationTracks).
  - transformOffsets = per-track byte offset into each block's
    packed data; blockOffsets = per-block byte offset into
    dataBuffer. THIS is the decode key for the dense grid:
    block b at dataBuffer+blockOffsets[b]; track t's bitstream
    inside the block at transformOffsets[t]; DOF bit widths from
    qFormat bit-width table.

  NEXT: map ANIX clip sub-blocks onto {staticMask, staticDOFs,
  transformOffsets, blockOffsets, dataBuffer}. The 48B clip header
  {v1[3],v2[3],0,0,0,duration_ms,cnt,pad} likely carries
  {bounds?, duration, numBlocks}; the u16[cnt] array = per-block
  frame counts; the dense region = dataBuffer blocks.

## r93 — clip internal layout mapped (orc_sync_infightkill_victim clip1)

  Clip1: sent=0xb60, subs=[136, 6384, 52304, 52320], dur=5033ms,
  150 frames, count=20, bones=55.

  Layout (offsets rel sentinel):
    [+4, +136)      channel-group track table: 12x {count,
                    data_pos_ms, flags, cum} + terminal {7,0,12,cum}
    [+136, ~+6256)  element-index table: u16 sequential ramp
                    0..~55+ with flagged/non-sequential entries —
                    static/dynamic DOF index map (cf. Havok
                    staticMask + element indices)
    [~+6220..6384)  f32 vec4 runs: per-track quantization
                    range/reference values (repeating bounds-like
                    triples, e.g. (-0.3,-2.7,9.4))
    [+6384, +52400) data buffer:
                    +0x00.. ~40B: f32 range tail continuing
                    ~+0x28:  {u32 dur_ms=5033, u32 frames=150,
                              u16 0, u16 100(?maxFramesPerBlock)}
                    +~0x30:  u16[150] frame-time deltas (33/34ms,
                             30fps; sums to duration)  [VERIFIED]
                    +~0x200: another ~300B u16 table (abs ms times?)
                    +~0x62c: dense grid begins: 150 rows x 304B =
                             19 rec x 16B per frame  [VERIFIED]
    [+52304,+52320) 16B trailer
    [+52320, next)   ms time table / trailing data

  The 304B/frame record rows are per-frame fixed-size => this
  variant stores UNIFORM 16B-per-track-per-frame records (not
  variable-bit delta blocks). Delta semantics still apply per-DOF
  within each record (bit-packed inside 16B).

  NEXT: correlate the element-index ramp + range tables against
  track DOF counts; then bit-read each 16B record as packed DOFs
  and fit vs c5m0 ground truth (slot1 = local==model for root).

## r94 — exhaustive negative: no byte-aligned field matches runtime

  Sweep: all 18 marker-delimited dense-grid runs in the orc bank
  (75..187 frames each), every byte position 0..303 of each 304B
  row as u16 LE (aligned + odd offset), std-filtered, correlated
  (sliding-window, all arena offsets) against all 160 ground-truth
  curves (20 slots x {quat4,pos4}) of c5m0_frames.bin — RAW and
  frame-DIFF modes.

  Result: max |r| ~0.016. Nothing. Conclusion: the per-frame
  record data is bit-packed at sub-byte granularity (per-DOF
  bit-width streams per hkaDeltaCompressedAnimation semantics).
  Byte-lane analysis is exhausted; only the bit-level block
  format (offset+scale+packed values per DOF per block) remains.

  Implication for next session: stop trying to match raw bytes to
  floats. Either (a) get the decompressor from the exe/live vtable
  (reads dataBuffer bit-exactly), or (b) implement the documented
  delta-block layout {per DOF: f32 offset, f32 scale, N-bit packed
  values} and iterate bit widths against ground truth. (a) is far
  cheaper — recommend the short read-only live dump.

## r95 — sentry clip [sub0,sub1) region decoded (biped variant)

  sentry.anix clip2 (sent=0x92f0, subs=[64,2256,43664,43760,43776],
  171 frames):

  [sent+64, sent+2256) layout VERIFIED by direct byte read:
    +0x00  channel-group descriptors {count u16, data_pos_ms u16}:
           (23,1066) (28,2666) (27,3400) (31,0) — identical values
           to the sent+4 track_records — then terminator {4, 816}
           (flags=4, cum key total).
    +0x24  shared BE-u16 time ramp: ~171 entries, smooth monotonic
           0x004e..0x25ff (~9472ms quanta = ~55ms/frame? or ticks),
           ends with 00 00 pad.
    +0x17c f32 key pool begins: vec3-scale floats in [-2,2]
           (translation/rotation channels, 4B aligned).
           f32 runs are interleaved with u16/u32 fields — the pool
           is per-channel-group structured, not one flat array.
    Prior "s16 quat-word runs" in this region were f32 pool
    false positives — scratch that interpretation.

  Orc bank ([136,6384) region) is DIFFERENT: element-index ramp +
  f32 range bounds + u16 frame-time deltas + dense 16B grid —
  two distinct cooked layouts coexist (quantized anim on sentry,
  delta-packed on orc?). Bank variants differ in cooked payload.

  NEXT: sentry pool structure — find per-channel {key_count,
  key_offset} and test whether keys are vec3 pos or quat sets;
  orc bank remains the delta-format frontier (needs bit-level
  decode or live vtable read).

## r96 — unified cooked layout across banks (variant-B grid confirmed)

  sentry.anix clip2 (sent=0x92f0): the [sub1,sub2) data buffer is
  IDENTICAL in shape to the orc bank's:
    +0x00  f32 bounds vec3 pair {(72.0,26.1,3.9),(72.3,26.4,4.17)}
           — then a third partial vec3 (72.6,26.66); likely
           track-range mins/maxs (pos extents ~72u? or motion bounds)
    ~+0x20 {u32 dur=5666, u32 frames=171, u16 0, u16 100}
    ~+0x2c u16[171] frame deltas (33/34ms — 30fps) [VERIFIED]
    then  second ~350B u16 table (absolute ms times)
    then  dense grid: marker sig `34 fc bf ff f7 ff f0 ff
          fe df ff 03` spacing 240B = 15 rec x 16B [VERIFIED]

  Grid: 171 rows x 15 rec x 16B. Byte-constancy map over ALL
  frames: rec00 = {8B const signature, 8B varying}; recs 1-14 vary
  in every byte (earlier "static lanes" were adjacent-frame
  coincidence). So ~15 dynamic transform tracks per frame
  (count=20 minus statics/helpers).

  Both banks therefore share: dataBuffer = {bounds?, dur, frames,
  maxFramesPerBlock(100?), frame-delta u16[n], ms table, dense
  frames x tracks x 16B packed grid}. Per-record 16B carries the
  track's quantized transform for that frame, bit-packed (byte-
  lane correlation exhaustively ruled out, r94).

  The 16B record almost certainly = Havok 128-bit packed
  transform: pos quantized + quat (smallest-3 or 4x) + flags,
  with the `fe df ff 03`-family tag = the `sel`/flags field.
  Dequant range likely = the f32 bounds vecs at buffer head +
  per-track ranges in the [sub0,sub1) metadata region.

  NEXT (unchanged priority): decode the 128-bit record field
  split — try pos=3x~20b + quat 3x16b + scale + tag bitfields,
  then validate vs ground-truth LOCAL transforms (needs skel
  hierarchy; orc_base skel + slot map already known).

## r97 — bit-level record analysis + why all correlations failed

  Bit-flip profiles across 171 frames (sentry clip2, 15x16B recs):
  every record shows repeating low-to-high flip ramps — classic
  fixed-width quantized DOF fields, ~8-16 bits each, LSB-first.
  ~10 fields/record fits pos3 + quat3/4 + scale3 in 128 bits.
  rec00 = 64-bit constant signature + 2 short fields (the `fe df
  ff 03` header IS a record — likely a frame/anchor tag).
  Many raw fields are SMOOTH (raw-smooth ~170 = max) => absolute
  quantized values, NOT deltas (despite the delta type name).

  WHY ALL CORRELATIONS FAILED (important reframe): the runtime
  arena `frames.bin` is a SHARED buffer across members playing
  DIFFERENT clips of the same bank; AND file data is LOCAL
  transforms while the arena grid is MODEL-space (dst matched it
  verbatim, and dst is model-space). So file->runtime comparison
  needs BOTH (a) clip identity per arena segment AND (b) the
  skeleton FK compose. Never had both — no wonder byte-lane corr
  was zero. It was never going to work.

  Path forward (concrete):
  1. Field-split a record: find contiguous bit-runs of smooth raw
     fields (~10 per record). Decode: quantized -> range bounds
     (f32 vec pairs at data-buffer head / metadata region).
  2. LOCAL transforms per frame per track.
  3. FK compose with orc_base skel (hierarchy known) -> model grid.
  4. Compare vs frames.bin arena segments — corr now meaningful.
  5. Clip identity: whichever file clip's decoded grid matches a
     contiguous arena segment self-identifies.

  tools/anix_bitscan.py: bit-level field scanner (flip profile +
  raw/cumsum smoothness candidates) — committed this session.

## r98 — FULL decompressor semantics recovered (HavokLib reference)

  Located PredatorCZ/HavokLib `source/hka_delta_decompressor.cpp`
  (GPL — reference only, NOT vendored into repo). Exact semantics:

  * Quantized data stream layout: FOR each block (blockSize frames,
    numBlocks = ceil(frames/blockSize)) FOR each dynamic track:
    `numPreserved` x f32 seed values, then ceil(bitWidth*(numItems-1)/8)
    bytes of bit-packed unsigned values, LSB-first within u32 words
    (values cross u32 boundaries).
  * Dequant is CUMULATIVE DELTA: v_i = v_{i-1} + offset +
    q_i * scale/(2^bitWidth - 1); block seeds with preserved f32(s).
    => raw-field smoothness in r97 was misleading; correct model is
    cumsum + per-track (offset,scale) f32 pair.
  * StaticMask = u16/track: bits0-1 pos type, 2-3 rot, 4-5 scale
    (0=dyn,1=static,2=identity); bits 6..15 = per-component flags
    posZ,posY,posX,rotW,rotZ,rotY,rotX,scaleZ,scaleY,scaleX
    (1=component in dynamic stream, 0=static f32 in static buffer).
  * Quat special case: W == +-2.0 marks smallest-three encoding —
    W = +-sqrt(1 - x^2 - y^2 - z^2), sign preserved.
  * Serialized members (classgen): numberOfPoses u32, blockSize u32,
    qFormat{u8 maxBitWidth, u8 preserved, u32 numD, u32 offsetIdx,
    u32 scaleIdx, u32 bitWidthIdx}, quantizedDataIdx/Size,
    staticMaskIdx/Size, maskDOFsIdx/Size, [numStaticTransformDOFs,
    numDynamicTransformDOFs (HK600+)], totalBlockSize, lastBlockSize,
    dataBuffer[]. Index fields = byte offsets into dataBuffer.

  IMPORTANT: the ANIX clip does NOT serialize this object verbatim —
  no qFormat/u32 numFrames signature found in the victim bank. The
  file is WB's own cooked repacking, but the dequant semantics still
  apply. Confirmed cooked buffer header at clip1 (file off 0x2474):
  `{u32 dur_ms=5033, u32 frames=150, u16 0, u16 blockSize=100}`
  preceded by per-track f32 range/reference vec3s and zero pad.

  NEXT: interpret the metadata region ([sub0-sentinel .. buffer]) as
  the offset/scale/staticmask/static-DOF tables, then apply the
  cumulative-delta dequant over the 304B/frame grid. The '16B record'
  reading may still be right (per-track-per-frame packed DOF group)
  OR the stream may be block-major — test both against decoded
  smoothness once (offset,scale) assignment is known.

## r99 — frame grid exact + delta-field structure decoded

  GRID PINNED (orc victim bank clip1, file abs offsets):
  - cooked buffer @0x2450: f32 range vecs -> 16B zero pad ->
    {u32 dur=5033, u32 frames=150} @0x2474, {u16 0, u16 blockSize=100}
    @0x247c, u16[148] frame-ms deltas (33/34, sum~4933) @0x2480
  - dense grid @0x2600: 150 rows x 304B = 19 recs x 16B, frame-major
  - marker rec13: bytes 12-15 = `fe df ff 03` EVERY frame (150/150)
  - marker rec14: ends `fe df ff 13` in 71/150 frames
  - grid span 45600B + ~528B pad to next clip @0xda30

  FIELD STRUCTURE (bit-flip profiles, this session):
  - animated records (15-18 etc) show field boundaries at multiples
    of ~13 bits: ~9-10 DOF fields of 12-14b each in 128 bits
    (fits pos3 + rot4 + scale3 = 10 DOFs, or pos3+rot3+scale3+sel)
  - rec13/14 = partially-constant records (static tail bytes) —
    the constant `fe df ff` tail = static DOF fields, not a tag
  - cumsum-dequantized fields are smooth (std(d2)/std(d1) ~0.11-0.29
    vs ~0.9 raw) => delta coding WITHIN the record stream confirmed
    (per r98 semantics: v_i = v_{i-1} + offset + q*scale/(2^w-1))

  OPEN: exact per-record field boundaries/widths, signed-vs-offset
  delta convention, and the (offset,scale) f32 table assignment.
  Two interpretations still live: (a) 9-10 DOF fields/record
  frame-major (WB-cooked), (b) block-major hka layout. The 304B
  arithmetic regularity strongly favors (a).

## r100 — exact field boundaries; per-track layouts differ

  Flip-edge segmentation on clip1 grid gives consistent field starts
  at bits {0,13,26,42,55,68,84,97,110} across animated records —
  i.e. 9 fields of widths 13,13,16 (x3 groups) + ~5b tail.
  Hypothesis: pos{13,13,16} + rot{13,13,16} + scale{13,13,16}
  with a 2-bit smallest-three `sel` in the tail.

  BUT: active-field sets differ per record (rec13: fields at
  26,42,55,68 only; rec14: 42,55,68; rec18: lacks 84,97) =>
  per-track StaticMask gates which DOF slots are populated, exactly
  per r98 semantics. Records are NOT a homogeneous 10-DOF pack.

  Quat invariant test (range-normalized cumsum, smallest-3): only
  rec12[6:9] (0.8) and rec18[0:3] (0.73) partially validate.
  Still missing: real (offset,scale) per DOF — observed-range
  normalization is too crude; need metadata f32 table assignment.

  METADATA REGION [0xbe8,0x2450): element table ~150 u16 entries
  {idx u8, tag u8}, 99 distinct idx, 51 tagged extras; then f32
  blob (~1400+ floats — more than offsets+scales need; probably
  static transform values AND ranges). Need the clip descriptor to
  get numTracks/numDynTracks to carve these arrays exactly.

## r101 — clip header decoded

  Clip1 header @sent(0xb60), verified field-by-field against track
  table and clip2 (@0xda30, same layout):
    +0x00 u32 2 (version), +0x10 u32 1, +0x20 u32 1
    +0x24 u16 20 = numTracks (matches 19 recs + identity)
    +0x26 u16 0x88 flags
    +0x28 u16 6384 = sub1 (data buffer) offset
    +0x2a u16 52304 = sub2 offset
    +0x2c u32 52320 = clip size
    +0x30 u32 0x2b99dbc1 = clip ID hash (same in every clip)
    +0x38 u16 13 = channel-group count (+1), u16 99 = num elements
    +0x40.. u16[13+1] quad rows {0, cum, count, pos_ms} — the SAME
      channel-group data as the u32 table at sent+4, duplicated in
      u16 form. Rows 0-8 verified identical to parser track_records
      (cum=77/89/159/184/220/244/256/328/343, count, pos match).

  NOTE: channel groups ARE per-track timing channels: 12 groups,
  cum total 444 events across the clip. 'pos' = start time in ms.
  These are sparse event channels SEPARATE from the dense grid —
  probably the spline/keyframe timing layer referenced by the
  element table tags.

## r102 — offset-binary discovered; runtime corr needs clip-id first

  Field value distributions: 13-bit raw values cluster at ~4096
  (= 2^12, mid-range) => OFFSET-BINARY coding, zero at 2^(w-1).
  rec13/14 static fields = exactly 0xFFF/0xFFFF -> the `fe df ff`
  bytes are saturated-ones fields. rec09 (near-static track):
  all 9 fields constant ~4096+-30 => constant near-zero deltas.

  Runtime side: c5m0_frames.bin = 51 frames x 20 slots x 32B
  {pos vec4, quat vec4}, slot0 = identity root every frame.
  BUT c5m0_asrc.bin contains clip descriptor with dur 13166ms —
  the member plays a MULTI-CLIP SEQUENCE (~395 frames), not one
  bank clip. So single-clip windowed corr vs the arena is
  structurally impossible for this capture: must slice the arena
  by clip segment boundaries first (frame-time deltas or
  discontinuity detection on slot1's pos curve).

  NEXT best offline step: decode rec0 -> candidate quat/pos via
  offset-binary + smallest-three (W=sqrt(1-x2-y2-z2)), FK-free
  for slot1, then slide along the WHOLE c5m0 arena (51 frames
  only covers part; earlier 409-frame arena.bin captures are the
  real targets). Alternatively crack the static-mask location so
  field->DOF assignment becomes deterministic.

## r103 — offset-binary decode model validated on frame 0

  Encoding model (working): each 16B record = per-track per-frame
  pack of 9 DOF fields at bits {0,13,26,42,55,68,84,97,110},
  widths 13,13,16 x3, tail ~5b. Values are OFFSET-BINARY
  (v = (q - 2^(w-1)) / 2^(w-1), range [-1,1] for quat comps).

  Evidence:
  - Static/near-static fields sit at ~4096 (=0.0) or 0xFFF exactly
    (saturated); rec13's `fe df ff` marker bytes = const 0xFFF
    fields. rec09 fully static track = all 9 fields constant ~4096.
  - Frame-0 spot check vs runtime slot1 quat (-0.003,0.154,-0.006):
    rec16 field6 = 4084 = EXACT encoding of -0.003 in 13-bit
    offset-binary. Real signal, not noise.
  - f32 metadata region [0xd50..0x2450) = smooth vec3 TRAJECTORIES
    (monotonic motion values, 0.0-padded triples) — preserved
    reference/key data, NOT offset/scale tables.

  Still open: which field triple is rot vs pos vs scale per record
  (quat-domain check is nearly vacuous for small components);
  whether DOF range is per-track (needs the table) or fixed;
  c5m0 rec<->slot identification.

  NEXT: for each record classify fields by temporal activity —
  scale fields should be ~constant 1.0 (q~mid or constant
  saturated), pos mostly static, rot varying; use rec16 as
  reference and diff records to assign semantics.

## r104 — DECODE WORKS. rot = fields 6,7,8 (smallest-3 quat)

  Record layout confirmed empirically on clip1 frame0 vs c5m0
  runtime slot quats: fields {6,7,8} = quat xyz, offset-binary
  v=(q-2^(w-1))/2^(w-1), w=13,13,13 (fields 6,7 are w13, f8 w13;
  the 16b slots are f2,f5). Reconstructed W = +sqrt(1-|xyz|^2).

  Match quality: recs 8,9,11,12,13,14,15,16,18 hit |dot|>=0.97 vs
  runtime slots (frame0). Near-identity local quats (recs 9,11,13,
  14,15,16 ~identity) matched slot0's identity — bones with small
  local rotations, correct behavior.

  Now obvious: most bones have near-identity LOCAL rotations in
  this clip's frame0 pose (victim hang/throw); the few animated
  ones (rec0 x=-0.8, rec5 y=0.93, rec17 x=-0.96) are the movers.

  Remaining: (a) rec<->bone mapping needs the skeleton's bind
  pose to disambiguate near-identity locals — trivially solvable
  now that rotation decodes; (b) pos/scale field assignment:
  fields {0,1,2} and {3,4,5} remain — pos likely one triple
  (scale the other, ~identity=4096 expected... but rec fields
  0-5 vary, so scale may be absent/sparse per static mask);
  (c) frame-1+ tracking to confirm temporal correctness.

  The 15-byte mystery is over: 16B = 9-DOF offset-binary pack,
  13/13/16 fields, quat last. anix pose can now emit real
  rotations pending (a)/(b).

## r105 — grid origin anchored: marker = rec0 tail; Rust decoder landed

  Grid origin correction: the dense grid's true row start is where
  EACH row's record-0 ends in `fe df ff 03` (bytes 12..16 of rec0).
  On orc victim clip1: marks at 304B stride, all 150 rows hit at
  row+12 -> grid_off = file 0x26d0 (a 0x26a0), NOT 0x2600 — r104's
  origin was shifted by 13 records; byte decode identical, only
  record indices permute. Second marker `fe df ff 13` sits at
  row+28 (rec1 tail) on ~69/150 rows = a second static record.
  This unifies with the sentry bank's 12B const + marker = rec0.

  Detection must tolerate stray `fe df ff 03` collisions inside
  row data (diffs 80/224/16 observed): stride = modal inter-mark
  gap; anchor = mark-12 maximizing per-row marker hits.

  RUST DECODER LANDED (clean-room, no GPL source used):
   - `GRID_FIELD_OFFSETS`/`_WIDTHS` = {0,13,26,42,55,68,84,97,110},
     widths 13/13/16 x2 + 13/13/13, offset-binary dequant.
   - `Clip::grid_quat`/`grid_quats`/`grid_frames`,
     `Anix::sample_grid_rotations` -> (rec_index, quat) pairs.
   - Fields 6-8 = smallest-three quat xyz, W=+sqrt(1-x2-y2-z2)
     clamped >=0. Sel bits not yet located (W-dominant only).
   - CLI: `anix <path> grid <clip|name> [frame]` dumps per-record
     quats; verified live on orc victim clip1 (19 recs, 150 fr).
   - Tests: synthetic offset-binary encode->decode roundtrip,
     saturated-record stability. `cargo test -p anduril-anix` green.

  TODO next: rec->bone order via bind pose; fields 0-5 pos/scale
  semantics + static masks; sel bits for non-W-dominant quats;
  wire grid rotations into `sample_rotations` once bone map lands.

## r106 — track hashes resolve to orc_base bones; rec->bone still open

  Verified the clip's `bone_hashes` list (block 0x00ce66fc) against
  orc_base.skel: every hash resolves via `name_hash` to a real bone
  (R_Index1-3 -> tracks 0-2, L_Weapon -> t5, L_Shoulder -> t8, ...,
  55 tracked bones total). So stream `track=i` == sorted-hash index,
  and the skeleton assignment is confirmed for clip1.

  rec->bone attempts this round:
   * bind-pose LOCAL quat match at frame0 vs all 100 skel bones:
     degenerates — ~half the records are near-identity locals which
     all collide onto identity bones (Null/attach).
   * same vs only the 55 tracked bones: same degeneracy.
   * fields 0-5 as pos candidates: saturated near +/-1 under the
     [-1,1] offset-binary read — they are NOT plain normalized
     translations; they need the per-track (offset,scale) range
     tables from the metadata region (f32 pairs) or a different
     interpretation. Pos decode stays open.

  Verified anchors from r104 stand: rec8->slot16, rec12->slot19,
  rec18->slot14 (|dot|>=0.97). If runtime slot index == skel bone
  index these are L_Index2/L_Middle2/L_Thumb3 — but slot->bone was
  never verified, so treat as UNVERIFIED.

  Next best lead: the u16 element-index ramp at metadata+~0x30 has
  sequential values with tagged interleaved entries (e.g. 0,1,2,
  0x6A02,3, 0x8303,4,...) — plausibly the transformTrackToBoneIndices
  / static-mask table the decompressor needs. Decoding its entry
  grammar gives rec->bone directly.

## r107 — element-table grammar cracked: indexed entries + tagged data bytes

  Metadata u16 ramp decodes as a self-indexing list (clip's
  "element count" = 99, entries 0..98). Byte-level grammar:
  each element = u16 `{idx, 0}` head, then ZERO OR MORE tagged
  continuation u16s `{idx, data}` — i.e. low byte echoes the
  index, high byte is a data byte. So each element carries a
  variable-length data-byte list (0-4 observed; el 98's trailing
  `ff ff` pair may be a table terminator).

  Element data-byte counts correlate loosely with animation
  activity (empty elements on static/silent channels), and every
  grid record on clip1 is nearly fully animated (15-16 varying
  bytes of 16 over 150 frames — static fields are the EXCEPTION
  on this clip, mostly rec0/rec1 tails and rec15/17 partials).

  Semantics of the data bytes unresolved — candidates:
  per-channel static masks, stream selectors, or bit-packed
  transformOffsets. 99 elements vs 19 grid recs vs 55 tracks vs
  13 channel groups vs 444 channel-group entries: no integer
  match, so elements are probably DOF- or key-level, not
  track-level.

## r108 — metadata region content map refined

  Post-element-table region [sent+136+300B on]: f32 vec3 trajectory
  data, not quantization ranges. Big contiguous sane run (~130 vec3s)
  reads as smooth curves — X≈0, Y monotone 158->93->254, Z -5..255.
  These are the "preserved f32" / reference-motion values the Havok
  delta scheme seeds blocks with, in cooked form.

  Element-count survey across all 27 clips in the victim bank:
  elems (40..128) do NOT equal grid K (18-19) — elements are
  DOF/key-level, not track-level. Element-table entry = {idx,0}
  head + {idx,data}* continuation bytes; parsing consumed exactly
  150 u16s on clip1 before the f32 region starts.

  K=19 grid detection worked on most clips via modal-stride marks;
  a few clips report K=0/1 (marker absent or grid variant — flagged
  for follow-up: e.g. sent 0x29a40, 0x87370 parse as animated but
  no marker run found; probably a different record signature).

  Verified naming of tracks holds for all resolved hashes.

## r109 — decoder generalizes; sel still unlocated

  `anix grid` verified on a SECOND bank (bip_amb_sentry, K=12
  records/frame): parses cleanly, emits sane unit quats —
  decoder is not clip-specific.

  Smallest-three `sel` hunt: the 5-bit record tail (bits 123-127)
  is NOT the omitted-component selector — constant per record
  (0..3) regardless of whether |xyz|>1 at any frame. Per-record
  dominant-component varies (recs 6,7 on sentry clip exceed |q|>1
  in some frames -> those records' stored triple omits X/Y/Z, not
  W). Continuity self-selection test was inconclusive (>1 dots on
  saturated recs suggest wrong stored-component order, not noise).

  Where sel must live: NOT in the record tail. Candidates left:
  inside fields' reserved high bits, or in the element-table data
  bytes (r107) as a per-DOF track property. Since sel is fixed per
  Havok encoding scheme, ONE consistent rule should apply — likely
  "the stored triple is always the same 3 of 4 components for a
  given record" rather than per-frame.

  Trajectory-correlation attempt for rec identification: no field
  0-5 curve correlates >0.85 with the metadata f32 trajectory
  block — pos isn't directly those curves, or needs the range
  tables first.

## r110 — per-record field semantics confirmed; quat slot moves

  Unit-triple test on sentry clip1 (which triple stays |q|<=1 over
  all frames): most records keep the quat at f6-8 as on the orc
  clip, but rec6 is unit-consistent at f0-2 and rec7 has NO valid
  triple (probably a float/leaf track, not a transform). So record
  slot semantics are per-track — the static mask decides which
  DOF slots carry which channel, and rot can land in a different
  field group when pos/scale are static.

  Also: the s>1 violations flagged in r109 on recs 3,4,9,10,11
  were all frame 157 — the LAST grid row. Verified: last row is
  padding/terminator (all records saturate identically). Grid
  effective frame count is cnt-1 live rows + 1 pad row on this
  bank (check `frame < grid_frames()-1` when sampling).

  Practical consequence for grid_quat(): a field-group validity
  check (|v|^2 <= 1.05) can detect "this triple isn't a quat
  here" and the correct rot slot should be resolved per track
  from the metadata mask instead of hardcoding f6-8.

  Universal Modder check: no ANIX animation handling found on
  the public repo/pages — UM covers gamedb/strdb/archives only.

## r111 — element-table data bytes = sparse key-time lists

  Re-decoded the element table as a BYTE-PAIR stream {idx,data}
  grouped by ascending idx (no required zero head). Both banks
  parse under one grammar. Counts: orc clip1 = 99 elements,
  sentry clip1 = 42.

  KEY FINDING: within an element, data bytes form a monotonically
  increasing ramp across [0,255] — quantized key TIMES (u8
  fraction of clip duration). Examples on sentry clip1:
    el15: 2a 54 7e a6 c8 e6        (6 keys, even spacing)
    el29: 1f 3a 51 64 77 8a 9d b1 c9 e2  (10 keys ~+0x17 each)
    el39: 10 1f 31 43 56 6b 82 9c b6 d0 e9 (11 keys ~+0x13)
    el41: 56 aa ff                 (3 keys)
  Element idx = a channel/DEF slot; data bytes = that channel's
  sparse key positions. Channels without data bytes are implicit
  (single key at t=0? static?).

  So each element is a KEYFRAMED CHANNEL — count (99/42) is close
  to active-DOF count, NOT track count (20) or grid K (19/12).

  track_records' {count, data_pos, flags, cum} look like the
  matching key-VALUE block descriptors: data_pos = byte offset
  into the value data stream, cum = cumulative key counts
  (sentry c1: 32/48/76/104 keys in 4 groups + a flags=8 tail row
  cum=904). Element key counts (144 excl. index pair) vs 104 —
  not 1:1, so granularity differs (per-channel times vs per-
  group value blocks). Next: check whether the f32 block after
  the element table sizes as key-values-per-track_record.

  IMPLICATION for the grid decode: the dense 16B records may not
  be the "true" key data at all — they may be a per-frame
  expanded delta/cooked layer, while the element table +
  track_records describe the sparse key source. Or vice versa:
  elements are the key indices, grid records are per-frame
  residual samples. Either way the field semantics question
  ("which slot is the quat") is now "which channel type does
  element i describe" — same wiring problem, new surface.

## r112 — sparse-key + value-block model confirmed

  track_records.data_pos resolved: it is an offset INTO the clip
  metadata region (sent+sub0..sub1), not the file. sentry clip1
  group0 (count=32, data_pos=1500) points at a run of clean f32
  vec3s — smooth key VALUES for a position channel:
    (79.9,27.3,-13.2) -> (80.2,35.1,-32.2) over 11 vec3s (132B).
  The byte gap to the next data_pos (1633-1500=133B) ~= 33 f32s,
  consistent with count=32/33 stored values per group.

  So the full picture is now:
    element table   = per-CHANNEL sparse key TIME lists
                      ({idx,data-byte} pairs, data = u8-quantized
                      position on the timeline; 42 channels on
                      sentry clip1, 99 on orc clip1)
    track_records   = per-GROUP key VALUE blocks
                      (data_pos -> metadata-region offset,
                      count = stored value count; first group is
                      raw f32, later groups may be quantized)
    dense grid      = per-frame residual/delta stream
                      (16B records, 9 offset-binary fields)

  This is exactly the cooked hkaDeltaCompressedAnimation scheme:
  sparse preserved values + dense residual bitstream. The f32
  "trajectory" blocks seen in r108 are the preserved/seed values.

  Correlation status: residual-vs-seed wiring still open. rec0's
  fields are static here; no direct field<->seed-curve match yet.
  The 11-vec3 block's channel (which record/triple carries its
  residuals) is not identified — needs per-channel scale from the
  same metadata or per-block seeding tests with the element's
  actual key times as boundaries.

## r113 — per-record field stats; boundaries may vary

  Field-level stats on sentry clip1 (158 live frames):
  - The two 16-bit fields (f2, f5) sit at |v|~0.85-0.95 with tiny
    std on nearly every record = near-saturated constants. Only
    on a few records (0,1,12) do they carry real data.
  - Several fields have lag-1 autocorr ~0 while decoded |v| is
    large (rec4 f1/f2, rec5 f2/f4/f5, rec6 f3-8, rec7 all). An
    absolute-value channel can't decorrelate that fast at 25fps.
    Likely cause: THAT record's bit packing differs — field
    boundaries are per-record (the 13/13/16 template is modal,
    not universal), so misaligned reads straddle two DOFs and
    produce near-saturated decorrelated junk.
  - rec6 f0-2 = (-0.11,+0.12,-0.88) + high corr = a real smooth
    absolute quat triple (w=+0.44 reconstructed) — CONFIRMING
    the r110 finding that rec6's rotation lives in f0-2.
  - rec1 is the highest-energy record: f0-5 all active (std up
    to 0.7) — probably the root/hips with pos+rot both dynamic.

  Open: exact per-record field template (widths/offsets likely
  derived from the element table's channel list — each element
  = one channel with a bit-width implied by...?). The element
  count (42) ~= active DOF count suggests: element i describes
  channel i, and channels pack contiguously per record.

  NEXT CONCRETE STEP: enumerate per-record active-field count n_r
  (number of non-saturated field groups). If record = concat of
  n_r channels each with fixed width, boundaries = cumulative
  widths. Then match quat triples by unit-norm scan over the
  record's actual field list.

## r114 — bit-activity map: template holds, data is mixed-rate

  Per-bit flip-rate map over 158 frames (sentry clip1): active
  bits cluster immediately after each template offset
  {0,13,26,42,55,68,84,97,110} on EVERY record — the field
  boundaries do NOT move per record; the const-bit runs are just
  unused high bits of small-value fields (offset-binary sign
  extension). Active-bit count per record: 42-119, tracking how
  many fields are live and how wide their values swing.

  But the field VALUES split into two populations:
  - smooth, high lag-1 corr (0.9+): absolute channels, incl. the
    quat triple (rec6: f0-2 = (-0.11,0.12,-0.88) -> w=0.44)
  - near-zero corr, |v| up to ~1.0, monotone-drifting cumsum:
    looks like residual/quantized-noise-level data — either real
    delta fields whose per-channel scale is much larger than
    |v|=1 (so cumsum drift is expected), or a different channel
    kind entirely (non-transform float tracks? flags?)

  So the honest r113 correction: boundaries are stable; the
  mystery is CHANNEL TYPE, not packing. Determining which fields
  are abs vs delta needs the per-channel descriptor — currently
  the element table (key times) + track_records (value blocks)
  are the only typed metadata found, and element count ~ active
  DOF count is the right order of magnitude for "one element =
  one channel".

  Cheapest decisive next step remains: a runtime read of the
  cooked animation object (or of the hkaDeltaCompressedAnimation
  it feeds) — the (channel type, bit width, offset, scale)
  quadruple per field is a table lookup there.

## r115 — Ghidra: loader + job chain mapped; decompressor still behind runtime dispatch

  Headless Ghidra (sow project, scripts in ~/ghidra_scripts —
  new: AndurilRefl/AndurilClassDump/AndurilFindImm/AndurilDisasmAt;
  NOTE: script args must be hex WITHOUT 0x for AndurilDump*).

  VERIFIED-STATIC:
  - ANIX stream loader = FUN_140480aec @ 0x140480aec (r24 had
    0x140480aeb). Reads 0x30B header via stream vtbl+8, checks
    'ANIX' + version 0x2c=44, allocs 0x28B descriptor
    (FUN_140480f5c ctor), then loops the block table: reads
    {u32 outer, u32 count, count x u32} and writes
    {inner_u32,outer} pairs indexed cumulatively — the
    decompression-block map. Then two verbatim section copies
    (FUN_140480c78 = seek-to, FUN_140480ec4/e6c = sized read).
  - Clip "sentinel" 0x2b99dbc1 has ZERO code references — it is a
    clip-name hash, not a structural tag.
  - hkReflect name-string registry table at 0x1426f2xxx confirmed:
    sequential qwords -> member-name cstrs. hkaDeltaCompressed
    Animation cluster @ 0x1426f2748 gives the full member list
    (endian, totalBlockSize, staticDOFsSize, maxFramesPerBlock,
    staticMaskSize, blockDuration, transformOffsets, qFormat,
    blockOffsets, numBlocks, lastBlockSize, extractedMotion,
    duration). NOTE: no embedded hkClass data records — only the
    name strings; member OFFSETS are not in the image this way.
  - Sample path chain: anim-player vt 0x141dc47b8 slot+0x18 =
    0x1403094fc -> 0x141154b34 = job submit (tag 0x50000000);
    slot+0x58 = 0x1407807ac = element-range job request ->
    FUN_14116eaf8 on global mgr 0x14272e8d8 -> job body
    FUN_14076b38c (tag 0xb0000000) -> FUN_140470a10 =
    dedup-compare only. The actual decode worker is registered
    under the job tags at RUNTIME (FUN_140251eec = job alloc/
    dispatch by tag id) — not statically reachable.
  - Eval-loop node sampler (FUN_1402fa10c vt+0x90) confirmed:
    intrusive-list advance on key time, dispatch per node type.
  - 0x1fff-mask scan: 415 hits, no obvious scalar unpacker — the
    decompressor is almost certainly SIMD bit-unpacking.

  CONCLUSION: static tracing of the decoder ends here — the job
  worker table is runtime-built. Two remaining routes:
  (a) write-watch the decoded 32B {pos,quat} grid during a
      forced rebind (needs game running, read-only),
  (b) identify the worker under job tag 0xb0000000 by scanning
      for registration sites that store a fn ptr next to the tag
      (pattern: `mov imm32 0xb0000000` near a `lea reg,[fn]`).

## r116 — Ghidra chain mapped + ms timing table found

  Static side (all VERIFIED-STATIC, headless Ghidra):
  - ANIX loader FUN_140480aec: 0x30B header, magic+ver44, 0x28B
    descriptor, block-map loop {u32 outer,u32 count,count x u32}
    -> cumulative {inner,outer} pairs, then 2 verbatim sections.
  - hkReflect name tables at 0x1426f2xxx: full delta-anim member
    vocabulary present (qFormat, staticMaskIdx, transformOffsets,
    preserved, blockOffsets, numBlocks...). Member VALUES are not
    embedded — names only.
  - Decode dispatch is runtime-built (job tags via FUN_140251eec)
    — the actual bitstream unpacker is not reachable by static
    xref. Static route to it = find registration site pairing a
    fn ptr with the job tag, or write-watch the decoded grid.

  File side (sentry clip1):
  - Between metadata end (sent+sub1) and grid_off sits a 496B
    u16 table: values are {33,34,66,67,100} = multiples of ~33ms
    = MILLISECOND key/frame deltas (the r22e "u16 ramp" is this
    table). So clip data = [element key-time table (u8)] +
    [u16 ms delta table] + [data buffer {dur,frames,blockSize}
    + u16 frame times + dense residual grid].
  - The 11-vec3 f32 pos block (track_records[0] data_pos) did
    NOT match any field triple as absolute values (best ~0.63)
    nor as single-scale deltas (~21% axis-1 residual). So the
    dense grid is neither plain absolutes nor plain deltas of
    these keys — the wiring needs the real blockSize/segmentation.

  NEXT (unchanged, now sharper): the decode is provably
  "sparse preserved values + dense residual stream"; residual
  scale/blocks live in metadata. A 5-min read-only runtime dump
  of the cooked animation object at bind resolves all remaining
  wiring at once. Until then, offline work on this front is
  exhausted — moving back to the wider Anduril roadmap is the
  better use of effort.
