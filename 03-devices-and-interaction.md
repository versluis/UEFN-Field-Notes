---
verified-on: UEFN v41.10 – v42.10
last-reviewed-against: UEFN v42.10 (3 Sep 2026)
---

# Devices & Interaction

The runtime-device constraint, button behaviour, tag discovery, pooling rationale, HUD & map devices. Verified as of UEFN v41.10 unless a later date says otherwise.

## Buttons & the moving-prompt pattern

- ❌ ~~`button_device` has **no Show/Hide and no TeleportTo**.~~ **WRONG — corrected 28 Aug 2026.** `button_device` → `creative_device_base` → **`creative_object`**, which provides **`TeleportTo`** (position+rotation, or transform) **and `MoveTo(…, OverTime)<suspends>`** for smooth interpolated movement. The original note was written from the class's *own* member list without walking the inheritance chain.
  - ⭐ **The lesson, and it has now cost us twice:** *a Verse class's listed members are not its full API.* Always walk the `:= class(parent)` chain before recording "X has no Y". This single wrong entry created the belief that "things that show a prompt cannot move", which shaped an entire design space toward a manager-owned architecture that was not needed.
- ✅ **A `button_device` CAN ride on a moving NPC.** Proven 28 Aug 2026: an NPC claims a tagged button and teleports it to its own transform in a loop, so the prompt travels with the character.
- ⚠️ **But not naively — the mesh blocks AI pawns.** Teleporting a button onto an NPC makes it shove the NPC around erratically (observed: NPC sprinting, NPC reversing away from the player, both stopping the instant the follow loop ended). **Cause, confirmed by reading the components:**
  - `SphereCollision` is **innocent** — `collisionEnabled: NoCollision`, responses all `ECR_Ignore` except `FCC_Trace_Interaction: ECR_Block`. It is the interaction trace volume, nothing physical.
  - **`ButtonMesh` is the culprit** — `collisionEnabled: QueryAndPhysics`, profile `FortBuildingMeshPhysics`, and critically **`FCC_Object_PawnAI: ECR_Block`**. It physically blocks AI pawns.
  - (`bCanEverAffectNavigation: false`, so it is *not* a navmesh problem — a plausible theory that turned out wrong.)
  - ✅ **The fix: set the device's `visibleDuringGame` to `false`.** Removes the mesh, removes the blocking, removes the geometry sliding through the terrain, and leaves a clean native prompt.
- ⭐ **`interactionRadius` IS IN METRES, and multiplies by 100 into the collision sphere.** Setting `interactionRadius := 250.0` produced `SphereCollision.sphereRadius = 25000` — a **250-metre** interaction volume. The default of `0` yields a `sphereRadius` of just **25 cm**, which is why an on-NPC prompt feels almost unreachable. **Sane value: 1.5–2.0 (= 150–200 cm).** Always read `sphereRadius` back after changing it. **Settled value for an on-NPC prompt: 0.75 (= 75 cm).** ⚠️ **Superseded 30 Aug 2026:** raised to **0.9** as latency compensation — the fatter trace sphere absorbs the remaining grant latency inside the player's own approach time. Accepted as slightly over-generous; watch for adjacent-NPC prompts in tight clusters and dial back to 0.8–0.85 if seen.
- ⭐ **Prompt stutter when a device follows an NPC is a platform characteristic, not your bug.** A device teleported to a moving NPC's transform tracks it in visible steps even at per-frame (`Sleep(0.0)`) update rates. **Epic's own wildlife pickup prompt behaves identically** — observed while following a chicken, which players can pick up and carry. **You are at parity with Epic's own implementation; do not spend an evening chasing this.** If it ever needs to be genuinely smooth, the answer is Scene Graph's `interactable_component` — which lives *on* the entity and so cannot lag behind it — not a faster teleport loop. *(See [04-scene-graph-and-prefabs.md](04-scene-graph-and-prefabs.md) for why that door is closed for Actor-geometry NPCs.)*
- **`button_device` surfaces useful authoring hooks** (read from a placed instance, 27 Aug 2026): *User Options* → `Interact Time`, `Activating Team`, `Trigger Sound`, **`Interaction Text`** (the on-screen prompt string); *User Options – Events* → **`On Interact`** (the binding target for `DeviceToolset.AddEventBinding`); *Functions* → `Enable` / `Disable`. Components: StaticMesh, Doorbell, SphereCollision, RangeSphere, ButtonMesh.
- `trigger_device` unsuitable for the proximity-prompt use case (evaluated and rejected).

## Tag discovery — FindCreativeObjectsWithTag

- ✅ **`FindCreativeObjectsWithTag` WORKS INSIDE `npc_behavior` — proven live 28 Aug 2026.** An NPC discovered a tagged level device at runtime. Epic ships a **first-class overload on `npc_behavior`**, documented *"will not find anything if called on an `npc_behavior` that is not simulating"*. **This is the sanctioned way to reach level devices from a sandboxed NPC** — discovery at run time, not wiring at author time.
- ⚠️ **The trap that likely cost the original project an evening.** Calling it without `using { /Fortnite.com/Devices }` fails with **"Unknown identifier `FindCreativeObjectsWithTag`"** — which reads exactly like a sandbox restriction and is nothing of the kind. It is a missing import. Add the using; no `Self.` qualification is needed.
- **The working recipe:** `using { /Fortnite.com/Devices }` + `using { /Verse.org/Simulation/Tags }` → declare `my_tag := class(tag){}` → **compile** (the tag class only becomes selectable in the editor after it builds) → on the target device add a **Verse Tag Markup Component** and assign the tag class *inside* it → call unqualified from the behavior.
- ⚠️ **UEFN does not use Unreal Actor Tags for this.** There is no "Tags" field on a device's Details panel. Verse tags come from the **Verse Tag Markup Component**, which is an empty container until a tag type is put in it. Adding the component alone does nothing.
- **CORRECTION to an earlier "deprecated but functional" note:** it is the **tag-instance** overload that is deprecated (*"use corresponding extension methods that take a tag type instead"*), and that deprecated form exists only for `creative_device`/`creative_device_base`. On `npc_behavior` **only the current tag-type form exists**. The modern API, not borrowed time. `entity` has the same method — that is the Scene Graph path later.
- ⚠️ **It returns *every* object with that tag**, not "the one near me". Per-NPC association (nearest, or distinct tag types per NPC) is a design problem the API does not solve for you. *(See the manager pattern in [02-npcs-and-ai.md](02-npcs-and-ai.md).)*
- ✅ **Deleting a tag class does not leave a dangling pointer.** Removing a tag class from Verse left the Verse Tag Markup Component on the device **simply empty** — no broken reference, no build error (verified 28 Aug 2026). The component can safely outlive the tag class that populated it.
- ✅ **PROVEN IN EFFECT 28 Aug 2026 (shipped in production).** `FindCreativeObjectsWithTag` returns a **`creative_object_interface`**; a working build discovers the tagged button, teleports it, and reacts to the interaction from inside `npc_behavior`, so the cast-down to `button_device` and the subscription both work in practice. *Exact cast form and `InteractedWithEvent` await recipe still to be transcribed here — flagged 29 Aug 2026 so the next reader does not re-test a settled capability.*

## Events & concurrency

- `Subscribe()` works only on built-in device events; custom events use the `event()` / `Await()` pattern.
- `race{}` runs multiple `suspends` branches concurrently and cancels the rest when one completes — the standard pattern for wander-loop vs interaction-loop.

## The runtime-device constraint (why pooling exists)

- ⚠️ **Devices CANNOT be created at runtime.** Digest-verified: every `Spawn*` API is a method on an already-placed device; only props (`SpawnProp`) and NPCs (spawners) exist dynamically. **Device count is fixed at edit time** — so any per-NPC-device pattern caps group size at wiring effort. Pool devices by *concurrent use* (≈ player count) instead: *a prompt only exists while an NPC has stopped for a player, and a player can hold only one interaction at a time* — so needed devices = concurrent interactions ≈ **player count, not NPC count**.

## HUD & map devices (29 Aug 2026)

- ✅ **`tracker_device` is the native quest HUD** — "description N/target" in the corner, no custom UI. `AssignToAll()`, `Increment(Agent)`, `IncreaseTargetValue(Agent)`, `CompleteEvent`. Needs the *agent who acted* — note `button_device.InteractedWithEvent` sends it.
- ✅ **`map_indicator_device`** marks the map, inherits `TeleportTo` (rides the follow pattern fine), `Enable`/`Disable`, plus **`ActivateObjectivePulse(Agent)`** — a native per-player trail pointing toward the device. **Display cleanup:** the device defaults to two "A" icons — a world-space icon and a map icon. **Set both default icons to None** to leave just the clean red minimap dot; an optional text label also exists.

## ⭐ `teleporter_device` — and the three different meanings of "off" (7 Sep 2026)

**`Teleport(Agent)` pulls an agent TO the device.** So a teleporter used purely as an *arrival point* needs no linking, no groups, no partner — place it where players should land and call it. It is **per-agent**, so co-op is just a loop over `GetPlayspace().GetPlayers()`. (`Activate(Agent)` is the other direction — sends the agent to *its* target group.)

### ✅ The recipe for a destination-only teleporter

1. **Teleporter Target Group → None.** With nowhere to send anyone, walking in does nothing — while `Teleport(Agent)` still delivers players to it. **This is the working answer.**
2. **Teleporter Rift Visible → unchecked.** Cosmetic tidy-up, safe *once step 1 is done*.

### ⚠️ Two approaches that FAIL, both tested — and they map three distinct meanings of "off"

| Approach | Effect | Verdict |
|---|---|---|
| `Disable()` in Verse | Kills the **whole device**, destination included — `Teleport(Agent)` silently stops working | ⛔ Too blunt |
| "Rift Visible" unchecked, alone | **Cosmetic only.** The device is still walk-in-able — a tester wandered into one they could not see and was teleported | ⛔ *Worse* than visible: an invisible trap |
| `OverlapCapsule.bGenerateOverlapEvents = false` | The 22 cm entry capsule stops reporting overlaps — **and entry still worked** | ⛔ Not the entry path |

⭐ **The lesson, which generalises past teleporters:** *"off"* is not one thing on a UEFN device. **Disabled** (dead to everything, Verse included), **invisible** (purely cosmetic), and **deaf to overlap** (component-level) are three separate axes — and on this device *none* of them is the one that stops a player using it. **The functional switch was a user option in the Details panel, not an API call or a component flag.** Check the Details panel's own options before reaching for Verse or component surgery.

*Method note: the failed capsule experiment was reverted (`bGenerateOverlapEvents` back to `true`) rather than left in place — a setting that changes nothing is worse than no setting, because the next person has to work out why it is there.*

## 🖥️ UI: HUD devices vs Verse-authored widgets (16 Sep 2026)

Three ways of putting information on screen, auditioned in one build, keeping the best of each.

- ⛔ **`tracker_device` (the blue quest box) IGNORES Verse `Increment` unless Stat to Track is None.** Placed fresh, the device defaults to **`statToTrack` = "Eliminations"**, which counts player-vs-player kills only. Symptom: the box sits at 0 forever while `Increment(Agent)` is called on every creature kill; `IncreaseTargetValue` still works, so the target climbs and the progress never does. **Fix: `statToTrack` = "None"**, then Verse drives the value entirely.
- **Useful tracker settings** (all writable over MCP with `ObjectTools.set_properties`): `showProgress` = **Remaining** counts *down* ("3 remaining") instead of up; `trackerCompletionCeremony` = false kills the fanfare; `whenTargetIsReached` = "Do Nothing" stops a completed tracker ending the round; `trackerTitle` / `descriptionText` are the two lines of HUD text; `sharing` = Individual / Team / All.
- ⚠️ **A completed tracker stops counting and shows a green tick, and `Reset` does not re-arm it** — neither did `RemoveFromAll()` + `AssignToAll()`. Counting *down* via Remaining sidesteps it, because reaching 0 left is the wanted end state anyway.
- **`hud_message_device` has no size control** — only `Show(Message, ?DisplayTime)` and `SetText`. For a big centre-screen announcement, a Verse `text_block` at `DefaultTextSize := 140.0` is ~5× what the device shows, and `SetVisibility(widget_visibility.Collapsed)` after a `Sleep` hides it again. Run it in a `spawn{}` so the surrounding loop isn't delayed.
- ⚠️ **Verse widget positioning is fussier than it looks.** The same `text_block`, in the same canvas as a banner that rendered perfectly, was **invisible** at `X := 0.5, Y := 0.04` (behind Fortnite's own top-centre compass) *and* at `X := 0.96, Y := 0.10, Alignment := (1.0, 0.5)`. Giving it the identical slot shape to the working banner (centred, `Alignment := (0.5, 0.5)`) made it appear immediately. **Debugging order that worked:** a `Print` in the build function to prove the widget is created at all, `DefaultText` at construction so it never depends on a later refresh, then copy a slot that already renders.
- **Don't forget the refresh calls:** a score that only updates in one handler shows a stale number everywhere else.
- ⭐ **`map_indicator_device`: disabling it does NOT clear the objective pulse.** The on-screen arrow keeps pointing at a finished objective until `DeactivateObjectivePulse(Agent)` is called **per player**; `Disable()` alone only removes the map icon. Symptom: *"I collected all the loot and the arrow still pointed at the empty location."*
- ⭐ **Map indicators can be told apart by colour and label:** `text`, `iconColor` and `textColor` are all writable over MCP as LinearColor `{r,g,b,a}` — so different objective types can read differently at a glance. `glowColor` exists but refused to be set over MCP. Also there: `showOnWhichMap` (Minimap / Overview Map / Both), `iconScale`, and `showObjectivePulseToInstigatorOnly`.
- **`supply_drop_spawner_device` defaults to `spawnDelay` = "Game Start"** — set it to **"Off"** or the crate falls at the start of the round instead of when Verse calls `Spawn()`. `LandingEvent` is the right hook for "the marker has done its job".
- 📚 **The next step up has its own file — see [09-custom-uis.md](09-custom-uis.md)** (started 20 Sep 2026): Verse-authored widgets, UMG Widget Blueprints, and how to debug them. Epic's docs: [In-Game User Interfaces in UEFN](https://dev.epicgames.com/documentation/fortnite/ingame-user-interfaces-in-unreal-editor-for-fortnite) and [User Interface Devices in UEFN](https://dev.epicgames.com/documentation/fortnite/user-interface-devices-in-unreal-editor-for-fortnite).
