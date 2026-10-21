# DLC content — what belongs to which add-on

> **Status:** **verified** — `DLC/Config` keys, `DLCn_*` record
> prefixes, and `GameData/Variables` gates are all confirmed in
> `game.gamedb`; product↔DLC-number mapping is INFERRED from name
> evidence (marked where direct).
> **Provenance:** GameDB record census (record-prefix → list table).

All DLC data ships inside the same `game.gamedb` and archives —
DLC ownership is expressed by **record naming** (`DLCn_*` prefixes
and per-campaign code prefixes) and runtime gating, not by
separate files.

## Product keys (`DLC/Config`, VERIFIED)

| key | content |
|---|---|
| `Blade` | Blade of Galadriel story expansion (= **DLC3**) |
| `Lithlad` | Desolation of Mordor story expansion (= **DLC4**) |
| `Forthog` | Forthog Orc-Slayer savior (now free — see `cut-content.md`) |
| `TribeSlaughter` | Slaughter tribe (= **DLC1**) |
| `TribeBlackguard` | Blackguard tribe (= **DLC2**) |
| `SwordOfDominion` | pre-order sword |
| `ChampionsOfWar` | Champions of War (Shadow Wars update, ≈**DLC5**) |
| `TrialMode` | Trials of War mode |
| `StoryPass` / `ExpansionPass` | pass entitlements |
| `Edition{Standard,Silver,Gold,Mithril,GOTY}` | SKU bundles |
| `HDContent` / `HDCinematics` / `HDLithlad` | high-res packs |
| `StarterBundle` | starter bundle |

## Per-DLC evidence (record census)

- **DLC1 = Slaughter tribe.** `DLC1_TribeBat_{Slaughter,Outlaw}`
  quests, `DLC1_Slaughter_Intro_*` ambushes/builders/role lists,
  `DLC1_StealthWarg_*` smart objects, `DLC1_TrlPsn_Ranged`,
  `DLC1_Vomit_Graug_Shake_Loop`. Gated by `DLC1_Active{,_PostRatbag1}`
  GlobalVarPredicates.
- **DLC2 = Outlaw tribe.** `DLC2_Outlaw_Intro_*` ambush chain,
  `DLC2_Interrogation_NoThrow`, `dlc2_fire_*` ambience. **Note:**
  `NemesisForge/FirebirdFactionRole` `Role_Massive_DLC2` /
  `Role_Vile_DLC2` — tribe-flavored nemesis roles ride the
  Nemesis Forge table.
- **DLC3 = Blade of Galadriel (Eltariel campaign).** `DLC3_*`
  records across ~2.5k lists — `DLC3_{Architect,Digger,ElfLover,
  Flint,Fixer}{IsDead,IsFollower}` vars (the recruitable named
  orcs), `DLC3_FortIsPoisoned`, `DLC3_IslandFort_*` siege vars.
  Content prefix `D3*` / `Blade2*` / `GP_Blade2_*` /
  `BladeOfGaladriel` (Eltariel's AI templates, perception sets,
  `GraftEltariel` model grafts) — dialog `dx_D3***_*_Galadriel`.
  The Gauntlet follower-trial channel is DLC3 (see
  `story-locked.md`).
- **DLC4 = Desolation of Mordor (Baranor campaign, Lithlad).**
  Largest prefix footprint (~3.4k records) — `D4*` stage codes
  (`D4RES`/`D4REC`/`D4INT`/`D4FIN`/`D4OAS`/`D4CAN` = region
  chapters), `Serka`/`MercDLC4_*` mercenaries,
  `DLC4_Savior`/`DLC4_PickerList_Savior`, DLC4 loot tables and
  `UIItemData` (the human-mercenary gear set),
  `BaranorGauntlet`/`BaranorNoGauntlet` appearance tags.
- **DLC5 = the Shadow-Wars-era update (Champions of War).**
  `SW_DLC5_Y{1,2,3}_{IS,MP,SM,VO}` SimEvent sequence items — the
  three Shadow Wars stages per region — plus `DLC5_Necromancer_
  SelfRes*` reward/predicates (necromancer self-resurrection),
  `DLC5_Y*_*_ChanceToGainTraits` reward lists,
  `DLC5_BetrayOverlord_WhenBodyguard`, `ShadowWars_DLC5_AnyCompleted`,
  and `DLC5_SW_Y0*_End` world-map camera sequences.

## Cross-cutting content

- **Savior channels** (`story-locked.md`): base GP/Bruz channels;
  `SerkaDLC4` (DLC4), `EnchantedDLC3`/`MercDLC4` variants — each
  has its own spawn event + cooldown items.
- **DLC gating pattern:** `Quest/GlobalVarPredicates`
  `DLCn_Active*` + `GameData/Variables` `DLCn_*`/`DLC_<name>` —
  the same gate mechanism story quests use; nothing is "patched
  in", it's all conditional records in the one database.
- **Naming conventions:** `DLCn_*` = hard DLC tag; `D3*`/`D4*` =
  per-campaign stage codes; `Blade2`/`GP_Blade2` = Galadriel
  campaign internals; `MercDLC4`/`SerkaDLC4` = Desolation.
- **Editions** (`Edition*` keys) bundle the same keys — e.g. a
  Silver/Mithril/Gold/GOTY flag just enables the constituent DLC.

## Open ends

- `DLC5` ↔ ChampionsOfWar is INFERRED from `SW_`/`ShadowWars`
  naming — no record literally ties the prefix to the product key.
- Whether `DLC1and2SoundBankList` implies shared DLC1/DLC2 audio
  packaging vs an audio-side convenience list is unverified.
- Some `PreDeath_Savior_*`/`AmbSavior` records have no DLC tag —
  base-game savior channels; don't assume every nameless `Savior`
  record is DLC.
