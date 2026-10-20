# Spec: Lua host — sandboxed `anduril` API surface

> **Status:** working — single-VM host running every enabled script
> **Provenance:** `anduril-lua` (`crates/anduril-lua/src/lib.rs`
> ScriptHost), API contract in `docs/MODDING.md`. Consolidated
> 2026-10-19.

One Lua 5.4 VM (`mlua`, vendored — no external DLL) hosts all script
mods. Each mod's `scripts/*.lua` run in a dedicated environment table
with whitelisted globals plus the `anduril` API — **no** `io`,
`os.execute`, `package`, `debug`, or unrestricted `load` (a wrapped
`load` stays inside the mod's environment). **VERIFIED.**

## Lifecycle

- `load_mod` runs scripts in filename order, then `on_load` if
  defined; `unload_mod` runs `on_unload` and drops the environment.
  **VERIFIED.**
- `anduril.on(event, fn)` subscribes; `ScriptHost::emit` fires
  subscribers. **VERIFIED.**
- Every script call is protected: a Lua error becomes a host warning
  and never aborts the host or other mods. **VERIFIED.**

## API surface (VERIFIED — contract in `docs/MODDING.md`)

- `anduril.log`, `anduril.mod` (own manifest), `anduril.mods()`,
  `anduril.mod_active(id)`, `anduril.feature(tag)`
- `anduril.vfs.exists/read` — read-only access through the VFS
- `anduril.db.lists/records/get` — merged GameDB reads;
  `anduril.db.set/setref/setcell` — writes as merge-time deltas
  (strings intern like TOML `set`; `setcell` is the raw-index escape
  hatch)
- `anduril.sim` — nemesis-sim surface (member/board queries; see
  `docs/MODDING.md`)

## Design rules

- The host is our API, not the retail `LuaQ` VM surface — retail
  scripts run under `anduril-scri`'s interpreter (`../../formats/detail/scri.md`);
  the mod surface is intentionally different and documented as such.
- Determinism: event dispatch order is stable (load order, then
  subscription order). **VERIFIED.**

## OPEN

- Per-mod capability flags beyond the global sandbox (e.g. a mod
  requesting write access) — not yet designed. **UNVERIFIED.**

## Andúril model

`anduril-lua` (`ScriptHost`) implements exactly this surface — the
spec is the API contract; `docs/MODDING.md` is the modder-facing
how-to for the same surface. Deliberate non-goal: binary-compat with
the retail `LuaQ` VM (`anduril-scri` covers retail scripts instead).
Parity n/a — our API by design.
