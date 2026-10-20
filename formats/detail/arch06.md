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
+0x14  byte[28]  reserved
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
+0x1c  u8    dummy      (always 1)
+0x1d  u8    flags      (semantics unknown; identical behavior observed)
```

## Folder entry (16 bytes)

```text
{ u32 name_off, u32 next_sub_folder_id, u32 next_folder_id, u32 sub_files }
```

Files are assigned to folders **sequentially**: each folder claims the next
`sub_files` entries from the file table; the file's VFS path is
`<folder>/<basename>`. The `next_*` links exist in the data but tooling only
needs the sequential `sub_files` counts (sufficient on all stock archives).

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

## Mount semantics (engine behavior we reproduce)

- `x64/default.archcfg` lists sources one per line, `;` comments; paths are
  relative to the `x64/` directory. Entries are archives *or* directories
  (a directory is a loose-file root).
- Search order is **bottom-up** — later lines have higher precedence.
- Loose `.arch06` files in the install root mount *after* the archcfg list.
- (Andúril extension) `Mods/` then layers on top: `Mods/*.arch06`, then the
  loose `Mods/` directory as the strongest source.

## Known gaps

- `flags` byte semantics unverified.
- Whether the `next_*` folder links encode a non-sequential order in some
  archives (none observed).
- LTAR *writing* is unimplemented by design — mods deliver loose files.
