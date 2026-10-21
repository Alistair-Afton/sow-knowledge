# Mode-exclusive content — base game vs Galadriel vs Baranor

> **Status:** **verified** — record names below are confirmed in
> `game.gamedb`; "absent in this mode" claims are named-evidence
> (INFERRED unless marked VERIFIED).
> **Provenance:** GameDB record census, VFS listing.

Three campaigns share one engine and one database. What makes each
mode distinct is which *systems* it enables and which named cast /
region content it owns — all gated by `DLC/Config` product keys
and `DLCn_Active*` GlobalVarPredicates.

## Base campaign — Talion & Celebrimbor

The systemic core everything else builds on:

- **Domination / branding** — the `Dominate` verb, `*BrandSupporters*`
  quest entries per region, `Domination_*` items, follower army
  recruitment. Exists in every region.
- **The freeform nemesis board** — seat grid, promotions, ambushes,
  blood-brothers, cheat-death, the full lifecycle
  (`faction-lifecycle.md`).
- **Fort conquest + Shadow Wars** — `Conquest_*` quests per region,
  `Invasion/Modes` (Community/Ranked/Friend), assault/defense
  events, `NeverEndingShadowWar` generation.
- **Raise-dead player skill** — `PS_Raise*`/`PS_RaiseRevenants*`
  skill nodes; `Resurrected_PlayerAbility_RaiseDead*` rewards.
- **Forthog Savior** — the Orc-Slayer rescue channel
  (`Savior_ForthogOrcSlayer`), formerly DLC, now free.

## Blade of Galadriel (DLC3, `Blade`) — Eltariel

Playable Eltariel (`AI/Templates BladeOfGaladriel`, own
perception/threat/combat sets, `GraftEltariel` model grafts).
Exclusive pockets:

- **The Gauntlet** — a private follower-trial nemesis channel:
  per-region builder sets `Gauntlet_*_Lvl{1,2,3}`, own SimEvents /
  promote verbs / screens (`story-locked.md`).
- **Named orc companions** — recruitable story followers with
  dedicated state vars: `DLC3_{Architect,Digger,ElfLover,Flint,
  Fixer}_{IsDead,IsFollower}` (+`FixerIsLoyal`,
  `FlintTinderIsLoyal`). Companions persist via vars, not board
  seats.
- **Poisoned-fort siege variant** — `DLC3_FortIsPoisoned`,
  `DLC3_IslandFort_{GateSealed,GrogPoisoned,FireTrapDisabled}`,
  `DLC3_Island_FirstTime` — island-fort siege flags the base game
  never sets.
- **Island region** content and `Eltariel` quest screens; savior
  channel `EnchantedDLC3` + `dx_Bl02_*_EltarielSavior`.
- **Domination exists but is dialed back (INFERRED):**
  `DLC3_Ignore_MightyShot_Brand` and `Revive_DLC3` menu entries
  show brand/revive mechanics present with DLC3-specific
  exceptions — Eltariel dominates selectively, Talion appears with
  `DLC3_AI_Talion_RaiseDeadAOE` in the crossover finale.

## Desolation of Mordor (DLC4, `Lithlad`) — Baranor

Playable Baranor — **mortal, no wraith powers**. The nemesis stack
is replaced by a human economy:

- **Permadeath** — `DLC4_Tip_Permadeath` is a shipped tip entry:
  Baranor's death is final (VERIFIED as a documented mechanic
  flag).
- **Mercenary hiring instead of domination** — `MercDLC4_*`
  mercenaries, `DLC4_PLBar_Coin_*` currency items, `HireMerc`
  tutorial item, `DLC4_PS_MercDiscount_*` skill tree,
  `DLC4_SkillGrant_Serka_MercDiscount`; `dx_seq_MercOasis*`
  outpost dialog (18+ merc banter pairs). No `Dominate` channel
  carries a `DLC4`/`D4` name — the verb is simply absent
  (INFERRED: replaced, not suppressed).
- **Outpost warfare** — `DLC4_OutpostCommander_Understudy_{C,E,N,
  NW,W}` builders, `DLC4_OutpostTakers`,
  `DLC4_OutpostAssault_Leader_IT`, `D4OAS_SiegeWarden_IT`,
  `Easterling_*` merc traits (`Easterling_Fear_AlliedMercFear`,
  `Easterling_Terror_AlliedMercDeath` — mercs fear *their own*
  allies dying).
- **Merc medic revive** — `DLC4_Easterling_Medic_Revive` attacks +
  `DLC4_ReviveAlly` hint + `DLC4_Sword_ButtonB_ReviveAlly` prompt —
  human medics fill the savior/self-revive niche Talion's wraith
  powers cover.
- **Serka** — companion (`DLC4_Serka` predicate), savior channel
  `SerkaDLC4` with its own cooldown, `PreDeath_Savior_SerkaCooldown`
  timer items.
- **Lithlad region** (`desert` board) and the `D4{RES,REC,INT,FIN,
  OAS,CAN}` chapter stage codes; DLC4 gear is a separate
  `Loot/LootItemLists`+`UIItemData` set (human equipment, not orc
  drops).

## Tribe & update add-ons (same modes, more members)

- **DLC1 = Slaughter tribe, DLC2 = Outlaw tribe** — tribe content
  inside the base campaign: intro ambush chains, members, roles
  (`Role_Massive_DLC2`/`Role_Vile_DLC2` ride
  `NemesisForge/FirebirdFactionRole`), `DLC1_StealthWarg` smart
  objects.
- **DLC5 = Champions-of-War-era update** — `SW_DLC5_Y{1,2,3}_*`
  Shadow Wars stage sequences, `DLC5_Necromancer_SelfRes*`
  (necromancer self-resurrection rewards), `DLC5_*_ChanceToGainTraits`
  reward lists.
- **Small keys:** `SwordOfDominion` (pre-order sword),
  `TrialMode` (Trials of War), `Edition*` SKU bundles,
  `StarterBundle`, `HD*` packs.

## How attribution works in data

- `DLCn_*` = hard DLC tag on records; `D3*`/`D4*`/`Blade2*`/`GP_Blade2*`
  /`MercDLC4`/`Serka`/`Eltariel`/`Baranor` = per-campaign naming;
  `DLC/Config` + `DLC_<name>` vars + `DLCn_Active` predicates =
  the runtime gate.
- Untagged names are base game or shared — never assume a plain
  `Savior`/`Ambush` record is DLC.
- INFERRED until traced: `DLC5`↔`ChampionsOfWar` product tie, and
  the exact `Blade`/`Lithlad`↔DLC3/DLC4 numbering (name evidence
  only — `D3` dialog is Galadriel, `D4` is Baranor).
