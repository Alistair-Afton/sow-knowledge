# Contributing

> **Status:** **living****
> **Provenance:** contribution contract — the evidence rules this corpus runs on.

This corpus is **evidence-reviewed documentation**, not a wiki. Every
claim is labeled with how strongly it's proven; the point of the
project is that a reader (human or AI agent) never has to guess what
to trust. Contributions are welcome in exactly one form: **a claim
plus its evidence**.

## The contract

1. **Every claim carries a label.** See `EVIDENCE.md` for the full
   vocabulary and what counts as proof for each label. When unsure,
   label *lower*, not higher.
2. **Evidence is replayable.** A VERIFIED or CONFIRMED claim must cite
   something a reviewer can re-run: a file + offset, a tool + command,
   a test, a hex dump location, a record path in `game.gamedb`.
   "It parses" is not evidence of semantics; "record X's `Range` cell
   `0x17ac` reads `[200.0, 200.0]` at heap offset `0x47+cell`" is.
3. **New observations land at PARTIAL or below.** Promotion to
   VERIFIED/CONFIRMED happens when the observation is reproduced on
   real install data — by the reviewer if you can't run the game.
4. **Disagreements are documented, not overwritten.** If your finding
   contradicts a doc, state both readings and what distinguishes them
   (file variant? build version? misread?). We keep variant models
   side-by-side until evidence resolves them — see the `.wldclnt` /
   `.wldmsh` sections of `formats/world.md` for the pattern.
5. **Clean-room, always.** Original wording only. Do not paste
   decompiled code, extracted game data, textures, archive contents,
   or third-party code into the docs. Cite sources by name (see
   `sources.md`), describe behavior in your own words, and quote only
   the minimal record/byte excerpts needed as evidence.
6. **Never distribute proprietary material.** No game files, no
   `oo2core` DLL, no `.arch06` contents. A reader must own the game.
7. **Subject = the game, the engine, the install.** Docs describe
   *Shadow of War* itself — the engine, every file format, runtime
   behavior, game systems, and the retail installation layout. They
   are never about a *project*: Andúril, Project Nemesis/nemesis.dll,
   sow-rs, or any other tool's internals, APIs, plugins, or mod
   machinery do not belong here, except as provenance citations.
   They are also not the place for session logs, method journals, or
   one tool's implementation notes. Evidence pointers (tool +
   command + location) belong in provenance; the doc body should read
   as the current best description of the system, not a history of
   how it was discovered.
8. **Second-source claims carry their own confidence.** Material
   ported from another project's notes (e.g. `runtime/` — Project
   Nemesis) keeps its per-claim tags (VERIFIED-STATIC, VERIFIED-LIVE,
   UNVERIFIED, MODEL, …) and is marked "not independently verified".
   Do not upgrade those tags without reproducing the evidence.

## What to contribute

- **Answers to `open-questions.md`** — each entry says what's missing
  and where to start. These are the highest-value targets.
- **New format/behavior docs** for undocumented systems, in the same
  structure (status banner → layout → verified samples → open
  questions).
- **Corrections** — mislabeled claims are bugs; a diff with evidence
  is a high-value contribution.
- **Probes/tools** that produce the evidence (scripts, tests,
  experiments against your own install). Describe them enough that a
  reviewer can rerun the measurement.

## For AI-agent contributors

- Follow the conventions mechanically — they exist for you:
  label first, evidence path second, prose third.
- When you can't run the game yourself, submit the claim at the
  label your evidence supports and say what would verify it.
- If you found the corpus through a task ("extract X", "mod Y"),
  `formats/index.md` + `open-questions.md` are the fastest map of
  what's known vs. what's unclaimed territory.
- Don't silently restate a doc's claim back to it as new evidence —
  independent reproduction is what upgrades a label.

## Workflow

- PRs: one subject per PR (a format, a question, a correction), with
  the evidence in the description. A maintainer reproduces the
  evidence on a retail install before the label is merged.
- Raw evidence files (dumps, screenshots) stay out of the repo —
  describe them, cite paths, keep the docs self-contained prose.
- Conventions: claims tagged inline; each doc ends with its open
  questions; `Status:` banner at the top of every doc.
