# Sources and credits

> **Status:** **living****
> **Provenance:** canonical provenance page — who found what.

Every claim in this corpus is original prose. Findings derive from two
hands-on projects plus published community work — all credited here
and inline per doc.

## Authorship

**Every document in this corpus is authored by Andúril (Alistair
Afton).** The corpus is Andúril's evidence documentation: findings
verified by its own parsers, probes, and renders against a legally
owned retail install — tests are `ANDURIL_GAME_DIR`-gated so nothing
proprietary is needed to read the code.

## Evidence sources

### sow-rs / OnSight's shared field notes

**OnSight** is the author of `sow-rs`, a parallel Rust/Bevy
reimplementation in development. He shared a folder of his reverse-
engineering field notes with this project — his *"OnSights Docs"*
bundle (`combat.md`, `hud-map.md`, `moveset-map.md`, `formats/…`) —
which we use as an *evidence source*. Where his notes inform a claim
it is credited inline (e.g. moveset record chains, combat formulas,
HUD map details, CADB traversal markup semantics, material censuses,
exe-verified loader landmarks). Where our work overlapped we
validated against retail bytes; every overlap agreed, and the
corrections we made (declared `.inst` layout not being universal,
f16 pad-lane variance, the LTAR 26-vs-30-byte entry stride) are noted
inline. Findings labeled **VERIFIED** cite a named source and method;
those sourced from his notes say so.

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
  sibling to this repo; the `runtime/` department and `reference/native-*`
  docs are ported from its model/function/struct notes. Those docs keep
  Project Nemesis' own per-claim confidence tags and are marked "not
  independently verified" — treat UNVERIFIED/MODEL entries as leads,
  not settled fact.
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
