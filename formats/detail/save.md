# Save files — `SOM3` / `SAVE` (M13 persistence)

> **Status:** **verified**** — real endgame saves parse; captain census extracted; semantics still expanding
> **Provenance:** real endgame saves parsed; 534-record captain census extracted.

Externally sourced spec (Universal Modder knowledge base, MIT —
`middle-earth-shadow-of-war/save-format-decrypt-and-collectible-quests.md`,
verified in-game by its author on build 3095001). **Container and
decryption verified by Andúril** — `anduril-save` decrypts all 6 retail
saves in `%LOCALAPPDATA%\WB Games\Shadow of War\` with exact CRC32 and
length agreement; chunk semantics below are from the external note.

## Container (`SOM3`)

```text
+0x00  char[4]  "SOM3"
+0x04  u32      crc32(plaintext)        — standard CRC32
+0x08  u32      ciphertext length
+0x0c  u32      plaintext length
+0x10  ...      AES-256-CBC ciphertext (PKCS#7)
```

- **IV** (16 ASCII bytes): `yuwgb@oftv@gx$t3`
- **Key** (32 ASCII bytes): `ad@210766@vac94Cd_?dVt5$alivjz$e` with
  eight bytes overwritten:
  - Steam account id as 4 LE bytes (also the hex prefix of the save file
    name) → key indices **10, 30, 22, 2**
  - plaintext CRC32 as 4 LE bytes → key indices **26, 11, 18, 17**
- Re-encryption round-trips byte-for-byte. The save exists in **two**
  places (`Steam\userdata\<acct>\356190\remote\` and
  `%LOCALAPPDATA%\WB Games\Shadow of War\`); the game writes both.

## Plaintext (`SAVE`)

- `SAVE` v2 header; total size at `+0x0C` and `+0x10` (= file − 20).
  **Verified**: `total == plaintext.len() - 20` on all 6 retail saves.
- Global chunks, then slot records `06000000 <u32 len> 00000000` each
  followed by a `SAVE` v31 block of `len` bytes.
  **Verified on retail**: 2 slots per save — slot 1 (campaign,
  ~1.5 MB, contains IVPD/WHDC/ASMC) and slot 2 (~288 KB to EOF).
  The external note's "3 slots" (u32 3 = slot count) describes the
  slot-count field; empty slots may have no `SAVE` block.
- Inside a slot: chunks `<4CC> <u32 ver> <u32 size> <u32 0> <u32 size-4>`.
  **Every size field changes when a chunk grows.**
- Observed chunks (verified): `GOXC` v5 global; slot-1 inner
  `IVPD` v4, `WHDC` v1190 (≈1.27 MB world/nemesis state — AttrHash
  probes place the live `FactionRole` roster inside), `ASMC` v8,
  `CBTC`, `GCPT`, `LPDC`, `UITC`, `LTMR`, `DIFF`, `IUC0`.
  (`PAIC` also exists in the editor's chunk model.)

## WHDC captain records (verified against retail save + dictionary)

WHDC is a tagged byte stream; u32s at any alignment that equal a
`name_hash` are embedded record/asset names. Records **start with a
bare 4-byte name hash** (no type tag — e.g. `Role_SlaveLover` =
`01 dd 22 2e` where the `01` is part of the hash, not a tag), followed
by scalar fields and a `00`-prefixed list of name-hash references.

A **captain record** reads (all verified, 534 records in a real save):

```text
[spawn-point name hash]     MP_FortFront, AD_Orc_Body_LOD, …
<Role_* hash>               Role_Torch, Role_SlaveLover, …
<TribeDef_* hash>           TribeDef_Dark, TribeDef_Terror, …
scalars                     small u32s / f32s (level, faction id, …)
00 <hash> …                 marker list: Marker_Potential_*,
                            ClassMarker_*, PersonalityMarker_*,
                            Marker_AppearanceTag_*, Marker_LearnSpeed_*,
                            Marker_OrcRarity_*, Marker_DominationShamed,
                            Marker_ShamedOnce/Twice, Picker_* (resists,
                            vulns, mounts, gangs, hates, behaviors),
                            Traits_*
```

Captains are identified to other systems by `GUID64A`/`GUID64B` u64
pairs (per the runtime dump), not by name hash — display names hash to
a different table. The `IVPD` chunk (before WHDC) holds the role pool:
a dense `<Role_* hash>` + small-u32 list (~914 entries).

CLI: `save <f> whdc` (full resolved-hash dump) / `save <f> captains`
(clustered captain roster). Requires `ANDURIL_HASHDB` pointing at the
harvested name dictionary.
- Value tags: `04` = u32, `00` = 32-bit AttrHash name (same
  `h*0x397 + map[c]` hash as gamedb/strdb), `0c` = bool, `10` =
  length-prefixed blob.

## Content chunks (from the note)

- **QSTD** — quest states: `04 <count>` then per quest
  `00 <hash> 02 <lead> (0c <bool>)×9`, sorted by hash.
  Completed = `lead 1, flags 0,1,x,0,1,0,0,1,0`. The pattern
  `1,1,x,1,0,0,0,0,1` = unlocked-but-not-done (looks done in data,
  isn't in UI).
- **IIUC** — world collectibles: ends with the list of pickups still
  in the world (`04 <n>` + n × `04 <hash> 0c 01`); the game prunes
  owned items on load.
- **IVMG** — inventories: `04 <entities>`, per entity
  `00 <id> 0c <has> 10 <u32 len> <blob>`; the player blob opens
  `00 c9b7c7b7 04 a 04 <entryCount>` where
  **entryCount = item records + trailing `0c` pairs + 1** (getting this
  wrong crashes the load). Item record:
  `00 <item hash> 04 <count> 04 <a> 04 <b> 0c <flag>`. IVMG inner size
  u32 at chunk offset +33.

## Open questions (from the note)

- Where "Shadows of the Past" statue completion and quest-line rewards
  are stored; skill unlocks are not plain IVMG items.
