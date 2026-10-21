# Hydra online services — transport, wire format, Garrison

> **Status:** **verified** — in-game verified by the Universal Modder
> project against build 3095001; addresses are exe-relative to image
> base `0x140000000`.
> **Provenance:** Universal Modder knowledge base field note "Offline
> mode and local Garrison" (MIT) plus its published `winhttp.dll`
> stand-in source — the protocol was recovered by replaying the game's
> own requests locally. Structural facts below are quoted as evidence,
> in our own words.

The game's online features (Garrison, Market remnants, Online
Conquest/Vendettas, telemetry) are not in-engine systems — they ride
on WB/Agora's **Hydra SDK** (`Hydra-Cpp/1.50.1`, `hydra::` /
`apiframework::` namespaces) over WinHTTP in async mode.

## Transport

- Hosts: `kraken-api.wbagora.com` (API) and `prod-event.wbplay.com`
  (telemetry).
- Requests are ordinary WinHTTP calls; a DLL-search-order proxy
  `winhttp.dll` can answer them locally without patching game code —
  every other host passes through untouched.
- `POST auth` → `{token, type}`; `POST access` → `{token, account{…},
  profile{…}}`. The game counts itself **signed in only when both
  `account` and `profile` are maps** (`0x141B5229C`).
- `ssc/invoke/get_server_time` must answer
  `{server_time (ms), query_start_MS (echoed)}`; without it the game
  polls ~1,600 times.

## Wire format — `application/x-ag-binary`

Type byte + big-endian payload (decoder `0x1409F6940`):

| tag | type |
|---|---|
| `02` / `03` | bool true / false |
| `10`–`17` | integers i8..u64 |
| `20` / `21` | f32 / f64 |
| `30`–`32` | strings, u8/u16/u32 length |
| `40` | date (u32 epoch seconds) |
| `50`–`52` | arrays, u8/u16/u32 count |
| `60`–`62` | maps, u8/u16/u32 count |

- **Hydra's integer reader (`0x140A01460`) accepts only int64 (`16`)**
  — ints of any other width silently read as 0.
- Value classes check a type id (String 3, Map 4, List 5); a `{}`
  where a list is expected is silently ignored — replies must be the
  right *kind*, not just the right shape.

## Remote kill switch and "store accessible"

- The `game-settings` broadcast channel is a cursor container
  `{cursor, start, count, total, results[]}` of messages
  `{slug, message_id, message, start_at, end_at, data{}}`. The game
  reads **`data.store_enabled`** (`0x1418368B0`), not the human
  `content` text — when false the Garrison shows "We have temporarily
  disabled the Garrison".
- Store accessible (`0x141893780`) additionally requires: signed in; a
  `store-front` channel layout
  (`data{version:2, title, act_i, act_ii, racks[]}`, builder
  `0x1418633BC`); and **non-empty lists** from `GET inventory/items`
  and `GET commerce/catalog/me/products` (else "Currently updating
  Garrison data" forever). Menu gating also needs `StoreAvailable`,
  Act 2+, no active quest; garrison assign needs
  `Player:HasDominationAvailable`.

## Garrison objects

- Object shape:
  `{id, type, owner_id, created_at, updated_at, data{FactionMember,
  Searchable, sys_creator, sys_name}}` — `data` carries the full
  ~5 KB member record (gear, appearance, traits).
- **List:** `GET objects/types/user-garrison/search_queries/
  garrison-get-collection/run` → cursor container of
  `results[{score, result: object}]`.
- **Store:** `POST objects/user-garrison` with the whole `data`;
  echo the full record back or the stored orc shows an icon but no
  gear/model/commands.
- **Read:** `GET objects/user-garrison/<id>` — the game reads objects
  through the `objects/<type>/<id>` path. The reader iterates `data`
  with no null check (`0x14097D7F0`) so every object reply needs a
  `data` map.
- **Delete:** `DELETE objects/user-garrison/<id>`.

## Deploy / Destroy-for-gear state machine

State names are exe strings; Deploy and Destroy share it:

`WaitForValidConnection → WaitForSave → CreateGarrisonMemberRemoval
(the DELETE) → WaitForGarrisonMemberRemoval →
WaitForCurrentNotifications → WaitForNewNotifications → AwardLoot →
RemoveNotification → Done | Failed`

- After the DELETE the game waits for a **server-pushed
  `garrison_removal` notification** whose `data.obj` is the removed
  object's id (checks `0x141C508A0` deploy / `0x141C509EC` destroy).
  Without it both paths reach `Failed` — but the orc is already gone.
- `GET accounts/me/notifications/unread` must be a **list** of
  `hydra::Notification` `{id, data, server_data, cmd, header, payload,
  state, consumed}` (`fromValue 0x14529A910`); entries without `cmd`
  are dropped. The notification manager (`0x141868A74`) keeps only
  `{id, header, data, consumed==false}` — so the object id must sit in
  **`data`**, not `payload`.
- Dismissal: `DELETE accounts/me/notifications/<id>` (bulk:
  `.../bulk/`; `0x14098F5F0`).
- Deploy rebuilds the captain from the already-fetched record and
  places it in the chosen slot (`0x141C5132C`); Destroy rolls the gear
  reward **locally from the orc's own equipment** (`0x141C51884`).

## Mirian wallet

Since the July 2018 Market shutdown the game migrates the save's
`PC_Mirian` into an online item `vc-earned`: it `PUT
profiles/<id>/inventory {modifications:[{command:"set_count", slug,
count}]}`, zeroes `PC_Mirian`, then reads the list back. Afterwards
the save is authoritative — the game pushes `set_count vc-earned =
PC_Mirian` and ignores a higher wallet value. A stand-in that answers
`{}` permanently loses the wallet.

## Nemesis member ids (`member+0x3D0`)

- Every faction member gets a string id at `member+0x3D0`, written
  only while empty, as `"<prefix>_NPC_<n>"`.
- `prefix` = a 32-hex session GUID at `analytics-manager+0xA81`
  (manager = `*(*(exe+0x26FFAA8)+0x6D18)`), set only while online;
  `n` = a global counter restarting at 0 each launch. Assigned at
  member creation / first telemetry report (`NPCProfile` events).
- **Offline consequence:** with no prefix every session re-issues
  `_NPC_0`, `_NPC_1`, … — ids collide across sessions. The
  `ReassignmentOrders_GUID` contract (`HasNPCGUID == False`) refuses
  to garrison a captain whose id is shared, which looks random.
- In saves, the ids are 1-byte-length-prefixed strings, one per
  member; rewriting duplicates to unique same-length strings restores
  garrisonability (length must not change).

## Scope notes

- `game-settings` replies can declare `vendettas_enabled: false`;
  Online Conquest/Vendettas and Market purchases are server-side
  features not modeled by any known local stand-in.
- Everything above describes the *client side* of the protocol as the
  game consumes it; server behavior is only observable through what
  the client requires.
