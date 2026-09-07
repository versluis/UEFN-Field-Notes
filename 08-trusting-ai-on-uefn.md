---
verified-on: UEFN v42.00 – v42.10
last-reviewed-against: UEFN v42.10 (6 Sep 2026)
---

# Trusting AI on UEFN

The strikes and the taxonomy. AI suggestions about UEFN APIs — including Epic's own in-editor assistant, and **including this repository** — are hypotheses until the digest confirms them.

## ⚠️ On trusting AI suggestions about UEFN APIs (28 Aug 2026)

Epic's own in-editor AI was consulted about a device-collision problem. Scored honestly:

- ✅ **Right on the diagnosis** (device collision) and ✅ **right on the editor settings** — `visibleDuringGame` and `interactionRadius` are real and were exactly the levers.
- ✅ `Sleep(0.0)` for per-frame following is sound.
- ✅ The Scene Graph `Interactable` pointer was the most valuable thing it said.
- ❌ **`DisableCollision()` does not exist on `button_device`** — it belongs to `video_player_device` alone. The suggested code would not compile.
- ❌ **There is no "Interaction Button Device."** The nearest thing is `skilled_interaction_device`, a different mechanic entirely.

**Third strike, 6 Sep 2026:** Epic's AI produced a complete, well-commented Verse "workflow" for multi-mesh prefab construction built on **`static_mesh_component`** — **a class that does not exist in any digest.** The real class (`mesh_component`) has an `epic_internal` constructor (compiler-proven 29 Aug), so the code is unbuildable under any name. Tell-tale: it claimed runtime-spawned children would be "configurable in the Details panel" — impossible for entities that don't exist until simulation. *The surrounding real APIs it cited (`transform_component.LocalTransform`, `MakeRotationFromEulerDegrees`, `/UnrealEngine.com/BasicShapes`) all checked out — fluent accuracy around a fictional core is the signature failure mode.*

**Fourth strike, 6 Sep 2026 — and the worst one, because it DOUBLED DOWN.** Told that UEFN's right-click menu had no *Create Level Instance*, Epic's AI explained *why*: "The option is often hidden in UEFN because Epic is moving toward Scene Graph… **Level Instances are still functional for purely visual set-dressing.**" They are not present in UEFN at all (real in UE5, real in Fortnite Creative, absent here — standing forum request). **It invented a plausible rationale for the absence of the thing it had invented.** That is more corrosive than the original error: it converts "I can't find it" into "I must be looking wrong."

**Fifth strike, same conversation:** an "alternative" manual workflow — *"Place your individual Static Mesh Entities in the viewport, drag them onto the Parent Entity in the Outliner."* There is no such placeable thing: everything dragged from the Content Browser becomes a **`FortStaticMeshActor`**, and the Outliner correctly refuses it with *"actor cannot be placed on this target"*. Actors and entities are separate object worlds — the same boundary that closed the Scene Graph interactable route ([04-scene-graph-and-prefabs.md](04-scene-graph-and-prefabs.md)).

**Sixth strike:** asked how to restore the (non-existent) Level Instance options, it answered *"enable the appropriate plugin under Edit → Plugins"* — **a menu UEFN does not have.** The conversation was closed there.

⭐ **Running score across one week: six confident answers, six different failure shapes, one right-click menu as the only honest witness.**

**Rule: treat any AI-suggested UEFN API as a hypothesis and grep the digest before building on it** — including suggestions from Epic's own assistant, and including these notes. Verification costs seconds; a hallucinated API costs a session.

## The taxonomy of failure modes seen so far

- **Fictional API on a real problem** — `DisableCollision()` on `button_device`; `static_mesh_component`. The code reads perfectly and cannot compile.
- **Fictional device/feature** — "Interaction Button Device"; Level Instances in UEFN (real in UE5 *and* Fortnite Creative, absent in UEFN — **a feature that is real-everywhere-but-here is prime AI-hallucination bait**; see [04-scene-graph-and-prefabs.md](04-scene-graph-and-prefabs.md)).
- **Fluent accuracy around a fictional core** — the surrounding real APIs all check out, which is exactly what makes the fiction convincing.
- ⭐ **Confabulated rationale for its own error** — explaining *why* the invented feature is missing/hidden (strike four). **The most dangerous mode**: it reframes your correct observation as your mistake, and it survives a docs search because the docs for the *other product* exist. **Trust the menu in front of you over any explanation of why the menu is wrong.**
- **Fictional UI path** — "Edit → Plugins", a menu UEFN does not have (strike six). Menus are the cheapest possible thing to verify: look.
- **Impossible editor claims** — "runtime-spawned children configurable in the Details panel": a statement that cannot be true of entities that don't exist until simulation. Claims about *when* something exists are worth extra suspicion.

**Countermeasures that work:** grep the digest (`Verse.digest.verse`, `Assets.digest.verse`, `Fortnite.digest.verse`); read the toolset Python source and its `tests/` ([05-editor-and-tooling.md](05-editor-and-tooling.md)); `SearchCVars` before assuming an editor behaviour has no switch; walk the `:= class(parent)` inheritance chain before recording "X has no Y" (that one cost twice — see [03-devices-and-interaction.md](03-devices-and-interaction.md)).
