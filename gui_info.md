# Dying Light — .gui File Format Reference

**Game:** Dying Light: The Beast (Chrome Engine)
**Source:** Static analysis of 1016 .gui files + in-game validation
**Working directory:** `F:\.dltbUnpack\data0\`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Package System (.uip)](#2-package-system-uip)
3. [File Structure](#3-file-structure)
4. [Root Fields](#4-root-fields)
5. [Component Reference](#5-component-reference)
6. [Object Types](#6-object-types)
7. [Layout System](#7-layout-system)
8. [Z-Order](#8-z-order)
9. [Resolution Scaling](#9-resolution-scaling)
10. [Action Graph](#10-action-graph)
11. [Action Node Reference](#11-action-node-reference)
12. [Field Path Types](#12-field-path-types)
13. [Data Context & Controller Binding](#13-data-context--controller-binding)
14. [Screen System](#14-screen-system)
15. [Texture & Color System](#15-texture--color-system)
16. [Confirmed Modding Rules](#16-confirmed-modding-rules)
17. [Known Controllers](#17-known-controllers)
18. [Practical Patterns](#18-practical-patterns)

---

## 1. Overview

`.gui` files define all UI elements in Dying Light. Each file defines one **symbol** — a self-contained UI unit that can be a full screen, a widget, a button, a list row, or a reusable component.

**Format:** Single-line JSON (no newlines in the raw file — use a JSON formatter to read).

**Mod system:** `.pak` files are ZIP archives. Place files at `data2.pak` through `data7.pak`. Higher numbers override lower. File paths inside the ZIP must exactly match the `data0/` layout (e.g., `gui/ingame_menu_pc/menuhints_pc.gui`).

---

## 2. Package System (.uip)

Each GUI directory has a `.uip` package manifest. It acts as a registry for everything that package contains.

```json
{
  "dependencies": ["menu_common"],
  "symbols": {
    "MenuHints": "menuhints_pc.gui",
    "hint_info_popup": "hint_info_popup_PC.gui",
    "dev_btn": "dev_btn_PC.gui"
  },
  "atlases": ["ingame_menu_compressed_0_PC.atlas"],
  "colors": {
    "faction_pk": "2C5FFAFF"
  },
  "effects": {
    "blur": "blur.guieff"
  }
}
```

**Key fields:**
- `dependencies` — other packages this one depends on (resolved at load time)
- `symbols` — maps symbol name -> .gui filename. All `sub_symbol` references must be registered here.
- `atlases` — texture atlases to load (contains all textures referenced by `Texture` fields)
- `colors` — named color constants usable in `ImageColorMul` / `TextColorMul`
- `effects` — named shader effects usable in `IEffectComponent`

**The UIP does NOT control which C++ controller binds to a screen.** That is determined by the C++ game layer based on screen slot, not by anything in the .gui or .uip files.

### Dependency chain (ingame menus)

```
common_pc.uip
    ^
menu_common_pc.uip
    ^
ingame_menu_pc.uip   <-- MenuHints, all dev menus, inventory, map, etc.
```

---

## 3. File Structure

Every `.gui` file has this top-level shape:

```json
{
  "version": 1,
  "symbol_uuid": "{09cb1ea0-1ed5-43fb-a295-3a855e1e512c}",
  "symbol_size": "1920.000000 1080.000000",
  "data_context_class": "MenuHintsController",
  "class": "gui::ISymbolRoot",
  "uuid": "{09cb1ea0-1ed5-43fb-a295-3a855e1e512c}",
  "Id": "MenuHints",
  "components": [ ... ],
  "objects": [ ... ]
}
```

The root element is always `gui::ISymbolRoot`. Its `components` and `objects` arrays define the full element tree.

---

## 4. Root Fields

| Field | Description |
|-------|-------------|
| `version` | Always `1`. Do not change. |
| `symbol_uuid` | Unique ID for this symbol. Used internally for cross-references. **Does not need to match the original** when replacing a screen — engine accepts any valid UUID. |
| `symbol_size` | Design canvas size in pixels (`"1920.000000 1080.000000"`). Does not restrict where content renders — use `IResolutionScalerComponent` for correct mapping. |
| `data_context_class` | C++ class name for the data context. Used by the GUI engine to resolve `field_path` types. **Does not determine which controller the game binds to the screen** — see §13. |
| `class` | Always `gui::ISymbolRoot` at the root level. |
| `uuid` | Same value as `symbol_uuid` for the root element. |
| `Id` | Logical name of this symbol. Must match the key in the `.uip` symbols map if referenced via `sub_symbol`. |
| `components` | Array of components attached to the root element. |
| `objects` | Array of child elements (groups, sub-symbols). |

---

## 5. Component Reference

Components attach functionality to an element. Every element can have multiple components.

### `gui::IElementComponent`

Controls visibility, opacity, and data context inheritance.

```json
{
  "class": "gui::IElementComponent",
  "uuid": "{...}",
  "Visible": "1",
  "Opacity": "0.800000",
  "InheritDataContextFromParent": "0"
}
```

| Field | Values | Notes |
|-------|--------|-------|
| `Visible` | `"0"` / `"1"` | Can be toggled by action graph nodes at runtime |
| `Opacity` | `0.0` – `1.0` | Multiplicative with parent opacity |
| `InheritDataContextFromParent` | `"0"` / `"1"` | If `1`, uses parent's DataContext |
| `DataContext` | pointer | Set at runtime by `CActionSetRawPtr` to bind a data object |
| `DesignTimeOnly` | `"1"` | Element only visible in editor, not at runtime |

---

### `gui::ILayoutComponent`

Controls size and position. The most important component for layout.

```json
{
  "class": "gui::ILayoutComponent",
  "uuid": "{...}",
  "Width": "400.000000",
  "Height": "60.000000",
  "MarginTop": "200.000000",
  "MarginLeft": "860.000000",
  "MarginAnchorX": "Center",
  "MarginAnchorY": "Center",
  "SizeModeX": "Fixed",
  "SizeModeY": "AutoSizeToElements"
}
```

| Field | Values | Notes |
|-------|--------|-------|
| `Width` / `Height` | float pixels | Design-space pixels (scaled by `IResolutionScalerComponent`) |
| `MarginLeft` / `MarginTop` | float pixels | Offset from anchor point. Can be negative. |
| `MarginRight` / `MarginBottom` | float pixels | Used with anchor combinations |
| `MarginAnchorX` | `"Center"`, `"Right"` | Positions element relative to parent center/right. Default = left. |
| `MarginAnchorY` | `"Center"`, `"Bottom"` | Positions element relative to parent center/bottom. Default = top. |
| `SizeModeX` / `SizeModeY` | see below | How element sizes itself |
| `LayoutScale` | `"x y z w"` | Scales the element visually |

**SizeMode values:**

| Value | Behavior |
|-------|----------|
| `"Fixed"` | Width/Height are exact (default when Width/Height specified) |
| `"Stretch"` | Fills parent size in that axis. Width/Height ignored. |
| `"AutoSizeToElements"` | Sizes to fit child objects |
| `"AutoSizeToComponents"` | Sizes to fit child components (text, images) |

---

### `gui::ITextComponent`

Renders text.

```json
{
  "class": "gui::ITextComponent",
  "uuid": "{...}",
  "TextValue": "CUSTOM MENU",
  "FontSize": "48.000000",
  "AlignmentX": "Center",
  "AlignmentY": "Center",
  "TextColorMul": "FFFFFFFF",
  "Multiline": "0",
  "Uppercase": "1"
}
```

| Field | Notes |
|-------|-------|
| `TextValue` | Literal string OR localization key `"&KeyName&"`. Use literal strings for custom text (always English). |
| `FontSize` | Float. Design-space pixels. |
| `AlignmentX` | `"Left"`, `"Center"`, `"Right"` |
| `AlignmentY` | `"Top"`, `"Center"`, `"Bottom"` |
| `TextColorMul` | RGBA hex `"FFFFFFFF"` or named color from `.uip` colors map |
| `Multiline` | `"0"` / `"1"` |
| `Uppercase` | `"0"` / `"1"` |
| `_preset_` / `_StylePreset_` | Named text style preset — sets font, size, color together |

---

### `gui::IImageComponent`

Renders a texture or solid color rectangle.

```json
{
  "class": "gui::IImageComponent",
  "uuid": "{...}",
  "Texture": "bg_box",
  "ImageColorMul": "000000C0"
}
```

| Field | Notes |
|-------|-------|
| `Texture` | Texture name from the atlas. If omitted, element renders as a solid color rectangle. |
| `ImageColorMul` | RGBA hex tint. Multiplied with the texture. On a solid-color (no texture) element, this IS the color. Format: `RRGGBBAA`. |

**To make a solid dark rectangle (background panel):** omit `Texture`, set `ImageColorMul` to desired RGBA. Example: `"000000C0"` = black at 75% opacity.

**Known useful textures:**

| Texture name | What it is |
|---|---|
| `1px_frame` | 1-pixel border/outline texture. Scales as a **border**, not a fill. |
| `border` | Thin line, used for separators |
| `bg_box` | Wooden/textured panel — has baked-in texture, color-mul tints it but pattern shows |
| `bg_main` | Full-screen background texture |
| `pause_menu_vignette` | Edge vignette gradient |
| `tooltip_bg_pattern` | Repeating background pattern |
| `tooltip_spacer` | Decorative spacer element |
| *(no Texture field)* | **Solid color fill** — correct way to make opaque colored rectangles |

---

### `gui::IButtonComponent`

Makes an element interactive (clickable, hoverable, focusable). Presence of this component is what makes an element a button. No additional fields required.

```json
{
  "class": "gui::IButtonComponent",
  "uuid": "{...}"
}
```

Fields readable by action nodes: `Pressed`, `Hovered`, `Focused`, `Disabled`.

---

### `gui::CScreenComponent`

Marks an element as a top-level full-screen menu. Required for screens that the game's screen stack can push/pop.

```json
{
  "class": "gui::CScreenComponent",
  "uuid": "{...}",
  "DataClassName": "MenuHintsController",
  "UsePadCursor": "1",
  "ShowScreenSound": "hud_hintsmenu_in",
  "HideScreenSound": "hud_hintsmenu_out",
  "DependencyScreen0": "OptionsBackground"
}
```

| Field | Notes |
|-------|-------|
| `DataClassName` | C++ class name — must match `data_context_class` on the root. Used for data binding lookup. |
| `UsePadCursor` | **`"1"` required for cursor-based mouse/gamepad interaction.** Without it, buttons are display-only and cannot be clicked. Original dev menus were missing this and needed to be patched. |
| `ShowScreenSound` / `HideScreenSound` | Audio event names played on screen open/close |
| `DependencyScreen0` | Another screen that must be active simultaneously (e.g., `"OptionsBackground"` for the blurred background) |

---

### `gui::ILogicComponent`

Contains the action graph — the visual scripting system that drives all interactivity.

```json
{
  "class": "gui::ILogicComponent",
  "uuid": "{...}",
  "data": [
    {
      "type": "CVariablesComponentSymbolData",
      "dyn_fields": [
        {"name": "my_var", "type": "int", "value": "0"}
      ]
    },
    {
      "type": "CActionGraphSymbolData",
      "uuid": "{...}",
      "hash": "",
      "nodes": [ ... ]
    }
  ],
  "dyn_fields": [ ... ]
}
```

**Critical rule: `ILogicComponent` must be on the ROOT symbol** (direct component of `gui::ISymbolRoot`) for `CActionOnButtonPress` to work. Placing it inside a child group makes button monitoring fail silently.

**`CVariablesComponentSymbolData`** declares local variables for use by action nodes (accessed via `"type": "GuiComponent"` field path referencing the `ILogicComponent` itself).

---

### `gui::IResolutionScalerComponent`

**Required for correct resolution-independent rendering.** Maps the design canvas (1920×1080) to the actual screen resolution. Without this, all pixel coordinates are raw physical-screen pixels anchored to the top-left corner.

```json
{
  "class": "gui::IResolutionScalerComponent",
  "uuid": "{...}"
}
```

No configuration fields. Its presence alone activates resolution scaling.

**Every full-screen menu in the vanilla game has this component.** It was the missing piece in early custom menu attempts that caused content to appear at the physical top-left of ultrawide monitors instead of centered.

---

### `gui::IWrapPanelComponent`

Controls auto-layout flow direction for children.

```json
{
  "class": "gui::IWrapPanelComponent",
  "uuid": "{...}",
  "Wrap": "0",
  "Orientation": "Vertical"
}
```

| Field | Values |
|-------|--------|
| `Orientation` | `"Vertical"`, `"Horizontal"` |
| `Wrap` | `"0"` / `"1"` |

---

### `gui::IScrollBarComponent`

Scroll bar. Fields accessible from action nodes: `Value`, `DocumentSize`, `ViewportSize`.

---

### `gui::IGridBoxComponent`

Grid layout container. Used by dev menus (e.g., `MenuDevPlayerCheats`) with `CElementTypePool` to display dynamically-populated grids of sliders or controls.

---

## 6. Object Types

Objects are the children in the `objects` array — the actual elements of the UI tree.

### `gui::IGroup`

A container element. No visual of its own. Has `components` and `objects`.

```json
{
  "class": "gui::IGroup",
  "uuid": "{...}",
  "Id": "my_group",
  "components": [ ... ],
  "objects": [ ... ]
}
```

`Id` is optional but needed if you want to reference this element from action nodes.

---

### `gui::CSymbolRoot`

A reference to another symbol by name. Loads and places a separate `.gui` file at this position in the tree.

```json
{
  "class": "gui::CSymbolRoot",
  "uuid": "{...}",
  "Id": "hint_info_popup",
  "sub_symbol": "hint_info_popup",
  "components": [
    {
      "target_component_uuid": "{uuid-of-component-in-target-gui}",
      "class": "gui::ILayoutComponent",
      "MarginLeft": "0.000000"
    }
  ]
}
```

| Field | Notes |
|-------|-------|
| `sub_symbol` | Must match a key in the `.uip` symbols map. Resolved to a filename at runtime. |
| `components` | Override components on the referenced symbol. `target_component_uuid` must match a real component UUID in the target `.gui` file. |

---

## 7. Layout System

The layout system positions elements within their parent's coordinate space.

### Positioning

Default anchor is **top-left**. `MarginLeft` and `MarginTop` are offsets from the anchor point.

**Changing the anchor:**
- `MarginAnchorX: "Center"` — element's left edge is placed at parent center X. Use negative `MarginLeft` to move left of center.
- `MarginAnchorX: "Right"` — element's left edge is placed at parent right edge.
- `MarginAnchorY: "Center"` — element's top edge is placed at parent center Y.
- `MarginAnchorY: "Bottom"` — element's top edge is placed at parent bottom edge.

**Resolution-independent centering (confirmed from yesnodialog, crosshair):**

```json
{
  "MarginAnchorX": "Center",
  "MarginAnchorY": "Center",
  "MarginLeft": "0.000000",
  "MarginTop": "0.000000"
}
```

This places the element's top-left at the parent's center. To truly center a 400×60 element: set `MarginLeft: -200`, `MarginTop: -30`.

**Alternative — manual math for a 1920×1080 design space:**
```
Center a 900×640 panel:
  MarginLeft = (1920 - 900) / 2 = 510
  MarginTop  = (1080 - 640) / 2 = 220
```
This works correctly at any resolution when `IResolutionScalerComponent` is present.

### Full-screen dark overlay trick (used by yesnodialog)

```json
{
  "Width": "12000.000000",
  "Height": "6000.000000",
  "MarginAnchorX": "Center",
  "MarginAnchorY": "Center"
}
```
An oversized element centered on screen with no texture and a dark `ImageColorMul` creates a resolution-independent full-screen overlay. The oversized dimensions ensure it covers the screen regardless of aspect ratio (including ultrawide).

---

## 8. Z-Order

**Objects render in reverse array order.** The **first** entry in the `objects[]` array is drawn **on top** (highest layer). The **last** entry is drawn **behind** everything else.

To put a background panel behind content:
```json
"objects": [
  { "Id": "title",   ... },   // rendered ON TOP
  { "Id": "buttons", ... },   // rendered second
  { "Id": "bg_panel", ... }   // rendered BEHIND everything (last)
]
```

This is the opposite of HTML/CSS and was a source of bugs where dark backgrounds obscured text.

---

## 9. Resolution Scaling

Without `gui::IResolutionScalerComponent`: coordinates are raw physical-screen pixels from the top-left. On a 3440×1440 monitor, `MarginLeft=960` places the element at 960 physical pixels from the left edge of the screen, not at screen center.

With `gui::IResolutionScalerComponent`: the engine maps the 1920×1080 design space to the full physical screen. `MarginLeft=960` means "960 design-space pixels from the left of the 1920-wide canvas" — which is the center, at any resolution.

**`MarginAnchorX/Y="Center"` anchors relative to the scaled parent bounds**, so it works correctly at any resolution when `IResolutionScalerComponent` is active.

---

## 10. Action Graph

Every interactive element uses action graph nodes defined inside `gui::ILogicComponent`. Nodes chain together via `"Next"`, `"True"`, `"False"` links (by node `id`).

### Basic structure

```json
{
  "type": "CActionOnButtonPress",
  "id": 1,
  "Next": 2,
  "field_path": {
    "type": "GuiComponent",
    "element": "btn_back",
    "element_uuid": "{uuid-of-GROUP-containing-IButtonComponent}",
    "component_class": "gui::IButtonComponent"
  }
}
```

```json
{
  "type": "CActionEmitGameDataEvent",
  "id": 2,
  "msg_rtti": "gui::CBackButtonAction",
  "field_path": {
    "type": "DataContext",
    "class": "MenuHintsController"
  }
}
```

### Node execution

- Nodes do not run continuously — they are **event-driven**. Trigger nodes (`CActionOnButtonPress`, `CActionOnFieldChanged`, `CActionGameDataEvent`) start a chain when their condition fires.
- Chains run synchronously to completion. There is no looping construct.
- Multiple trigger nodes in one `ILogicComponent` run independently (each starts its own chain).

---

## 11. Action Node Reference

### Trigger nodes (start a chain)

| Node type | Trigger condition |
|-----------|------------------|
| `CActionOnButtonPress` | Button with matching `field_path` is pressed |
| `CActionOnFieldChanged` | A data field changes value |
| `CActionGameDataEvent` | A specific `msg_rtti` event is received by this element |
| `CActionOnInitialize` | Fires once when element initializes |
| `CActionOnTimerFinished` | Timer countdown completes |
| `CActionOnListboxButtonPress` | Item in a listbox is pressed |

---

### Data read nodes

| Node type | Reads | Output property |
|-----------|-------|-----------------|
| `CActionGetBool` | bool field | `Value` |
| `CActionGetInt` | int field | `Value` |
| `CActionGetFloat` | float field | `Value` |
| `CActionGetString` | string field | `Value` |
| `CActionGetAnimation` | animation by uuid | `Animation` |
| `CActionGetRawPtr` | raw C++ pointer | `Value` |
| `CActionGetRawPtrVector` | vector of C++ pointers | `Value` |
| `CActionGetEnumEHintsMenuTabs` | enum field (specific type) | `Value` |

---

### Data write nodes

| Node type | Writes |
|-----------|--------|
| `CActionSetBool` | bool field |
| `CActionSetInt` | int field |
| `CActionSetFloat` | float field |
| `CActionSetString` | string field |
| `CActionSetRawPtr` | raw pointer field |
| `CActionSetBoolFieldFromPtr` | bool field on a C++ object reached via pointer |

`Value` can be a literal `{"value": "1"}` or piped from another node `{"src_node": N, "src_prop": "Value"}`.

---

### Logic/math nodes

| Node type | Function |
|-----------|----------|
| `CActionBranch` | If `Condition` true -> `True` node, else -> `False` node |
| `CActionNot` | Negates a bool |
| `CActionCompareFloat` | Compare two floats (`Greater`, `Less`, `Equal`) -> `Result` bool |
| `CActionCompareInt` | Compare two ints |
| `CActionAddInt` | Add two ints -> `Result` |
| `CActionSubtractFloat` | Subtract two floats -> `Result` |
| `CActionMultiplyFloat` | Multiply two floats -> `Result` |
| `CActionDivideFloat` | Divide two floats -> `Result` |
| `CActionModuloInt` | Int modulo -> `Result` |
| `CActionSelectUsingBool` | Return `Value on True` or `Value on False` based on bool |
| `CActionIsNotNull` | Check if pointer is non-null -> `Result` bool |

---

### Pointer nodes

| Node type | Function |
|-----------|----------|
| `CActionCastTo` | Try to cast a pointer to a specific type. `Success` / `Failed` branches. |
| `CActionVectorGetElementAt` | Get element at `Index` from a vector -> `Element` |
| `CActionVectorIsNotEmpty` | Check if vector has elements -> `NotEmpty` bool |

---

### Event/action nodes

| Node type | Function |
|-----------|----------|
| `CActionEmitGameDataEvent` | Send an event to C++ code (the controller or a data object). Core of all interactivity. |
| `CActionPlayAnim` | Play a GUI animation |
| `CActionStartTimer` / `CActionStopTimer` | Control timers |
| `CActionSetFocus` | Move keyboard/gamepad focus to an element |
| `CActionDebugLogText` | Print debug text (dev builds only, likely no-op in retail) |
| `CActionSwitchEnumEHintsMenuTabs` | Switch/case on an enum value with named branches |

---

### `CActionEmitGameDataEvent` — full field reference

This is the most important node. It sends an event to a C++ handler.

```json
{
  "type": "CActionEmitGameDataEvent",
  "id": 2,
  "msg_rtti": "GuiGenericEvent",
  "m_Text1": {"value": "cat"},
  "m_Text2": {"value": ""},
  "m_Int1": {"value": "1"},
  "m_Float1": {"value": "0.000000"},
  "m_Bool1": {"value": "0"},
  "m_Action": {"value": "BtnTeleportPlayer"},
  "m_Index": {"value": "0"},
  "field_path": {
    "type": "DataContext",
    "class": "MenuHintsController"
  }
}
```

| Field | Notes |
|-------|-------|
| `msg_rtti` | The C++ event class name. Determines which handler receives it. Must be a real class registered in the binary. |
| `m_Text1` / `m_Text2` | String parameters. `m_Text1` is used as the action selector by `GuiGenericEvent` handlers. |
| `m_Int1` | Integer parameter. Used for direction (`-1`/`+1`), index, or flags. |
| `m_Float1` | Float parameter. |
| `m_Bool1` | Bool parameter. |
| `m_Action` | String parameter specific to dedicated event types (e.g., `MenuDevStoryInvokeAction`). Different from `m_Text1`. |
| `m_Index` | Item index parameter (used by dev controller actions). |
| `field_path` | Which object receives the event — either the `DataContext` (controller) or a specific GUI component. |

**Silent failure:** If the `DataContext` is null (controller not bound), all `CActionEmitGameDataEvent` nodes that route to DataContext produce no effect — including `gui::CBackButtonAction`. The back button not working is the diagnostic indicator that DataContext is unbound.

---

## 12. Field Path Types

Field paths tell action nodes WHERE to read/write data.

### DataContext path

```json
"field_path": {
  "type": "DataContext",
  "class": "MenuHintsController",
  "field": "m_CurrCategory"
}
```

Reads/writes a field on the C++ controller object bound to this screen. `class` must match the `data_context_class` of the symbol.

For `CActionEmitGameDataEvent`, omit `field` — the event is sent to the controller itself.

### GuiComponent path

```json
"field_path": {
  "type": "GuiComponent",
  "element": "btn_back",
  "element_uuid": "{uuid-of-the-group}",
  "component_class": "gui::IButtonComponent",
  "field": "Pressed"
}
```

| Field | Notes |
|-------|-------|
| `element` | Slash-separated path from root to element by `Id`. Can be `"."` for the current element. |
| `element_uuid` | UUID of the target element. More reliable than path — the engine resolves by UUID first. **Must be the UUID of the GROUP (IGroup), not the UUID of the component inside it.** |
| `component_class` | Which component type on that element to access. |
| `field` | Which property on that component to read/write. |

**Critical rule:** In `CActionOnButtonPress`, `element_uuid` must be the UUID of the **`gui::IGroup`** that contains the `IButtonComponent`, not the UUID of the `IButtonComponent` itself. The engine resolves: "find element by UUID → look inside it for the named `component_class`".

### DataContext path with datasource traversal

```json
"field_path": {
  "type": "DataContext",
  "class": "MenuHintsController",
  "datasource_path": "m_Tabs",
  "field": "m_SelectedIdx"
}
```

Navigates through a sub-object (`m_Tabs`) to access a field on it.

---

## 13. Data Context & Controller Binding

### How it works

When the game opens a screen, the **C++ game layer** determines which controller class to instantiate based on the **screen slot**, not based on anything in the `.gui` file.

The `data_context_class` field on the root and the `DataClassName` on `CScreenComponent` are **type hints** for the GUI engine's field-lookup system only. They tell the engine "when this symbol accesses DataContext fields, assume the object is of this type."

### Controller binding is fixed per slot

The Hints menu slot **always** binds `MenuHintsController`. Specifying `data_context_class: "MenuDevStoryInvokeController"` does not cause the engine to instantiate that controller. It results in a null DataContext, and all DataContext-routed actions silently fail.

The same is true for dev menus: `MenuDevStoryInvokeController` only binds when the C++ game layer pushes that specific screen. The original `.gui` file must also include the required sub-symbols (listbox, filter, submenu etc.) or the controller fails to initialize its internal state.

### What determines which controller binds

Unknown from .gui analysis alone — this is pure C++ game logic. Based on testing:
- `MenuHints` slot -> `MenuHintsController` (confirmed)
- `MenuDevList` slot -> `MenuDevListController` (confirmed — tested via patched devlist.gui as hints replacement)
- Changing `Id`, `symbol_uuid`, or `data_context_class` has **no effect** on which controller binds

### How to use a different controller

Deploy the original dev `.gui` file (with `UsePadCursor: "1"` patched in) as a replacement for `menuhints_pc.gui`. The hints slot then loads a full dev menu including all its required sub-symbols, which allows the correct controller to bind.

---

## 14. Screen System

### Screen stack

The game maintains a stack of screens. `CBackButtonAction` pops the current screen. Pushing a new screen is handled by C++ (triggered by controller logic after receiving a GuiGenericEvent), not directly from GUI action nodes — there is no `CActionPushScreen` node type.

### Screen lifecycle events

These `msg_rtti` types are received BY a screen (not sent by it). Listen with `CActionGameDataEvent`:

| msg_rtti | When it fires |
|----------|--------------|
| `gui::CScreenAboutToShow` | Screen is about to become visible |
| `gui::CScreenAboutToHide` | Screen is about to be hidden |
| `gui::CScreenFullyShown` | Screen is fully visible (animation complete) |
| `gui::CScreenFullyHidden` | Screen is fully hidden |
| `gui::CHoverChangedEvent` | Mouse cursor hover state changed |

### `DependencyScreen0`

Specifies another screen (by `Id`) that must be active alongside this one. For example, `menuhints_pc.gui` depends on `"OptionsBackground"` which provides the blurred world background. Without it, the menu appears over the raw game world.

---

## 15. Texture & Color System

### Texture names

Texture names reference entries in `.atlas` files (texture atlases). The atlases are defined in the `.uip`. Textures not in the package's atlases cannot be used.

### Color format

All color values use RGBA hex: `"RRGGBBAA"` (8 characters, uppercase or lowercase).

```
"000000FF" = solid black
"000000C0" = black at 75% opacity (yesnodialog overlay)
"000000D0" = black at 82% opacity
"0D0D0DE6" = very dark gray at 90% opacity
"FFFFFF55" = white at 33% opacity (separator lines)
"FFFFFF44" = white at 27% opacity
```

`ImageColorMul` is multiplicative over the texture. On elements with no `Texture` field it is the solid fill color.

### Named colors

The `.uip` can define named color constants usable in place of hex values:

```
"text_white", "hud_white", "white_color", "menu_red", "menu_yellow"
"text_bg_black", "text_bg_fail"
"faction_pk", "faction_scav", "faction_survivor"
```

---

## 16. Confirmed Modding Rules

All rules confirmed through in-game testing.

| # | Rule | Source |
|---|------|--------|
| 1 | `ILogicComponent` must be at the **ROOT symbol level** for `CActionOnButtonPress` to work. Placing it inside a child group silently breaks button monitoring. | Variant C fail / Variant D pass |
| 2 | `element_uuid` in `CActionOnButtonPress` must be the **GROUP's UUID**, not the `IButtonComponent`'s UUID. Engine resolves: find element by UUID -> look inside for `component_class`. | Variant C fail / Variant D pass |
| 3 | `UsePadCursor: "1"` on `CScreenComponent` is **required for cursor-based button interaction**. Without it, buttons are display-only. All vanilla dev menus lacked this. | Dev menu testing |
| 4 | `gui::CBackButtonAction` works correctly from custom `.gui` files when DataContext is bound. | Variant D pass |
| 5 | `symbol_uuid` does **not** need to match the original. Engine does not validate it when loading a replacement. | Reference pak analysis |
| 6 | `CActionEmitGameDataEvent` **silently fails** when DataContext is null. No error, no crash — events are simply dropped. If BACK doesn't work, DataContext is unbound. | StoryInvoke wrapper test |
| 7 | `data_context_class` in the `.gui` file does **not** control which C++ controller the game binds. The C++ game layer determines the controller per screen slot. | Phase 6 investigation |
| 8 | `Id` in the root does **not** affect controller binding. Changing `Id` from `"MenuHints"` to `"MenuDevInvoke"` had zero effect. | storyinvoke_wrapper_v2 test |
| 9 | `gui::IResolutionScalerComponent` is **required** for resolution-independent positioning. Without it, coordinates are raw physical-screen pixels anchored to top-left. | Custom menu v1-v4 failures |
| 10 | Objects render in **reverse array order** — first entry = top layer, last entry = bottom/behind. | Custom menu v4 z-order bug |
| 11 | `MarginAnchorX/Y="Center"` works **only when `IResolutionScalerComponent` is present**. Without it, anchors are effectively ignored at the root level. | Custom menu centering investigation |
| 12 | `bg_box` texture has a baked-in wooden/textured appearance. Color-mul tints it but the texture pattern shows through. Use no-texture + `ImageColorMul` for solid backgrounds. | v2/v3 background bugs |
| 13 | `1px_frame` renders as a **1-pixel border/outline**, not a fill. It does not scale as a solid rectangle. | v3 background bug |
| 14 | The `.uip` file is a pure path registry. It maps symbol names to `.gui` filenames only. It contains no controller class information and does not affect controller binding. | UIP analysis |
| 15 | `debugconf.scr` / `debugconfdefault.scr` effect on dev controller binding is **unconfirmed**. Testing with and without it produced identical results. | Phase 6 testing |

---

## 17. Known Controllers

### MenuHintsController

**Screen slot:** Hints menu (always bound here regardless of `.gui` file content)

**Events it accepts (GUI -> controller):**

| msg_rtti | Parameters | Effect |
|----------|-----------|--------|
| `gui::CBackButtonAction` | none | Close menu |
| `GuiGenericEvent` | `m_Text1="cat"`, `m_Int1=1` | Next category |
| `GuiGenericEvent` | `m_Text1="cat"`, `m_Int1=-1` | Previous category |
| `GuiGenericEvent` | `m_Text1="show_leaderboard"` | Open leaderboard (online only) |

**Data fields it exposes (controller -> GUI):**

| Field | Type | Description |
|-------|------|-------------|
| `m_BtnOnlineEnabled` | bool | Online features available? |
| `m_ShowLeaderboards` | bool | Show leaderboard button? |
| `m_CurrCategory` | string | Current category name |
| `m_CurrentTab` | enum `EHintsMenuTabs` | Current active tab |
| `m_SelectedIdx` | int (watched) | Selected hint index |
| `m_Hints` | ptr vector`<GuiHintInfoData>` | All hint data objects |
| `m_Tabs` | ptr`<GuiGameplayTabs>` | Tab configuration |

**Per-hint data (GuiHintInfoData):**
`m_Id`, `m_Name`, `m_Category`, `m_Text`, `m_New`, `m_Selected`

---

### MenuDevListController

**Purpose:** Navigation hub — shows a list of dev screen names, navigates to selected one.

**Events accepted:**

| msg_rtti | Parameters | Effect |
|----------|-----------|--------|
| `GuiGenericEvent` | `m_Text1=""` (empty), `m_Int1=selectedIndex` | Item selected — C++ navigates to that screen |
| `gui::CBackButtonAction` | none | Close menu |

**Data fields:** `m_ScreenNames` (watched) — populates the listbox.

---

### MenuDevStoryInvokeController

**Purpose:** Teleport player, invoke story events, rollback campaign.

**Events accepted:**

| msg_rtti | `m_Action` | Effect |
|----------|-----------|--------|
| `MenuDevStoryInvokeAction` | `"BtnTeleportPlayer"` | Teleport to selected location |
| `MenuDevStoryInvokeAction` | `"BtnAction"` | Invoke selected story event |
| `MenuDevStoryInvokeAction` | `"BtnRollbakcCampaing"` | Rollback campaign (note: typo in original) |
| `MenuDevStoryInvokeAction` | `"ChangeShowInvokesFromOtherMaps"` | Toggle cross-map filter |
| `MenuDevStoryInvokeAction` | `"BtnToggleDialogs"` | Toggle dialog mode |
| `gui::CBackButtonAction` | — | Close menu |

**Note:** Uses `m_Action` parameter, not `m_Text1`. Different pattern from `GuiGenericEvent`.

**Requires sub-symbols to bind:** `DevInvokeSubmenu`, `dev_listbox`, `filter_text_symbol`, `dev_btn_nofocus`, `dev_btn`, `menu_dev_name`. Without all of these, DataContext is null and all actions fail silently.

---

### MenuDevPlayerProgressController

**Events accepted:**

| msg_rtti | `m_Text1` | Effect |
|----------|----------|--------|
| `GuiGenericEvent` | `"add"` | Add selected XP/skill |
| `GuiGenericEvent` | `"remove"` | Remove selected XP/skill |
| `gui::CBackButtonAction` | — | Close |

**Data fields:** `m_AllSkills`, `m_Variables`, `m_XpTypes`, `m_FilterData`

---

### MenuDevItemsController

**Events accepted:**

| msg_rtti | Effect |
|----------|--------|
| `MDevItemsAction` | Give/manage selected item |
| `gui::CBackButtonAction` | Close |

**Data fields:** `m_ItemsList`, `m_FilterData`, `m_ColorOption`

---

### MenuDevPlayerCheatsController

**Events accepted:** `gui::CBackButtonAction` only.
All interaction is via sliders bound directly to the DataContext. No discrete button events.

**Uses:** `IGridBoxComponent` with `CElementTypePool` (`m_DataClass: "gui::CMenuSliderControl"`) to render dynamic slider grids.

---

### SpawnEntityElementController

**Purpose:** Entity/prefab spawner. `sub_symbol: "SpawnEntity"` registered in the UIP.
**Events:** Uses a listbox + text input. Full API not yet analyzed.

---

## 18. Practical Patterns

### Minimal working full-screen menu

```json
{
  "version": 1,
  "symbol_uuid": "{your-uuid-here}",
  "symbol_size": "1920.000000 1080.000000",
  "data_context_class": "MenuHintsController",
  "class": "gui::ISymbolRoot",
  "uuid": "{your-uuid-here}",
  "Id": "MenuHints",
  "components": [
    {"class": "gui::IElementComponent", "uuid": "{uuid-1}"},
    {"class": "gui::ILayoutComponent",  "uuid": "{uuid-2}", "Height": "1080.000000", "Width": "1920.000000"},
    {"class": "gui::IResolutionScalerComponent", "uuid": "{uuid-3}"},
    {"class": "gui::CScreenComponent",  "uuid": "{uuid-4}",
     "DataClassName": "MenuHintsController", "UsePadCursor": "1",
     "ShowScreenSound": "hud_hintsmenu_in", "HideScreenSound": "hud_hintsmenu_out",
     "DependencyScreen0": "OptionsBackground"},
    {"class": "gui::ILogicComponent",   "uuid": "{uuid-5}", "data": [
      {"type": "CActionGraphSymbolData", "uuid": "{uuid-6}", "hash": "", "nodes": []}
    ]}
  ],
  "objects": []
}
```

---

### Working button (confirmed pattern)

```json
{
  "class": "gui::IGroup",
  "uuid": "{GROUP-UUID}",
  "Id": "btn_close",
  "components": [
    {"class": "gui::IElementComponent", "uuid": "{uuid-a}"},
    {"class": "gui::ILayoutComponent",  "uuid": "{uuid-b}", "Width": "200.000000", "Height": "60.000000",
     "MarginAnchorX": "Center", "MarginAnchorY": "Center", "MarginTop": "200.000000"},
    {"class": "gui::IButtonComponent",  "uuid": "{uuid-c}"},
    {"class": "gui::ITextComponent",    "uuid": "{uuid-d}", "TextValue": "CLOSE", "FontSize": "28.000000",
     "AlignmentX": "Center", "AlignmentY": "Center"}
  ],
  "objects": []
}
```

And in the ROOT `ILogicComponent` nodes:

```json
{"type": "CActionOnButtonPress", "id": 1, "Next": 2,
 "field_path": {"type": "GuiComponent", "element": "btn_close",
   "element_uuid": "{GROUP-UUID}", "component_class": "gui::IButtonComponent"}},
{"type": "CActionEmitGameDataEvent", "id": 2,
 "msg_rtti": "gui::CBackButtonAction",
 "field_path": {"type": "DataContext", "class": "MenuHintsController"}}
```

---

### Solid centered dark panel (confirmed pattern)

```json
{
  "class": "gui::IGroup",
  "uuid": "{uuid-panel}",
  "Id": "panel_bg",
  "components": [
    {"class": "gui::IElementComponent", "uuid": "{uuid-a}"},
    {"class": "gui::ILayoutComponent",  "uuid": "{uuid-b}",
     "Width": "900.000000", "Height": "640.000000",
     "MarginLeft": "510.000000", "MarginTop": "220.000000"},
    {"class": "gui::IImageComponent",   "uuid": "{uuid-c}", "ImageColorMul": "0D0D0DE6"}
  ],
  "objects": []
}
```

Place this as the **last** entry in `objects[]` so it renders behind all other content.

---

### Live DataContext field -> text display

```json
{"type": "CActionOnFieldChanged", "id": 7, "Next": 9,
 "field_path": {"type": "DataContext", "class": "MenuHintsController", "field": "m_CurrCategory"}},
{"type": "CActionGetString", "id": 8,
 "field_path": {"type": "DataContext", "class": "MenuHintsController", "field": "m_CurrCategory"}},
{"type": "CActionSetString", "id": 9,
 "Value": {"src_node": 8, "src_prop": "Value"},
 "field_path": {"type": "GuiComponent", "element": "cat_display",
   "element_uuid": "{TEXT-GROUP-UUID}", "component_class": "gui::ITextComponent", "field": "TextValue"}}
```

Note: `CActionOnFieldChanged` only fires on **changes**. The initial value won't display until the field changes at least once (e.g., press NEXT category to force an update).

---

### UUID generation

```python
import uuid
for i in range(10):
    print("{" + str(uuid.uuid4()) + "}")
```

Format must be: `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}` (lowercase hex, with braces).
All UUIDs within one `.gui` file must be unique. UUIDs across different `.gui` files do not need to be unique.
