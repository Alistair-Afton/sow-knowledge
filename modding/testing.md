# Testing a mod

> **Status:** **living**
> **Provenance:** launch checks observed on the reference install;
> save-safety items from shipped-mod incident reports (THaych).

## Go offline first

A modded database must never be exercised while signed in to WB
services. The game's WB/Hydra traffic rides WinHTTP — a search-order
`winhttp.dll` proxy answers it locally (offline sign-in, local
Garrison, local wallet), no code patching needed. Protocol:
`../engine/detail/hydra-online-services.md`.

## Launch health check

After every launch, `x64\modmenulog.txt` should show:

```
Injected successfully!
Offsets ... 52/52 resolved, 0 failed
```

Missing either line = a plugin failed to load; the game may still
boot but your mod isn't active.

## Functional test pattern (LordMonstey)

One mechanism at a time, each with its own check:

1. Fresh natural acquisition (e.g. a loot drop), not a console grant
2. Equip / sheathe / attack
3. Spectral mode
4. A second instance of the item
5. Full save → restart → reload cycle

A controlled grant proves none of the loot path.

## Save safety — known ways mods eat saves

- **Never write the player's save from a buff/data mod**, and never
  give a buff an `InventoryItemRemove` on a story item — a shipped
  mod lost `Talion_Isildur_Ring` from a save this way.
- Saves live in **two** places — `Steam\userdata\<acct>\356190\
  remote\` and `%LOCALAPPDATA%\WB Games\Shadow of War\` — the game
  writes both; back up **both** before testing save-adjacent mods.
- Save format is encrypted SOM3 but round-trips byte-for-byte;
  edits are possible (`../formats/detail/save.md`) — still treat it
  as read-only unless that is the point of the test.

## Quick failure table

| Symptom | Likely cause |
|---|---|
| Mod ignored entirely | Not in `archivepatchloosefiles.txt`; archcfg line dropped by Steam repair; wrong VFS path (case/backslash) |
| Crash on boot | Declared size/count inconsistent in a repacked file; bad `.bndl` size |
| Records missing / mod dropped | GADB name-hash sort or link repoint wrong on rebuild |
| Patch not applied | `.gamedb_patch` lineage stamp mismatch vs base `+0x20` |
| Works once, then stale | A loose overlay won over your newer archive — check mount order |
