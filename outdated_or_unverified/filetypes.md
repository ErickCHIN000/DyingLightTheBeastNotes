# File Type Reference — Dying Light UI Modding

This document covers each file extension relevant to UI/menu work.
For each format: what it does, how it looks, whether you can edit it, and its menu relevance.

---

## `.gui` — UI Element Definition

**Purpose:** Defines a single UI widget or screen (called a "symbol").
Each `.gui` file describes one visual/interactive element: a full-screen menu, a button, a list item, a popup, etc.

**Readability:** Plain JSON (single-line, no newlines in raw files). You can edit with any text editor.
Use a JSON formatter (e.g. Notepad++ with JSON Viewer plugin) to make it readable.

**Format structure:**
```json
{
  "version": 1,
  "symbol_uuid": "{09cb1ea0-...}",
  "symbol_size": "1920.000000 1080.000000",
  "data_context_class": "MenuHintsController",
  "class": "gui::ISymbolRoot",
  "uuid": "{09cb1ea0-...}",
  "Id": "MenuHints",
  "components": [...],
  "objects": [...]
}
```

**Key fields:**
- `Id` — logical name for this symbol (used in `.uip` and `sub_symbol` references)
- `symbol_size` — width/height in pixels at 1080p
- `data_context_class` — the C++ class that provides data to this element. You cannot create new ones; must reuse existing classes.
- `components` — list of component objects attached to the root element
- `objects` — child elements (recursive; same structure as root)

**Component types seen:**

| Component class | Role |
|-----------------|------|
| `gui::IElementComponent` | Visibility, opacity, data context binding |
| `gui::ILayoutComponent` | Size, position, margins, stretch/auto-size modes |
| `gui::ILogicComponent` | Action graph (behavior/event handling) |
| `gui::ITextComponent` | Text display (value, font, color, trimming) |
| `gui::IImageComponent` | Texture display |
| `gui::IButtonComponent` | Interactivity (hover, press, focus) |
| `gui::IMaskComponent` | Clip masking |
| `gui::IMaterialEffectComponent` | Apply a `.guieff` shader |
| `gui::IWrapPanelComponent` | Layout wrapping (horizontal/vertical flow) |
| `gui::IResolutionScalerComponent` | Scale to resolution |
| `gui::CScreenComponent` | Marks this as a full screen (has show/hide sounds, dependencies) |
| `gui::CHintPositionComponent` | Special positioning for hint elements |
| `gui::IMouseReceiverComponent` | Receives mouse events |

**Child element types:**

| Object class | Role |
|--------------|------|
| `gui::IGroup` | Container group (no visual of its own) |
| `gui::CSymbolRoot` | Reference to another `.gui` symbol by name (via `sub_symbol` field) |

**Action graph nodes (inside `ILogicComponent`):**

Events (triggers):
- `CActionOnFieldChanged` — fires when a data field changes
- `CActionOnButtonPress` — fires when button is pressed
- `CActionOnInitialize` — fires once on element init
- `CActionOnTimerFinished` — fires when a timer completes

Data access:
- `CActionGetString`, `CActionSetString`
- `CActionGetBool`, `CActionSetBool`
- `CActionGetFloat`, `CActionSetFloat`
- `CActionGetRawPtr` — get a raw C++ pointer
- `CActionCastTo` — cast a pointer to a specific type

Control flow:
- `CActionBranch` — conditional split (True/False)
- `CActionStartTimer`, `CActionStopTimer`
- `CActionDebugLogText` — log message

Game communication:
- `CActionEmitGameDataEvent` — sends an event to C++ game code. This is how buttons trigger in-game actions.
  - `msg_rtti`: event type (e.g. `GuiGenericEvent`, `GuiHintInfoEvent`)
  - Parameters: `m_Text1`, `m_Text2`, `m_Int1`, `m_Float1`, `m_Bool1`

**Relationships:**
- Referenced by: `.uip` (in `symbols` map), other `.gui` files (via `sub_symbol`)
- References: other `.gui` files (via `sub_symbol`), textures by name (resolved via `.atlas`), effects by name (resolved via `.uip` `effects` map)

**Editing viability:** High — JSON format, readable with formatter. But:
- Do NOT change `symbol_uuid` — it is used internally
- Do NOT add new `data_context_class` values — must be existing C++ class names
- Button actions (`CActionEmitGameDataEvent`) can only trigger events the game engine already handles

**Menu relevance:** Critical — every visual element of the Hints menu is a `.gui` file.

**Example files:**
- `gui/ingame_menu_pc/menuhints_pc.gui` — full Hints menu screen
- `gui/ingame_menu_pc/hint_info_popup_pc.gui` — hint detail popup
- `gui/ingame_menu_pc/hint_info_cat_pc.gui` — hint category header
- `gui/ingame_menu_pc/menu_hint_symbol_pc.gui` — individual hint row

**Confidence:** High

---

## `.uip` — UI Package Manifest

**Purpose:** A package manifest that defines all resources belonging to a UI module.
The engine uses it to know which symbols, atlases, effects, and SVGs belong together.

**Readability:** Plain JSON (formatted, multiline). Directly editable.

**Format structure:**
```json
{
  "dependencies": ["menu_common"],
  "colors": { "faction_pk": "2C5FFAFF", ... },
  "symbols": {
    "MenuHints": "MenuHints_PC.gui",
    "hint_info_popup": "hint_info_popup_PC.gui",
    ...
  },
  "atlases": ["ingame_menu_compressed_0_PC.atlas", "ingame_menu_textures__PC.atlas"],
  "svg": { "crafting_component_frame": "crafting_component_frame_PC.guivg" },
  "effects": {
    "blur": "blur.guieff",
    "outline": "outline.guieff",
    ...
  }
}
```

**Key fields:**
- `dependencies` — names of other `.uip` packages this one depends on (loaded first)
- `colors` — named color constants used by `.gui` files in this package
- `symbols` — logical symbol name → `.gui` filename. This is the registry of all screens/widgets.
- `atlases` — list of `.atlas` files that provide textures for this package
- `effects` — logical effect name → `.guieff` filename
- `svg` — vector graphics name → `.guivg` filename

**Relationships:**
- Referenced by: game engine (loaded by name when entering a menu state)
- References: `.gui`, `.atlas`, `.guieff`, `.guivg` files

**Editing viability:** High — adding a new symbol entry here would register a new `.gui` file.
Modifying existing entries would redirect which `.gui` file handles a given symbol name.

**Menu relevance:** Critical — the `MenuHints` symbol is registered here.
To add a new menu screen, add a new entry to `symbols`.

**Example files:**
- `gui/ingame_menu_pc/ingame_menu_pc.uip`
- `gui/common_pc/common_pc.uip`
- `gui/menu_common_pc/menu_common_pc.uip`

**Confidence:** High

---

## `.scr` — Script File

**Purpose:** Plain-text game scripts. The most common file type (5615 files).
Used for gameplay logic, hint definitions, campaign config, player variables, animation configs, and more.

**Readability:** Plain text. C-like syntax. Directly editable in any text editor.

**Format:**
```javascript
import "otherfile.scr"

sub functionName()
{
    Command("value", argument);
    // comments use //
    /* block comments */
}

sub main()
{
    use functionName();
    Param("VariableName", "value");
}
```

**Key syntax elements:**
- `import "file.scr"` — include another script
- `sub name() { ... }` — define a subroutine
- `use name()` — call a subroutine (like an include/macro)
- `Param("name", "value")` — set a named parameter
- `Hint("name") { ... }` — define a hint entry
- `HintId("&Key&")` — register a hint ID with localization key
- `Trigger(EEvent_X)` — bind hint to a game event
- `Text("&LocalizationKey&")` — localized string reference
- `Condition(ECnd_X)` — condition that must be true for hint to show
- `//` and `/* */` — comments

**Relationships:**
- `.scr` files frequently `import` other `.scr` files
- `gameplayhints.scr` imports `inventorystuff.scr`
- Hint text uses localization keys (`&KeyName&`) resolved from `.binsloc` files

**Editing viability:** Very high — plain text, no special tools needed.
This is the primary modding target for script-based mods.

**Menu relevance:** High for hint content — `gameplayhints.scr` defines all hints.
Low for menu layout/structure (that's `.gui`).

**Important limitation:** DL does NOT support hot-reloading. Restart the game after every change.

**Example files:**
- `scripts/tutorials/gameplayhints.scr` — all gameplay hint definitions
- `scripts/tutorials/loadinghints.scr` — loading screen hints
- `scripts/tutorials/deathhints.scr` — death screen hints
- `hints.scr` — root hint ID registry
- `scripts/player/player_variables.scr` — player stats

**Confidence:** High

---

## `.atlas` — Texture Atlas Manifest

**Purpose:** Maps named texture identifiers to their pixel coordinates within a larger texture sheet (atlas).
The GUI system uses texture names (not file paths) — the atlas resolves names to actual pixels.

**Readability:** JSON (single-line). Readable with a JSON formatter.

**Format:**
```json
{
  "filename.dds": {
    "w": 128, "h": 128,
    "rectangles": {
      "texture_name": {
        "x": 0, "y": 0, "w": 128, "h": 128,
        "s": "Size_4k",
        "glyph": {"adv": 112, "off_x": 0, "off_y": -112, "asc": 112, "dsc": 0, "size": 16, "col": false}
      }
    }
  }
}
```

**Key fields:**
- Top-level key: source texture file (`.dds` or `.png`)
- `rectangles`: named regions within that texture
- `glyph`: metadata for using the texture as a font glyph (for icon fonts)
- `s`: size category (`"Size_4k"` = scalable across resolutions)

**Two atlas types observed per UI package:**
- `*_compressed_0_pc.atlas` — likely references compressed/streamed textures
- `*_textures__pc.atlas` — references uncompressed/direct textures; also includes story images

**Relationships:**
- Referenced by: `.uip` (in `atlases` list)
- References: `.dds` and `.png` texture files (in `.rpack` archives — NOT directly moddable via `.pak`)

**Editing viability:** Medium — you can add new entries if you add new textures.
However, the actual texture files are in `.rpack` archives, which require separate tools to modify.
For layout-only mods, you don't need to touch `.atlas` files.

**Menu relevance:** Medium — required if you want to change or add textures/icons.
Not needed for structural or behavioral changes to the menu.

**Example files:**
- `gui/ingame_menu_pc/ingame_menu_textures__pc.atlas`
- `gui/ingame_menu_pc/ingame_menu_compressed_0_pc.atlas`

**Confidence:** High

---

## `.guieff` — GUI Shader Effect

**Purpose:** Defines a visual shader effect applied to a GUI element.
Used for things like blur, color inversion, dissolve transitions, masks, outline glow, etc.

**Readability:** Plain text DSL (similar to `.scr` syntax).

**Format:**
```javascript
sub main()
{
    FloatVar("alpha_mul", "1.0", 9, 0, "");  // shader parameter
    FloatVar("alpha_add", "0.0", 9, 1, "");

    Image()
    {
        Material("gui_blur.mat", BLUR_LAYER);
    }
    Text()
    {
        Material("gui_blur.mat", BLUR_LAYER);
    }
}
```

**Known effects:**
`blur`, `circular_mask`, `desaturate`, `dissolve`, `distort`, `greenscreen_movie`,
`gui_default_effect`, `horiz_blur_5`, `horiz_blur_7`, `hsv_colorize`, `invert_colors`,
`mapicon`, `mapicon2`, `mask_colorize`, `mask_mul`, `mask_overlay`, `outline`, etc.

**Relationships:**
- Referenced by: `.uip` (in `effects` map), `.gui` files (by effect name in `IMaterialEffectComponent`)
- References: `.mat` material files (shader files, binary, not modder-editable)

**Editing viability:** Low — modifying these is advanced. The effect names are referenced by name in `.gui` files. You can reference existing effects by name without touching `.guieff` files.

**Menu relevance:** Low for beginners — the blur background and outline effects on the Hints menu use these, but you don't need to change them to modify the menu's content or behavior.

**Example files:**
- `gui/materials/effects/blur/blur.guieff`
- `gui/materials/effects/outline/outline.guieff`
- `gui/materials/effects/dissolve/dissolve.guieff`

**Confidence:** High

---

## `.rscr` — Resource Registration Script

**Purpose:** Registers texture resources for a UI package — maps texture names to file paths in the game's data hierarchy.

**Readability:** Plain text DSL. Starts with `import "ResourcePackCfg.scr"`.

**Format:**
```javascript
import "ResourcePackCfg.scr"

sub main()
{
  configuration("pc")
  {
    res(_TEXTURE_, "texture_name.dds", "gui/ingame_menu_pc/texture_name.dds", "", false, "ph_ft/work/data/gui/ingame_menu_pc");
    res(_TEXTURE_, "another_texture.png", "path/to/file.png", "", false, "source_path");
  }
}
```

**Parameters of `res()`:**
1. Resource type (e.g. `_TEXTURE_`)
2. Logical texture name (referenced in `.atlas` and `.gui` files)
3. Relative file path to the texture
4. Unknown (usually empty)
5. Boolean (purpose unclear)
6. Source path (likely build system path, ignored at runtime)

**Relationships:**
- Referenced by: UI package system (loaded alongside `.uip`)
- References: `.dds`, `.png` texture files

**Editing viability:** Medium — needed if adding new custom textures. Not needed for structural changes.

**Menu relevance:** Low for beginners — only needed if adding new textures to your custom menu.

**Example files:**
- `gui/ingame_menu_pc/gui_ingame_menu_pc.rscr`
- `gui/common_pc/gui_common_pc.rscr`

**Confidence:** High

---

## `.def` — Definition / Constant File

**Purpose:** Defines named constants, enums, or typed parameters. Multiple usage contexts.

**Readability:** Plain text. Context-dependent syntax.

**Formats observed:**

*Constant definitions (e.g. `pad_elements.def`):*
```
// synchronized with EPadElement
$_BUTTON_A     (i, 0)
$_BUTTON_B     (i, 1)
$_BUTTON_X     (i, 2)
```
Macro-like named integer constants, synchronized with C++ enums.

*Interface/type definitions (e.g. `interface/barrier.def`):*
Short structured definitions for engine object types.

**Editing viability:** Low to Medium — constants are synced with C++ so changing values breaks things.
These are primarily read-only references for modders.

**Menu relevance:** Low — `menu/controls/pad_elements.def` defines gamepad buttons, relevant if you want custom controller binding references, but not for basic menu modding.

**Example files:**
- `menu/controls/pad_elements.def` — gamepad button constants
- `campaigns.def` — campaign definitions
- `characters.def` — character definitions

**Confidence:** High (format), Medium (full behavior)

---

## `.gds` — Game Data Set / Animation Set

**Purpose:** Defines animation set data tables — maps combinations of input parameters to animation sequences.
**NOT related to GUI or menus.** Found only in `ai/animsets/`.

**Readability:** Plain text DSL.

**Format:**
```javascript
sub main()
{
    Definition()
    {
        Input("Role", "int_range", "");
        Input("Action", "enum", "Attack,Grab,Crawl");
        Output("sequence", "sequence", "");
    }
    Data("[0,0]::Attack", "animation_name@script_file.scr");
}
```

**Menu relevance:** None.

**Confidence:** High (format and irrelevance to menus confirmed)

---

## `.scd` — Script C Description

**Purpose:** Editor metadata — provides display descriptions, ranges, and UI hints for variables shown in the game editor.

**Readability:** Plain text. Uses `Description()` calls.

**Format:**
```
Description("v_ambient", "f:EDIT;e:EDIT|SPINNER|EXPANDABLE|BROWSE_COLOR;m:0.0 0.0 0.0;x:1.0 1.0 1.0;...")
```

**Menu relevance:** None — editor-only metadata.

**Confidence:** High (irrelevance confirmed)

---

## `.ares` — Unknown (2319 files)

**Purpose:** Unknown. High count (2319) suggests a major game system format.
No `.ares` files found in `gui/` directories — likely not directly related to UI.

**Readability:** Unknown — needs binary inspection.

**Menu relevance:** Unknown. Not a priority for basic menu modding.

**Confidence:** Low — not yet investigated.

---

## Summary table

| Ext | Editable | Menu relevance | Priority |
|-----|----------|----------------|----------|
| `.gui` | Yes (JSON) | Critical | Must learn |
| `.uip` | Yes (JSON) | Critical | Must learn |
| `.scr` | Yes (text) | High (content) | Must learn |
| `.atlas` | Yes (JSON) | Medium (textures) | Learn if adding new textures |
| `.rscr` | Yes (text) | Low | Only if adding textures |
| `.guieff` | Yes (text) | Low | Reuse existing names only |
| `.def` | Risky | Low | Reference only |
| `.gds` | Yes (text) | None | Ignore for UI work |
| `.scd` | Yes | None | Ignore |
| `.ares` | Unknown | Unknown | Investigate later |
