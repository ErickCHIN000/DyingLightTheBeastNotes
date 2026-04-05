# Format Notes — .gui Files

Deep-dive reference for the `.gui` file format used in Dying Light's UI system.

---

## Overview

A `.gui` file defines a single "symbol" — a UI element that can be a full screen,
a widget, a button, a list item, or a reusable component.

Files are **single-line JSON** (no newlines in raw form). Use a JSON formatter to read them.

---

## Top-level fields

```json
{
  "version": 1,
  "symbol_uuid": "{09cb1ea0-1ed5-43fb-a295-3a855e1e512c}",
  "symbol_size": "1920.000000 1080.000000",
  "data_context_class": "MenuHintsController",
  "class": "gui::ISymbolRoot",
  "uuid": "{09cb1ea0-1ed5-43fb-a295-3a855e1e512c}",
  "Id": "MenuHints",
  "components": [...],
  "objects": [...]
}
```

| Field | Notes |
|-------|-------|
| `version` | Always 1 — don't change |
| `symbol_uuid` | Unique ID for this symbol — don't change, used internally |
| `symbol_size` | Width and height in pixels (at 1080p baseline) |
| `data_context_class` | C++ class name that provides data. Cannot be invented — must be an existing class. |
| `class` | Always `gui::ISymbolRoot` for the root element |
| `uuid` | Same as `symbol_uuid` for root — don't change |
| `Id` | Logical name. Must match the symbol name in the `.uip` file. |
| `components` | Array of component objects attached to the root |
| `objects` | Array of child elements |

---

## Component reference

### `gui::IElementComponent`

Controls visibility, opacity, rotation, and data context binding.

```json
{
  "class": "gui::IElementComponent",
  "uuid": "{...}",
  "Visible": "1",
  "Opacity": "0.800000",
  "RenderRotation": "0.000000 0.000000 0.000000 -1.000000",
  "InheritDataContextFromParent": "0"
}
```

Fields you can safely change:
- `Visible` — `"0"` or `"1"`
- `Opacity` — float 0.0–1.0

### `gui::ILayoutComponent`

Controls size and position.

```json
{
  "class": "gui::ILayoutComponent",
  "uuid": "{...}",
  "Height": "1080.000000",
  "Width": "1920.000000",
  "MarginTop": "24.000000",
  "MarginBottom": "8.000000",
  "MarginLeft": "8.000000",
  "MarginRight": "8.000000",
  "MarginAnchorX": "Center",
  "SizeModeX": "Stretch",
  "SizeModeY": "AutoSizeToElements"
}
```

- `SizeModeX/Y`: `"Fixed"`, `"Stretch"`, `"AutoSizeToElements"`, `"AutoSizeToComponents"`
- `MarginAnchorX/Y`: positioning anchor

### `gui::ITextComponent`

Renders text.

```json
{
  "class": "gui::ITextComponent",
  "uuid": "{...}",
  "_preset_": "H1_HeaderTitle",
  "_StylePreset_": "H1_HeaderTitle",
  "TextValue": "&Cash_TooltipText&",
  "FontSize": "14.750000",
  "TextColorMul": "text_white",
  "Multiline": "0",
  "TrimmingMode": "Animate",
  "ShadowOffset": "1.000000 1.000000",
  "Uppercase": "1",
  "LineSpacingAdjust": "-1.000000",
  "CharacterSpacingAdjust": "2.650000"
}
```

- `TextValue`: either a literal string or a localization key `"&KeyName&"`
- `_preset_` / `_StylePreset_`: references a text style preset (font, size, color)
- `TextColorMul`: a named color from `.uip` `colors` map, or a hex value

### `gui::IImageComponent`

Renders a texture.

```json
{
  "class": "gui::IImageComponent",
  "uuid": "{...}",
  "Texture": "tooltip_spacer",
  "ImageColorMul": "B7B7B7FF"
}
```

- `Texture`: a texture name resolved via the `.atlas` system
- `ImageColorMul`: RGBA hex tint color

### `gui::IButtonComponent`

Makes an element interactive (clickable/hoverable).

```json
{
  "class": "gui::IButtonComponent",
  "uuid": "{...}"
}
```

The presence of this component is what makes an element a button.
Logic nodes reference `IButtonComponent` fields like `"Pressed"`, `"Hovered"`, `"Focused"`.

### `gui::CScreenComponent`

Marks an element as a top-level screen.

```json
{
  "class": "gui::CScreenComponent",
  "uuid": "{...}",
  "ShowScreenSound": "hud_hintsmenu_in",
  "HideScreenSound": "hud_hintsmenu_out",
  "DependencyScreen0": "OptionsBackground",
  "UsePadCursor": "1",
  "DataClassName": "MenuHintsController"
}
```

- `ShowScreenSound` / `HideScreenSound`: audio event names
- `DependencyScreen0`: another screen that must load alongside this one
- `DataClassName`: C++ class name for this screen's controller

### `gui::ILogicComponent`

Contains the action graph.

```json
{
  "class": "gui::ILogicComponent",
  "uuid": "{...}",
  "data": [
    {"type": "CVariablesComponentSymbolData", "dyn_fields": [...]},
    {"type": "CActionGraphSymbolData", "uuid": "{...}", "hash": "", "nodes": [...]}
  ],
  "dyn_fields": [...]
}
```

### `gui::IWrapPanelComponent`

Controls layout flow direction.

```json
{
  "class": "gui::IWrapPanelComponent",
  "uuid": "{...}",
  "Wrap": "0",
  "Orientation": "Vertical"
}
```

---

## Object types

### `gui::IGroup`

A container with no visual of its own. Can have `components` and `objects`.

```json
{
  "class": "gui::IGroup",
  "uuid": "{...}",
  "Id": "my_group",
  "components": [...],
  "objects": [...]
}
```

### `gui::CSymbolRoot`

A reference to another symbol. The `sub_symbol` value must match a registered symbol name in the `.uip`.

```json
{
  "class": "gui::CSymbolRoot",
  "uuid": "{...}",
  "Id": "btn_back",
  "sub_symbol": "legend_button",
  "components": [
    {
      "target_component_uuid": "{...}",
      "class": "gui::ILayoutComponent",
      "MarginLeft": "0.000000"
    }
  ]
}
```

The `components` on a `CSymbolRoot` **override** components on the referenced symbol.
`target_component_uuid` must match a component UUID in the target `.gui` file.

---

## Action graph node reference

### Field path structure

Most action nodes operate on a `field_path`:

```json
"field_path": {
  "type": "DataContext",
  "class": "MenuHintsController",
  "field": "m_ShowLeaderboards"
}
```

or:

```json
"field_path": {
  "type": "GuiComponent",
  "element": "default_buttons/group1/btn_leaderboard",
  "element_uuid": "{e00c663a-...}",
  "component_class": "gui::IElementComponent",
  "field": "Visible"
}
```

- `element`: slash-separated path from root to element by `Id`
- `element_uuid`: UUID of the target element (more reliable than path)

### Common action nodes

| Node type | What it does |
|-----------|-------------|
| `CActionOnInitialize` | Trigger: fires once when element initializes |
| `CActionOnFieldChanged` | Trigger: fires when a field value changes |
| `CActionOnButtonPress` | Trigger: fires when a button is pressed |
| `CActionOnTimerFinished` | Trigger: fires when a timer completes |
| `CActionGetString` | Read a string field → outputs `Value` |
| `CActionSetString` | Write a string field |
| `CActionGetBool` | Read a bool field → outputs `Value` |
| `CActionSetBool` | Write a bool field |
| `CActionGetFloat` | Read a float field → outputs `Value` |
| `CActionSetFloat` | Write a float field |
| `CActionBranch` | If `Condition` is true → `True` node, else → `False` node |
| `CActionStartTimer` | Start a countdown timer |
| `CActionStopTimer` | Stop a timer |
| `CActionEmitGameDataEvent` | Send an event to C++ code |
| `CActionGetRawPtr` | Get a raw C++ pointer |
| `CActionCastTo` | Try to cast a pointer to a type (Success/Failed branches) |
| `CActionDebugLogText` | Print a debug message |
| `CActionMultiplyFloat` | Multiply two floats |

### `CActionEmitGameDataEvent` — the key to triggering in-game actions

```json
{
  "type": "CActionEmitGameDataEvent",
  "m_Text1": {"value": "show_leaderboard"},
  "m_Text2": {"value": ""},
  "m_Int1": {"value": "0"},
  "m_Float1": {"value": "0.000000"},
  "m_Bool1": {"value": "0"},
  "field_path": {"type": "DataContext", "class": "MenuHintsController"},
  "msg_rtti": "GuiGenericEvent"
}
```

This sends a `GuiGenericEvent` to the `MenuHintsController` with `m_Text1 = "show_leaderboard"`.

**You can only trigger behaviors the controller already handles.**
Known `m_Text1` values for `MenuHintsController`:
- `"show_leaderboard"` — opens the leaderboard

---

## Modding `.gui` files — what's safe to change

**Safe to change:**
- `TextValue` in `ITextComponent` — change button labels, descriptions
- `Visible` in `IElementComponent` — hide/show elements
- `Opacity` in `IElementComponent` — change element transparency
- Layout values in `ILayoutComponent` (Width, Height, Margin*) — reposition elements
- `ShowScreenSound` / `HideScreenSound` in `CScreenComponent` — change menu sounds
- `m_Text1` values in `CActionEmitGameDataEvent` — IF the controller handles the new value
- Adding/removing `gui::IGroup` containers
- Adding/removing `gui::CSymbolRoot` references to existing symbols

**Risky:**
- Changing `symbol_uuid` or `uuid` values — will break cross-references
- Changing `data_context_class` — must be a valid C++ class name
- Changing `sub_symbol` to a symbol not registered in `.uip` — will fail at runtime
- Changing `target_component_uuid` in `CSymbolRoot` overrides — must match real UUIDs

**Do NOT:**
- Invent new C++ class names in `data_context_class` or `DataClassName`
- Invent new `msg_rtti` event types
- Remove the `symbol_uuid` field

---

## Practical: adding a new button to the Hints menu

1. In `menuhints_pc.gui`, find the `default_buttons/group1` group
2. Add a new `gui::CSymbolRoot` entry:
   ```json
   {
     "class": "gui::CSymbolRoot",
     "uuid": "{your-new-uuid-here}",
     "Id": "btn_my_action",
     "sub_symbol": "legend_button",
     "components": [...]
   }
   ```
3. In the `ILogicComponent` action graph, add nodes:
   - `CActionOnButtonPress` targeting your new button
   - `CActionEmitGameDataEvent` with an `m_Text1` value the controller handles
4. The button label is controlled inside `legend_button_pc.gui` — you'd need to override the text component

Note: Generating a valid UUID — use any UUID generator (all zeros will work for testing, but may cause issues).
