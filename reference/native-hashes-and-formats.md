# Hash functions & engine data reference (for modders)
> **Status:** **partial** — Project Nemesis source. Hash claims **verified by Andúril**: the FOLD table was independently located in `ShadowOfWar.exe` (unique candidate; `FOLD['/']=0x34`, `FOLD['\\']=0x2a`, A–Z/a–z→1–26, digits→27–36, 95 nonzero entries — identical to the table in `engine/engine.md`), and the check vector `namehash("UI_Frontend_Benchmark")=0xa0014283` resolves to `RUN BENCHMARK` in retail strdb. The claimed routine at `0x140423e70` was **byte-confirmed**: `movzx/test` loop containing literal `imul r8d, r8d, 0x397` + `movsx edx,[fold+rax]`, and its `lea r9` references the FOLD table at VA `0x141DB50D1` — matching the documented `+0x1DB50D0`. Remaining Nemesis-only: the 2,250-pair event-hash table and the intern-map hash.


> **Provenance:** Project Nemesis runtime-model documentation (native
> `nemesis.dll` live-game forensics), ported into the corpus. `sdk.*`,
> `Mods/`, `recon/`, and `tools/` names inside refer to that project's
> plugin API and scripts — cited as evidence of the claims, not as game
> surface. Per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE /
> VERIFIED-INGAME / MODEL / UNVERIFIED / INFERRED) are theirs.


> Audience: modders working at the engine level (`sdk.debug`, `sdk.db`,
> `sdk.strings`, direct disasm). For the friendly Lua surface see
> Project Nemesis `docs/design/lua-api.md`. Everything here is for
> build `0x5B7F5782` (`Patch_09`, exe `2025-05-05 04:27`) unless noted —
> addresses are image VAs (base `0x140000000`) and shift on exe updates.

## Hash functions

The engine uses **three different 32-bit hashes** for different domains.
Using the wrong one is the #1 source of "record not found" bugs.

### 1. `name_hash` — records, attrs, paths, .bvr names

Used for: gamedb record names, attribute names, `.bvr` field names,
loose-file index paths, `sdk.db` name lookups.

```python
# tools/namehash.py — reference implementation
def name_hash(s: str) -> int:
    h = 0
    for c in s:
        h = (h * 0x397 + FOLD[ord(c)]) & 0xFFFFFFFF
    return h
```

- `FOLD` = 256-entry case/separator-folding table at exe `+0x1DB50D0`,
  mirrored in `extracted/namehash_fold.txt`. It folds `'A'..'Z'`→`'a'..'z'`
  (case-insensitive) — but **not** path separators: `FOLD['/']=0x34` vs
  `FOLD['\\']=0x2a`, so `'/'` and `'\\'` hash differently (checked against
  the extracted table; an earlier version of this doc claimed they fold).
- Routine live at `0x140423e70` (`u32 namehash(const char*)`).

### 2. Event hash — script/gameplay events

Used for: Nemesis/event system ids (`SpectatorDepartedNode`,
`SpawningComplete`, …). **Not** the same function as `name_hash`.

- Algorithm not yet fully recovered; we ship a captured
  **2,250-pair lookup table**: `extracted/event_hashes.tsv`
  (`hash<TAB>name`). Match by table, not by recomputing.
- Names also appear in script constants as
  `__global_<name>_Script_<hash8>` / `_<area>_<event>_Script_<hash8>`.

### 3. `.strdb` key hash — localization strings

Used for: string-database lookup keys (`START`, `OPTIONS`, every UI
label).

- **Same function as #2** — the strdb key hash IS the gamedb name hash
  `h = h*0x397 + fold[c]` (verified live 2026-09-24:
  `namehash("UI_Frontend_Benchmark") = 0xa0014283` = the "RUN BENCHMARK"
  entry; `UI_Menu_Quit`→"QUIT", `UI_MainMenu_Start`→"START NEW GAME").
- `sdk.strings.{set,get,unset}` accept a key **name** directly —
  `sdk.strings.set("UI_Frontend_Benchmark", "MOD MENU")`.
- `sdk.strings.register_named("UI_MyMod_Label", text)` stages an entry
  the engine's own name→hash path resolves — records referencing that
  name see `text`.

### 4. Nemesis-minted hashes — `sdk.strings.register`

`sdk.strings.register(text)` → `u32`. We generate a fresh hash ourselves:

```text
hash = fnv1a_32("nemesis:" + utf16le(text))   then linear-scan
       +1 until unused in the staged set AND absent from live dbs
```

- Deterministic for the same text (dedup), collision-checked against the
  real string DBs, `0`/`u32::MAX` reserved.
- These hashes are valid anywhere the engine takes a **raw u32 string
  hash**. For name-keyed strings (records resolving a localization key),
  use `register_named(name, text)` — that stages under
  `namehash(name)` so engine name lookups hit it.

### Intern-map hash

`0x1898e8` — hashes interned name objects for the gamedb name map. Only
relevant when registering new names (G41); `sdk.db.inject`/`append` are
gated on this.

## `.strdb` — string database format

```
+0x00  u32 magic      "SKDB"
+0x04  u32 version    0x01000000
+0x08  u32 idx1_off   (0x1c)      — offset of {hash,u32} index
+0x0C  u32 idx1_count
+0x10  u32 idx2_count (0 retail)
+0x14  u32 str_chars              — u16 CHARS (engine reads *2 bytes)
+0x18  u32 blob3_size  (0 retail)
idx1:  count × {u32 hash, u32 char_idx}
strbuf: UTF-16LE chars; str = strbuf + char_idx*2. NO flag bits —
        odd indices are suffix-dedup pointers into mid-string.
```

- Loader `0x141d23228`; lookup `0x140472104` = `strbuf + off*2`;
  manager `0x140472090` walks the db list newest-first.
- `tools/strdb.py` — load/build; `tools/strdb_dump.py` — TSV dump;
  `tools/strdb_patch.py` — `.strdelta` → merged strdb (all verified
  live: dumped → rebuilt → booted → text renders).

### `.strdelta` — mod-friendly delta format

```
keyhash=0x0123abcd
"0x0123abcd"="replacement text"
"old text"="new text"          # find by content
""="brand new string"          # add (hash auto-minted)
```

`sdk.strings.set/unset/apply/get/clear/register` is the live runtime
equivalent (no file write — a synthetic db object spliced into the
manager's list; newest db wins lookups).

## `.bvr` — object schema format

```
'...' + u32x4 + u32 objcount
per obj: u32 namehash + u32 schemaidx + u64[]
then:    u32 fieldcount + u32×fields
then:    u32 typecount + types {u32 namehash, u32 size, u16, u16}
strings: u8 len-prefixed, indexed from the field table
```

Field→object mapping is per-object `u64` bitmask into the field table;
`tools/bvr.py` decodes it. Names use `name_hash` (#1).
Layouts: `docs/evidence/formats/bvr.md`, `docs/model/gamedb/schemas.md`.

## `ShadowOfWar.exe` — image layout (repacked)

The shipped exe is **section-repacked** — the real code does **not** live
in `.text`. When scanning for code/xrefs, use:

| Section | VA | Purpose |
|---------|-----|---------|
| `.arch` | `0x140001000`–`0x141d87c00` | **main code + rodata** (~30 MB) |
| `.xtext` | `0x141d92000`–`0x1425aba00` | **secondary code + rodata** (~8 MB) |
| `.text` | `0x147299000` | stub only (0x14 bytes) — **dead end** |
| `.idata` | `0x1425ac000` | rdata/strings blob |
| `.srdata` | `0x1472ae000` | string/resource data |

Cstrs and code are interleaved across `.arch`+`.xtext` — scan both for
xrefs **and** for strings.

> Earlier "scan .text for xref/imm" results were void — rescan over
> `.arch`+`.xtext`. File on disk is packed; runtime bytes differ —
> `sdk.db.dumpmem` pulls unpacked code from the live image.

### Known registration sites (build `0x5B7F5782`)

- Menu event enum table: entries at `0x142a4b830` (+0x10 each),
  registered via `0x14117e9b4`; `MenuEnter` = enum `0x20`.
- `Interface/Menu/MenuSelection/Action` list handle resolved at
  `0x141b0f4c5`/`0x141b0f4f3` → global `~0x142701268` (the
  `g_ActionList`).
- `Engine.Benchmark` console command thunk `0x141179608` → handler
  `0x141179618` (aspect-ratio helper `0x140331b5c`).

## `.arch06` / `LTA` — archives

- `tools/ltar.py` — `l`/`x`/`xh` list/extract members.
- `tools/archcfg.py` — the real mount list (`default.archcfg`); loose
  `Mods/` dir must be an archcfg line.
- `tools/looseidx.py` — loose-file index (`.loosefiles`) format.

## `.scripts` — HavokScript bundles

- `SCRI` header → `luaq` bytecode (HavokScript 5.1) or text prototypes.
- Function header has `instrCount`/`unk` swapped vs stock hksc;
  op = `raw>>25` (7-bit opcode), `A = raw&0xff`, B/C in middle bits.
- Disasm: `tools/hks_disasm.py` (SoW-aware), strings:
  `tools/hks_mine.py`. Deep dive: `docs/evidence/scriptvm/havokscript.md`.

## Engine VM ABI (verified)

- Lua 5.1 ABI; `LUA_GLOBALSINDEX = -10002`; TValues are 16-byte
  `{tag:u32, pad, value:u64}`; stack `top@L+0x48`, `base@L+0x50`.
- Tag map: 0 nil, 1 bool, 2 lightuserdata, 3 **f32** number, 4 string
  (`len@+8` &0x3fffffffffffffff, Jenkins `hash@+0x10`, `chars@+0x14`),
  5 table (32B nodes `{key_tv,val_tv}`, mask@+0x1c, nodes@+0x20),
  7 userdata (mt@+0x18), 9 ifunction, 10 cfunction, 11 ui64,
  **12 struct** (engine C++ object, mt@+0x20).
- `dostring` `0x1406a682c` `(L, buf, len:u32, name, flag:u8)→bool`;
  `lua_pcall` thunk `0x141159294`; `getglobal` `0x14115931c`;
  `setglobal` `0x14114e4a8`; `index2addr` `0x141d00500`.
- Full map: `docs/evidence/scriptvm/index.md`,
  `docs/evidence/scriptvm/api-map.md` (~1400 binding stubs documented),
  `api-signatures.md`.
