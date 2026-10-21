# `.arch06` — LTAR v4 archive

> **Status:** **proven** — byte-identical extraction vs reference tooling on real archives
> **Provenance:** every `.arch06` in a retail install; extraction diffed byte-identical vs reference tooling.

Container used by Shadow of War for all packed game data. Verified against
every archive in a retail install. All integers are little-endian.
Implemented by `anduril-ltar`.

## Container

```text
+0x00  char[4]   magic "LTAR"
+0x04  u32       version = 4
+0x08  u32       names_size     byte size of the name blob
+0x0c  u32       folder_count
+0x10  u32       file_count
+0x14  u32       1 in stock archives; 0 in community-built archives
                 (which load fine — not validated)
+0x18  byte[24]  zero
+0x30  name blob: packed NUL-terminated UTF-8 strings; a "name_off"
       anywhere is a byte offset into this blob
+..    file entry table: file_count x 30-byte entries
+..    folder table: folder_count x 16-byte entries
+..    file payload chunks
```

## File entry (30 bytes)

```text
+0x00  u32   name_off   -> name blob (basename)
+0x04  u64   offset     absolute payload position in the archive
+0x0c  u64   zsize      bytes on disk (== size means stored raw)
+0x14  u64   size       decompressed size
+0x1c  u8    dummy      (1 in stock archives)
+0x1d  u8    flags      (9 on stock compressed entries)
```

Community-built archives write `0, 0` here and store files raw; the
engine serves them anyway, so neither byte gates loading. Stock
archives also leave **6 zero bytes per file** between the folder
table and the first payload; community archives omit them (payload
offsets are absolute, so the gap is never read). CONFIRMED on
`Patch_00` vs `primalroar`/`orcgrenades_all`; THaych documents the
same stock layout.

**Retail read path (Ghidra `FUN_1411ae284`, byte-verified):** the
loader checks `LTAR` + `version==4`, then reads each file entry as
`{u32 name_off; u64 offset; u64 zsize; u64 size; u8; u8 flags}`
into an 0x18-stride runtime record with the flag byte packed into
the **top byte** of the stored u64 (`flags<<56 | size`) — so the
runtime keeps sizes as 56-bit values and the flags live in the
high byte of each u64. A second 0x30-stride folder table is read
afterwards via `FUN_1411ae154`.

## Folder entry (16 bytes)

```text
{ u32 name_off, i32 first_child, i32 next_sibling, u32 sub_files }
```

Folder names are **full paths** (`fx\characters`; folder 0 is the root,
name `""`); `-1` terminates child/sibling links. The table is a
depth-first tree with children sorted alphabetically, and the engine
finds folders by binary search on the full path (THaych; the full-path
names and `-1` sentinels CONFIRMED). Files are assigned to folders
**sequentially**: each folder claims the next `sub_files` entries from
the file table (alphabetical within a folder); the file's VFS path is
`<folder>/<basename>`.

Paths are **VFS-rooted**: a leading `/` refers to the root of the mount, not
an OS-absolute path. Normalization: `/` separators, case-insensitive
matching (the engine's filesystem semantics).

## Payload chunks

`size == zsize` → `size` raw bytes at `offset`.

Otherwise `offset` points at a chunk stream:

```text
per chunk:  { u32 chunk_zsize, u32 chunk_size }  then chunk_zsize data bytes
```

- `chunk_zsize == chunk_size` → raw; else Oodle-compressed (`OodleLZ_Decompress`
  from the install's own `oo2core_5_win64.dll` — we load it at runtime, we do
  not ship it).
- Next chunk position: `entry_offset + align4((data_off + chunk_zsize) -
  entry_offset)` — i.e., each chunk's `(8 + zsize)` record is 4-aligned
  relative to the entry's data start.
- The stream ends when `zsize` bytes are consumed; output is truncated to
  `size`.
- Alignment padding is ASCII **`'X'`**; chunks hold at most **64 KiB** of
  raw data (CONFIRMED on `Patch_00`).
- Compressed chunks start with an Oodle header: **`8C 06` = Kraken**
  (decodable by open-source `ooz`) and **`8C 02`** = an older Oodle codec
  used by `.bndl` and some text files that only the game's own
  `oo2core_5_win64.dll` decodes (both CONFIRMED present in `Patch_00`;
  decoder support per THaych).

## Mount semantics (engine behavior we reproduce)

- `x64/default.archcfg` lists sources one per line, `;` comments; paths are
  relative to the `x64/` directory. Entries are archives *or* directories
  (a directory is a loose-file root).
- Search order is **bottom-up** — later lines have higher precedence.
- Loose `.arch06` files in the install root mount *after* the archcfg list
  (community-observed; THaych flags it as unverified).
- Directories are only sources when listed — plain loose files dropped in
  the install root are ignored (THaych). A `..\Mods` line in archcfg is
  what makes a `Mods\` folder a source.
- A mod archive serves a file only if its own
  `archivepatchloosefiles.txt` lists it — **except `game.gamedb`, which a
  mod archive always supplies** (THaych; layout in `runtime/vfs.md`).
- Also per THaych (unverified): `-arch <file>` on the command line mounts
  an extra source.

## Known gaps

- `flags` byte semantics unverified (stock `9`, community `0`, both load).
- LTAR writers exist community-side (THaych's `arch06.py`, packing
  Kraken through the game's `oo2core_5_win64.dll`); stored-raw archives
  with a correct `archivepatchloosefiles.txt` are enough for a working
  mod.
