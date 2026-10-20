# Spec: Mod manager — manifests, load order, dependency diagnostics

> **Status:** working — implemented and exercised; real-install mod
> **Provenance:** `anduril-mods` + `docs/MODDING.md` (manifest schema,
> load-order rules, CLI contract). Consolidated 2026-10-19.

Every mod is a folder under `Mods/<id>/` with a `mod.toml` manifest,
optional `files/` overlay, `deltas/` record patches, `scripts/` Lua,
and `*.arch06` payloads. **VERIFIED.**

## Load order

- `Mods/loadorder.txt` lists enabled mod ids, one per line, **weakest
  first, strongest last** — the same bottom-up rule as
  `default.archcfg`. `#` comments a mod out (that *is* the disabled
  state). **VERIFIED.**
- Absent the file, order is derived: `load_after`/`load_before`
  resolved topologically, ties by `priority` (higher = mounts later =
  stronger) then id. A dependency cycle is a **hard error**, reported,
  never silently resolved. **VERIFIED.**
- Discovery never writes; `loadorder.txt` is created on the first
  mutating command (`enable`/`disable`/`move`). **VERIFIED.**

## Manifest rules

- `depends = [...]` — required ids; a missing dep skips the mod (and
  reports it) and also implies ordering. **VERIFIED.**
- `integrates = [...]` — soft ordering: mount after if present, fine
  if not. **VERIFIED.**
- `conflicts = [...]` — both-sides-enabled is a diagnostic, surfaced
  by `anduril mods check`. **VERIFIED.**
- `provides = [...]` — feature tags (`orc-names`); other mods can
  depend on the tag rather than a specific id, and Lua sees them via
  `anduril.feature`. **VERIFIED.**

## Overlay conventions

Two coexist deliberately: the stock flat `Mods/<vfs-path>` engine
overlay (always active) and the managed `Mods/<id>/files/<vfs-path>`
packaged overlay (gated by loadorder.txt). `files/` is the convention
for anything new; flat is kept for compatibility with existing drops.
**VERIFIED** — see `vfs-mount-order.md` for where each layer sits.

## OPEN

- Hot reload of load-order changes without restart (watch `loadorder.txt`
  + manifests) — design intent, not yet implemented. **UNVERIFIED.**

## Andúril model

`anduril-mods` implements the manifest/order/diagnostics contract
verbatim — this spec *is* our design (there is no retail counterpart;
the retail engine only knows the flat `Mods/` overlay). The flat
overlay is kept for compatibility with existing drops. `anduril mods`
is the management CLI. No deviations to document beyond the OPEN
hot-reload item.
