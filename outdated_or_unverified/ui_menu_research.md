# UI & Menu System Research Notes

Detailed findings about the Dying Light UI/menu system, based on direct file inspection.

---

## How the UI system is organized

The UI system is structured as **packages**. Each package corresponds to a directory in `gui/` and has:

| File | Purpose |
|------|---------|
| `*.uip` | Package manifest — lists all symbols, atlases, effects |
| `*.rscr` | Resource registration — maps texture names to file paths |
| `*.gui` files | Individual UI element/screen definitions |
| `*.atlas` files | Texture atlas manifests |
| `*.guieff` files | Shader effects (shared, in `materials/effects/`) |

### UI Packages found

| Directory | Package name | Contents |
|-----------|-------------|----------|
| `gui/common_pc/` | `common` | Shared widgets (buttons, slots, icons, loaders) |
| `gui/menu_common_pc/` | `menu_common` | Shared menu widgets (hint buttons, common dialogs) |
| `gui/ingame_menu_pc/` | `ingame_menu` | All in-game menus (inventory, map, hints, skills, etc.) |
| `gui/main_menu_pc/` | `main_menu` | Main menu screens |
| `gui/hud_pc/` | `hud` | In-game HUD elements |
| `gui/dev_menu_pc/` | `dev_menu` | Developer debug menus |

### Dependency chain
```
common_pc.uip
    ↑
menu_common_pc.uip
    ↑
ingame_menu_pc.uip   ← This is where MenuHints lives
```
(The `.uip` file has a `"dependencies": ["menu_common"]` field.)

---

## The `.gui` symbol system

Every UI element is a "symbol" — a `.gui` file. There are two kinds:

### 1. Symbol Root (`gui::ISymbolRoot`)
A self-contained, independently defined symbol. Has its own `symbol_uuid`, `symbol_size`, and `data_context_class`.

### 2. Symbol Reference (`gui::CSymbolRoot`)
A reference to another symbol by name. Uses the `sub_symbol` field (which must match an `Id` in the target `.gui` file). The reference is resolved at runtime using the `.uip` registry.

Example from `menuhints_pc.gui`:
```json
{
  "class": "gui::CSymbolRoot",
  "Id": "hint_info_popup",
  "sub_symbol": "hint_info_popup"
}
```
This tells the engine: "Place a `hint_info_popup` symbol here" — which maps to `hint_info_popup_PC.gui` via the `.uip` file.

---

## The action graph system

The `ILogicComponent` on any element contains a `CActionGraph` — a visual scripting system.

Nodes connect via:
- `"Next"` — runs the next node
- `"True"` / `"False"` — branch result

Each node has a `"type"` field and other parameters.

### How data binding works

Nodes use `"field_path"` to reference data:
- `"type": "DataContext"` — access the C++ data object bound to this element
- `"type": "GuiComponent"` — access a component on a specific GUI element by UUID

Example: when the user hovers over a hint item, the logic fires `CActionEmitGameDataEvent` with `msg_rtti: "GuiHintInfoEvent"` and `m_Seen: 1` and `m_Id: <hint id>`. The C++ code on the `MenuHintsController` receives this and marks the hint as seen.

### How buttons trigger game actions

From `menuhints_pc.gui`:
```json
{
  "type": "CActionOnButtonPress",
  "Next": 93,
  "field_path": {
    "type": "GuiComponent",
    "element": "default_buttons/group1/btn_leaderboard",
    "component_class": "gui::IButtonComponent"
  }
}
```
When the button is pressed, node 93 runs:
```json
{
  "type": "CActionEmitGameDataEvent",
  "m_Text1": {"value": "show_leaderboard"},
  "field_path": {"type": "DataContext", "class": "MenuHintsController"},
  "msg_rtti": "GuiGenericEvent"
}
```
The `MenuHintsController` C++ class receives a `GuiGenericEvent` with `m_Text1 = "show_leaderboard"` and acts on it.

**Key insight:** Buttons can only trigger game actions that the C++ controller already handles. You cannot add new behavior beyond what the controller responds to.

---

## The `gui::CScreenComponent`

Full-screen menus have a `gui::CScreenComponent` component. This marks the element as a top-level screen.

From `menuhints_pc.gui`:
```json
{
  "class": "gui::CScreenComponent",
  "HideScreenSound": "hud_hintsmenu_out",
  "ShowScreenSound": "hud_hintsmenu_in",
  "DependencyScreen0": "OptionsBackground",
  "UsePadCursor": "1",
  "DataClassName": "MenuHintsController"
}
```

- `ShowScreenSound` / `HideScreenSound` — audio events played on open/close
- `DependencyScreen0` — another screen that must be active alongside this one (the background)
- `UsePadCursor` — enables gamepad cursor navigation
- `DataClassName` — the C++ class that drives this screen's data

This component tells the engine "this is a top-level screen with its own lifecycle."

---

## Hints menu structure (full tree)

File: `gui/ingame_menu_pc/menuhints_pc.gui`
Data context class: `MenuHintsController`
Root size: 1920×1080 (full screen)

```
MenuHints (gui::ISymbolRoot)
├── hints_menu_template (group) — template/placeholder
├── title (group) — the menu title area
│   ├── text — title text element
│   └── tooltip_spacer — decorative separator
├── tabs_group_symbol → tabs_group_symbol.gui — tab selector at top
├── menu_hints_container_symbol → menu_hints_container_symbol_pc.gui
├── screen (group) — main content area
│   ├── popup_group (group) — contains the detail popup
│   │   └── hint_info_popup → hint_info_popup_pc.gui — detail view
│   ├── text — section title/label
│   ├── listbox (group) — scrollable list of hints
│   │   ├── viewport (group)
│   │   │   └── content (group)
│   │   │       ├── hint_info_symbol → hint_info_symbol_pc.gui — hint row
│   │   │       └── tutorial_name_selection_symbol → tutorial_name_selection_symbol_pc.gui
│   │   ├── gradient — visual fade gradient
│   │   └── top_gradient
│   ├── quest_scrolbar → quest_scrolbar.gui — scrollbar
│   └── menu_hints_bighint_symbol → menu_hints_bighint_symbol_pc.gui — large preview area
├── default_buttons (group) — bottom action buttons
│   ├── group1
│   │   ├── btn_back → legend_button.gui — Back button
│   │   └── btn_leaderboard → legend_button.gui — Leaderboard button
│   └── group2
├── permadeath_info_symbol → permadeath_info_symbol.gui
├── group3 — background layers
│   ├── pause_menu_vignette
│   ├── pause_menu_vignette2
│   └── bg
├── dark_edges → dark_edges.gui — screen edge vignette
└── blur_simple — background blur
```

---

## Hints data flow

1. Game C++ code creates a `MenuHintsController` object when the player opens the Hints menu
2. The controller reads hint definitions from `gameplayhints.scr` (compiled into game logic)
3. Each hint is represented as a `GuiHintInfoData` object with fields:
   - `m_Name` — display name (localized string key)
   - `m_Category` — category name (localized string key)
   - `m_Text` — body text (localized string key)
   - `m_Id` — hint identifier string
4. The list in the menu is populated from these objects
5. When the player hovers/selects a hint, the action graph fires events back to the controller
6. The detail popup (`hint_info_popup`) binds to the selected `GuiHintInfoData`

**Localization:** Text values use `&KeyName&` syntax. The actual text is in `.binsloc` files.

---

## Hint script structure (gameplayhints.scr)

Located at `scripts/tutorials/gameplayhints.scr`.

```javascript
import "inventorystuff.scr"

sub DefaultParams()
{
    Cooldown(120);
    ShowDelayTime(0.5);
    MaxQueueTime(5);
    MinDisplayTime(6.5);
    Priority(1);
}

sub main()
{
    CharsPerSecond(20);
    SetCategory("&MenuHints&");

    Hint("HintName")
    {
        Text("&LocalizationKey&");
        use DefaultParams();
        Trigger(EEvent_SomeGameEvent);
        Condition(ECnd_SomeCondition);
        ShowInMenu(true);   // whether hint appears in the Hints menu
        Priority(8);
        Cooldown(120);      // seconds before hint can show again
        MinDisplayTime(6.5);
    }
}
```

### Available events (partial list from file comments):
`EEvent_LowStamina`, `EEvent_LowHealth`, `EEvent_WeaponBroken`, `EEvent_ChaseStarted`,
`EEvent_NightStart`, `EEvent_EnteringDarkZone`, `EEvent_LockPickStart`,
`EEvent_ItemTaken_Medkit`, `EEvent_FuryActivated`, `EEvent_JoinedCoop`, etc.

### Key hint parameters:
- `Trigger(EEvent_X)` — game event that triggers this hint to show
- `Text("&Key&")` — localized text body
- `ShowInMenu(true/false)` — whether this hint appears in the browseable Hints menu list
- `Cooldown(seconds)` — minimum time between re-shows
- `Priority(n)` — higher = shown before lower priority hints
- `MinDisplayTime(seconds)` — minimum display duration
- `MaxQueueTime(seconds)` — how long hint waits in queue before being discarded
- `Condition(ECnd_X)` — only show if condition is true
- `NotCondition(ECnd_X)` — only show if condition is false
- `PrologueHint(true)` — marks as prologue-only
- `DisableShownTimes(n)` — number of times to show before auto-disabling

---

## Other hint-related files

| File | Role |
|------|------|
| `hints.scr` | Registers hint IDs: `HintId("&SurvivalHint_1&")`, etc. |
| `scripts/tutorials/loadinghints.scr` | Loading screen tip definitions |
| `scripts/tutorials/deathhints.scr` | Death screen tip definitions |
| `gui/hud_pc/gameplay_hint_widget_pc.gui` | In-HUD hint popup (during gameplay) |
| `gui/hud_pc/onboarding_hud_hint_widget_pc.gui` | Onboarding tutorial hint overlay |
| `gui/common_pc/fullscren_tutorial_widget_pc.gui` | Full-screen tutorial overlay |
| `gui/ingame_menu_pc/time_freezing_tutorial_text_menu_pc.gui` | Time-freeze tutorial overlay |

---

## Open questions

See `docs/open_questions.md` for a full list.

Key unknowns:
- Can a modder add entirely new screens (not just modify existing ones)?
- What events does `MenuHintsController` respond to beyond `show_leaderboard`?
- How does the game map `DataClassName` in `CScreenComponent` to a C++ object?
- How are menu tabs (the `tabs_group_symbol`) configured — are they in the `.gui` or the `.scr`?
