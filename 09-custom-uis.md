---
verified-on: UEFN v42.20
last-reviewed-against: UEFN v42.20 (20 Sep 2026)
---

# Custom UIs

Verse-authored widgets and UMG Widget Blueprints — the two ways to put custom UI on screen, how they bind to Verse, and how to debug them when they won't compile.

ℹ️ *Provenance: started 20 Sep 2026 while working through Epic's `UserInterfaces_Demo` sample project on v42.20. For HUD devices — `tracker_device`, `hud_message_device`, `map_indicator_device` — see [03-devices-and-interaction.md](03-devices-and-interaction.md), which this file continues from. For the MCP toolsets that author widgets programmatically (`UMGToolSet`, `VerseFieldsToolset`, `MVVMToolset`, `WidgetAnimationToolset`), see [05-editor-and-tooling.md](05-editor-and-tooling.md).*

> ⚠️ **Start here if an AI told you this is impossible.** Epic's own in-editor assistant asserts that widgets are immutable and that bindings don't exist in UEFN. Both are false, and both are disproven below. See [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md), ninth entry, for why that shape of error is the most expensive one.

## ⭐ Diagnosing a Widget Blueprint that won't compile (20 Sep 2026)

**Symptom.** Compiler Results says which asset failed but not which part of it:

```
Binding '<none> <- <none>': A source path is required, but not set.
Binding '<none> <- <none>': A source path is required, but not set.
Compile of UW_CustomButton_Hotbar02 failed. 2 Fatal Issue(s) 0 Warning(s)
```

- ⭐ **The human-facing signal is the View Bindings panel** (button in the editor's bottom bar). The broken row is **outlined in red** with *"No field selected"* on both sides. ⚠️ **The trap: that panel was ~100 rows long here, and the bad row was near the bottom** — it reads as "no visual indication" until you scroll all the way down. Scroll before concluding the editor isn't telling you.
- ⭐ **The agent-facing route is `MVVMToolset.ListWidgetViewBindings`**, which returns every binding row with its source path, destination path, conversion function, GUID and `bEnabled`/`bCompile` flags. Feed it the widget's refPath. `MVVMToolset.RemoveWidgetViewBinding` deletes one by GUID.
- ⚠️ **"Empty source path" is NOT by itself the bug — this is what makes the panel so misleading.** On this widget **19 of 45 bindings had a completely empty source path and were all perfectly healthy.** When a binding uses a **conversion function**, the source comes from that function's own graph inputs rather than a direct property path, so an empty `sourcePath` is normal and expected.
  - ⭐ **The real discriminator: empty source AND no conversion function.** Filtering on "empty source" alone gave 20 suspects; filtering on both gave exactly **1**. That one was blank in *both* directions — a `+ Add Binding` row that was never filled in.
- ⚠️ **The fatal-issue count is not the number of broken rows.** One blank binding produced **two** identical diagnostics. Don't go hunting for a second fault because the panel said 2.
- ✅ **Verified by watching the number move** (the compile-probe discipline from [05-editor-and-tooling.md](05-editor-and-tooling.md)): before — 45 bindings, 1 blank; after removing it — **44 bindings, 0 blank, the 19 conversion-bindings untouched**, widget compiles, `BuildAll` returns `[]`.
- **Sibling tools for the same class of "it names the asset, not the part" error:** `MVVMToolset.ListWidgetViewEvents` (event bindings), `VerseFieldsToolset.ListVerseFields` (the Verse↔UMG field side), and `MVVMToolset.FixupMVVMData` — the repair call, for when the **generated graphs** report errors rather than the binding data itself.
- ℹ️ *How it got there: file timestamps showed every sibling widget in that folder untouched since the project download, and the failing one alone saved later — so it was a stray edit in the Bindings panel, already committed to disk, surfacing later because compiling the project compiles every widget. **Checking asset mtimes against their siblings is a cheap way to answer "is this mine or Epic's?"***

## ⭐ Verse-authored UI: how to update a widget that is already on screen (20 Sep 2026)

The question a first Verse UI raises: to change *"Score: 0"* into *"Score: 1"*, do you rebuild the widget or edit it?

- ⚠️ **`DefaultText` is initialisation-only and is IGNORED after construction.** Epic's own digest comment on `text_block.DefaultText`, and on every sibling `Default*` field: *"The text to display to the user. Used only during initialization of the widget and not modified by SetText."* The same wording appears on `DefaultTextColor`, `DefaultTextSize` and `DefaultTextOpacity`. **So mutating a `Default*` field after the widget is on screen does nothing** — which is the trap that makes destroy-and-recreate look like the only option.
- ✅ **There ARE live setters — this is the lighter path.** Digest-verified on `text_block` (`/UnrealEngine.com/Temporary/UI`): `SetText(InText:message)`, `GetText():string`, `SetTextColor(color)`, `SetTextSize(float)`, `SetTextOpacity(…)`, and matching getters. Keep a reference to the `text_block` itself, build the `canvas` **once**, and call `SetText` on each update.
- **Destroy-and-recreate also works** — `PlayerUI.RemoveWidget(OldCanvas)` then build a fresh `canvas` and `AddWidget` it — and is what a first working example tends to land on. It is correct but heavier: it rebuilds the whole widget tree per update and means tracking `[player]canvas` rather than `[player]text_block`.
- ✅ **Status: RUNTIME-VERIFIED 20 Sep 2026.** Put through the compile-probe recipe on v42.20: wrote a probe file, `BuildAll` → **0 errors**; then planted `ThisIdentifierDoesNotExist()` and rebuilt → `Error 3506 Unknown identifier` **reported against the probe's own path**, proving the file really was compiled and the clean result meant something. ⭐ **Two things it proved beyond `SetText` itself:** a `[player]text_block` map is legal — widget references can be held per player — and `GetPlayspace().PlayerAddedEvent().Subscribe(…)` type-checks against a plain `(Player:player):void` handler. **Both versions now run**, and the `SetText` version behaves identically on screen without the per-update rebuild.
- **`SetText` takes a `message`, not a `string`.** Hence the helper any working example needs: `ScoreMessage<localizes>(Score:int):message = "Score: {Score}"`. **The `<localizes>` specifier is what makes a function produce a `message`.**

## Per-player shape, and the late-joiner gap (20 Sep 2026)

- **`GetPlayerUI[Player]` is per-player and failable** — there is no single global UI surface. Every widget you add belongs to one player's UI, so anything on screen is tracked in a `[player]…` map.
- ⚠️ **A device that only initialises UI in `OnBegin` gives late joiners nothing.** `OnBegin` iterates `GetPlayspace().GetPlayers()` once, at round start; a player who joins afterwards never gets a canvas and never sees a score. **Fix:** `GetPlayspace().PlayerAddedEvent()` and `PlayerRemovedEvent()` both exist and return `listenable(player)` — subscribe to build and tear down per player. *(Compile-verified 20 Sep 2026, then shipped and running.)*
- **`set Map[Key] = Value` is failable**, which is why working code is littered with `if (set PlayerScores[Player] = 0) {}`. The empty `{}` is the idiom for "run it, ignore whether it succeeded".

## Effects: why `OnBegin` is written the way it is (20 Sep 2026)

`OnBegin<override>()<suspends>:void` — both specifiers are inherited obligations, not choices.

- **`<override>`** — `creative_device` already declares `OnBegin<native_callable><public>()<suspends>:void`. Your class inherits it, and `<override>` says "this replaces the inherited one". Verse makes this **explicit and mandatory**: you cannot accidentally shadow a parent method.
- **`<suspends>`** — an **effect specifier** marking the function asynchronous: allowed to take more than one update to finish, and to use `Sleep`, `Await`, `race`, `sync`, `spawn`, `branch`, or a `loop` that sleeps. ⭐ **You must write it even if your `OnBegin` never sleeps** — the signature has to match the one being overridden. It is not "I need async", it is "the parent said async".
- ⭐ **The effect is statically checked, and that shapes the whole file.** A `<suspends>` function can only be called from another async context. `listenable.Subscribe` takes `Callback:type {_(:t):void}` — **no `<suspends>`** — so **an event handler cannot sleep or await**. That is why a button handler is written `OnButtonInteracted(Agent:agent):void` with no effect specifier. If a handler ever needs to wait, wrap its body in `spawn{}`.
- **`OnEnd():void` has no `<suspends>`** (digest-verified) — cleanup must complete immediately.
- **Event handlers receive an `agent`, not a `player`.** Hence the failable cast `if (Player := player[Agent])` before anything per-player. `agent` is the broader type (it covers NPCs); `player` is what a `[player]…` map and `GetPlayerUI` need.

## ⭐ The Widget Blueprint path, end to end — BUILT AND RUN 20 Sep 2026

Proven in `UserInterfaces_Demo` on v42.20, with a **Verse-built `canvas` and a UMG Widget Blueprint on screen at the same time**, both showing the same score, updated by two entirely different mechanisms. ⭐ **Both update in place. Neither needs teardown. The whole "rebuild the widget to change it" idea is wrong on both paths.**

Six steps. The traps are in 2 and 4.

### 1. Build the widget — but do NOT drag it into the level

⚠️ A **screen-space** Widget Blueprint is *never* placed in the world. Verse instantiates it and adds it to a player's UI. (Placing UMG in the world is a different feature — see Epic's `SceneGraphUIExamples` — and not this.) Coming from Unreal, "drag it into the viewport" is the instinct to unlearn.

### 2. ⛔ Add a Verse field — THE PANEL SILENTLY DISCARDS IT

On the widget's `+ Add` panel (top right), typing a name and choosing a type is **NOT enough**. The row must be committed with the **✓ button beside the type dropdown**. Click away instead and the field is thrown away — **no warning, no error, no prompt**. *(Hit 20 Sep 2026, then deliberately reproduced with a second variable.)*

- **Symptom:** the field is not offered as a binding source, and `VerseFieldsToolset.ListVerseFields` returns `[]`.
- ⚠️ **Compile does NOT rescue it, and Compile is NOT at fault.** It compiles the Blueprint correctly — there is simply no field to compile. **A green Compile tick right after this trap is honest and thoroughly misleading**, and will send you hunting in the bindings panel for a fault that isn't there.
- ⭐ **The tell that a field is REAL: a `var` pill appears beside its name** in the fields list. The uncommitted editing row has no pill.
- *Why it's so easy to miss: the ✓ and ✗ controls don't read as buttons — they scan as cluttered UI decoration and the eye glides past them.*

### 3. Bind the widget property to the field

⭐ **A binding row reads DESTINATION ← SOURCE.** Left = what gets written (always a widget property). Right = where the value is read from (your Verse field). The direction control sits between them. That layout is the reason the panel is hard to read at first.

Correct signature, as `MVVMToolset.ListWidgetViewBindings` reports it:

```
source:      source: "SelfContext", memberName: "ScoreDisplay", bSelfContext: true
destination: source: "Widget", widgetName: "ScoreValue", memberName: "Text"
mode:        OneWayToDestination
conversion:  none
```

- ⭐ **Match the types and no conversion function is needed.** `TextBlock.Text` is a `message`, so make the Verse field `message` and format the number on the Verse side with a `<localizes>` helper. Epic's own `MessageText → UEFN_TextBlock_Message.Text` is exactly this shape.
- ✅ **"Is Variable" is NOT required** on the destination widget. Epic's bound `UEFN_TextBlock_Message` reports `bIsVariable: false`. Don't go looking for that checkbox.
- ⚠️ **If the field doesn't exist yet, the source picker offers only the widget's built-in members** (`Is Enabled`, `Visibility`, `Transform`, child widgets…) — and you will end up binding something to itself, e.g. `ScoreValue.Text ← ScoreValue.Text`. **That is silently accepted** and only surfaces later as the fatal `'<none> <- <none>'` class of binding error at the top of this file.
- ⭐ **Doing steps 2–3 over MCP instead** — both worked first time and are the fast path when the panel fights you: `VerseFieldsToolset.AddVerseField(fieldName, fieldType, defaultValue, visibility, writeAccess, bIsVar)` then `VerseFieldsToolset.BindWidgetPropertyToVerseField(verseFieldName, targetWidget, widgetPropertyPath, mode)`. The second creates the SelfContext binding directly and recompiles. `RemoveVerseField` reverses the first, so it is safe to try.

### 4. ⛔ Declare the module, or the asset is INVISIBLE to Verse

An asset folder must be declared as a `<public>` module, mirroring the Content folder structure, before Verse can reference anything inside it. Without it:

```
Error 3593: Invalid access of internal module '(.../MyStuff:)Widgets' from control scope ...
```

✅ **Proven recipe (compile-probed 20 Sep 2026):** a file at **Content root**, alongside Epic's own `ui_helper.verse` which does the same job for its own folder:

```verse
MyStuff<public> := module:
    Widgets<public> := module{}
```

- ⭐ **This is what Epic's otherwise-baffling `ui_helper.verse` is for** — a file containing nothing but nested empty module declarations. **It is load-bearing, not clutter.** Don't delete it, and add an entry for each new asset folder.
- **How to tell from the digest:** in `*-Assets.digest.verse`, a declared folder appears as `Widgets<public> := module:`; an undeclared one appears as `(/…/MyStuff:)Widgets := module:` — qualified path, no `<public>`.

### 5. Declare it in Verse — ⭐ and NO `@editable`

```verse
@editable
ScoreButton : button_device = button_device{}        # placed in the level -> @editable

ScoreWidget : MyStuff.Widgets.UW_MyWidget            # an asset class -> constructed in code
            = MyStuff.Widgets.UW_MyWidget{}
```

⭐ **The rule:** `@editable` exists so a *human* can point the device at a specific thing **placed in the world** — there are many buttons, so you must say which. A widget is not in the world; it is an asset **class**, and that second line is a constructor call Verse runs itself. Nothing to wire. Epic's own hotbar device declares its widget the same way, with no `@editable`.

Then `PlayerUI.AddWidget(ScoreWidget)` — the same call as a Verse-built canvas.

### 6. Update it — one field assignment

```verse
set ScoreWidget.ScoreDisplay = ScoreValueMessage(NewScore)
```

No `SetText`, no `RemoveWidget`, no rebuild. The MVVM binding carries the value to the bound widget property. **This is the closest thing UEFN has to Unreal's "bind a text field to a variable", and it behaves the way that does.**

### Two things only running it will teach you

- ⚠️ **A Verse field with no default renders BLANK, overwriting the design-time text.** `ScoreDisplay` defaulted to `""`, and the binding pushed that empty message into `ScoreValue.Text` on construct — wiping the placeholder the widget was designed with. The widget showed an empty value until the first button press. **Fix: set the field once at init**, next to `AddWidget`. The Verse-built path doesn't suffer this because `DefaultText` is baked in at construction.
- ⚠️ **ONE widget field = ONE shared instance across all players.** `PlayerUI.AddWidget(SharedWidget)` in a loop gives every player the *same* instance, so a per-player score map driving a single widget field makes everyone see whoever acted last. **Identical to the per-player version when testing solo, divergent the moment a second player joins.** Epic's hotbar shares one instance deliberately; a per-player UI needs a `[player]<widget_class>` map with a fresh `<widget_class>{}` per player. **Per-player widget instances are NOT yet tested here** — and v42.20's note about multiple instances of a widget class only updating one of them ([00-version-watch.md](00-version-watch.md)) is the first suspect if they misbehave. *(That note is about **in-world** UMG, so it may not apply at all.)*
- ℹ️ **Let the widget own its labels.** A static "Score: " TextBlock in the widget plus a `"Score: {N}"` message from Verse renders *"Score: Score 30"*. Keep the label in the widget and send only the value (`"{N}"`) — the widget owns presentation, Verse owns data. **The duplication is a sign the split is right, not wrong.**

## ⚠️ "I pushed my change and the UI didn't move" (20 Sep 2026)

**Push Verse Changes swaps the code. It does NOT rewind and re-run initialisation that already happened, and it does not re-evaluate a widget already on screen.** Anything written in `OnBegin` or an init function keeps the value the *previous* run gave it until something re-triggers it. The screen looks identical, which reads exactly like a failed push.

- ⭐ **The 5-second test that separates "my code is wrong" from "my code didn't arrive": trigger the update path once.** If the display corrects itself, the push landed and the bug is in the init path. If it stays stale, go looking at the push.
- **Where to confirm a push for real** — `%LOCALAPPDATA%\UnrealEditorFortnite\Saved\Logs\UnrealEditorFortnite.log`:
  - `LogSolLoadCompiler: … (incremental, 1 packages compiled …) finished: SUCCESS` — the compile that picked up the edit
  - `LogValkyrieBeacon: … [Push Verse Changes] operation successful.` — the push itself
  - ⚠️ `Global Verse compile finished: No packages found requiring compilation` is **normal** on a repeat push with no further edits — it is not a failure.
- **If you need init to re-run, restart the session** rather than pushing.
- ℹ️ *Caught this way 20 Sep 2026: a label-duplication fix was applied to the update call site but not the init one, so the initial render stayed wrong while the post-press render was right. The log proved both pushes had succeeded, which pointed straight at the code instead of the toolchain.*

## Named Slots — Epic uses them ZERO times (surveyed 20 Sep 2026)

Machine-counted across Epic's `UserInterfaces_Demo`: **114 Widget Blueprints, 0 with any named slot** (`UMGToolSet.GetNamedSlots` on every one). That covers menus, shops, conversations, hotbars, in-world UMG and the art examples — every family in the sample.

- **The distinction that matters:** Verse fields + MVVM bindings give you **dynamic values in a fixed layout**. Named slots give you **dynamic composition** — injecting unknown content into a placeholder. Those are different problems, and almost all game UI is the first one.
- ⭐ **Where Epic goes instead, in the two places you would most expect a named slot:**
  - **Variable-looking lists → fixed slots with per-slot fields.** The hotbar has eight hardcoded `Slot01…Slot08` sub-widgets with their own icon and selection fields, not a dynamic item container.
  - **Swappable pages → swap the whole widget.** The Menu example has separate page widgets and Verse classes (main menu, settings, upgrade, vehicle select, countdown) and adds/removes them from the player UI, rather than injecting pages into one shell.
- ⚠️ **This is absence of evidence, not proof the feature is broken.** Named Slot is in the widget palette and `UMGToolSet` ships three tools for it (`GetNamedSlots`, `SetNamedSlotContent`, `ReplaceWidgetWithNamedSlot`), so it exists and is scriptable — **untested here**. The finding is that Epic did not reach for it once in a project written to teach UEFN UI.
- **Verdict for now:** bind a field for changing values; add or remove whole widgets for changing structure. Revisit named slots only for a genuinely reusable shell (a dialog frame wrapping arbitrary content), and test before committing to it.
