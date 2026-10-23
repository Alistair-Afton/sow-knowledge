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
- **Archive list of dead/pool members**: after the live
  member's own context block, the post-tail gap runs tokenless
  archive records — spaced ~2 KB, each ending in a
  `{Faction_<r>, Faction_<r>, AI_<role>?, <r>_Row_<seat>?}`
  tuple (the dead member's last faction/role/seat). They
  decode with the full member field grammar (`05` blobs, name
  hashes, bools, gear-entry signatures) — complete member
  records minus the `0f` ASCII token, **retaining identity**:
  each carries one `{kind=0 | owner flagword | GUID64B
  ending 00 | 0}` blob (~2 KB stride), so dead members keep
  GUID64B + last state flagword and stay joinable against
  `05` refs and event-instance ids. The `{Faction_X,
  Faction_X}` context-pair census enumerates them: ~523 pairs
  on one endgame save ≈ 165 live + ~360 dead — dead per
  region: desert 120, MtnPass 50, volcano 53, island 39,
  MinasIthil 42, snowMtn 45, Demovolcano 15; total serialized
  member space ~525 vs max index 613 (the rest were purged
  transients). A live member's own faction/row/AI is the
  **first** tuple; later tuples are archive records.
- **Dead members are still referenced**: 322/328
  dead-member GUID64Bs appear at other WHDC sites (top
  example cited from 9 offsets across the chunk) — refs to
  dead members persist after death (VERIFIED; the
  `..040000`-pattern `b` family in raw scans is a different
  small-index layout, not a GUID). Splitting on the owner
  flagword: 339 citations are sections inside the dead
  member's own archive record; **182 external refs**
  (flagword values 0/8/0x200/0xc0/0x800 — the edge *kind*
  is the low 24 bits of `a`, see the census below) point
  at **119 dead members** from other entities'
  records — relationship/mission history survives death.
  **But the archive is a rolling graveyard, not cumulative**
  (VERIFIED — GUID join of two saves ~10 weeks apart):
  ~118 GUIDs left the live roster between the saves and
  **zero** of them appear in the later save's archive.
  Dead records purge out of WHDC entirely (mechanism
  undecoded — plausibly the same removal path as the
  `ET_CanRemove_*` transient deletes), so
  the ~360 archive holds only *recently* dead members —
  live-roster diffing undercounts total deaths, and
  killer attribution across long gaps is impossible
  because both the dead records and the edges into them
  are garbage-collected.
- **Edge kind census** (165-member endgame save, `save refs`):
  the `a` word is `{instance-id:u32 | kind:u32}` — the
  high word is 0 for plain refs or a shared event-instance
  id (bits 0/8/0xc0/0x200/0x800 seen as residual values on
  some), kind values on the low 24 bits. 156
  live→live + 306 live→dead edges; the dominant kinds are the
  `0x?022a4`/`0x400080` family (generic member refs, both
  directions). Two discriminators stand out:
  - **`0xbb02af` edges are dead-only** (n=14) and their
    flagword is a **shared event-instance id** (VERIFIED by
    collision): `0x581030eb` appears on NPC_12, NPC_13 *and*
    NPC_14 all citing the same dead member — three live
    members + the dead one = one 4-participant event;
    `0x123747b9` links NPC_13+NPC_3+the same dead member;
    two further instance ids are singletons. So
    `0xbb02af` = *dead-participant event edge* — a member's
    record of an event instance whose roster included a
    now-dead member (INFERRED; consistent with mission
    participant memory, incl. the fatal mission itself —
    the shared instance means the killer would be *among*
    the participants, not identified by the edge alone).
    All 14 edges target just two dead members. Co-
    participants of one instance do **not** hold pairwise
    live→live edges to each other — the instance id rides
    on the dead-participant edge only. Small nonzero high
    words (0x8/0x200/0xc0/0x400/0x800) on the ordinary
    `0x?022a4` family are likely instance *counters* on
    the same field — common events reused, vs the large
    hash-like ids of unique instances.
  - **Personal history is ~95% cross-faction** (406 cross
    vs 21 same-faction after excluding the `0x400000`
    registry and `-gap-` sources) — nemesis edges are
    almost entirely *cross-board* encounters (invasions,
    roaming, missions), not intra-board structure.
    Consistent with henchmen/bonds being marker records
    rather than `b`-edges — board-mates don't cite each
    other.
- **DLC WHDCs carry no live member records** (VERIFIED,
  same save's slot 1/2 chunks): each holds exactly one
  member token (the `FORT_0` region record) — no
  `NPC_n` identities at all. BoG/DoM have no orc nemesis
  board; their WHDCs keep only fort/region state plus a
  **dead archive** — slot 2's 38 edges are all `-gap-`
  registry refs into `dead:Faction_desert_DLC4` records
  (the `0x?02?a4` kind family replicates there, so the
  edge model is format truth, not a base-game quirk).
  `save refs slot=N` selects the Nth WHDC chunk
  (0 = base campaign) and prints a token census.
  - **`0xc0400000` edges are the death registry**
    (VERIFIED shape, Aug-7 save): emitted from `-gap-`
    regions (non-member archive bookkeeping) and from two
    anchor member records (46 and 54 refs each), the
    targets enumerate the whole dead roster
    (`dead:Faction_*` members + a few live). This is the
    archive's index layer, not personal nemesis history —
    it dominates an early-campaign save (115 of 168
    edges) and shrinks to a minority later as real
    member-to-member history (`0x?022a4` family) accrues.
  - **Kind=0 group edges**: 5 members cite one shared
    target (`0x5254997c00`, a member of a *different*
    faction than most citers) all carrying the same
    flagword `0xc2000000` — one shared instance/binding
    across a 6-member group (INFERRED; the flagword-as-
    instance-id interpretation from `0xbb02af` fits).
    **Caveat: `NPC_N` names collide across factions** —
    `save refs` now emits `name@faction` on both ends
    because per-faction numbering otherwise merges
    distinct members (two different `NPC_17`s etc.).
  - **Kind `0x?02284` is a mutable role-binding edge, not
    history** (VERIFIED by a 45-minute in-session save
    series): the edge `{instance 0x200c862 | kind 0x2284}
    -> NPC_4` moved from member NPC_3 to a freshly spawned
    member when it appeared, and back to NPC_3 when that
    member left ~13 minutes later — identical `a`, `b`, and
    flags throughout. The edge sits on *whichever member
    currently holds the role* toward the target in that
    event instance; `b` equals the target member's own
    head GUID here. Edges are only rewritten on membership
    events — two further diffs in the same series (8 min,
    and 5 days with no intervening play) show **zero** edge
    churn, so the `0x?022a4` history family accrues on
    event scale, not per-tick.
- **Bonds are not `b`-edges**: the Bruz brothers' GUID64Bs are
  cited only by their own records + one kind-0 ref — the
  blood-brother link is marker-driven, not an `05` edge.
  Caveat: `b` is **not token-unique** — `0xe65e14fd00` heads
  both an Undead Olog (`Marker_Resurrected_PlayerAbility_
  RaiseDead`) and Bruz (`TribeDef_Bruz`) — two unrelated
  entities sharing one head `b`. Duplicates cluster:
  `NPC_0` sentinel records each share `b` with one captain,
  and one 5-member group shares a single `b` — the working
  read is head `b` = **bound-entity reference** (self for
  ordinary members, another entity for proxies, shared for
  multi-member bindings), not a unique id (INFERRED). Bound
  members are not seat-specific (warchief + roamers alike) —
  what picks the sentinel's member is UNRESOLVED.
- **Transient members are deleted, not archived** (cross-save
  diff): a save holds one extra member — an Epic Marauder
  Champion-Assassin carrying `ET_CanRemove_ChampionAssassin` —
  fully absent from a later save (token nowhere in the
  plaintext). `ET_CanRemove_*` members are removed wholesale
  and their singleton group prefix disappears with them —
  consistent with the prefix being a spawn-batch id
  (INFERRED). A 45-minute in-session save series sharpens
  this: a `Faction_desert_Real` Marauder captain (level 75,
  a singleton record group) appears in one save and is gone
  ~13 minutes later — no corpse, no archive entry — having
  held a `0x2284` role edge during its stay. Mission-spawned
  members are written into WHDC for the duration of the
  event, then deleted (VERIFIED). Member bodies also serialize marker/picker
  vocabularies directly: `Marker_Potential_*`,
  `ClassMarker_*`, `PersonalityMarker_*`, `Marker_OrcRarity_*`,
  `Marker_StatusEffect_*`, `RoleMarker_SpecializationProb_*`,
  then `Picker_Resist_<school>_<level>` pairs.
- **FORT tails serialize siege assault slots**: `Offense_nn` /
  `Defense_*` entries pair name hash + bool + `04 u32`
  **global member index** (the `n` in `_NPC_n`) — the assault
  team's slot assignments, `0xffffffff`-padded. Referenced
  indices exceed the live token set (dead/archived members) —
  the graveyard indices remain addressable.
- **One FORT field increments across saves** (5-slot
  cross-save diff): a u32 in the MinasIthil/island `FORT_0`
  steps 24950 → 25050 in timestamp order with non-uniform
  increments while all other FORT/NPC fields are static —
  a game-time tick or accumulator (INFERRED).
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
  unique (GUID64B) — **verified stable across saves**: joining a
  save pair 10 weeks apart on each record's first-`b` value links
  32 of ~150 live members across the two files, making `b` the
  practical member-identity key for cross-save diffs (the
  `_NPC_<n>` slot index is NOT stable — it is reused by new
  spawns). Caveat: `b` is a bound-entity reference, not strictly
  unique — board-sentinel records share their bound member's `b`,
  and resurrection/group bindings can share one `b` across
  several records (the duplicate-`b` census shows every
  `NPC_0` sentinel pairs with a specific captain).
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
