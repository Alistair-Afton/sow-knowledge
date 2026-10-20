# Evidence labels — definitions and examples

> **Status:** **living**
> **Provenance:** the label vocabulary used across the whole corpus.

Every claim in this corpus carries a label. The labels are strict on
purpose: they are what makes the corpus worth trusting. A claim's
label describes **how we know**, not how confident we feel.

## The labels

### CONFIRMED

Independently reproduced on real install data — byte-level checks on
real files, multiple samples, or a working decode producing correct
output. The reproducing project is named inline (e.g. "CONFIRMED
(Andúril)").

Example: PRFB `.inst` submesh descriptors are 52-byte D3D9-style
draw-parameter records — verified field-by-field on real files
(min/nv/idx_off/base/ntri all satisfy their stride relations on
multiple files) and by textured renders that use the decoded material
index.

### VERIFIED

Verified by a named source with a stated method and corpus — and not
independently reproduced by a second party. Treat as reliable, cite
the source.

Example: `.gfx` Scaleform tag walks — sow-rs reports exact walks over
all 19 installed files (47,559 tags). No second decoder exists yet,
so the tag layouts are VERIFIED (sow-rs), not CONFIRMED.

When a VERIFIED claim is later reproduced independently on real data,
it is promoted to CONFIRMED (the original provenance stays).

### PARTIAL

Real evidence exists but the interpretation is incomplete, checked on
few samples, or known to have exceptions.

Example: `.wldclnt` placement records — the canonical 76-byte layout
is verified on Minas Ithil, but other regions/files show additional
record layouts; the doc keeps both models with their evidence.

### HYPOTHESIS

A consistent guess — fits all observations, mechanism not proven.
Never implement it as fact.

### PLACEHOLDER

A working approximation used in code, known to be wrong or
incomplete. (E.g. the spec→PBR channel mapping `spec=R, roughness=B`:
produces sane output, is not the game's real BRDF.)

### OPEN

Known unknown; no reliable answer yet. `open-questions.md` collects
these with starting points for investigation.

## What counts as evidence (per label)

| To claim | You need at minimum |
|---|---|
| PARTIAL | A real observation on real data (one sample, a parse, a render) with the source file named |
| VERIFIED | A method + corpus stated by a credited source (e.g. "exact tag walk, all 19 files"), or an independent reproduction described precisely enough to rerun |
| CONFIRMED | The reproduction *was run* on retail install data — the test/probe must exist and be cited |

## Evidence is about *semantics*, not just structure

- "The file parses to EOF" → proves container layout, says nothing
  about field meanings.
- "The decoded mesh renders without artifacts" → stronger, still not
  proof the fields mean what we named them (a wrong-but-consistent
  decode can render fine). Pair renders with field-relation checks.
- "A scan of all N files succeeded" → evidence the layout is
  universal *for that build*; variant flavors may still exist
  (see `fishandguts.inst`).
- A claim about *runtime behavior* (formulas, meters, AI) needs data
  + a mechanism — records alone tell you what the game *stores*, not
  what it *does*.
