# Script-side nemesis surface (string mining, patch09)

> **Status:** **verified** — string-mined identifier reference
> **Provenance:** 8,848 unique strings from 460 patch09 `.scripts` LuaQ constant pools (`anduril scri dump`).

Identifiers recovered from the constant pools of all patch09 `.scripts`
LuaQ chunks (`anduril scri <f> dump`, 8,848 unique strings, 460 files).
These are the globals/fields/events the game's own scripts touch — the
vocabulary a nemesis-sim Lua surface has to speak.

## Roles / spawners

`Captain`, `captain`, `aiCaptain`, `aiCaptain1`, `aiCaptain2`,
`aiBackupCaptain`, `aiAmbushCaptain`, `aiWarchief`, `aiOverlord`,
`cCharCaptain`, `enforcer`, `myGrunts`, `gGrunts`, `NumOrcs`,
`MaxOrcs`, `Orc_FactionA`, `GetOverlordTribe`, `OverlordsUnlocked`,
`WarchiefsUnlocked`, `WarchiefCount_01..06`

## Mission lifecycle

`MissionCheck`, `MissionEnd`, `MissionEndScreenFinished`,
`Mission_0`, `CompleteMission`, `HuntingMission`,
`MI_GondorianOriginMissions`, `ForceScenarioShutdown`,
`SetAllowWarchiefReplay`, `bCaptainSuccess`, `CaptainOnly`,
`CaptainOnlyOnSpawn`, `bDoubleCaptain`, `fCaptainChance`,
`bFirstInnerWarchiefSpawned`

## Captain / hierarchy events

`CaptainDeathRelay`, `CaptainDied`, `CaptainDispatch`,
`CaptainDispatched`, `CaptainDominationEndCheck`, `CaptainExists`,
`CaptainFlee`, `CaptainHit`, `CaptainLookedAt`, `CaptainTeleport*`,
`CaptainTooFar`, `EjectWarchief`, `WarchiefFailed`, `WarchiefHit`,
`WarchiefIntro_RL`, `WarchiefTarget`, `Warchief_Betrayed_IT`,
`Warchief_SpychiefBetrayer_IT`, `OverlordHitByAttack`,
`OverlordInterlude`, `OverlordLookedAt`, `OverlordPresentationStart`,
`OverlordBarrelStateChange`

## Nemesis stories

`Nemesis`, `NemesisStory_Ambush`, `NemesisStory_SpychiefFailure_IT`,
`NemesisStory_SpychiefFailure_AuxBanner_IT`

Full string list: `target/tmp/scri_strings.txt` (generated, not
committed). Method-level calls (`SELF`/`GETFIELD` targets) are visible
in `anduril scri <f> dump` output.
