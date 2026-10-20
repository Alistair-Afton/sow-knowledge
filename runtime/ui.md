# UI model — Scaleform GFx + data-driven menus
> **Status:** **partial** — Project Nemesis source doc; per-claim confidence tags (VERIFIED-STATIC / VERIFIED-LIVE / VERIFIED-INGAME / MODEL / UNVERIFIED) preserved inline. Not independently verified by Andúril.

> **Provenance:** Project Nemesis runtime-model documentation (native `nemesis.dll` live-game forensics), mirrored and maintained in the corpus. Their `VERIFIED-STATIC`/`MODEL` tags carry over; unverified entries are marked inside each doc.


*Verified 2026-09-24 against live game + `ingame.embb` assets.*

## Stack

SoW's UI is **Scaleform GFx 4.5** (`GFX\x08` header) driven by gamedb records.

- Movies are `.gfx` files packed inside `interface\bundles\gfx\ingame.embb`
  (in `UI_Assets.Arch06`). a corpus tool list|extract` handles the bundle.
- `interface\flash\sp.gfx` is the root movie; it imports widget libraries.
- `interface\flash\widgets\componentlibrary.gfx` holds shared `LIB-*` widgets
  (menu list items, prompts, `LIB-Version` …).
- `interface\flash\widgets\mainmenu.gfx` holds `MAIN-*` symbols
  (`MAIN-SaveMenu`, `MAIN-Prompts`, `MAIN-PersonalProgress`,
  `MAIN-InitialStartPrompt`, `MAIN-GameLogo`).
- Textfields carry HTML-formatted default text
  (`<p align=…><font face="$HydraHUD" …>text</font></p>`) — many defaults are
  placeholders baked into the movie (`"Talion is awesome"`, `"300h 52m"`,
  `"WWWWWW…"`). Some fields bind to `$`-prefixed variables
  (`$Menu_UI_TotalPlaytime`) that resolve through gamedb.

## Text sources — three kinds

| Kind | Where the text lives | How to change it |
|------|----------------------|------------------|
| strdb-bound | `localisation*.strdb`, resolved by key hash | runtime text patch` |
| gfx-baked | default text inside the `.gfx` tag stream | patch the movie file (loose override) |
| programmatic | engine writes the field at runtime (e.g. `1.21`) | hook the writer / change source data |

**Caveat (live-verified):** widgets fetch their text once at
menu-build time and cache it — a runtime override only affects
*new* lookups, so text must be patched before the frontend builds.

## Menu records (gamedb)

The menu is fully data-driven; no item positions live in the movie.

- `Interface/Menu/MenuLayer/<Name>` — one layer per screen
  (`FrontEndRoot` = main menu, `InitialStartScreen` = title,
  `GameOptions`, `Gold`, `WBPlay`, …). Records hold `ContentSymbol`,
  `SelectionTree`, widget refs.
- `Interface/Menu/MenuTreeNode/GenericAction` — a menu item:
  label strz (`UI_Menu_Quit`, `UI_MainMenu_Start`, `UI_WBID_Menu` …),
  icon texture, action linkage. Frontend items: `FrontEnd_Start`,
  `FrontEnd_Continue`, `FrontEnd_Options`, `FrontEnd_WBPlay`,
  `FrontEnd_Quit`, `FrontEnd_Difficulty`, `FrontEnd_ChangeUser`, …
- `Interface/Menu/MenuSelection/Prompt` — selectable prompt record;
  carries the label name (e.g. `LaunchBenchmark` → `UI_Frontend_Benchmark`).
- `Interface/Menu/MenuSelection/Predicate/Node` — input predicate;
  `LaunchBenchmark` fires on `MenuEnter`/`Mouse1`.
- `Interface/Menu/MenuSelection/Action*` — the action records
  (`Options_LaunchBenchmark`, `QuitToFrontEnd`, `Options_ApplyChanges`…).
- `Interface/Menu/MenuTreeNode_State`, `MenuTreeNode_Attribs`,
  `MenuTreeNode_FX`, `ComponentButton` — presentation/state glue.

Labels resolve name → strdb hash via the gamedb name hash
(`h*0x397 + fold[c]` — verified: `namehash("UI_Frontend_Benchmark")`
equals the "RUN BENCHMARK" key `0xa0014283`). So the Nemesis plugin API accepts
key names directly: the Nemesis plugin API("UI_Frontend_Benchmark", "MOD MENU")`.

### Inline key-prompt tokens

strdb text can embed input-glyph tokens: `{MenuEnter}`, `{MenuAux6}`,
`{MenuEscape}`, `{CombatPrimary03}` etc. follow the `{TokenName}`
format. **Substitution is renderer-dependent, NOT a string-layer
feature** — verified 2026-09-24: the Nemesis plugin API(0xc86e43da,
"…{MenuEnter}…")` renders the braces literally on the title screen.
Real key icons in menus come from Prompt records (`Enabled=0xd` → a
separate icon widget), not inline text. For plain text fields, copy
the vanilla approach: `<font color='#74CDF3'>[SPACEBAR]</font>` markup
(the stock `0xc86e43da` entry uses exactly this).

### Adding a tree entry (verified recipe)

`Options_DetailTree_Video` is a `TreeBase` record whose `BaseNodes`
attr is an **inline record-entry-pointer array** — live cell layout is
slot-indexed via the record's hash table (entry+0x08 → sorted
`{key, meta}` pairs; `meta` bits 48+ = slot, bits 32–47 = element count
for array attrs). Schema index ≠ live slot (`BaseNodes`: schema 4,
slot 5; `Input`: schema 5, slot 20). the Nemesis plugin API handles the
insert: relocates the data block, bumps post-array slots in a cloned
hash table, repoints entry+0x10/+0x08 under thread quiesce, journals
for re-apply on rebuild, and self-idempotents on existing elements.

A button's live chain (entry pointers all the way):

```
ComponentButton node (e.g. Video_LaunchBenchmark)
  data cell3  → TreeNode_State entry      (PrimaryState — visuals+wiring)
  data cell7  → ComponentData record      (label carrier)
     cell2    → label element
       cell1  → help-name cstr            (side-panel description)
       cell2  → label-name cstr           (THE ROW TEXT)
TreeNode_State (e.g. Component_AutoConfig = compact Options styling)
  data cell4  → attribs  cell5 → Predicate.Node entry
  data cell6  → ElementData  cell7 → FX
Predicate.Node
  data cell3  → Action record (e.g. Options_LaunchBenchmark)
  data cell6  → Prompt entry
Prompt — element array, 4-qword groups {flags, name-cstr, nil, ctx}
  element0 name-cstr = PromptID → footer text ("[icon] <text>")
  element0 flags: 0xd = keyboard+gamepad+clickable → shows key icon
```

Three independent text sources per row: **row label** =
ComponentData element cell2, **footer prompt** = Prompt element0
name, **help panel** = ComponentData element cell1. Cloning must
deep-copy each (see the Nemesis plugin API/`add_button` in
`runtime/sdk/menu.lua`) — the prompt's PromptID does NOT feed the row
label, which is why the first clone kept showing the main-menu string.

### Live-verified override

```lua
the Nemesis plugin API(0xa0014283, "MOD MENU")   -- "RUN BENCHMARK" label
the Nemesis plugin API()                      -- link synthetic db (autoload!)
```

Renders in the main menu after a restart. the Nemesis plugin API(hash)` reads
through the engine path (synthetic db wins, then locale-gated real dbs).

## Version label (`1.21`, title screen bottom-right)

- Widget: `LIB-Version` in `componentlibrary.gfx` — a `Field` textfield,
  right-aligned 18pt HydraHUD. Registered engine-side as
  `BuildVersionWidget` (name-hash registration at `0x140c8c510`).
- Text is **programmatic**: `Interface.Shared.PublicBuildVersion` = `1`
  is only a show/hide flag. `Profile/GlobalOptions` carries
  `MajorVersion`/`MinorVersion` records — `1.21` is formatted from those.
- `0x1f1cab9f` → `"Hydra Version: {1!s}"` is a *different* template, not
  the title label.
- The live UTF-16 buffer ("1.21") is a heap string object; changing it
  post-hoc does not re-render the field. Changing the version at the
  source (MajorVersion/MinorVersion cells) or the widget writer is the
  path — GFx set-text entry points are not yet mapped (see GAPS).

## Mod-menu integration (MOD MENU entry)

**Done (2026-09-24):** the main-menu benchmark slot now reads "MOD
MENU" (the Nemesis plugin API("UI_Frontend_Benchmark", …)`) and benchmark
access is preserved via a cloned "Run Benchmark" entry at the bottom of
Options → Video (Advanced). The clone chain, styling, strings, and
idempotence are implemented in `runtime/sdk/menu.lua` (live-verified through the game's own records)
and applied by `runtime/nemesis_ui.lua` — framework code, not a mod.

**Dispatch: solved 2026-09-24.** `nemesis-sdk::actionwatch` hooks the
action-context builder (`exe+0x2775c0`) — every button activation queues
its private `Action` record entry to Lua; the Nemesis plugin API(entry, fn)`
observes it (notification, not replacement — the native action still
runs). For injected rows we bind `RefreshOptionsTree` (harmless) purely
as a dispatch signal and do the real work in Lua. One physical press can
dispatch twice — handlers debounce (~500 ms window).

**Label repaint model (verified live):** row text is resolved once at
widget build; `RefreshOptionsTree`, focus changes, strdb writes, and
element-name repoints do NOT repaint. Only a full tree rebuild
re-resolves — driven by the Nemesis plugin API (posts RIGHT+LEFT via
the Nemesis plugin API, ~360 ms/key; the tab-switch animation eats faster
input). `postkey` = in-process `PostMessageW` tap: keydown now, keyup
flushed on the 33 ms wndproc timer (~60 ms hold — a same-tick down+up
reads as "never pressed").

**Instance-list limit (G47):** screens do NOT iterate the record's
`BaseNodes` at render time — each screen materializes a private
node-entry list when it is built. Frontend Options builds lazily on
first open → `add_button` rows render/select/dispatch/repaint there
(end-to-end verified). The pause Options instance was built during
engine boot — before Lua init's `arrinsert` — so injected rows never
appear in pause. The record-side row array `0x651eb390` (live, 2026-10-04d)
already contains the injected `Display_RunInBackground` — only the
render list is stale. Workarounds: push a custom tree/layer (its list
builds at push time, post-insert — the the Nemesis plugin API recipe),
or repoint an existing pause row's dispatch binding — but NOT via the
record: **VERIFIED-LIVE 2026-10-04d**, `Pause_Extras.DestinationLayer`
repoint (data cell14 → mod layer clone) was ignored on press; no
aligned copy of the layer-entry ptr exists in any component object;
proper heap-pointer scans (`scanmem(string.pack("<I8",p))` —
`scanptr` walks module images only, `scanmem` with a text arg finds
ASCII echo) show NO non-DB back-reference to the button's record entry
or data block at all — the press binding is copied payload or an
index, not an address. `DestinationLayer` itself is vestigial (see
component dispatch below).

**Remaining for MOD MENU:** bind the retitled slot to a custom layer
push (`Action.Menu` / `LayerToPush`) or repoint an existing pause row —
the manager UI itself should be a pushed screen, not injected rows.

**Live-list location (2026-10-01, VERIFIED-STATIC):** the pause menu's
screen set is declared by the `Interface/Menu/MenuManager :: Menu`
record — 46 attrs, one `*Layer` slot per screen (`OptionsLayer` at
attr 41, `QuestLogLayer` 14, `HierarchyLayer` 17, …). The native
`MenuManager` object instantiates each slot at boot — that is the
boot-snapshot origin. The GFx registry `[0x1427017b0]` holds only ONE
collection ("Shared" — movies); screen instances are NOT there.
Locator staged: a live probe order-scores every heap array
that references Options_OverviewTree row identities (entry / label
cstr / ComponentData refs) to find the live 0x28-stride node list.

**Native schema registry (2026-10-01, VERIFIED-STATIC):** every
schema class has a static registration record in `.idata`:
`{attr_table VA, count, cppname, desc, schema_name, parent_reg}` —
2157 classes enumerated by a corpus tool →
the extracted corpus, with parent links resolving to the
inheritance chain (e.g. `MenuLayerOptions` → `MenuLayer`). The attr
tables are fixed-VA BSS populated at boot; a live probe
(`_G.schemadump()`) dumps them all in one live pass → the complete
native field map per class.

The plugin console overlay (`nemesis`) remains a viable interim surface
for mod management until native layers are safe.

## Action dispatch model (2026-09-24, live)

- `MenuSelection/Action` records are **identity tokens** —
  `Options_LaunchBenchmark` has *no* cells and its name/hash is absent
  from the exe. Dispatch is **not** name- or hash-keyed in code.
- The *layer* binds which action triggers which native behavior:
  `Frontend_OptionsLayer` (schema `MenuLayer.Options`) carries one attr
  per hardcoded behavior — `CommitSettingsAction`, `AutoConfigAction`,
  `ViewPersonalInfoAction`, `RestoreSettingsAction`,
  **`LaunchBenchmarkAction`**, `LPM{Install,Uninstall,Help}Action`,
  `SpeakerTest`. Each cell holds the matching `Action` record pointer.
  `findrefs(Options_LaunchBenchmark)` → only `Frontend_OptionsLayer@256`.
- Expected dispatch shape: predicate fires → generic selection code gets
  the action record → active layer's handler compares it against its
  action cells → runs the matching slot's native routine.
- **Action subtypes have real payload fields** (dispatch by record type):
  `Action.Menu` (43, push a `MenuLayer`), `Action.MessageBox` (27),
  `Action.Faction` (6 — fire FactionEvents natively, per old research:
  `AddResurrectFactionEvent` etc.), `Action.Game`/`Quest`/`Video`/
  `Region`/`Tree`/`HierarchyDialogue`, `Action.Menu.LayerData.*`.
- `ComponentButton` nodes have a `LayerToPush` cell — buttons can push
  a `MenuLayer` directly without the action path.
- Menu input events are an enum: registration table at `0x142a4b830`
  (16-byte entries, `call 0x14117e9b4`), `MenuEnter` = `0x20`.
- The `Interface/Menu/MenuSelection/Action` list handle resolves at
  `0x141b0f4c5` → global `~0x142701268`.
- **Exe is repacked**: code+rodata live in `.arch`/` .xtext`, `.text` is
  a 0x14-byte stub — see `docs/reference/hashes-and-formats.md`. All
  prior "scan .text" nil results must be re-run over `.arch`+`.xtext`.

**Resolved 2026-09-24:** the action-context builder at `exe+0x2775c0`
is hooked (`actionwatch`) — it queues each dispatched action's private
record entry; the Nemesis plugin API delivers it to Lua. Checkbox/cycle
rows go through a different element-activation path (hook at
`exe+0x1951708`, `menupoll`/`on_activate`) — `element+0x10` is the bound
field object whose vtable teardown calls unconditionally, which is why
cloned checkboxes AV on teardown (datasource must stay shared/valid).

**the Nemesis plugin API (built, awaiting first live verify):** clones
a `TreeNode.ComponentCheckbox` (`Video_DOF` default template) — node
cells 6/7 are the Checked/UncheckedComponentData variants (row label +
tick icon live inside), cells 10/11 the Pane* help-panel record entries
(kept shared — record entries, not embedded structs), cell 9 the
datasource (kept shared → teardown-safe). The mark is driven by
pointing BOTH compdata cells at one private variant — whichever branch
the dead binding resolves shows our state. `on_toggle` runs Lua-side,
the engine press is swallowed, and `nemesis_ui` uses it for the
Display "Run in Background" row (persisted via the Nemesis plugin API). Full
native control vocabulary: `MenuComponent.{Button×78, Checkbox×25,
Cycle×18, Slider×15, TextEntry×5}` — same clone recipe applies.

**Control record schemas** (schema dump, VERIFIED-STATIC 2026-09-26 —
all share the base 19 attrs `Platform…SizeOverride`, then):

- `MenuComponent.Cycle`: `[19] LabelClip`, `[20] Values` — the
  option list. Clone a donor (`Display_Mode`'s cycle) → rewrite
  Values + label.
- `MenuComponent.Slider`: `[19] ShowValue`, `[20] ValueRange`,
  `[21] HasPreferred`, `[22] PreferredValue`, `[23] NumTicks`,
  `[24]/[25] OnActivate/DeactivateAnimation`, `[26] SliderFX`.
- `MenuComponent.TextEntry`: 5 component records exist
  (`WBID_CreateNew_{Email,Password,PasswordRepeat}_TextEntry`,
  `WBPlay_EmailTextEntry`, `FactionMemberSaveTextEntry`) — `Field`,
  `Label`, content element `WBID-TextEntry`, mode `in`/`out`, style
  `Password`/`Email`/`Normal`. **No `TreeNode.ComponentTextEntry`
  records exist** (the list is empty — verified 2026-09-26): text
  entry cannot be a tree ROW because there is no donor node type to
  clone and a clone's record type is fixed by its entry header.
  Text input is screen-scoped only: a `MenuLayer` carrying a
  `MenuComponent.TextEntry` component (donor: `FactionMemberSave`
  layer — the game uses it for name-entry dialogs — or
  `MenuLayer.OnlineSearch`, which also carries `TextInputState`@39 +
  `SearchHeaderText`). `add_screen` can clone such a layer; the typed
  value would be read back via the layer's `TextInputState` record
  (live-gated — G45/G47 session).
- `MenuComponent.Button`: base + `[19] LabelClip`, `[20] LabelAltClip`.

## Pushing a custom screen (decoded 2026-09-24)

`Action.Menu` `Push*` records are NOT hardcoded — they carry the screen
in cells: cell5 → a `MenuLayerList` record (e.g. `PushStatsLayer` →
`Stats_Single`, `PushCreditsLayer` → `Credits_Single`,
`PushQuestLog` → `QuestLog_Single`), cell7 → an optional
`MenuLayerState` record. `PushLayerFromData` is an empty variant.

`MenuLayerList` (**VERIFIED-INGAME** 2026-09-25, corrected): a
transition table of repeating `{count, layer-entry…}` groups — NOT
named per-context attrs. `QuestLog_Single` decodes as `{2,
Root_QuestLog, QuestLog}, {2, …}` — each group's count precedes its
layer-entry refs (root layer first, content layer second).
`PushQuestLog`.cell5 → `QuestLog_Single`; its `QuestLog` group
member → `MenuLayer.QuestLog::QuestLog` record. (The earlier
`ValidInPauseMenu`/`ValidInMainMenu` reading was a schema-name
mislabel — same attr-name drift class as the subtype slots below.)
Safe mutation: clone the list, replace every cell holding the source
layer's entry addr — `add_screen` does this by entry-scan, no attr
names needed.

`MenuLayer` (~33 attrs) carries ContentSymbol (GFx), SelectionTree
(a TreeBase), Components, NavPane, transitions, sounds, prompts,
`DefaultLayerState`, `OverrideLayers`, HUD state.

**Subtype attr → live slot** (**VERIFIED-INGAME** 2026-09-25):
`MenuLayer.QuestLog::QuestLog` declares `OverviewTree` at schema index
30 but the value lives at live data slot 11 — resolved through the
record's hash→slot table (entry+0x08), whose keys' lo32 = the attr
namehash (`namehash("OverviewTree")` = 0x063ad377 = key lo32 → slot
11; the value there is `TreeBase::QuestLog_OverviewTree`). Declared
schema indices are NOT live cell offsets — always resolve through the
slot table. the Nemesis plugin API does this via `live_slot()`.

`MenuLayer.Options` subtype (**VERIFIED-INGAME**, ~60 cells):
`Components`@18 → `TreeBase::FrontEnd_Options_OverviewTree`;
cells 19-49 are subtype fields — a second `TreeBase` (detail tree
`Options_DetailTree_Sound`), `MenuWidget`/`MenuWidget.Prompt`
records, `MenuSelection.Action` records (`Options_AutoConfig`,
`LPM_{Install,Uninstall,Help}`, `ShowNetworkPersonalInfo`,
`Options_LaunchBenchmark`), `MenuSelection.Predicate` records
(`Options_ApplyChanges`, `Back`), a `MenuTransition`
(`UpgradesMenu`), overlay pointers, and a `MenuLayer.FrontEndRoot`
ref. The recon's attr NAMES misalign over subtype slots (two
name-streams overlap: e.g. `CustomerIDWidget`@32 really holds the
`Options_LaunchBenchmark` action) — trust ref types, not names.

**Inline component/node arrays** (**VERIFIED-INGAME**): layers and
trees keep their elements as contiguous record-refs inside the data
block, and schema attr names mislabel element slots.
- `MenuLayer::MM_Options` cells 19-28 = 10 contiguous
  `MenuComponent.Button` refs (Sound/Video/Advanced/Game/
  KeyboardBindings/PC/360/XBOne/PS3/PS4) — the main-menu options
  nav list. `NavPane`@19 is really element[0].
- `MenuLayer::FactionMemberSave` has a
  `MenuComponent.TextEntry` as a component — text-entry dialogs
  are layer components.
- `TreeBase::Options_DetailTree_Display` cells 5-14 = 10 contiguous
  `TreeNode.Component*` refs in display order (Display_Mode,
  Display_Mode_UWP, Display_Resolution, Display_Adapter,
  Display_Gamma, Display_SafeZone, Display_VSync,
  Display_ConsoleRenderPref, Display_DynamicResolution,
  Display_MaxFPS) — the row array `tree_insert` splices.
  `BaseNodes`@4 is the array marker; a `TreeBase_Input` record
  binds the tree's input handling.
- Node rows are `TreeNode.Component{Button,Checkbox,Cycle,Slider,
  KeyRebind}` records.

**Native mod menu recipe** (the Nemesis plugin API —
**VERIFIED-INGAME** chain, 2026-09-25): clone `MenuLayer.
QuestLog::QuestLog` → write the private tree entry into the
`OverviewTree` live slot (11, via hash→slot) → clone
`QuestLog_Single` and entry-scan-swap the source layer refs for the
clone → clone `PushQuestLog` (name kept — actions are name-keyed) and
set cell5 to the list clone → `repoint_pred_action(pred_entry,
push_clone)` on the frontend slot's predicate. A pushed screen
materializes its node list AT PUSH TIME — post-insert — so every
cloned row appears natively (dodges the G47 pause-instance snapshot
problem entirely). Templates are overridable via opts
(`layer_src`/`tree_attr`/`list_src`/`push_src`).

### Private trees (2026-09-24, offline-verified)

A pushed layer references its tree by **pointer**, so the tree needs
no list registration — a leaked clone keeps mod rows out of every
vanilla tree. the Nemesis plugin API(src, name)` clones entry+data
verbatim; `tree_insert`/`tree_clear` replicate
`array_insert_inline`'s semantics on the unregistered clone:

- elements are spliced into the data block (every later slot shifts);
- the record's hash→slot accelerator (`entry+0x08` — a separate alloc
  of sorted 16-byte `{hash, meta}` nodes; meta bits48..63 = live slot,
  bits32..47 = inline-array count) is cloned with bumped slots/count;
- `entry+0x10`/`entry+0x08` are repointed; the source is never touched.

Caveat: metas exceed f64's 53-bit mantissa — they round-trip as **hex
strings** end to end (`write_u64` accepts them since 2026-09-24;
`menu.hex_write_ok()` probes so old DLLs degrade instead of corrupting).
`build_node` (extracted from `add_button`) produces insertable rows
without touching a registered tree. `runtime/modmenu_screen.lua`
consumes all of it: one toggle row per mod, a two-tap Reorder Mode
(single-press semantics only — pick up, drop), Reload Mods.

Assembly: the Nemesis plugin API{tree_ctx, layer_src, context, ...}`
does the full recipe — clone layer → write tree-attr cell (live index
via `cells()`, no marker math) → clone a MenuLayerList and write the
layer clone into its `ValidInPauseMenu`/`ValidInMainMenu` cells
(context-gated by `opts.context`, live indices) → clone push action →
write cell5 → returns the push entry for `repoint_pred_action`.
`nemesis_ui` prefers a persisted `modmenu.push` over the PushQuestLog
v1 fallback. Best layer templates: `MenuLayer.LanguageSelect`
(`SelectionTree`@29) and `MenuLayer.GameModeSelect` (`Tree`@30 +
`Node`@31) — simple single-tree list screens. Remaining unknowns =
the MenuLayerList Layers element shape + template choice → G48,
decoded by a live probe.

## Frontend component dispatch (2026-09-24)

- `MenuComponent.Button` records (the main-menu list entries) have **no
  action field** — 24 attrs are presentation-only (label, sounds, FX,
  `DestinationLayer` — empty even on working push buttons).
- The semantic binding is a **`Predicate.Node`** record shared with the
  tree-row model: `LaunchBenchmark` (`cell0` = input-event name
  `MenuEnter`, cell1 `Mouse1`, cell3 = ResultAction). TreeNode_State
  cell5 → Predicate.Node; frontend components bind the same way.
- `Action` records are **empty identity records** (`Options_LaunchBenchmark`
  has zero attrs) — dispatch is name-keyed: the executor resolves the
  record's name against a registered handler table, payloads come from
  the action's subtype cells (e.g. Push* cell5/cell7).
- Verified live: pressing the MOD MENU slot read NEITHER the action
  record entry NOR the component record (watchpoints, 0 hits) — the
  frontend resolves its binding before/at dispatch differently than the
  options-tree path (the `exe+0x2775c0` ctx-builder hook only sees
  tree/predicate dispatches).
- the Nemesis plugin API(pred, action_entry)` rewrites a
  Predicate.Node cell3 — the write target for MOD MENU. `pred` accepts
  a name (base `Predicate` list), `{data=}`/`{addr=}` table, or a raw
  record ENTRY addr (name-map node). `nemesis_ui` repoints
  `LaunchBenchmark` → a private `PushQuestLog` clone (name preserved —
  dispatch is name-keyed), keeping `Component_LaunchBenchmark`'s
  Video-row chain on a private pred clone so Options→Video
  'Run Benchmark' still works.

### Subtype lists are not enumerable (VERIFIED-LIVE 2026-09-25)

- `Predicate.Node` (offline `Interface/Menu/MenuSelection/Predicate/
  Node`, 92 records) has **no runtime list** — `db.lists()` shows only
  the base `Predicate` (74 live = 58 base + patches; subtype records
  do NOT roll up) and `Predicate.FactionMember` (113). `Predicate.Quest`
  likewise absent.
- The records ARE interned in the global name→record map: ref cells
  store the 0x28-byte map ENTRY `{key=namehash<<32|tag, +0x08 aux,
  +0x10 record data, +0x18 chain, +0x20 name cstr}`. Verified:
  `TreeNode_State::Component_LaunchBenchmark` cell5 → entry
  `0x638a59f0` whose `+0x10` record has cell3 = the
  `Options_LaunchBenchmark` action entry.
- Resolution path for unlisted records: follow a known referrer's ref
  cell (live-verified through the game's own records), or `findrefs`. nemesis_ui's
  MOD MENU wiring resolves `LaunchBenchmark` via
  `Component_LaunchBenchmark`.cell5 — `findrefs` shows that state is
  the pred's ONLY record consumer; the frontend slot resolves it by
  name (or entry) outside gamedb, so repointing the record's cell3
  redirects both. The state's own cell5 is moved to a private
  `Nemesis_Benchmark_Pred` clone first so the Options row keeps the
  original chain.
- Timing: frontend menu records (`TreeNode_State` et al) are NOT
  loaded at Lua init — `nemesis_ui` wiring runs as a retry-until-done
  the Nemesis plugin API("ui")` pump; `modmenu.pred_entry` is persisted in the Nemesis plugin API
  for F8-reload idempotence (clone detected by entry-name).
- `MenuLayer::MM_Options` cells 19-30 = component ENTRY array
  (`MM_FrontScreen_Benchmark` @cell30); `MenuLayer.Options::
  Frontend_OptionsLayer` cell32 (byte 0x100) holds the
  `Options_LaunchBenchmark` action entry directly.
- Extent caveat: `entries()` sibling-distance extents explode when a
  record's data block is its arena's highest — `Options_DetailTree_
  Display` reported 1.7GB. `clone_with_array` caps at 0x2000 and falls
  back to the slot table's live range (hi slot+count).

## Input model (corrected 2026-09-24)

- USER32 imports have **no GetForegroundWindow/GetFocus/PeekMessage** —
  the engine pumps `GetMessageA` and cannot check real foreground. There
  is no OS-focus input gate: `PostMessage` keys always reach the wndproc.
- Every "input ignored while unfocused" observation this session was
  the **in-game console** (overlay::wndproc consumes keys while open,
  `~` toggles) or the GENETICZZ2K ImGui overlay — not a focus gate.
- the Nemesis plugin API()` therefore only posts the fake-activation trio
  (WM_NCACTIVATE/WM_ACTIVATE/WM_SETFOCUS) as insurance — it never calls
  real SetForegroundWindow/BringWindowToTop (user's desktop untouched).
  `postkey` auto-asserts it when another app owns the real foreground.
- the Nemesis plugin API(msg,w,l)` = raw PostMessageW to the game window.
- the Nemesis plugin API()` is the complementary guarantee: the wndproc
  swallows the deactivation message set so the engine never *drops* its
  believed-focused state — keeps synthetic input digestible AND stops
  auto-pause on alt-tab.

## Frontend synthetic input — ROOT CAUSE (VERIFIED-INGAME 2026-09-30m)

The earlier claim "no synthetic input variant advances the title" was
misattributed to the frontend. Real mechanism:

- The GENETICZZ2K `SoW_ModMenu.dll` overlay auto-opens at boot and eats
  keyboard **downstream of our wndproc hook** (its subclass sits after
  ours in the chain): msgwatch records every posted/real key arriving,
  but the game's own handler never sees them. That's why postkey,
  keybd_event, attached SendInput and real focused SPACE all failed
  identically. The overlay's toggle is VK_F6 via GetAsyncKeyState's
  edge bit — real-device-only, so posted keys can never close it — and
  its visibility state is heap-owned (the `mod+0xf58a*` bytes probed
  earlier are toggle-intent/ImGui `p_open` bools, not the render gate).
- With `SoW_ModMenu.dll` moved aside pre-launch, the SAME postkey drive
  (SPACE→ENTER→SPACE, 2 cycles) reaches in-world cold — verified twice
  (a corpus tool, exit 0). No raw-input/XInput mystery: the
  frontend digests ordinary WM_KEYDOWN fine.
- `PlayerInput.SetSimInput` was tested live (gated `console.exec`) and
  confirmed no-op at title — it requires an active player
  (`[mgr+0xe90]+0x40` index, `[arr[idx]+0x38]==3`), so it stays an
  in-world tool, not a title bridge.
- a corpus tool now moves the overlay DLL aside for launches it
  owns (restored after plugin enumeration) and warns on attach when
  `modbase('SoW_ModMenu.dll')` is loaded.

## Console toggle binding — scancode-authoritative (2026-09-30)

The in-game console toggle (`nemesis.ini console_key`, default
scancode `0x29`) matched on **hardcoded OEM VK fallbacks** in both the
wndproc and the `GetAsyncKeyState` poll path. On a German layout
those VKs are ordinary typing keys — ö=`0xC0`@sc`0x27`,
ß=`0xDB`@sc`0x0C`, <>=`0xE2`@sc`0x56` — so typing them flickered the
console open/closed (only the 300 ms debounce capped the rate).
Fix: the scancode is the binding. The poll path only accepts a
fallback VK when `MapVirtualKeyExW(vk→vsc) == console_key`; the
wndproc is scancode-authoritative and only reverse-map-checks the VK
for synthetic posts carrying `scancode == 0`. `console_vk` (explicit
VK binding) is still honored verbatim.

## GFx movie runtime objects (2026-09-26, VERIFIED-STATIC)

UI context singleton: `0x1426ffb08`, a `0x860`-byte object allocated
by `0x14188bab0` (ctor `0x141889674`).

**This is the master game-context singleton**, not a UI-only object
(2026-09-29 unification): the same root is read by the faction
event-source dispatch `0x1403511c0` — `ctx+0x7e8` holds the event
-source container pointer (0x40-stride `{ptr,u32}`-keyed entries,
see `docs/model/nemesis-memory.md`). UI movie slots live at
`ctx+0x360..+0x388`. Other subsystem pointers presumably occupy the
rest of the `0x860` block — worth mapping as recon continues.

### ctx field map (rip-relative ref scan, 897 sites — 2026-09-29)

| field | refs | identified |
|-------|------|------------|
| `+0x10` | 76 | ptr — compared/stored; session/state sub-object |
| `+0x30` | 109 | manager ptr — `0x14016b63c(ctx+0x30)` validator; sub-ptrs at `+0x128/+0x130` |
| `+0x50` | 131 | manager ptr — `+0x578` flag check; heavy entity-path use |
| `+0x70` | 294 | **presentation manager** — `ShowEnemyFeared` handler registered here via `0x14032e3b8` |
| `+0x80` | 80 | subsystem ptr — `0x14046e270(obj, [x+0x368])` calls |
| `+0x360/0x378/0x380/0x388` | | UI movie slots (G45) |
| `+0x518` | 5 | UI-region object (0x14189* sites) |
| `+0x7e0` | | subsystem ptr → `0x141932cd8` |
| `+0x7e8` | | **faction event-source container** (0x1403511c0) |
| `+0x7f8` | | subsystem ptr — identity-check helper `0x1406bb6f4` |
| `+0x800`/`+0x810`/`+0x828`/`+0x840` | | subsystem ptrs (lea/mov users) |

Scan method: `mov reg,[rip→0x1426ffb08]` sites, first `[reg+off]`
follow. Full count is in the 2026-09-29 journal.

Movie storage inside the context:

- Slot array at `ctx+0x360` (entries are 8-byte pointers to
  movie-entry objects; entry `+0x48` → movie object; `+0x38` →
  root GFxValue/display object). Accessor `0x14188a3f0(ctx+0x360, idx)`
  returns slot pointers; `0x1418927f0` iterates the array calling
  `vtable+0x88` on each live root.
- Cached aliases: `ctx+0x378` = slot[0] (**main** movie),
  `ctx+0x380` = slot[1] (**nav** movie), `ctx+0x388` = slot[2]
  (third movie — likely HUD/ingame).

Root-object accessors (used by the `UI.Main.*`/`UI.Nav.*` console
commands): `0x14188daa0` (main) / `0x140701ea4` (nav), i.e.
`[[[0x1426ffb08]+0x378|0x380]+0x48]+0x38`.

Movie-array init `0x14188baf0` enumerates the named collection
**"Shared"** from registry object `[0x1427017b0]` (lookup helper
`0x14114b534` → `0x140618f7c`), creating one movie-entry per element
(`0x141889cb0`/`0x141889250`/`0x14188a044`), then fills the three
aliases. The registry is NOT gamedb — `Interface/Movies` gamedb only
holds `Movies.Events.*` presentation records; the collection is fed
by native UI bundle data.

GFx method slots observed on movie/root objects: `+0x38`
(PrintMovieChildren), `+0x40` (DumpMovieTree JSON), `+0x48`
(PrintMovieState), `+0x88` (16-byte-arg broadcast called per root —
candidate event/var set).

### Movie object graph (2026-09-28, VERIFIED-LIVE)

Four-level chain under the slot entry. Both main (`ctx+0x378`) and
nav (`ctx+0x380`) movies share all vtables; only instance data
differs.

| level | fetch | class | vtable | slots |
|-------|-------|-------|--------|-------|
| slot entry | `*[ctx+0x378]` | movie-slot registry entry | — | — |
| movie | `*[slot+0x48]` | movie container | — | — |
| root | `*[movie+0x38]` | ASMovieRootBase-ish (display root) | `0x141e6ee80` | 82 (`0x00`–`0x288`, `+0x290`="createTe" str) |
| impl | `*[root+0x18]` | MovieView wrapper (`MovieImpl`) | `0x141e73420` | 71 (`0x00`–`0x230`, `+0x238`="Image" str) |
| asroot | `*[impl+0x18]` | **AS movie root — the variable API** | `0x1422ca568` | ~71 |
| core | `*[impl+0x50]` | display-state worker (frame/pause flags) | `0x141db9340` | 129 (`0x00`–`0x400`) |

Vtable dumps: `extracted/gfx_vt_{movieimpl,root,asroot,core}.tsv`
(slot → target VA); classifier output via a corpus tool;
call-site string harvest via a corpus tool.

Method dispatch is layered: `impl` view-level slots mostly tail-call
`[impl+0x50]` (core) for display state or `[impl+0x18]` (asroot) for
AS values. Confirmed forwardings on impl: `+0x10`→`core+0x360`
(GetCurrentFrame — leaf getter of `core+0x11c`), `+0x18`→`core+0x390`
(HasLooped — bit `[+0x138]>>1`), `+0x20`→`core+0x378` (GotoFrame —
clamped setter via `+0x380`=GetFrameCount), `+0x30`→`core+0x398`
(SetPlayState — writes `+0x118`), `+0x48`→`core+0x1d0` (SetVisible —
bool flag + propagate), `+0x50`→`core+0x1d8` (GetVisible — bit
`[+0x6a]>>0xe`). This matches the GFx `Movie` base order at indices
2–9 (GFx 4.5 ≈ CommonLibSSE `GFxMovieView` layout, with a few
game-specific slots inserted in the `0x40`–`0x50` range).

### Variable/Invoke ABI (VERIFIED-LIVE 2026-09-28)

Named by real game call sites (disasm of callers loading AS path
cstrs), then called live at title without faults:

| asroot slot | identity | signature | evidence |
|-------------|----------|-----------|----------|
| `+0x188` | **SetVariable** | `(asroot, const char* path, GFxValue* value, SetVarType type)` | game site `0x141288bf4` loads `"_global.gfx_ime_candidate_list_state"` + built GFxValue + type int; live call with zeroed GFxValue OK |
| `+0x190` | **GetVariable** | `(asroot, GFxValue* out, const char* path) -> bool` | disasm requires rdx≠0 (out) and treats r8 as path; live `GetVariable("_root")` → ret `1`, fills `out+0`=managed ptr, `out+8`=0x4a tag |
| `+0x1c8` | **Invoke** | `(asroot, const char* name, GFxValue* result, GFxValue* args, u32 numArgs@stack)` | game site `0x1411bfd5b` loads `"Selection.setFocus"`, `result=NULL`, `args=&zeroed`, `numArgs=1`; thunk resolves target via `0x1402abfe0` then vcalls `asroot+0x1f0` |
| `+0x1f0` | Invoke worker | `(resolvedClip, leafName, args, result, numArgs)` | called by the +0x1c8 thunk |
| `+0x48` / `+0x50` | chain forwarders | remap `(rcx,rdx)` via `+0x38→+0x20` / `+0x10→+0x30`, tail into `0x14128e18c` / `0x14128e21c` | reached via `impl vt+0x188`; likely the Movie-level SetVariable/GetVariable workers |
| `+0x198`–`+0x1b0` | Invoke/array family | `(resolve×2 → vcall+0xf0 → string init → write rdx)` motif | SetVariableArray/GetVariableArray/InvokeArgs candidates |

GFxValue is **0x18 bytes** (array stride confirmed by the +0x1f0
worker's arg walk `lea rdx,[rcx+rcx*2]; lea r12,[rcx+rdx*8]` =
`i*0x18`): `ObjectInterface*` @+0x00, `u32 Type` @+0x08 (bit `0x40` =
managed), `u64 union` @+0x10. Zeroed buffer = undefined value — safe
as `result`/`args` placeholder.

Type codes confirmed against the game's own GFxValue ctors
(`0x140468954` base init writes `Type` at +8; `0x14115b7c0` number,
`0x1411beee8` boolean, `0x14116e6d0` string, `0x1411bf0b1` stringW):
`0`=undefined, `1`=null, `2`=bool (byte), `3`=int (dword), `5`=number
(f64), `6`=string (char*), `7`=stringW, `8`=object, `10`=displayObject
(`_root` read back `0x40|0x0a`).

**Managed strings:** a `Type&0x40` string's union ptr is a pool-node
pointer, not the chars — `node+0x00` = char*, `node+0x20` = length
(verified live: `setvar('_root.__nemesis_str','mordor')` → node
`0x7ae0f358`, `[node]` → `"mordor"`). Unmanaged strings are direct
char* — that's what `setvar` hands the movie; it copies into its pool.

View-level `impl vt+0x188` is itself a forwarder
(`mov rcx,[rcx+0x18]; jmp [rax+0x48]` → asroot+0x48), so the public
surface works at both levels; calling `asroot` directly skips a hop.

**Unloaded-movie behavior:** at splash (before title menu movie
loads) the same calls return a sentinel pointer and leave the
out-buffer untouched — fail-closed, no crash.

**Known-unsafe slots (do NOT call):** `impl+0x1a0` and `impl+0x1a8`
AV deterministically under generic arg shapes (they dispatch vcalls
through arg objects); `asroot+0x008` and `asroot+0x068` AV at title
(lifecycle/foreign-object methods). All fatal probes faulted at
`ShadowOfWar.exe+0x1215f32`/`+0x221fa6`/`+0x2abf56`-family sites —
async, `pcall`-proof.

## Texture/image path for thumbnails (2026-09-26, VERIFIED-STATIC)

Native texture registry exists — `Marketplace.Icon` records are
`{Texture: "interface\textures\store\x.tex", Color}` and
`Marketplace.Image` adds `Bundle`. ~90 `Icon_*` records map names to
`.tex` paths for the store UI. So a mod thumbnail = an `Icon_<mod>`
record pointing at a `.tex` shipped in the mod's VFS `files/` dir —
the record side needs no GFx API. Consumption (widget binds
icon-name → texture) is through GFx widgets — the icon render
binding itself is still unmapped (G45).

Screen-space draw primitives also exist as console commands:
`UI.SetScreenMaterial [Path] [L T R B]` / `UI.ClearScreenMaterial`
(UIMgr.cpp) — renders a material over a screen rect, and
`UI.SetRegionMaterialValue` (WorldMapMaterialController) — viable
for splash/badge overlays without widget binding.

### Help-pane images — the GFx-free thumbnail path (2026-09-26)

The options-tree **help pane** natively renders images — no GFx
binding needed. Schema (`schema_dump.tsv`/`blend_fields.tsv`):

- `TreeNode.DLC`: `PaneDesc`@7, `PaneImage`@8, `PaneBundle`@9
- `TreeNode.Stats`: `PaneDesc`@19/20, `PaneImageIfLocked`@21,
  `PaneBundleIfLocked`@22, `PaneMovie`@24, `PaneImage`@25,
  `PaneBundle`@26
- `TreeNode.CommunityChallenge`/`StoreMOTD`: `PaneImage`,
  `PaneLockedImage`, `PaneImageBundle`, `PaneLockedImageBundle`
- `ComponentCycle/Checkbox/TextEntry`: `PaneDescDisabled`,
  `PaneBundle` (pane *layout* bundle, not the image)
- `CycleNodeData` (cycle option elements): `PaneDesc`@1,
  `PaneMovie`@2, `PaneImage`@3, `PaneSound`@4 — **each cycle option
  can carry its own pane image/movie/sound**
- `Interface.Predicates.LegendNode`: `HasPaneMovie`/`HasPaneImage`
  gating predicates

`PaneImage`/`PaneBundle` cells are **class-4 string cells** — they
hold the `.tex`/`.bndl` path cstrs directly on the node (VERIFIED
by serialized record content: `CommunityChallenge` stores
`…\modlogo.tex`, `…\monolith.tex`, `…\locked.bndl` in its string
fields). Two earlier readings were schema-index→live-slot misreads:
the `TreeNode_State` ref seen in pane cells was `PrimaryState`
(field 0), and the `Marketplace.Image` record the wrong cell
resolved. `Marketplace.Image` records `{Texture, Bundle}` still
exist as the *store's* icon registry — unrelated to node panes.
the Nemesis plugin API{node, tex, bundle}` writes cstrs into the
node's live slots (namehash→slot-table); `SilverWarChest` ships
`Bundle` empty — loose `.tex` over VFS is plausible without a .bndl.
Live confirmation armed in a live probe §9 (`dump_strs`
prints node cells whose values resolve to cstr paths).

### `.tex` format (VERIFIED-STATIC, orc_grenade_d.tex)

`TEXR` magic + u32 version(8) + header fields + source-name cstr at
+0x40 (`...tex0` variant name) + more fields, then a **standard DDS
payload embedded at +0x69** (here: 512×256 BC). `.tex0`/`.texuhd`
siblings are mip-tier alternates. So `.tex` = trivially craftable:
fixed TEXR header + name cstr + DDS bytes — mods can ship real
thumbnails (DDS source) via the VFS `files/` path.

The dedicated mod-logo slot is the game's own
`interface\textures\hud\mod\modlogo.tex` (128×128 DXT5); a crafted
`.tex` keeps the same header + source-name layout, and DDS art must
match the donor's dims/format.

A pane **`.bndl`** is a trivial manifest (`BNDL` v6): type
`Bundles/Types:UI_QuestStartPoint` + resource path cstrs + their
byte sizes (verified byte-identical round-trip against the game's
own queststart bndls). Most retail bndls have no `.embb`
payload — the manifest resolves its resource paths through the VFS,
so a loose `files/` `.tex` + authored `.bndl` pair is sufficient.

**Thumbnail recipe (armed, live-verify pending)**: craft
`.tex`(DDS) → `Mods/<mod>/files/interface/textures/mods/x.tex`;
pack `x.bndl` → `files/interface/bundles/mods/<mod>/x.bndl`;
declare `thumbs = {"x.bndl"}` in `manifest.lua` (or drop files in
`Mods/<mod>/thumbs/`). `modmenu_screen` writes the first thumb into
the mod row's `PaneBundle` cell via the Nemesis plugin API —
`ComponentButton` rows carry `PaneBundle` (declared 8) but **no**
`PaneImage`, so mod-row thumbs are bundle-only. Entries containing a
path separator are used as VFS paths verbatim; bare filenames map to
`interface\bundles\mods\<name>\<file>`, and a `.tex` entry resolves
to its sibling `.bndl`. For pane-bearing row types
(`TreeNode.Stats`/`DLC`/`CommunityChallenge`), `set_pane_image` can
also write `PaneImage` tex paths directly; per-option images use
`CycleNodeData` pane cells.

**Row template choice**: `ComponentButton` has NO image cell
(`PaneDescDisabled`/`PaneBundle` only). `TreeNode.DLC` is the
thumbnail-capable interactive template — same activation shape as
buttons (`PrimaryState`@0, `AuxStateOverrides`, `Children`,
`CreationPredicates`, `EnabledPredicates`, `ComponentData`) PLUS
`DLC`@4, `PaneDesc`@7, `PaneImage`@8, `PaneBundle`@9. Whether a DLC
node's press reaches the element-activate hook (and swallows the
same way) is live-only verification — if it does, mod-menu rows can
be DLC-node clones carrying per-mod `PaneImage` clones.
`TreeNode.CommunityChallenge` carries THREE image cells (donor
record shows pane bundle + two `.tex` refs incl.
`interface\textures\hud\mod\modlogo.tex` — a dedicated mod-logo
slot shipped in-game) + `PaneLockedImage*`.
