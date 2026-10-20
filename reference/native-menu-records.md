# Menu record layouts — TreeBase / TreeNode / Predicate / Action
> **Status:** **partial** — Project Nemesis source doc; per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE / VERIFIED-INGAME / MODEL / UNVERIFIED) preserved inline. Not independently verified by Andúril.

> **Provenance:** Project Nemesis runtime-model documentation (native `nemesis.dll` live-game forensics), mirrored and maintained in the corpus. Their `VERIFIED-STATIC`/`MODEL` tags carry over; unverified entries are marked inside each doc.


**Tag:** VERIFIED-LIVE (title + pause Options, build `0x5B7F5782`,
2026-09-25..28)
**Evidence:** live menu-tree probes, `fe_tree*.lua`, `fe_ga.lua`;
`runtime/sdk/menu.lua` builders.

Consolidated layout of the menu-records we mutate. All are GameDB
record entries (`entry+0x10` = data block, `entry+0x20` = name cstr);
"cell N" = `data + N*8`.

## Record entry (all types)

| Off | Type | Name | Meaning |
|-----|------|------|---------|
| 0x00 | u64 | key | hi32 = namehash, lo32 = record index |
| 0x08 | ptr | slot_table | sorted `{hash, meta}` pairs; meta bits 48–63 = live cell slot, bits 32–47 = element count |
| 0x10 | ptr | data | cell array (qwords) |
| 0x18 | ptr | owner | owning list descriptor |
| 0x20 | ptr | name | name cstr |

Declared schema index ≠ live cell index — resolve via the slot table.

## TreeBase (e.g. `FrontEndRoot`, `Options_DetailTree_*`)

- `BaseNodes` = **inline** array in the data block (elements are node
  entry pointers). Grows via the Nemesis plugin API (relocates data +
  slot table into the leak pool; journaled).
- `FrontEndRoot.BaseNodes` stock children (7): `FrontEnd_Start`,
  `FrontEnd_Social_RestoreSave`, `FrontEnd_StartDemo`,
  `FrontEnd_WBPlay`, `FrontEnd_Options`, `Video_LaunchBenchmark`,
  `FrontEnd_Quit`.

## MenuTreeNode.GenericAction (title-menu rows)

| Cell | Content |
|------|---------|
| 0 | row label — strdb-name cstr (e.g. `UI_Menu_Options`) |
| 1 | icon texture path cstr |
| 4 | `TreeNode_State` record ref (state → pred → action chain) |

Cells 5+ partially decoded — `fe_ga.lua` dumps the full chain.
Press dispatch follows the record chain (node → state → pred →
action); cloning the node + repointing pred.cell3 retargets it.

## MenuTreeNode.ComponentButton (e.g. `Video_LaunchBenchmark`)

- cell3 → `TreeNode_State` (e.g. `Component_LaunchBenchmark`)
- cell11 → component record ref (e.g. `Component_AutoConfig`)
- cells 4–7, 12–15: embedded arrays with zeroed payloads
- **Frontend-variant caveat:** in the *title* tree this node's press
  dispatches through embedded/native component bindings — repointing
  its state's pred.cell3 does NOT retarget the press (VERIFIED-LIVE
  2026-09-28, still launched the benchmark). Works normally in
  Options detail trees (add_button uses exactly this chain).

## TreeNode_State (e.g. `Component_LaunchBenchmark`)

| Cell | Content |
|------|---------|
| 5 | `Predicate/Node` entry (the activate pred) |
| 6 | `MenuSelection/Prompt` entry (footer prompt record) |

## Predicate/Node (e.g. `LaunchBenchmark`, `PushLayerFromData`)

| Cell | Content |
|------|---------|
| 0..n | event-name cstrs: `MenuEnter`, `Mouse1`, `MenuAux*`,
  `MenuEscape`, `Map`, … |
| 3 | **ResultAction** record entry — "what fires on press" |
| 6 | prompt record (in ComponentButton-style preds) |

`PushLayerFromData` + sibling `Action.Menu::PushLayerFromData` form the
generic data-driven push pair used by GenericAction items.

## MenuSelection.Action.Menu (push actions, e.g. `PushQuestLog`)

| Cell | Content |
|------|---------|
| 5 | `MenuLayerList` entry — the pushed screen chain |

Pushed screen = `Action.Menu → MenuLayerList → {count, layer…} →
MenuLayer → tree attr → TreeBase`. the Nemesis plugin API assembles
this whole chain around a private cloned tree.

## MenuSelection.Prompt

| Cell | Content |
|------|---------|
| 0 | flags qword |
| 1 | `Enabled` strdb-name cstr (footer text) |

## ComponentCheckbox — UNSAFE to clone (VERIFIED-LIVE 2026-09-28)

- Binding is per-entry (a datasource map keyed on the entry address),
  not record-carried and not name-keyed. Any clone — even same-name —
  renders label `"Option"` and crashes teardown at
  `ShadowOfWar.exe+0x698e8c` on category switch. Gated behind
  `opts.experimental` in the Nemesis plugin API. Repair needs
  the binding-map insert path decoded (D006 seam).

## TreeNode.GenericAction (frontend title items)

Verified live 2026-09-28 by bisect (`FrontEnd_Options` → OPTIONS press):

| Cell | Content |
|------|---------|
| 0 | label strdb-name cstr (`UI_Menu_Options`) |
| 1,8 | icon cstr |
| 3 | **`MenuLayer` entry pushed directly on press** — no pred chain, no name resolution (`FrontEnd_Options.cell3 → Frontend_OptionsLayer`) |
| 4,11 | `TreeNode_State` (visual state only — `FrontEnd_PauseRoot`) |
| 5,6 | embedded arrays (visibility preds incl. `IsReplayMode`) |
| 12,13 | embedded arrays (region/mode gate preds) |
| 7,15 | cstr |
| 10 | record ref (e.g. `WBPlay`) |

Adding a title row: clone this record, write cell0 (label cstr) +
cell3 (layer clone entry), `arrinsert` into
`TreeBase::FrontEndRoot.BaseNodes`. The press pushes whatever layer
cell3 names — a cloned layer works (VERIFIED-LIVE:
`Frontend_OptionsLayer` clone → OPTIONS screen).

## MenuLayer subtype records

- `Frontend_OptionsLayer`: `OverviewTree` attr at **live slot 18**
  (`MenuLayer.Options` schema; hash lo32 `0x063ad377`). **Not**
  `Components` — that attr resolves to live slot 48 and is unrelated
  to the content tree (declared-name ≠ live-name, again).
- `MenuLayer.QuestLog::QuestLog`: `OverviewTree` at live slot 11.
- Detail trees (e.g. `Options_DetailTree_Display`) hold flat row
  nodes (`Display_*` component records) — the shape `build_node`
  rows match. Overview/section trees hold sub-tree records
  (`OverviewTree_BladeSide` carrying `Quest_*` refs; the quest screen
  builds rows live from quest state — wrong template for row screens).

## tree_clear + tree_insert — slot-table semantics (VERIFIED-LIVE)

- `clone_with_array` captures `(arr_slot, count)` from the array
  attr's hash→slot node (meta bits 48..63 = slot, 32..47 = count).
- `tree_insert` splices at `insert_slot = arr_slot + count`, bumps
  slot of every table node `>= insert_slot` **except the array attr's
  own node** (`lo32(hash) == attr_lo32`), and bumps that node's count.
- **Bug fixed 2026-09-28**: without the exclusion, inserting into a
  cleared tree (`count=0` → `insert_slot == arr_slot`) bumped the
  array node's slot → `BaseNodes` resolved one qword right of the
  element → engine walked tail data as a record → `exe+0x274af9` AV.
  Populated-tree inserts never hit it (`insert_slot > arr_slot`).
- `tree_clear` removes the element run, shrinks slots `>= tail_slot`
  by `count`, count meta → 0. Cleared trees render blank safely.

## ComponentButton press semantics (VERIFIED-LIVE 2026-09-28)

Press chain on a pushed `Frontend_OptionsLayer` screen:

```
row.cell3 (PrimaryState, live slot 3, TreeNode_State)
  → state.cell5 (Predicate.Node)
  → pred.cell3 (Action record)      — WRITE TARGET
  → executor resolves action BY NAME via intern map (exe+0x1898xx)
  → registered handler runs
```

- `pred.cell1`/`cell2` = activation event namehashes (stock
  `0xb8dd77a1`/`0x72e112df` — clones inherit working gates).
- Actions with **interned stock names** fire (e.g.
  `Options_LaunchBenchmark` → benchmark). Clone names are
  uninterned → silent drop.
- `LayerToPush` = live slot **2** — set on stock
  `Display_SafeZone`/`Display_Gamma` (`SafeZoneLayer`/`GammaLayer`),
  but does NOT make a `Video_LaunchBenchmark` clone press-push;
  different element wiring (don't write it on our clones — dead
  cell, harmless but inert).
- `Action.Menu` push records do not fire from FE row presses
  (context-gated or absent from the FE handler table; one press
  preceded an `exe+0x748b19` exec-fault — fragile, avoid).
- `ComponentButton` schema attrs: PrimaryState(0), AuxStateOverrides,
  Children, CreationPredicates, EnabledPredicates, ComponentData,
  SystemInfoData, PaneDescDisabled, PaneBundle, LayerToPush — live
  slots for SafeZone: LayerToPush=2, PrimaryState=3, ComponentData=7.

## MenuTreeNode.Legend (pane-bearing row) — VERIFIED-LIVE 2026-09-28

Schema (declared idx): PrimaryState(1) AuxStateOverrides(2) Children(3)
CreationPredicates(4) BeginNode(5) NavMarker(6)
NavMarkerCharacterStates(7) Texture(8) SizeScalar(9)
TintIfProgressListUnavailable(10) DetailTree(11) Region(12) Item(13)
EndNode(14) BeginHighlightPane(15) PanePopulateFromProgress(16)
UseChallengeHeaderFrame(17) PaneHeader(18) PaneHeaderIfLocked(19)
PaneDesc(20) PaneDescIfLocked(21) PaneImageIfLocked(22)
PaneBundleIfLocked(23) PaneDescAppendProgress(24) PaneMovie(25)
PaneImage(26) PaneBundle(27) ShowPaneMovieImgIfLocked(28)
EndHighlightPane(29) BeginProgress(30) …Progress*(31..42)
BeginLock(43) LockCommonPredicates(44) LockPredicates(45)
CanHighlightIfLocked(46) EndLock(47) BeginModel(48) ModelConfig(49)
ModelBundle(50) ModelOverviewCamera(51) ModelDetailCamera(52)
ShowModelIfLocked(53) ShowModelInOverview(54) EndModel(55)
BeginMedia(56) Video(57) VideoBundle(58) Audio(59) EndMedia(60)
BeginTelemetry(61) TelemetryAlias(62) EndTelemetry(63).

Live cells on `Locations_MountDoom` (data-dump verified):
- c5 = `…\locations_mountdoom.bndl`   (PaneBundle — pane layout bundle)
- c7 = `.tex` cstr                  (PaneImage — RENDERS in pane;
  wrote `modlogo.tex`, drew on highlight, live-verified)
- c8 = `pointofinterest.tex`        (row icon or PaneImageIfLocked)
- c10 = `App_Desc_MountDoom`        (PaneDesc strdb id)
- c13 = `App_Name_MountDoom`        (PaneHeader strdb id)
- c14/c15 = lockimage.tex + locked.bndl (IfLocked variants)
- c23 = `Apdx_Locations_MountDoom`  (progress/gate record ref)
- c28 = `Appendices_Detail`         (DetailTree — SPACEBAR View pushes)
- c40 = `collectible.tex`           (Texture or marker)
- c50..c52 = Collectible*/DoorPuzzle refs (progress item lists)
- c53 = "island"                    (Region)
- c63 = `StatusHeader_Tree`

`set_pane_image` takes the record's ENTRY addr; `live_slot` resolves
attr names through `entry+0x08`'s hash→slot table (meta: slot=u16[63:48],
count=u16[47:32]).

## AppendicesLayer (MenuLayer.Appendices) as screen host — VERIFIED-LIVE

- `AppendicesLayer` carries `OverviewTree` (attr 34 → live cell 12,
  `Appendices_OverviewTree`) — self-contained tree attr, so the FE
  direct-push (title cell3 → MenuLayer) materializes it.
- Wrapping our private tree in an `AppendicesLayer` clone renders FULL
  appendices chrome: title bar, category icon row, entry list frame,
  model backdrop (DefaultCameraState/ModelState), currency strip,
  prompt bar. ComponentButton clones inside render as a normal list.
- **Lock gating**: appendices entries/categories are progress-driven —
  at the frontend (no save loaded) `LockPredicates` evaluate locked
  (lock icon, empty list). Renamed clones additionally lose name-keyed
  progress binding. For mod rows the lock machinery (attrs 4,10,16,19,
  21,22,23,24,30..42,44..47) must be stripped or bypassed.
- **Entry model**: `Appendices_Overview_<Category>` nodes do NOT hold
  child node refs — their data embeds inline entry STRUCT GROUPS
  (~10 cells each: image .bndl, icon .tex, locked-icon, desc strdb,
  locked-name, name strdb, lockimage .tex, locked .bndl, progress-ref
  record like `Col_SpiderPickup_10`/`Appendices_Shelob`/`SCO_*`).
  Authoring mod rows in this layer means decoding+writing those groups
  (or finding the ProgressItemList attr that lists them).
- `ComponentButton.PaneBundle` (attr 9) is NOT consumed by this pane —
  only Legend-type Pane* fields are. `PaneImage` also absent on
  ComponentButton slots.

- Progress refs on Legend nodes/categories (`Apdx_*`, `Col_*`,
  `SCO_*`, `Appendices_*` cells) are **`Inventory.Item` records** —
  lock/unlock gates ride inventory possession, hence everything locks
  at the frontend (empty inventory).
