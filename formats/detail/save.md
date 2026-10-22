# Save files — `SOM3` / `SAVE` (M13 persistence)

> **Status:** **verified** — real endgame saves parse; captain census extracted; semantics still expanding
> **Provenance:** real endgame saves parsed; 544-record captain census extracted.

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

A **captain record** reads (all verified, 544 records in a real save):

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

## Member identity tokens and relationships (verified on a retail save)

Each member record is preceded by a length-prefixed ASCII token
`0f <len> <32-hex>_NPC_<n>` (tag `0f`, 39-char value). The 32-hex
prefix is a per-board faction GUID — shared by every member of one
army, distinct per region board, with separate prefixes for the
Gondorian ally faction (`GP_Baranor`, `AI_Gondorian_*` members) and
special records. `NPC_<n>` is the per-board member index; this token
is the on-disk form of the `GUID64A`/`GUID64B` pair.

Relationship persistence — verified negative results:

- **No guid cross-references.** A captain's full token (ASCII) and its
  16-byte binary form occur only inside that captain's own record.
  Nemesis bonds, blood brothers and killer references are not stored
  as guid cross-references.
- **No numeric partner index.** Scanning captain records for aligned
  u32s equal to another member's `NPC_<n>` index yields only
  coincidence-level noise (levels, counts, array data); the
  reciprocal-pair check (A→B and B→A at the same field offset — the
  bilateral bond signature) finds no consistent structural slot.
- **What is stored:** relationship *state*, not edges —
  `Marker_Generate_BloodBrother` (the brother-generator side),
  `BloodBrother_SpawnIn` (partners pulled in by the spawn channel),
  `Marker_Instance_{BloodBroAmbush,BetrayalAmbush,ForcedBetrayal}`
  (live instance flags), and `Ambush_Presented_{BloodBro,
  BloodBro_Killed, Betrayal_BloodBro}` (presented-event history).
  Story captains ride the same mechanism: `BloodBrother_BazGaz`
  appears on the NPC_611/NPC_612 pair (consecutive indices — spawned
  together), alongside `BruzBrothers_{Baz,Gaz,Daz}{,Dead}` quest
  markers.
- **No shared pair-key hash.** Grouping every resolved *and* plausible
  unresolved name-hash field by owning record finds no hash shared by
  exactly two generated captains that could act as a bond key; the only
  verified shared relationship tokens are story-scripted
  (`BloodBrother_BazGaz`, `BruzBrothers_GazDead`). Shared-by-2
  unresolved values are all field-data artifacts, not AttrHash-shaped.
  Pairing is runtime-reconstructed from role + marker state, not a
  serialized key.
- Story-captain records use `Role`-containing names without an
  underscore suffix (`SQ_ZogRoleDef` — Zog, Sorcerous tribe,
  `Picker_RaiseDead`); a census keyed on `Role_` alone misses them.
- Gear items carry `05`-tagged 18-byte binary GUIDs (one per piece).
  A `CharEvent_*` hash hit inside one gear blob was a byte
  coincidence — but nemesis memory *is* persisted, as the `03`-tagged
  `(category, event-name-hash)` event log in the record layout below.
- **Cross-save diffs are permutations, not mutations**: aligning the
  same captain's record across two saves shows byte differences that
  are pure swaps — `Picker_*` hash entries trade positions. The
  marker/picker lists are unordered hash sets re-serialized in a
  different order on each save; positional diffing is not a field
  oracle.

## Member record layout (verified, `save rec` field walk)

```text
0f <len> <32-hex>_NPC_<n>   identity token (faction-guid + member idx)
09 <u64>                  type/category id — shared by members of the
                          same kind; all NPC_0 board sentinels share
                          one value. NOT a unique member GUID.
04 <u32> ×3, 0c <bool>,   head scalars, two (often empty) `0f` strings,
0f <str> ×2, 08/04 pair   then `08 <u32>` + equal `04 <u32>` pair
02 <u8>, 05 <18B>, 00     flag, 18-byte blob, one scalar field
00 <TribeDef_* hash>      tribe record
04 <count> +              marker list, count verified exact:
  00 <Marker_* hash> ×n     potential/class/personality/appearance/
                            learn-speed/rarity/specialization markers
04 <count> +              picker list, count verified exact — each
  00 <Picker_* hash> +      entry = ability/resist/vuln name hash
    00 <u32 param=0>        followed by a (zero) param field
04 <count> + 00 <name>…   trait names (Traits_*, Orc_<personality>,
                          sometimes led by Default_Act*)
08 <u32> ×9               stat array
03 <u24 cat> <u32 hash>   **event log — the member's nemesis memory**:
                          `03`-tagged (category, event-name-hash) pairs
                          grouped by log channel — 0x15f sim events
                          (PromoteDead_*, Chase_Clear, Raid_Checkpoint_*,
                          ActiveKillPlayer_*), 0x160 sub-events,
                          0x161 encounter history (GuardWarCouncil_*,
                          ActiveKilledOnSimEvent, interrogation/wraith-
                          world ids), 0x162 tail, 0 = identity/template
                          refs (Orc_Spearman, OlogHai_Infantry) and
                          story quest-state (BruzBrothers_* on NPC_611).
                          Fixed-length array (~31 slots); zero entries
                          are empty slots.
02 <u8> + 00 <hash>…      appearance meshes (orc_anim_base, Orc_Stk_*),
                          then AD_* attachment names
04 <count> + 04 <u32>…    body-parameter block — 0xff……-prefixed
                          values are ARGB colors
00 <Tag_* hash> +         appearance-tag counts
  08 <u32 count>
00 <TraitLevelOffset_*>   level-offset marker
ff ff ff …                tail (record ends at next `0f` token)
```

Head scalars (roster census): the `08`+`04` pair after the two
`0f` strings reads `power/level` — EQUAL on 160/165 members
(9/9 … 84/84), diverging only on story captains: the Bruz trio
79/89-90, a `TribeDef_Sauron` captain 84/90, and the follower-state
Bruz record NPC_32 (`TribeDef_Bruz`) at 80/18 (power-vs-level
direction INFERRED; divergence pattern verified). `f4b`/`f4c` are
near-always 0; `f4a` groups by army but not uniquely — possibly a
board/region slot index.

Bruz brothers: faction `fe44e382` is a three-member story army —
Baz (NPC_611, `BruzBrothers_Baz`+`BazDead`), Gaz (NPC_612,
`Gaz`+`GazDead`), Daz (NPC_613, `BruzBrothers_Daz`), all sharing
`09` type id `0x19f82a858e5` and the same Warmonger/Olog
BeastMaster build (`Marker_SpecialButterfly`).

Field-tag vocabulary: `00`=32-bit (hash or scalar), `02`=u8,
`03`=event-log entry (`u24 category` + `u32 name hash`), `04`=u32,
`05`=18-byte blob, `08`=u32, `09`=u64,
`0c`=bool u8, `0f`=len-string, `10`=len32-blob.

CLI: `save <f> whdc` (full resolved-hash dump) / `save <f> captains`
(clustered captain roster) / `save <f> npcs` (`_NPC_` token census) /
`save <f> links` (cross-record guid scan) / `save <f> relscan`
(numeric member-index scan) / `save <f> names <substr>` (resolved-name
hits with owning record) / `save <f> pairs` (shared-marker
grouping) / `save <f> rec|ids` (record walk /
head-id crossref) / `save <f> recdiff <fB> <NPC_n>` (cross-save
record diff) / `save <f> roster` (TSV census: token, typeid, head
scalars, tribe, marker/picker/event/trait counts) /
`save <f> hex|raw <off> <len>` (dumps).
Requires `ANDURIL_HASHDB` pointing at the harvested name dictionary.
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
