# Format Notes — .scr Files

Deep-dive reference for the `.scr` (script) file format in Dying Light.

---

## Overview

`.scr` files are the primary modding target for gameplay changes.
They use a plain-text scripting language with C-like syntax.
5615 `.scr` files exist in `data0` — covering nearly everything from player stats to UI hints.

---

## Basic syntax

```javascript
// Single-line comment
/* Multi-line
   comment */

import "other_file.scr"   // Include another script file

sub FunctionName()
{
    Command("argument1", "argument2");
    Param("name", "value");
}

sub main()
{
    use FunctionName();    // Call a sub (like a macro/include)
    Command("value");
}
```

- `import "file.scr"` — include another script (path relative to this file's location)
- `sub name() { }` — define a reusable block
- `use name()` — call/expand a sub
- `main()` is the entry point
- Strings in double quotes
- Comments with `//` and `/* */`

---

## Player variable scripts

Example: `scripts/player/player_variables.scr`

```javascript
Param("InVisibleToEnemies", "0");
Param("MaxHealth", "100.0");
Param("StaminaMax", "100.0");
```

- Boolean params: `"0"` = false, `"1"` = true
- Float params: decimal values in quotes
- Comment changes with `//YourTag - original was X`

---

## Hint definition scripts

Located at `scripts/tutorials/gameplayhints.scr`.

### Hint block syntax

```javascript
Hint("HintIdentifier")
{
    Text("&LocalizationKey&");
    use DefaultParams();

    // Timing
    Cooldown(120);               // Min seconds between re-shows
    ShowDelayTime(0.5);          // Delay before showing after trigger
    MaxQueueTime(5);             // Max time to wait in queue (0 = never queue)
    MinDisplayTime(6.5);         // Min time to display

    // Triggering
    Trigger(EEvent_SomeEvent);   // Game event that triggers this hint
    Condition(ECnd_SomeCondition);      // Must be true to show
    NotCondition(ECnd_SomeCondition);   // Must be false to show

    // Behavior
    Priority(1);                 // Higher = shown first
    ShowInMenu(false);           // Hides from Hints menu. Default is VISIBLE (no call needed to show)
    AlwaysVisibleInMenu(true);   // Always in menu regardless of other flags/discovery state
    PrologueHint(true);          // Only show during prologue
    DisableShownTimes(2);        // Auto-disable after N shows
}
```

### Known game events (triggers)

Copied from the comment block at the top of `gameplayhints.scr`:

```
EEvent_LowStamina
EEvent_LowImmunity
EEvent_LowHealth
EEvent_HealthOk
EEvent_TimeInDarknessNoFlashlight
EEvent_DamagedByAINormalAttack
EEvent_DamagedByAIPowerAttack
EEvent_WeaponBadlyDamaged
EEvent_WeaponBroken
EEvent_UseWpnInSafeZone
EEvent_NewCraftingAvailable
EEvent_UnspendSkillPointsAfterTime
EEvent_UnspendInhibitorsAfterTime
EEvent_UnspendFuryPointsAfterTime
EEvent_WpnChangedToRangedWpn
EEvent_ChaseStarted
EEvent_ChaseEnded
EEvent_NightStart
EEvent_NightEnd
EEvent_EnteringDarkZone
EEvent_LeavingDarkZone
EEvent_EnteringSafeZone
EEvent_LeavingSafeZone
EEvent_AntizinDraining
EEvent_AntizinRegenerating
EEvent_EnemyKilled
EEvent_FirearmIronsights
EEvent_ItemTaken_Medkit
EEvent_ItemTaken_CraftPlan
EEvent_ItemTaken_FireCrackers
EEvent_ItemTaken_Collectable
EEvent_ItemTaken_QuestCollectable
EEvent_ItemTaken_Outfit
EEvent_LockPickStart
EEvent_LockPickEnd
EEvent_NoLockpicks
EEvent_ReleaseCable
EEvent_NewLocationsOnMap
EEvent_PlayerDiedWithWhiteWeapon
EEvent_LandingAlmostDied
EEvent_PlayerInChemicalZone
EEvent_MedkitUsed
EEvent_FallOnDestroRoof
EEvent_NewOutfitAvailable
EEvent_EnteringRestrictedAreaQuest
EEvent_EnteringAreaQuest
EEvent_ItemAddedToInventory
EEvent_TimeInterval
EEvent_ZoneChange
EEvent_Flashlight_On
EEvent_Flashlight_OnAttempt
EEvent_Flashlight_Off
EEvent_UvFlashlight_On
EEvent_UvFlashlight_OnAttempt
EEvent_UvFlashlight_Off
EEvent_FuryActivated
EEvent_FuryDeactivated
EEvent_FuryReady
EEvent_PlayerEnteredOWA
EEvent_PlayerIsOnStreetLevelAfterTime
EEvent_MenuMapUnlocked
EEvent_JoinedCoop
```

Special trigger with parameter:
```javascript
Trigger(EEvent_TimeInDarknessNoFlashlight, 30);  // 30 seconds in darkness
Trigger(EEvent_ItemAddedToInventory, "wpn_*");   // wildcards supported
```

---

## Hint ID registry

File: `hints.scr` (at data0 root)

```javascript
!HintId(s)
HintId("&SurvivalHint_1&")
HintId("&SurvivalHint_2&")
```

Purpose: unclear — may be a pre-registration system for hint IDs.
`!HintId(s)` is likely a declaration/type specifier.

---

## Localization keys

Text values in `.scr` files (and `.gui` files) use the pattern `"&KeyName&"` to reference localized strings.
The actual text is stored in `.binsloc` files (binary localization format, not directly editable).

For modding text content, you have two options:
1. Use existing localization keys (the text is whatever the game has for that key)
2. Hardcode literal text: `Text("Your custom text here")` — works but only in English

---

## Campaign definition scripts

Campaign files define which maps/missions are available:
- `campaigns.scr`, `campaign_main.scr`, `campaign_story.scr`
- Not directly relevant to menu modding

---

## Common modding mistakes

1. **Breaking the format** — any syntax error causes the game to ignore the entire file and use the vanilla version. Test one change at a time.
2. **Hot-reload doesn't exist** — always restart the game after changes
3. **Live events override mods** — active in-game events take priority. Test offline.
4. **Typos in event names** — `EEvent_WeaponBroken` (not `Broken` alone). Copy from the comment block.
5. **Wrong file path in .pak** — the folder structure inside your .pak must match `data0.pak` exactly.

---

## Practical: adding a new hint to the Hints menu

1. Open `scripts/tutorials/gameplayhints.scr`
2. Add a new `Hint()` block inside `sub main()`:
   ```javascript
   Hint("MyCustomHint")
   {
       Text("This is my custom hint text.");
       Cooldown(0);
       // Do NOT add ShowInMenu(false) — hints appear in menu BY DEFAULT
       Priority(5);
       MinDisplayTime(5.0);
   }
   ```
3. Package the modified `gameplayhints.scr` into a `.pak` at the path `scripts/tutorials/gameplayhints.scr`
4. Test in-game — your hint should appear in the Hints menu list

Note: The hint `Name` field shown in the menu comes from the `m_Name` data field in `GuiHintInfoData`.
This is populated by the C++ controller from the hint's identifier or a localization key.
If using literal text in `Text()`, the display name may be the identifier string itself.
