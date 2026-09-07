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
