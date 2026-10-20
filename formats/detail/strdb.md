# `string.strdb` — SKDB v9 string database

> **Status:** **proven**** — full-corpus read + `.strdelta` merge
> **Provenance:** full `.strdb` corpus read + `.strdelta` merge on the real install.

Per-locale table mapping `key_hash -> UTF-16LE text`, found at
`database/string/<locale>/string.strdb` inside the archives and
overridable via `Mods/database/string/<locale>/string.strdb`.
Implemented in `anduril-strdb`.

## Layout

All u32 little-endian. Header is 7 words (0x1c bytes), sections follow
in order:

```
+0x00  char[4]  magic = "SKDB"
+0x04  u32      version = 9
+0x08  u32      index_offset        (0x1c — index starts right after)
+0x0c  u32      index_count
+0x10  u32      index2_count        (0 in retail files)
+0x14  u32      string_char_count   (in u16 CHARS, not bytes)
+0x18  u32      trailing_blob_size  (0 in retail files)
+0x1c  {u32 key_hash, u32 char_index}[index_count]  sorted by hash
+..    index2[index2_count]         same entry shape (empty in retail)
+..    string space: UTF-16LE, u16-NUL terminated,
       exactly string_char_count u16 chars
+..    trailing blob (empty in retail)
```

Retail scale: the enus file in `HotChunk.Arch06` is ~7.8 MB with
68,547 index entries and 0x374e8e string chars.

## Index and string space

Index entries are `{key_hash, char_index}` pairs sorted ascending by
hash (lookup is a binary search). `char_index` is a **u16 character
index** into the string space: `text = strbuf + char_index * 2` bytes,
decoded UTF-16LE to the first u16 NUL.

There is no flag bit. Odd char indices are suffix-deduplication
pointers into the middle of longer stored strings — e.g. a key whose
text is `Game` can point at char 9 of a stored `Continue Game`.
A few hundred retail entries land mid-string; all are in-bounds.

## Key hash

The key hash is the engine's record-name hash:

```
h = 0
for each byte c of the key:  h = h * 0x397 + fold[c]   (mod 2^32)
```

`fold` is a 256-entry table recovered from the executable that folds
case and remaps punctuation — `name_hash("Predicate.Not")` equals
`name_hash("predicate.not")` = `0x929db77b`. `anduril_strdb::name_hash`
implements it.

## Writer rules

- `string_char_count` is a char count — the reader consumes exactly
  `field * 2` bytes for the string space. Writing a byte count there
  doubles the stream length and corrupts the file.
- Strings are stored once per unique text (dedup); index offsets are
  `byte_pos / 2`.
- Terminator scanning must step by 2 from an even offset —
  `find(b"\x00\x00")` false-matches the odd overlap of an ASCII char's
  high byte against the next string's terminator.
- Index entries must stay sorted by hash for binary search.

## String deltas (`.strdelta`)

Mods ship mergeable deltas under `strings/**/*.strdelta` instead of
whole strdb copies, so multiple mods can patch strings. One directive
per line; `#` comments and blank lines are fine; fields are
tab-separated with `\t` `\r` `\n` `\\` escaping:

```
set      0x<hash>   <text>          add or replace an entry
del      0x<hash>                   remove an entry
replace  <old text> -> <new text>   rewrite every entry with that text
```

Warnings are emitted for `set` over an existing different text, `del`
of an absent hash, and `replace` matching nothing. Applied in mod
load order — later mods win.

## Verification notes

Decoding was cross-validated two ways: field-exact agreement with the
independent reference parser on the 63,089-entry extracted corpus file,
and a hash-keyed diff against the retail `HotChunk` strdb mounted
through the real VFS — 63,031 shared keys, only 792 differing, all
legitimate localization revisions between patch versions (e.g.
"Nazgul" -> "Nazgûl"), not parse artifacts.

Corpus warning: the Nemesis `extracted/strdb/string.tsv` golden file is
itself a product of an earlier buggy dump (~half its hash->text
mappings are wrong, and `enus.tsv` is UTF-16-as-UTF-8 mojibake). The
reference *parser* is right; that TSV is not trustworthy as a golden.
