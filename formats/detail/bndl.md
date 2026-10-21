# `.bndl` + `.embb` — BNDL v6 bundles (asset packages)

> **Status:** **verified** — manifest + section decode verified
> **Provenance:** manifest + section decode on the real `.bndl` corpus.

Every packaged asset group is a `.bndl` manifest paired with a
same-named `.embb` payload inside the LTAR archives — ~225k pairs
across the install (`WorldQuest` sections, fort kits, UI images,
cinematics, weather sets). The manifest alone answers "what does this
package contain" without touching the payload. Implemented in
`anduril-bndl`; `anduril bndl <path>` / `anduril embb <path>` in the CLI.

## `.bndl` (v6)

All u32 little-endian:

```
+0x00  char[4]   magic "BNDL"
+0x04  u32       version = 6
+0x08  u32       bundle_type_offset   (relative to section_base)
+0x0c  u32       resource_ptr_offset  (relative to section_base)
+0x10  u32       resource_count
+0x14  u32[count] resource_size[]     per-resource payload sizes
section_base = 0x14 + 4*count
  string section: NUL-terminated resource-path cstrings (empty/locale
    variant slots collapse to shared offsets), then the bundleType
    cstring at section_base + bundle_type_offset
  ptr table @ section_base + resource_ptr_offset: u32[count], each an
    offset from section_base to that resource's path cstring
```

Caveat: an older spec revision describes `{size, offset}` record
pairs; retail v6 ships the bare u32-offset table described above.

**Retail read path (Ghidra `FUN_140504c70`, byte-verified):**
`{BNDL, ver==6}` exact, 0x14-byte header read in one shot. The
loader bulk-reads the `u32[count]` size array and the whole string
section, walks `count` offsets each **bounds-checked against the
section size**, and copies a 16-byte record per offset. The
`bundle_type` string is located by scanning the section for a
`0x3a` (`:`) byte — bundle types are `Name:Variant` cstrings.

## `.embb`

```
+0x00  char[4]   magic "EMBB"
+0x04  u32       version
+0x08  u32       path_array_size
+0x0c  u32       file_count
+0x10  char      path_array[path_array_size]   (concatenated cstrings)
+..    u32       path_offset[file_count]       (into path_array)
+..    per file: u32 size + raw data           (contiguous, in order)
```

The `.embb` payload inside an archive is usually Oodle-compressed at
the LTAR level — `anduril-vfs` decompresses it transparently before
the container parser ever sees it; entries inside are stored raw.

## Bundle types (island census)

`WorldQuest` (114), `FlairWorld` (28), `ThroneRoomBarrows` (17),
`QuestCinBundle` (17), `FortTraits` (10), `KrakenBaseKit` (7),
`KrakenBaseLevel` (6 — region master manifests), plus smaller
siege/DLC/UI types. `KrakenBaseLevel`'s `island.bndl` lists 3,876
resources — the per-region dependency graph.

## Load-order semantics (externally sourced — Universal Modder KB,
`packet-loader-mods-without-packet-loader.md`; not yet re-verified
byte-level by us, but consistent with `default.archcfg` bottom-up
mount order)

- **Newest `.bndl` wins.** When several archives manifest the same
  resource path, the latest `.bndl` in archive/mount order decides the
  effective set. To *remove* an asset, ship a newer `.bndl` that omits
  it; to *replace* one, provide the loose payload and update its size
  field in **every** `.bndl` that references it — one stock path can be
  carried by several bundles.
- `resource_size[]` must match the replacement payload size; a stale
  size is a load failure, not silent ignore.
- Texture replacement must cover `.tex`/`.tex0`/`.texuhd` companions
  consistently or the engine picks up mismatched mip strips.
- `Flair_*MB` bundle-type strings are per-budget pools the loader uses
  for flair streaming.
- **New resources must be bundle-listed.** A new UI texture path
  shows **magenta** in game unless a loaded bundle lists it; adding
  the path + size to `interface\bundles\ingame_tex.bndl` (type ending
  in `UI_Textures`) makes it load (THaych, in-game — patched in
  memory via the BNDL loader hook).

## Verification

- Corpus diff vs the independent reference parser: 200/200 manifests
  in `extracted/bndl_island` — bundle type, count, and every resource
  path field-exact.
- `.embb` verified on all 5 extracted payloads plus a live
  Oodle-compressed `bundles/goldenpath/carnun.embb` read through the
  VFS.
- Malformed inputs (bad magic, wrong version, truncated tables, OOB
  string offsets, file-size overflow) return errors, never panic.
