# Sources and credits

> **Status:** **living**
> **Provenance:** canonical provenance page — who found what.

Every claim in this corpus is original prose. Findings derive from two
hands-on projects plus published community work — all credited here
and inline per doc.

## Primary sources — who found what

Roughly speaking: **Andúril is the primary source for file-format and
container structure; sow-rs is the primary source for gameplay/runtime
semantics.** Where both projects examined the same artifact we
validated against retail bytes; every overlap agreed, and corrections
(declared `.inst` layout not being universal, f16 pad-lane variance,
the LTAR 26-vs-30-byte entry stride) are noted inline.

### Andúril (author: Alistair Afton)

Clean-room open-source Rust engine reimplementation for *Middle-earth:
Shadow of War*. Findings labeled **CONFIRMED** are verified by
Andúril's own parsers, probes, and renders against a legally owned
retail install — tests are `ANDURIL_GAME_DIR`-gated so nothing
proprietary is needed to read the code.

Andúril-derived findings in this corpus include: the archive/container
layer (LTAR v4 corrected entry stride, EMBB v5, SLVT v4, PCK), the
GADB record format and GameDB structure, `.strdb` and the complete
`name_hash` algorithm + FOLD LUT, `.wldclnt` placement records and the
tail's region-index structure, `.terrain` quadtree node order and the
level-aware tile layering rule, OBJR section map + CADB v3 full decode
+ group-key derivation + world-space edge linking, `.scri` script
data, `.save` save structure, and the structural specs for `.bvr`,
`.bnk`, `.gfx`, `.uis/.uiwd`, `.veg*`, `.hkai`, `.subb`, `.vib`,
`.vid`, `.pmo_rcv`, `.fx`, `.mat`, `.tex` — the per-format detail docs
under `formats/` are Andúril's evidence docs mirrored verbatim.

### sow-rs (author: OnSight)

A parallel Rust/Bevy reimplementation in development by **OnSight**.
His field notes are the primary source for gameplay and runtime
semantics: combat formulas, the nemesis/captain systems, traversal
markup semantics (CADB action sets/masks), Scaleform tag walks,
material censuses, the wraith shader model, moveset record maps,
skinned-mesh attributes, and exe-verified engine landmarks/loader
addresses. Findings labeled **VERIFIED** are his, credited per claim.

> Both sow-rs and Andúril are works in progress and not yet publicly
> released — citations name them as *sources of evidence*, not as
> fetchable dependencies.

## Community documentation and tooling used as references

- **Universal Modder knowledge base** — field notes incl.
  "Goblins of War" and "Offline mode and local Garrison".
- **SoW Modding Community** — `ImHex-Patterns` and `MeshParser`
  (binary format references).
- **LordMonstey** — MIT-licensed Monstey Mesh Publisher and
  sow-anix-format (mesh + animation format notes; the Havok
  sibling-block palette-order detail is theirs).
- **Luigi Auriemma** — `shadow_of_mordor.bms` QuickBMS script v0.4
  (LTAR layout reference).
- **WarpZephyr** — MIT-licensed MordorFormats / MordorTemplates.
- **Adobe SWF File Format Specification v19** (public) — the standard
  SWF tags under `.gfx`.
- **Project Nemesis** — the native modding framework + knowledge base
  sibling to this repo; engine API surface and live-game memory reads.
- **`ShadowOfWarDllLoader.dll`** — the community plugin loader (a
  generic `dll_loader` build): enumerates `x64/plugins/*.dll` and
  `LoadLibrary`s each at startup; the entry point for all native mods.
- **Ghidra + REA (Reverse-Engineer-Anything)** — read-only executable
  analysis for loader addresses and format confirmation.

## Legal position

- Everything here is documentation of *facts observed* in a legally
  owned install — the same category of interoperability research the
  modding community has always produced.
- No game files, extracted assets, archives, textures, or verbatim
  decompiled code are distributed. Numbers quoted as evidence are
  field values, not asset data.
- Oodle (`oo2core_5_win64.dll`) is proprietary RAD Game Tools code —
  never distributed; consumers load it from their own install.
- Record/asset names appearing in docs (e.g. `PL_Walk`,
  `Orc_Infantry`, `powerbar1.mat`) are identifiers, cited for
  technical accuracy.
- Contributing to this corpus means writing original prose and
  citing evidence, never copying proprietary or third-party content —
  see `CONTRIBUTING.md`.
