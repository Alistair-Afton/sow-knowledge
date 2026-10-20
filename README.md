# Shadow of War — Shareable Knowledge Base

A community knowledge base of everything known about *Middle-earth:
Shadow of War* (Steam appid 356190, retail build 1.0.8874 / content
build 3095001): file formats, runtime behavior, and the modding
surface — written for modders, engine programmers, and AI agents
working on any goal: reimplementation, tooling, data extraction,
mods, or research.

**Legal:** documentation only. No game files, extracted assets, or
decompiled code. Everything here was derived by observing the files of
a legally owned install and (where noted) disassembling the executable
read-only. Original wording throughout.

## Evidence status labels

Every claim carries one of these tags — the full contract (what counts
as proof for each, why structure ≠ semantics) is in `EVIDENCE.md`:

| Label | Meaning |
|---|---|
| **CONFIRMED** | Independently reproduced on real install data (by at least one named project — byte-level checks, multiple samples, or a working decode) |
| **VERIFIED** | Verified by a named source with a stated method and corpus; not independently reproduced |
| **PARTIAL** | Real evidence exists but interpretation is incomplete or only checked on a few samples |
| **HYPOTHESIS** | A consistent guess; not proven |
| **PLACEHOLDER** | A working approximation known to be wrong/incomplete |
| **OPEN** | Known unknown; no reliable answer yet |

## Contributing

Yes — contributions are wanted, including from AI agents. This corpus
is **evidence-reviewed**: contributions arrive as *claims + evidence*,
not free text. Read `CONTRIBUTING.md` (the submission contract) and
`EVIDENCE.md` (the labels). `open-questions.md` is the work queue.
`sources.md` credits every project this corpus stands on.

## Sources

- **Andúril** (author: **Alistair Afton**): clean-room open-source
  Rust engine reimplementation
  (in development, not yet publicly released). **Every doc in this
  corpus is authored by Andúril** — findings verified against the
  install by its own parsers, probes, and renders.
- **sow-rs** (author: **OnSight**): a parallel Rust/Bevy
  reimplementation, also in development. OnSight shared a folder of
  his field notes which we use as an *evidence source* (moveset
  record map, combat notes, HUD map, format notes) — credited inline
  per claim. Where the two projects' work overlapped, claims were
  validated against retail bytes; every overlap agreed.
- Universal-modder field notes ("Goblins of War"), QuickBMS
  `shadow_of_mordor.bms` v0.4, SoW Modding Community ImHex-Patterns /
  MeshParser, LordMonstey's Mesh Publisher, WarpZephyr's
  MordorFormats — used by upstream projects as documentation.

## Reading map

| Doc | Contents |
|---|---|
| `engine/engine.md` | Coordinate frame, units, name hash, VFS mount order, Oodle |
| `formats/index.md` | Every known file type, magic, version, status |
| `formats/archives.md` | LTAR `.arch06`, archcfg, EMBB/BNDL bundles |
| `formats/gamedb.md` | GADB binary (`game.gamedb`, `.bvr`, embedded GADB) |
| `formats/geometry.md` | MMSH `.mesh`, PRFB `.inst`, MESH `.wldmsh`, vertex declarations |
| `formats/animation.md` | SKEL `.skel`, ANIX `.anix` (all codecs) |
| `formats/materials.md` | LTMI `.mat`, TEXR/TEXA/SLVT `.tex*`, shaders, texture conventions |
| `formats/world.md` | WDSC `.wldclnt`, OBJR `.objres`/OBJL/CADB, TERR `.terrain`, VEGG |
| `formats/ui.md` | Scaleform `.gfx`, HUD records and widget map |
| `formats/detail/` | Full field-level spec per format — 31 detail docs linked from `formats/index.md` |
| `gameplay/detail/` | Retail-behavior specs for game systems (faction generation/lifecycle/pickers/spawn/XP rewards, moveset, combat, traversal, HUD map, nemesis data) |
| `engine/detail/` | Retail engine/runtime specs (VFS mount order, spawn pipeline, apply-event write path) |
| `runtime/` | Live-game runtime model ported from Project Nemesis forensics: nemesis memory/event log, spawning, console commands, script VM, UI, VFS, gamedb, engine map, native function decodes — per-claim tags preserved; **not independently verified** |
| `reference/` | Script classes, event surface, event-subscription tables, native schema/registry + gamedb intern maps |
| `gameplay/combat.md` | Stat formulas, evaluation rules, moveset chains, fight circle, meters |
| `gameplay/traversal.md` | Movement constants, context-action edges, traversal clips |
| `gameplay/nemesis.md` | Nemesis gamedb categories, captain generation, save data |
| `modding.md` | How to actually modify the game: loose files, load order, plugins, caveats |
| `open-questions.md` | Consolidated list of what is NOT known — the work queue |
| `EVIDENCE.md` | The evidence-label contract (what counts as proof) |
| `CONTRIBUTING.md` | How to contribute — claims + replayable evidence |
| `sources.md` | Credits and provenance for every project this stands on |

## Quick orientation for AI agents

- The game is **data-driven to an unusual degree**: combat, skills,
  meters, HUD composition, weather, and AI parameters all live in a
  giant typed record database (`game.gamedb`, GADB format). Most
  "how does X work" questions are answered by following record links.
- Geometry/animation/texture formats are individually simple; the hard
  part is the *binding* (which mesh group uses which material at which
  LOD) — documented where proven.
- All asset names resolve through one case-insensitive name hash
  (`engine/engine.md`). Paths in records are keys, not filesystem lookups.
- Real installs differ from any single doc: use tolerant parsers, and
  treat corpus scan counts as evidence for structure only.
