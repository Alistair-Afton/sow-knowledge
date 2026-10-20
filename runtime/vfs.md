# VFS & Asset Loading — canonical model
> **Status:** **partial**** — Project Nemesis source doc; not independently verified by Andúril.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


> Status: **MODEL** — canonical current knowledge. Claims carry per-line
> confidence tags; `UNVERIFIED` entries are experiments, not facts.
> Raw findings and code archaeology live in `docs/evidence/formats/` and
> `docs/evidence/engine/exe-xrefs.md`; this doc records what they *mean*.

without being packed or converted into ungodly formats.* This doc is the
strategy and the evidence ledger for it.

## Compatibility requirement (user, 2026-09-24)

Whole-file shadowing is the wrong granularity for *shared* files:
`string.strdb`, `game.gamedb`, `.scripts` etc. are single packed blobs
that many mods want to touch — last-one-wins is not acceptable. So the
delivery model splits by granularity:

- **File-granular assets** (textures, meshes, audio, videos — replaced
  wholesale): loose-file override is the correct native mechanism.
  Conflicts only arise when two mods replace the *same* asset, which is
  inherent.
- **Record-level content** (gamedb records, strdb strings, script
  functions): mods ship **deltas**, not file copies — e.g.
  `Mods/<name>/db/<list>.toml`, `strings/<lang>.strdelta`. The
  framework merges them post-load through the native mutation paths
  (`sdk.db.setcell`, runtime strdb patch, script VM injection) with a
  defined conflict policy (explicit priority, warn on overlap).
  **Strings are working today**: `tools/strdb_patch.py` merges
  `.strdelta` files (`set`/`replace`/`del`) into a rebuilt strdb and
  `Mods/` serves it — verified in-game 2026-09-24.

This is a design constraint on the modding layer, not just a VFS
question: the loose-file mechanism matters most for assets; the merge
layer is what makes record content composable.

## What we know

- `x64/default.archcfg` is the archive mount manifest; its own header
  comment says search order is **bottom-up** (later entries win).
  `VERIFIED-STATIC` (file contents).
- The loose `game/` directory is a VFS source — it already contains
  loose engine config (`SKU.cfg`, `projectconsole.cfg`) and loose
  cinematics. `VERIFIED-STATIC` (files present).
- `archivepatchloosefiles.txt` exists **inside every archive** (it's an
  LTAR member, VFS-rooted path `/archivepatchloosefiles.txt`) and is a
  per-archive index of paths that may resolve loose on disk — the
  engine's own shipped loose-override mechanism. `VERIFIED-STATIC`.

### Native mount/lookup path (disasm, 2026-09-24)

At mount, per archive (`0x140c28a57` region):

```text
0x1403ddad8(archive_obj, "archivepatchloosefiles.txt")  — open index via
        the loose-name lookup itself (bsearch over a sorted dir table)
0x1411ae510          — map/read the file body
0x141182428          — parse index: requires version==1, reads count,
                       iterates {u32 hash, u32 blob_offset} entries
                       building a sorted table at [archive+0x30]/+0x38
```

Lookup at open time:

```text
0x1403ddad8(this, path) — normalize path, find last component
0x1403ddb90             — bsearch a sorted 0x30-stride directory-entry
                         table at [src+0x40], count at [src+0x48]
```

`VERIFIED-STATIC` (`tools/disasm_va.py`, `tools/xref.py`) — this is the
**per-archive** index lookup, not the dir-source path. The full
resolution model is decoded below.

### Resolution model (disasm + live trace, 2026-09-28; cache model corrected same day)

The resolver chain for every asset open:

```text
vt[0x80] on the FS sub-object (singleton 0x1426fefd0 → vt[8])
→ 0x14045a1f4          — CACHE TRY: delegate to asset-manager global
                         0x142a86758 → 0x1401beaac → 0x14017ac5c
                         (hash-map get; bucket fn 0x14114b914,
                         comparator 0x14114b924); returns node or 0
miss → slow path       — walk dir sources and GetFileAttributesW /
                         CreateFileW each <source_root>\<vfs_path>
                         candidate (probe loop ~0x140c40770–0x140c40a5c;
                         attrs call at 0x140c40a27), then archive
                         index; insert result node into the cache
```

`VERIFIED-STATIC` (call chain) + `VERIFIED-INGAME` (probe behavior).

**The map is a resolve cache, NOT an eligibility index** (corrected
2026-09-28 evening): `map = *(0x142a86758)` (live object `0x0b103138`
this boot), circular-bucket hash map, 2048 buckets × 24 B, head at
`bucket+8`, `node+8` = next, `node+0x18` = path cstr, fields at
`node+0x28`/`+0x30`/`+0x52`; `node+0x38` = per-node stream/reader
descriptor (vtable `0x141e6d910`). It **grows on demand**: 11,660
entries post-boot → 27,025 after one benchmark + one save load
(walk with `tools/vfsidx.py`; terminator is `node == bucket`, not
null — the lists are circular).

**Loose probing is NOT index-gated** — the earlier claim was wrong:

- `interface\videos\kraken\options\soundoptions.vib` is in **zero**
  `archivepatchloosefiles.txt` indices (all 41 archives checked —
  `.vib` paths are never indexed), yet it probed every dir source
  and **resolved from `plugins\nemesis\files`** (`A ok` + `C ok`,
  played in Options→Sound). `VERIFIED-INGAME`.
- `interface\textures\hud\mod\modlogo.tex` — a novel path existing
  only in `plugins\nemesis\files` — sits in the live cache
  (`node 0xac19960`): the engine resolved a brand-new loose file.
  `VERIFIED-INGAME`.
- The four probe crashes earlier today were **call-convention bugs**:
  hand-calling `vt[0x80]` (the cache try) returns null for cold paths,
  and the open path then unconditionally calls `vt[0xa0]`
  (`0x14119f3a4` → `0x1403df304` derefs the null resolve handle at
  `0x1403df326`). The engine's own miss path never produces a null
  handle. Do not hand-call resolver internals.

**Consequence**: `files/` roots serve **any** path — loose mod assets
(textures, bundles, videos, scripts) work with no index registration
step. What the cache stores is the resolution result (descriptor +
metadata); nothing needs pre-registration.

**Dir-source loose probe** — `VERIFIED-INGAME` via `vfswatch` (module
file-API tracer, `NEMESIS_VFS_TRACE=1`, log `x64/plugins/vfs_trace.log`):

- For every requested path the engine probes **each directory source**
  lazily at lookup time: `GetFileAttributesW` then `CreateFileW` on
  `<source_root>\<path>`. There is no per-source file table to build —
  dir sources are just a root path + child-archive vector. Not gated by
  any index (see cache model above).
- **Probe order is descending registry index; first hit wins** — the
  loop stops at the first `ok`. Live trace shows
  `..\Mods\faction_probe\files` (idx 91) probed before
  `plugins\nemesis\files` (90) before `..\Mods` (89) before the `Game`
  dir sources — matching archcfg "bottom-up wins" semantics, extended
  to runtime registrations: **later-registered sources take precedence,
  so `sdk.vfs.add_source` roots outrank every archcfg source.**
- `src+0x108`/`+0x110` on a dir source is a **child-archive vector**,
  filled once at boot by the wildcard glob pass `0x1411972c9` (per-source
  pattern build at `0x141197304`, dedup `0x141197150`). `..\Mods` holds
  2 children = the 2 `.arch06` files in `Mods/` — **`Mods/*.arch06`
  mounting is native**, closing open question 3. Runtime-added dirs
  show `+0x108=0` because the glob ran before they existed and they
  hold no archives — *not* a defect.

### Runtime source registration (disasm, 2026-09-25)

The `archcfg → source` chain is fully decoded — and the globals are now
named correctly (earlier notes mislabeled them):

```text
argv collection   0x1426fefe0   — command-line args, NOT sources.
                                  vtbl[0x30]=count, vtbl[0x38](i)=arg i
argv loop         0x14119720b   — for i: _stricmp(arg[i],"-arch")==0
                                  → mount arg[i+1] (0x1403ddd34 normalize
                                  → 0x140c28560 → 0x1410dcd60)
archcfg resolver  0x141196fe0   — arg "-archcfg" → next arg is the
                                  manifest name; else "default.archcfg"
archcfg reader    0x141203bc0   — open path → read whole file → parse fn
line parser       0x141203a98(buf,size) — split lines (\r\n|\n|\r),
                                  ";" comment-to-EOL, ≤0x400 chars
per-line mount    0x141197464(line,0) — ★ the registration entry point:
                                  empty-string guard → 0x120 source ctor
                                  (0x141196f10) → bounded path copy
                                  (0x104 B, 0x14115eec4) → registry
                                  singleton → insert
source registry   0x142a55cb8   — TLS-init'd singleton (getter 0x141197084),
                                  insert via 0x141196b2c
```

**`-arch <name>` mounts an extra source from the command line; `-archcfg
<file>` picks a different manifest.** `VERIFIED-STATIC`.

`sdk.vfs.add_source(path)` wraps `0x141197464` on the game thread —
resolved via `signatures::VFS_MOUNT_LINE` (unique on 0x5B7F5782), so an
exe update degrades to an "unresolved" error, never a wild call. The
exact same call the parser makes per archcfg line, so relative paths
(`..\Mods\name\files`, resolved against `x64/`) and dir-source semantics
are inherited. This is the per-mod-root mechanism (open question 2):
each mod can mount its own `files/` dir at runtime without touching
`default.archcfg`.

**Post-boot registration is fully verified live** (2026-09-28): sources
added mid-session appear in the registry tail (`plugins\nemesis\files`
at idx 90, `..\Mods\faction_probe\files` at 91 of 92 total), and the
loose probe reaches them on every subsequent indexed lookup — 33,028
probes into the two runtime dirs during one save load. Positive
end-to-end hit: `interface\videos\kraken\options\soundoptions.vib`
served from `plugins\nemesis\files` over `..\Mods`/`..\Game` (probe
stopped at first hit). `VERIFIED-INGAME`.

### Loose-index census (2026-09-24)

Extracted every archive's index (`tools/looseidx.py` + `ltar.py`):

- 43 archives carry an index; **73,940 indexed paths total, 31,031
  unique** — essentially all content: `bundles\` (~25k), `.scripts`
  (2,770), `.bvr` (305), `database\` (213) including
  `database\game\game.gamedb` and every `.strdb`, `.mat`/`.fxo`/`.tex0`,
  cinematics, `.anix` animation packs.
- The shipped loose `.vib` cinematics under `game/` are **not** in any
  index — `game/` is a separate *unindexed gap-fill source*, consistent
  with `..\Game` sitting at the top of `default.archcfg` (bottom-up
  search ⇒ weakest precedence).

### Live experiment: candidate loose roots (2026-09-24, negative)

Dropped full copies of `database\string\enus\string.strdb` (indexed in
`hotchunk.arch06`) — first with patched strings, then **truncated to
1 KiB** — at four candidate roots and booted:

| Candidate root | Result |
|---|---|
| `game/database/string/enus/` | not read — text normal |
| `<install>/database/string/enus/` | not read |
| `<install>/x64/database/string/enus/` | not read |
| `<install>/hotchunk/database/string/enus/` | not read |

Truncated copies would garble every string if served; nothing changed
`VERIFIED-INGAME` (screenshots, truncated-file test).

**Why all four were structurally incapable of working**: `default.archcfg`
orders sources bottom-up, and both dir sources (`..\Game`, `Game`) sit at
the *top* of the file — below every archive, weakest precedence. A
dir-source can only win for paths no archive provides. `string.strdb`
exists in every stack, so no dir at those positions could ever serve it.

### archcfg dir-source experiment (2026-09-24, POSITIVE)

Appended `..\Mods` as the **last** (strongest) entry of
`default.archcfg` and deployed a patched
`database/string/enus/string.strdb` under `Mods/`:

- Boot 1: the deployed copy was accidentally **1 byte short**
  (7,797,583 vs 7,797,584). The entire UI rendered tofu boxes — every
  string lookup failed. `VERIFIED-INGAME`: **the engine mounted
  `..\Mods` and served the loose strdb over every archive**; the strdb's
  hash/index data lives at the file tail, so a 1-byte tail truncation
  breaks all lookups. This also proves indexed archive content *is*
  loose-overridable from disk.
- Boot 2: same file at full length with 49 marker-patched title
  strings → all text normal in-game; marker observation pending.

Conclusions `VERIFIED-INGAME`:

1. **`default.archcfg` lines that are directories mount as dir-sources**,
   same precedence rules as archives — bottom of file = strongest.
2. **Loose files in a mounted dir override archive members** for indexed
   paths (strdb proven; the index is very likely just a "patchable"
   allowlist — see open question 1).
3. Relative entries resolve against `x64/` (`..\Mods` = install-root
   `Mods/`, `..\Game` = install-root `game/` — the shipped weak source).

The native loose-mod mechanism is therefore: **one added line in
`default.archcfg` + plain files in `Mods/`** — no packing, no plugin
code, no archives. This is the Tier-1 delivery the project wants; the
record-level merge layer (below) still applies for shared structured
files. The archcfg edit itself is a config change (backup at
`default.archcfg.nemesis-bak`), not a binary patch.

### `archivepatchloosefiles.txt` format (decoded 2026-09-24)

```text
u32  version          (=1; earlier "v4" bytes seen were a misread of a
                      different archive's header — actual magic is this u32)
u32  blob_len
u8[blob_len]          NUL-terminated paths, VFS-rooted, backslash-separated
u32  count            (= number of paths)
{u32 hash, u32 blob_offset}[count]   sorted by hash — bsearch index
```

- `hash` = the game's record-name hash `h = h*0x397 + fold[c]` over the
  path bytes as stored (backslashes, lowercase corpus paths hash
  identically as-is or lowered). `VERIFIED-STATIC` — reproduced all
  entries in the Patch_04 index (`tools/looseidx.py --verify`).
- `blob_offset` indexes into the path blob (offset of the path's first
  byte), so the index is a hash→string bsearch table: the engine hashes
  the requested VFS path and resolves loose hits in O(log n).
- Contents observed in Patch_04: `.mat`, `.fxo`, `.tex0` — i.e.
  **compiled/asset types are loose-resolvable**, not just text. The
  index lists which of the archive's own members may be shadowed by a
  file on disk.
- Parser/tool: `tools/looseidx.py` (dump + `--verify` hash check).
- Extra `.arch06` files in the install root mount without being listed
  in `default.archcfg` (existing mods like `Patch_IncreasedCap.arch06`
  work). `VERIFIED-INGAME` (they function) / `INFERRED` (wildcard-mount
  mechanism assumed, not located in the exe).
- `Mods/` — two `.arch06` files sit there and reportedly load; whether
  `Mods/` is a second wildcard source, a loose-file source, or handled
  by a third-party plugin (`mesow_patches.dll`? Mod Menu?) is
  `UNVERIFIED`.

## Open questions (each is a concrete experiment)

1. ~~**Does `archivepatchloosefiles.txt` gate loose resolution?**~~
   **ANSWERED 2026-09-28**: **no** — the loose probe fires for every
   requested path regardless of index membership. Proof: `.vib` paths
   exist in zero archives' loose-indices yet `soundoptions.vib` probed
   + served from a runtime dir source; the novel `modlogo.tex`
   resolved+cached. The global map is a resolve cache (grows on
   demand), not an eligibility set. Novel loose files work today —
   no index registration needed.
2. ~~**Per-mod subdirectories**~~ — **SOLVED 2026-09-28**: enumeration is
   lazy per-lookup, not eager at insert. `add_source` registrations are
   probed for every indexed path the moment they're registered
   (`VERIFIED-INGAME`, live trace). Precedence = descending registry
   index → late adds are the *strongest* sources, outranking `..\Mods`.
   Remaining nit: duplicate adds aren't deduped by `0x141197464` — the
   framework must guard (call-site responsibility, already noted).
3. ~~**Is `Mods/*.arch06` mounting native or plugin-provided?**~~
   **ANSWERED 2026-09-28**: native — the boot-time glob pass
   `0x1411972c9` walks each dir source's root for `.arch06` children
   and mounts them into the source's `+0x108` child vector. `..\Mods`
   holds exactly the 2 `.arch06` files present in `Mods/`.
   `VERIFIED-STATIC` (disasm) + `VERIFIED-INGAME` (2 children observed
   live). Caveat: the glob runs once at boot — archives dropped into a
   runtime-added dir *won't* auto-mount post-boot (would need a
   re-glob or explicit mount).
4. **Asset-type matrix** — extend the loose-override test per type
   (`.mat`, `.bndl`, `.bvr`, `.scripts`, `.tex0`) now that a working
   root is known. `.bndl`/`.tex`/`.vib`/`.embb`/`.bnk` probes all
   observed firing against runtime sources in the 2026-09-28 trace.
5. **Marker visual confirmation** — superseded by a stronger result:
   a **writer-produced** merged strdb (68,547 entries via
   `tools/strdb_patch.py` + a `.strdelta` file) was deployed to
   `Mods/` and the game rendered all UI text perfectly (verified
   2026-09-24, WBPlay Newsletter + menus). Parser + writer + delta
   merge + loose override are all proven end-to-end. The
   `NEMESIS LOOSE OVERRIDE` marker was set on the title-prompt string
   but that string rotates with promos — cosmetic confirmation only.

## Asset-type matrix (fill in as experiments land)

| Type | Loose | Archive | Notes |
|------|-------|---------|-------|
| `.lua` (our mods) | yes — `Mods/` flat tree, our loader | n/a | Our own mechanism, `VERIFIED-INGAME` |
| `.cfg` / `.txt` | likely — `game/` ships loose cfgs | yes | `game/` is read for engine config |
| `.mat` / `.fxo` / `.tex0` | **yes** — indexed by `archivepatchloosefiles.txt` | yes | `VERIFIED-STATIC` (Patch_04 index); loose root on disk `UNVERIFIED` |
| `.bndl` (bundles) | **yes** — probed against runtime sources live | yes (extractor exists: `tools/ltar.py`) | indexed (~25k paths); probe firing `VERIFIED-INGAME` 2026-09-28 |
| `.vib` (video) | **yes** — served from `plugins\nemesis\files` | yes | `VERIFIED-INGAME` 2026-09-28 — full probe→hit→open chain in Options→Sound |
| `.bvr` (behavior) | UNVERIFIED | yes (`tools/bvr.py`) | 305 indexed |
| `.strdb` (strings) | **yes** — `Mods/` mounted via archcfg | yes (`tools/strdb_dump.py`, `tools/strdb.py`, `tools/strdb_patch.py`) | `VERIFIED-INGAME` 2026-09-24 — read **and write** proven in-game; prefer `.strdelta` merges over whole-file copies |
| `.gamedb` | UNVERIFIED | yes | Runtime mutations go through `sdk.db`, not file overrides |
| textures/models | UNVERIFIED | container-internal | `.tex0`/`.mat` already indexed loose — promising |

## Investigation tooling

- `tools/ltar.py` — `.arch06` list/extract (for reproducing what a loose
  override would need to shadow).
- `tools/looseidx.py` — parse/verify `archivepatchloosefiles.txt`
  (`--verify` reproduces the index hashes).
- `tools/archive_census.py` — inventory of install archives.
- `tools/vfsidx.py` — external `ReadProcessMemory` walk of the live
  global index map (2048-bucket circular lists; dump paths/node fields).
- `tools/vfsapis.py` — locate runtime file-API slots/call sites (the
  `.xtext` bound table at `0x141d9xxxx`, NOT the `.idata` IAT).
- `tools/vfsroot.py` — summarize `vfs_trace.log` by root/API/extension.
- `vfswatch` (`NEMESIS_VFS_TRACE=1`) — patches every qword slot in the
  exe holding a resolved file-API address (covers `.xtext` bound table +
  late-bound slots via ~500 ms rescan window); logs `C`/`A`/`P`/`F`/`N`
  + ok/err + tid + path to `x64/plugins/vfs_trace.log`. Diagnostic,
  off by default.
- Recon scripts: `runtime/recon/` (deploy with `deploy.ps1 -Recon`).
- The exe is metadata-rich — grep `extracted/` corpora and the
  `docs/evidence/reference/exe-*.txt` lists for VFS/archive symbols
  before disassembling blind.

## Guardrails

- Never modify stock archives or `ShadowOfWar.exe` — overrides only.
- An experiment that writes into the install gets reverted or documented
  in the journal the same session.
- Claims in this doc get re-tagged as experiments land; don't let a
  `UNVERIFIED` row age into assumed-fact.
