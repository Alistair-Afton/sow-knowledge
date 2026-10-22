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
  **Verified on retail**: 3 slot records per save — slot 1
  (~1.5 MB), slot 2 (~655 KB), slot 3 (~288 KB; identical byte
  length on all six saves). Slot records are NOT 4-aligned
  (slot 3 begins at an odd offset) — scanners must check every
  byte position.
- Chunk headers are `<4CC> <u32 ver> <u32 size> <u32 0> <u32 size-4>`,
  packed **with no alignment** — a chunk ends at `off + 16 + size`
  and the next tag follows immediately. Aligned scans silently miss
  most chunks.
- Every slot carries the **same ~26-chunk battery** — the slots are
  three parallel world states, not different data kinds. Per-slot
  order (verified `07010497_1`): `IVGD v3, IVSH v1, PPDC v3,
  PDIC v9, PAIC v1, IEUC v1, IVMG v43, IVPD v4, TTRL v10, LTMR v1,
  WHDC v1190, FAST v1, FOTP v1, ASMC v8, DIFF v9, CBTC v1,
  GCPT v5, INVA v9, QSTD v23, PDCC v5, LPDC v4, AUDI v1, IIUC v48,
  UITC v3, STOR v159`. Global-only chunks: `GOXC` v5, `PERF` v6,
  `PICB` v8, `GPDC` v5, `GMSC` v5. (`IUC0` seen on other saves;
  `SAVE` v31 is the slot inner header.)
- The three WHDC chunks are **the three campaigns** (verified by
  resolved names): **slot 1 (≈1.27 MB)** = base-game nemesis world
  (544 member tokens, 165 captain records — the only slot whose
  members use `<guid>_NPC_<n>` identity tokens); **slot 2
  (≈488 KB)** = *Blade of Galadriel* (DLC3) world — `DLC3_Architect`,
  `DLC3_ArchitectIsDead`, `Real_Faction_MtnPass_DLC3`,
  `DLC3_MtnPass_Row_Overlord`, `BecomeFollower`,
  `IsPlayerFollowerApply`, `Picker_Follower_GrabAndThrow`; **slot 3
  (≈122 KB)** = *Desolation of Mordor* (DLC4) world — `DLC4`,
  `DLC4_Desert_Overlord`, `Faction_desert_DLC4`,
  `DLC4_desert_Row_Overlord`, `DLC4_Role_Overlord`,
  `DLC4_OverlordArmSever_GrantClawArm`, `Role_Lithlad`,
  `Role_Desolation`. Slot 3 is byte-length-identical on all six
  saves — DoM was likely never played on this profile.
- All three WHDC chunks open with the **same prologue**: `Player`,
  `QuestReplay`, then a region/encounter name table (`BG_*`,
  `SM_*`, `D_*`, `IS_*`, `MI_*`, `VC_*`, `SON_*`, …) — a shared
  header, not per-campaign data.
- DLC nemesis states do NOT use `_NPC_` member tokens (verified:
  `_NPC_` occurs only in slot 1). Eltariel's allied captains and
  Baranor's nemeses are keyed differently — probably by
  `Role_`/`Real_Faction_`-keyed records (UNVERIFIED).
- Identity tokens generalize to `<32-hex-guid>_<KIND>_<n>`; besides
  `_NPC_` the saves carry `_FORT_` tokens. Slot 1 holds fortress
  records per region army (`<faction-guid>_FORT_0/1`, and the guid
  matches the captain-roster faction ids); slots 2 and 3 each carry a
  single bare `_FORT_0` record (no guid prefix) even on profiles that
  never started those campaigns. FORT records use one extra tag:
  `0a` = f32, paired with `00` name-hashes — the list resolves to
  the `Faction.DynamicWeight` vocabulary (`OriginDeathTaunt_*`,
  `Transformed_*` at 0 = locked out, `AdvancedClass_*`,
  `Weight_*_BlownOutInitially`), i.e. the save-persisted live
  weights. The tail serializes the region board
  (`Faction_<region>`, `FortAssault`, `<region>_Row_Captain_*` +
  u32 index list), then a `01`-tagged (second u32 class) section
  opens the **siege board**: `Warchief_*` slot assignments
  (`WarchiefCount`, `Warchief_1/2`, `WCI_Col*`/`WCO_Col*`),
  `Offense_*`/`Defense_*` upgrade families (Siege, Onslaught,
  Beasts, Battlefield, Army, Archers; PoisonMines, FireSpouts,
  PoisonSiegeBeasts…), `InnerGarrisons`, `Keep_Large`,
  `SauronOffense`/`PlayerOffense` and `*_Mon_*` + bool unlock
  flags. One `FORT_1` ran ~3900 fields to the WHDC chunk end —
  the largest record type in the world state.
- **Tokenless sections dominate WHDC**: only ~17% of slot-1's
  payload is member records' event sections — ~1.05MB sits between
  a record's event-log tail and the next token. The gaps are
  coherent tagged streams of member-shaped data (`08` stat
  arrays, `FactionMember_RoleSelected_*`, `03` event logs incl.
  a `cat=0x3f` channel, mesh names). UNRESOLVED: post-tail
  continuation sections of the same record vs. tokenless
  archived/pool member records — a mid-gap walk stays
  self-consistent and lands exactly on the next `0f` token.
- **Member-reference graph**: records embed `05`-blob refs to
  other members — `<u64 a> <u64 b> <u16>` where `b` = the target's
  GUID64B and `a` = `{u32 kind-flagword | u32 owner-state-
  flagword}` — BOTH halves compose flag bits (kind examples:
  `0x208 = 0x200|0x8`, `0x104ff = 0x10000|0x400|0xff`; owner:
  `0xC0000000` marker + ~14 member-state bits, shared across
  the record vocabulary). Usually the owner half equals the
  containing record's own flagword; foreign owners occur (a
  section inside one member's span carrying another member's
  flagword under a different kind — verified). Filtered
  census of plausible blobs: `0x0` ×512, `0x8` ×74, `0x200`
  ×44, `0x400` ×31, `0xc0` ×35, `0x800` ×15 (byte-scan kinds
  ≥ 0x10000 are mostly misparse artifacts — the field triple
  `02 u8 5`+`04 ffffffff`+`04 1` inside gear entries fakes
  a `05` tag reading kind `0x104ff`). ~157 candidate edges
  on a retail save; individual kind-bit meanings undecoded
  (UNVERIFIED).
- **Member context block** (in the post-record gap after each
  `<guid>_NPC_<n>` record tail): (a) board-context name hashes resolving to
  `InvasionFaction_<region>`, `Faction_<region>`, `AI_<role>`,
  `<region>_Row_Captain_<i>` — faction + board seat + AI role,
  and (b) ~0x45-stride gear entries `{name-hash,
  AD_<name>-hash, 05 blob, bools, counters}`
  (`Tracker_Shoulders`, `Fire_Helmet`, `Orc_Def_*`,
  `Orc_Shld_*`…). VERIFIED **post-tail** ownership: a
  `Faction_MinasIthil` ref sits in an a5ebbd08 (=MinasIthil)
  member's post-record gap while the following token is a
  different faction — the block continues the preceding
  member's record.
- **FORT tails serialize siege assault slots**: `Offense_nn` /
  `Defense_*` entries pair name hash + bool + `04 u32`
  **global member index** (the `n` in `_NPC_n`) — the assault
  team's slot assignments, `0xffffffff`-padded. Referenced
  indices exceed the live token set (dead/archived members) —
  the graveyard indices remain addressable.
- 4CC scans produce false positives inside payload data — `FYJB`,
  `TZEB`, `JNXO`, `LODL`, `LODS` appear inside slot-1 WHDC/global
  regions with garbage version/size fields; validate by
  end-chaining (`off + 16 + size` == next tag) before trusting.
- **Full chunk-tag registry** (exe, verified static): the binary
  carries a table of `mov eax,'TAG'; ret` thunks — 38 entries:
  `PERF PDIC PPDC IVGD DIFF ASMC FAST WHDC CBTC IVMG IVPD IVSH ENT2
  TTRL SAVE QSTD LPDC GPDC PICB GOXC` (main table) plus
  `ATI1 ATI2 AUDI FOTP GCPT GMSC IEUC IIUC INVA LTMR PAIC PDCC REEL
  STOR UITC` elsewhere. ATI*/DXT* are texture-codec tags; the rest
  are save/subsystem chunk ids. With an unaligned scan, nearly
  every registry tag is now observed on retail — the exceptions
  are `ENT2` and `REEL` (never emitted in the six saves).

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
`0f <len:u8> <32-hex>_<KIND>_<n>` (tag `0f`). `<KIND>` is `NPC` for
captains; fortress records use `<guid>_FORT_<n>` in slot 1 and bare
`_FORT_<n>` (no guid) in the DLC slots. The 32-hex prefix is a
**record-group GUID — NOT the member's faction** (corrected:
members of 15+ distinct prefixes all seat into MtnPass). On FORT
records it does match the faction GUID (`2ce24c19_FORT_1` ↔
`Faction_snowMtn`); on members its semantics are UNVERIFIED
(origin/spawn-group id is the working read). `NPC_<n>` is the
global member index (dead members keep their indices — FORT
assault slots reference untokened members).

Observed faction census on an endgame save (165 members, faction
= each member's post-tail `Faction_<region>` ref): MtnPass 51
(the active siege board), volcano 30, island 28, desert_Real 25,
MinasIthil 13, Demovolcano 12, snowMtn 6.

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
- `05` blobs decode as `<u64 a> <u64 b> <u16>` — `a` = `{u32
  section-kind | u32 owner-state-flagword}`; the low32 flagword
  is `0xC0000000` marker + ~14 member-state bits (shared
  vocabulary, not an id — bit census verified), `b` per-member
  unique (GUID64B).
  Other records carry the same pair with flag bits OR'd into `id_a`
  (e.g. `a|0x08`) — the on-disk member-reference form used by
  tokenless mission/instance records (INFERRED).
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
0f <len:u8> <32-hex>_<KIND>_<n>  identity token (NPC/FORT; DLC FORTs drop the guid)
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
                          `cat` is a **shared, sequenced event-instance
                          id**, not a category — one gameplay event
                          (e.g. cat=0x2d4: PromoteDead_BuilderTransform_
                          Simulate → Promote → Clear) logs under the
                          same id on every member it touched (verified
                          across NPC_1 and NPC_2). cat=0 carries
                          identity/template refs (Orc_Spearman,
                          OlogHai_Infantry) and story quest-state
                          (BruzBrothers_* on NPC_611). Observed event
                          vocabulary: Promote, PromoteDead_*, Raid_*,
                          HenchmenKilled,
                          FortDefenseMoveIntoOverlordWarchief,
                          RewSub_SubSucOth{Fled,Died}_Backup,
                          ActiveKillNamedAI, ActiveInitialPlayerThreat,
                          ActiveHateTriggeredDefeat, Chase_Clear,
                          GuardWarCouncil_*, ActiveKilledOnSimEvent,
                          interrogation/wraith-world ids.
                          Fixed-length array (~31 slots); zero entries
                          are empty slots.
02 <u8> + 00 <hash>…      appearance meshes (orc_anim_base, Orc_Stk_*),
                          then AD_* attachment names
04 <count> + 04 <u32>…    body-parameter block — 0xff……-prefixed
                          values are ARGB colors
00 <Tag_* hash> +         appearance-tag counts
  08 <u32 count>
00 <TraitLevelOffset_*>   level-offset marker
03 ff ff ff …             tail — a `03`-tagged event slot with
                          cat=0xffffff; every census record ends
                          here (record ends at next `0f` token)
```

Head scalars (roster census): the `08`+`04` pair after the two
`0f` strings reads `power/level` — EQUAL on 160/165 members
(9/9 … 84/84), diverging only on story captains: the Bruz trio
79/89-90, a `TribeDef_Sauron` captain 84/90, and the follower-state
Bruz record NPC_32 (`TribeDef_Bruz`) at 80/18 (power-vs-level
direction INFERRED; divergence pattern verified). `f4b`/`f4c` are
near-always 0; `f4a` tracks **creation time**, not seat or
level — the early-conquered snowMtn army reads ~5–7 across all
34 members, MinasIthil's late refills 117–128, story members
scatter 1–29 (INFERRED creation tick/batch index).

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
`save <f> hex|raw <off> <len>` (dumps). `roster` and `rec` accept
an optional trailing hex arg selecting a non-first WHDC chunk by
its tag offset; `names <substr>` accepts the same arg.
`save <f> find <hex>` searches the whole plaintext for a byte
pattern.
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
