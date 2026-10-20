# Spec: VFS mount order and precedence

> **Status:** verified — engine-faithful order reproduced; extracts
> **Provenance:** `x64/default.archcfg` header comment + file search
> order on the real install; `anduril-vfs` (`Vfs::mount_install`,
> `crates/anduril-vfs/src/lib.rs`); `docs/MODDING.md` precedence rules.
> Consolidated 2026-10-19.

The virtual filesystem resolves a file by walking mounted sources from
strongest to weakest; the first source containing the path wins.
**VERIFIED.**

## Mount order (weakest → strongest)

1. `x64/default.archcfg` entries, in file order. Each entry is a path
   relative to the archcfg's directory; an entry that resolves to a
   directory mounts it as a source dir, one ending `.arch06` mounts the
   archive. The file's own header documents bottom-up precedence
   (later entries stronger). **VERIFIED** — matches stock install
   behavior.
2. Loose `*.arch06` files in the install root, sorted by name,
   skipping any path already mounted. **VERIFIED.**
3. Loose `*.arch06` in `Mods/` (engine-faithful layer). **VERIFIED.**
4. The flat `Mods/` directory itself — the engine mounts it via the
   `..\Mods` archcfg line, so `Mods/<vfs-path>` files shadow packed
   content directly. **VERIFIED** — stock engine behavior, always
   active regardless of the mod manager.
5. Each enabled packaged mod, in `loadorder.txt` order: within a mod
   its packed `*.arch06` payloads first, then its `files/` overlay.
   **VERIFIED** — Andúril convention layered on engine semantics; see
   `mod-load-order.md`.

## Rules

- Whole-file shadowing: a file in a stronger source replaces the same
  VFS path entirely; there is no partial merge at the VFS layer.
  Record-level merging is a separate concern (see
  `gamedb-overlay-merge.md`). **VERIFIED.**
- Mount failures are warnings, never fatal — a missing archive or dir
  is reported and skipped so one bad drop can't kill the install.
  **VERIFIED** (`mount_install` returns `Vec<String>` warnings).
- Duplicate archives (same path reachable twice) are deduplicated at
  mount time. **VERIFIED.**

## Notes

- `anduril mounts` prints the resolved order for sanity checks.
- Game assets stay read-only at runtime; the VFS never writes into
  sources.
