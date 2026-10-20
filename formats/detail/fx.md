# `.fxo` / `.fxp` — shaders and effects

> **Status:** **partial**** — bank structure decoded; shader payloads not recompiled
> **Provenance:** shader bank structure decoded on the real `.fx` corpus.

## `.fxo` — SRHR v2 compiled-shader bank

A shadertree's compiled output: the fxc command line plus every variant
permutation as raw D3D bytecode, plus the constant/variant name table that
binds `.mat` properties to shader registers.

```
header    "SRHR" u32ver(2) u32(1) "DX11" u32(0x14) "DX11" u32 zeros → 0x44
preamble  optional {u8,u8,u8,u16 len} fxc args record
          ("/Qstrip_debug  /O3  /Tcs_5_0  /E<entry>  /Fo"tmp" "src.tmp"")
          + shared resource/constant-buffer table
entries   N × { binding preamble, u32 dxbcSize, DXBC[dxbcSize] }
tail      variant-switch table — u16-len name strings (not NUL-terminated)
```

- Every DXBC is preceded by a `u32` equal to its internal size field —
  invariant on all 767 files.
- DXBC payloads are standard `DXBC` containers (ISGN/OSGN/SHEX chunks) —
  usable directly by D3D tools/disassemblers.
- The args record is optional (`graug_disp.fxo` has none); parse by
  scanning for the first `/`-leading printable string.
- **The tail names the whole binding surface:** `k_*` constants
  (`k_fRoughnessMultiplier`, `k_mWorldToClip`, `k_vClothForce`), `t*`
  textures (`tDiffuseMap`, `tNormalMap`), `s*` samplers
  (`sAniso8X_TriJuicing15_WRAPWRAPCLAMP_Sampler`), and pass variants
  (`fulldeferred`, `shadowmap`, `shadowmappoint`, `fulldeferred_meshtype4/5`).
  These are exactly the names `.mat` properties set — `anduril-mat` property
  names ↔ `anduril-fx` variant names is the material↔shader contract.

## `.fxp` — LTFX v8 effect bank

```
"LTFX" u32ver(8) "GADB" u32ver(17) u32[9] u16   (FX mini-header)
"FXDB" u32(0)   FX record payload → EOF
```

- The embedded GADB header is **not** the master-gamedb layout — fields:
  `[fxdb_size, entries, u1..u3, block_count, u5..u7]` + u16.
- The FXDB payload starts with a type-name cstring (`FxGlobalProps`) then
  hash-keyed property records — record format not yet field-decoded.
- All 1379 files: `LTFX@0`, `GADB@8`, `FXDB@0x36` — invariant.

## Verified

- `.fxo`: all **767** files parse; every DXBC size-prefix checked.
- `.fxp`: all **1379** files parse.
- Total **2146/2146** through the VFS (Oodle path).

## Unknowns

- `.fxo` preamble internals: the constant-buffer binding table between
  args and each DXBC (`{cstr name, u32 hash, u16?, byte[] permutation}`).
- `.fxo` header bytes 0x08/0x18/0x1c–0x44 and the small records
  (`02 00 04 05` etc.) preceding the args string.
- `.fxo` tail structure beyond the name strings (variant-select bits).
- `.fxp` FXDB record encoding and the GADB mini-header field semantics.
