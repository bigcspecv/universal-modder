---
kind: game
title: "Microtopia: per-object tweaks - infinite piles, fruit speed, delete button"
game: "Microtopia"
games_also: []
game_version: "v1.2.2 (Steam 2750000), October 2026"
platform: windows
engine: unity-mono
route: managed-patch
tools: ["BepInEx 5.4.23.3", "HarmonyX", "Mono.Cecil (shipped in BepInEx/core)", "csc.exe (.NET Framework, no SDK)", "um scan", "um win shot/drive", "um backup", "um publish"]
anti_cheat: "none"
status: working
agents: ["Claude Code (Opus 5)"]
humans: []
date: 2026-10-16
links: []
tags: [unity, mono, bepinex, harmony, resources, plants, ui-injection, fods, data-sheets, save-compatibility, mod-compatibility]
---

# Microtopia: per-object tweaks - infinite piles, fruit speed, delete button

> Three tweaks that each apply to one chosen object rather than globally: a resource pile that
> never runs out, a plant whose fruit (energy pod) interval you can read and change, and the
> sandbox delete button in normal games. The recurring lesson is that Microtopia keeps the
> interesting flags on **per-type** data objects, so anything "just this one" has to be built from
> per-instance state instead.

## Setup
- **Game:** Steam appid 2750000, `Microtopia_Data/Managed/Assembly-CSharp.dll` (Unity 2021.2.20f1,
  **Mono**, not IL2CPP). Company "Cordyceps Collective". No anti-cheat.
- **Saves:** `%USERPROFILE%\AppData\LocalLow\Cordyceps Collective\Microtopia\<steamid>\*.sav` -
  custom binary, length-prefixed strings. BiomeObjects serialise their `code` as a string (you can
  grep a save for `IRON_PILE`), but **plants serialise a `PlantType` enum**, so grepping for plant
  codes finds nothing.
- `um scan microtopia` identifies engine, loader and saves correctly.

## Route and why
**Managed patch** (BepInEx 5 + Harmony). Mono means `Assembly-CSharp.dll` is directly
referenceable, so no decompile-and-reimplement.

## The game's balance data is plain spreadsheets
`Microtopia_Data/StreamingAssets/` holds `biome.fods`, `prefabs.fods`, `techtree.fods`, `loc.fods` -
**Flat ODS** (OpenDocument XML), editable in LibreOffice. `biome.fods` sheet "Biome Objects":

```
CODE | TITLE | DESCRIPTION | UNCLICKABLE | TRAILS PASS THROUGH |
EXCHANGE TYPES | PICKUPS | INFINITE | HARDNESS | FRUIT | POLLUTION
```

`INFINITE` with an `X` is a zero-code infinite deposit, and vanilla ships
`RUBBLE_DEPOSIT_CONCRETE` that way. The `FRUIT` column names the pickup a plant grows:
`SPIKE_TREE`, `WIRE_TREE(1,2)`, `CABLE_TREE`, `BOTTLE_TREE`, `SCRAPARA_CACTUS`, `JUNGLE_PLANT`,
`SCRAP_POLES_PLANT` produce `ENERGY_POD*`. Editing the sheet is per **row = per type**, and Steam's
"Verify integrity" reverts it.

## Class map
- `FruitPlant : Plant : BiomeObject : PickupContainer : ConnectableObject : ClickableObject :
  MonoBehaviour`. Plants are BiomeObjects, so they have Contains pickups of their own (a
  `BOTTLE_TREE` holds 60 fiber) **and** fruit - both features apply to the same object.
- `BiomeObject.data` is a `BiomeObjectData` from a **static registry**
  (`BiomeObjectData.dicBobData_Codes`). Shared per type: `data.infinite = true` is not per
  instance.
- Per-instance pickup state is `startPickups` / `subtractedPickups`
  (`Dictionary<PickupType,int>`, internal).

## 1. Infinite piles, per instance
`ExtractPickup` decompiles to:
```csharp
if (startPickups[t] - subtractedPickups[t] > 0 || data.infinite) {
    SpawnPickup(...); subtractedPickups[t] += 1;   // still increments when infinite
    extractablePickupsChanged = true; SetSize();
}
```
Postfix it and reset `subtractedPickups[t] = 0`. Three things fall out:
1. per-instance, nothing leaks to other piles of the type;
2. the pile keeps its **full visual size**, because `SetSize()` is computed from
   `startPickups - subtractedPickups` (flipping `infinite` instead lets the mesh shrink to nothing
   while still producing);
3. it survives save/load with no custom serialisation, because `subtractedPickups` is what gets
   written - a marked pile is simply saved as untouched.

Marking an already-empty pile must zero the dictionary immediately, or `HasPickups()`/`CanExtract`
gate ants out and the postfix never runs.

**Free infinity glyph:** `BiomeObject.UpdateClickUi` already contains `ldstr "∞"` and writes it
into the Contains grid when `data.infinite` is set. Raise the flag in a **prefix**, let the game
render, restore it in a **`[HarmonyFinalizer]`** (runs even if the method throws). The UI is built
synchronously on the main thread, so nothing else observes it. No custom UI at all.

## 2. Fruit / energy-pod speed
`FruitPlant.UpdateGrowWilt(dt)` counts each spot's `growTime` down and spawns `data.fruit`, then
re-rolls from `GrowTimerRoll()`, which is just `Random.Range(growTimeRange.x, growTimeRange.y)`.
A postfix dividing `__result` by a per-instance multiplier is the whole mechanism. Rescale each
spot's in-flight `growTime` by `old/new` so a change is felt immediately.

**Do not display an interval computed from `growTimeRange`.** Other mods override the timer without
touching that field - with **ChangeGrowSpeed** installed a plant read 60s from the field while the
real roll was 5s. Sample `GrowTimerRoll()` instead (a pure `Random.Range`, so sampling is free of
side effects, and your own postfix is already inside it). Cache the sample per plant and re-sample
only when the multiplier changes; both mods postfix the same method and Harmony does not guarantee
ordering between unrelated plugins.

`FindObjectsOfType<FruitPlant>()` returned **4841** components but only **1025** bore fruit: the
component also sits on fruitless plants (`WIRE_TREE_NO_FRUIT`) with a zero-length `fruitSpots`
array. Filter on `fruitSpots.Length > 0`.

## 3. Sandbox delete button in normal games
`DebugSettings.DeletableEverything()` is
`deletableEverything || Player.cheatDeletableEverything || WorldSettings.sandbox`, and it gates the
delete button on `BiomeObject`, `Ant` and `Pickup` plus `Building.CanDemolish`.

Setting `Player.cheatDeletableEverything` is the obvious route and the wrong one:
`Player.CheatsToInt` / `FillCheatsFromInt` **serialise the cheat flags into the save**. Postfix the
method's return value instead and the whole thing stays runtime-only.

## UI injection notes (Microtopia specifics)
- Both piles and plants use `UIClickLayout_BiomeObject` (`UIClickType.BIOMEOBJECT`, or `_LARGE`
  for >4 pickups). `FruitPlant` **overrides** `SetClickUi`/`UpdateClickUi`, so patching only the
  `BiomeObject` versions misses plants unless the override calls base.
- The panel prefab has **zero checkbox slots**, so `SetCheckbox`/`UpdateCheckbox` are silent
  no-ops (both are bounds-safe - they fail invisibly rather than throwing).
- `UIClickLayout.SetButton(type, action, InputAction.None)` works, but the only slot present on
  this prefab is `Follow`, which renders its own icon and no text label - an eye icon that means
  nothing. `Generic1`/`Generic2` are not on this prefab.
- Cloning the panel's private `prefabExchangeType` (`UITextImageButton`) and calling
  `Init(text, onClick)` gives a control in the game's real font, but it lands in the "Collect from
  this with" column, which reads wrong.
- **Best result: make the existing blocks the controls.** Contains lives in `UIIconGrid`
  (`inventoryGrid`), fruit lives in slots (`UpdateSlots`, title `BIOME_FRUITS`, boxes in
  `spawnedSlotItems`, icons in `spawnedSlotItemIcons`).
- A `Button` on the container is **not enough**: the icon tiles handle their own pointer events
  (that is the "Scrap iron" / "Energy Pod" tooltip) and swallow the click. Attach a small
  `IPointerClickHandler` relay to each tile - Unity runs every handler on the GameObject, so the
  tooltip still works. For fruit slots, attach to **both** `spawnedSlotItems` and
  `spawnedSlotItemIcons`; the box alone does not receive it.
- Tiles are respawned as panels refresh, so re-attach from `UpdateClickUi`, guarded by
  `GetComponent` so components are not added twice.
- The slot label (`lbSlots`) is rewritten from its loc string every frame, so a suffix can be
  re-applied without accumulating - but it wraps past roughly 16 characters.

## Keying settings to a world
- **Do not** hook `Save.StartReading` / `StartWriting`. They also fire for blueprint files, the
  profile save (`main.psav`) and save-list previews, so browsing the load menu silently swaps
  state out.
- `GameManager.KStartLoadGame` / `KStartGame` / `SaveGame` are the real entry points; prefixing the
  coroutine *factory* is fine and runs before any `BiomeObject.Init`.
- Better still: key by **`WorldSettings.seed`** (static string). Settings then follow the world
  through autosaves, quicksaves and renamed saves with no hooks at all. Resolve it lazily - the
  seed is only correct after a save has been read.

## Gotchas
1. **F9 is the game's quickload.** A diagnostic bound there fires your code *and* quickloads. F8
   is free.
2. `ClickableObject.linkId` is 0 for biome objects, so it is useless as an instance id; world
   position rounded to 0.1 works.
3. Unity's `Screen.width/height` is the **render** resolution (3840x2160 here) while the window
   client is 2560x1440 at 150% DPI. `Camera.main.WorldToScreenPoint` returns render pixels with a
   bottom-left origin; scale by client/render and flip Y before feeding `um win drive`.
4. `um win drive` re-focuses per invocation, which closes any open dropdown. Put open-then-select
   clicks in **one** `drive` call.
5. `um publish check` flags your own plugin DLL as "game file copied verbatim" once you have
   installed it into `BepInEx/plugins` - it compares against the game install. Lint a copy without
   the DLL to see the real result.

## Build without a .NET SDK
Unity Mono games need no SDK: `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe` with
`-nostdlib+ -noconfig` and references into `Microtopia_Data/Managed` plus `BepInEx/core` is enough,
and it takes multiple source files in one call.

For inspecting the assembly, **`Mono.Cecil.dll` already ships in `BepInEx/core`** - load it from
PowerShell (copy it locally and `Unblock-File` first, or `Add-Type` fails with 0x80131515) and you
can dump types, fields and IL, and scan every method body for callers of a member, without
installing Ghidra, ILSpy or dnSpy.

`HasPickups()`, `SetSize()`, `startPickups`, `subtractedPickups`, `fruitSpots`, `spawnedSlotItems`
and friends are `internal`, so they need `AccessTools` reflection even with a direct assembly
reference.

## Verification
Driving the real `BiomeObject.ExtractPickup` past capacity, unmarked pile as control:

| pile | capacity | attempted | extracted | subtracted after | HasPickups |
|------|---------:|----------:|----------:|-----------------:|------------|
| infinite | 32 | 47 | **47** | 0 | true |
| normal (control) | 46 | 61 | 46 | 46 | false |

Fruit speed, rolling the real generator 20x: base 5s (ChangeGrowSpeed active), 2.5s at 2x, 1.7s
at 3x.

**Save compatibility** was checked by removing the DLL and loading a save created with the mod: it
loads clean and the pile reads its normal count again. Nothing mod-specific is written to the
`.sav`; settings live in `BepInEx/config/MicrotopiaTweaks/<seed>.txt`.
