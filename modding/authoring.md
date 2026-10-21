# Authoring & round-tripping files

> **Status:** **living**
> **Provenance:** tool commands verified on the reference install
> (Andúril CLI); community tools per `../sources.md`.

Creating a file is always three steps: **out of the binary** →
**edit in a sane form** → **back into the binary layout the game
loads**. This page is the per-format recipe; per-format byte specs
are in `../formats/`, delivery in `packaging.md`.

## The golden rule

You almost never hand-edit the game's binary formats. Every format
here has (or needs) a **decoded intermediate form** — JSONL records,
TOML deltas, PNG, OBJ, TSV strings — that you edit, then a tool
re-emits the binary. The formats that lack a round-trip writer are
marked NO WRITE PATH; for those you build the emitter yourself from
the `../formats/detail/` spec.

## `game.gamedb` — the sane way (no hex, no insanity)

The database is the worst format to hand-edit and the most valuable —
so the whole pipeline is built around **never touching it raw**:

```
1. Inspect:   anduril db rec Role_Slaver          # one record, decoded
              anduril db refs Trait_Fire          # who links to it
              anduril db emit db.jsonl            # whole DB -> JSONL
              anduril db lists Faction            # browse categories
2. Author:    write TOML deltas — Mods/<mymod>/db/mypatch.toml:

              [[record]]
              list = "Faction/Roles/Definitions"
              name = "Role_Slaver_Mine"        # clone + tweak
              new  = true
              clone = "Role_Slaver"
              set  = { "BaseWeight" = 200.0 }

              [[record]]
              list = "Combat/AOE"
              name = "SomeRecord"
              set  = { "3" = 42 }              # raw cell index
              ref  = { "1" = "Refs/Things/X" } # repoint a link cell

3. Merge:     anduril db merge            # apply all enabled mods, report
                                          # conflicts/warnings first
              anduril db write game.gamedb   # emit one merged valid file
4. Ship:      drop at Mods\database\game\game.gamedb (see packaging.md)
```

`new = true` + `clone` copies a donor record so required cells are
never missing; `set`/`ref` then patch it. Unnamed struct records use
`name = "#<idx>"` (or `new` with no name to append). Cells accept raw
indices or schema field names — prefer names; index cells can't be
kind-checked.

To target the **patch route** instead of a merged file: the same
authoring produces your added records — a `.gamedb_patch` emitter is
the missing tool (spec in `../formats/gamedb-patch.md`); the merged
`db write` output is the proven route and needs no lineage stamp.

**Strings** round-trip the same way: `anduril str emit out.tsv` →
edit TSV → author `Mods/<mymod>/strings/*.strdelta` (`set 0x<hash>
<text>`, `replace <old> -> <new>`, `del 0x<hash>`) →
`anduril str merge out.strdb` → ship loose.

## Textures

```
out:  anduril tex <path> png out.png    # mip0 -> PNG (BC1/BC3/R16/RGBA)
      anduril tex <path> dump out.dds   # raw DDS payload
edit: any image tool; re-encode DDS (BC1/BC3/BC5/BC6H;
      normals = BC3 X-green/Y-alpha)
back: wrap the DDS back into the TEXR container (spec:
      ../formats/detail/tex.md); `.texa` = DX10 arrays
```

## Meshes & skeletons

```
out:  anduril mesh <path> obj out.obj [lod]   # Wavefront OBJ + .mtl
      anduril mesh <path> meshes              # layout/verts/groups
      anduril skel <path> bones
edit: Blender/Max against a native template (same rig + grip)
back: MMSH write — Monstey Mesh Publisher exports it; recompute the
      mesh allocation counters (../formats/detail/mesh.md)
view: anduril mesh <path> view  (interactive wgpu) | render out.png
```

## Animations

```
out:  anduril anix <path> clips -v            # codecs, tracks, events
      anduril anix <path> sample <clip>       # engine-exact evaluation
      anduril anix <path> skin <skel> <mesh> <clip> out.png  # visual check
edit: pose data is decodable; authoring clips = emit a `U*`
      uncompressed-codec track (simplest format) — toolchain immature
back: ANIX writer does not exist yet (NO WRITE PATH beyond U*)
```

## Scripts (`.scri`/`.bndl` script bundles)

```
out:  anduril scri <path> protos   # hstructure prototypes -> TSV
      anduril scri <path> chunks   # LuaQ chunk extents
edit: source-level Lua (the VM runs 81.7% of retail bodies — see
      ../runtime/scriptvm.md)
back: HavokScript/LuaQ compiler — NO WRITE PATH; mod logic normally
      lives in plugin Lua instead (packaging.md route 3)
```

## Audio

```
out:  .pck/.bnk are Wwise containers — `anduril bnk <path>`
      summarizes; wem payloads extract with standard Wwise tooling
back: Wwise-authored .bnk (version match matters) — PARTIAL
```

## Placements / world

```
out:  anduril embb <path> objects   # OBJL record names in a region bundle
edit: .wldclnt records are 76 B typed structs — editable per
      ../formats/detail/wldclnt.md
back: same file, in place in your override archive/loose tree —
      write path exists but world-edit workflows are young
```

## Saves (for inspection/migration only)

```
out:  anduril save <file.sav> chunks   # decrypt SOM3 -> chunk map
edit: chunk-level edits; every size field must stay consistent
back: re-encrypt round-trips byte-for-byte (../formats/detail/save.md)
      — but mods should never write saves; see testing.md
```

## No-write-path formats

`.fx`/`fxo` shader banks (DXBC — no compile path), most of
`.scri`, and `.hkai` navdata have readers but no writer. For these,
the practical "edit" is a plugin hook at load or a neighboring
data record that redirects the reference.

## The general recipe

For any format with no tooling:

1. Read `../formats/detail/<fmt>.md` — layouts here are
   byte-verified, labeled, and tolerant-read safe.
2. Extract samples: `anduril extract <substr> outdir` or QuickBMS.
3. Write the smallest possible change (one field, one record) with a
   hex editor or a 50-line script.
4. Diff-test: boot offline (testing.md), watch for silent ignores —
   a malformed file usually loads *unpatched*, not crashed.
5. Once one edit round-trips, automate: the format is simple enough
   that a writer is a weekend script, not a research project — that's
   the design intent of the whole corpus.
