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
- ⭐ **Stale doc comment closing a live route** (8 Sep 2026, and this one was ours, not Epic's assistant) — the inverse of the confabulated rationale. The AI reasons *correctly* from a real, quotable source that happens to be out of date, and declares a working route impossible. It survives a digest search because the digest *is* the source. The tell: the verdict "blocked" was never compiled. **If a compile can settle it, compile before writing it off.** Detail below.

**Countermeasures that work:** grep the digest (`Verse.digest.verse`, `Assets.digest.verse`, `Fortnite.digest.verse`); read the toolset Python source and its `tests/` ([05-editor-and-tooling.md](05-editor-and-tooling.md)); `SearchCVars` before assuming an editor behaviour has no switch; walk the `:= class(parent)` inheritance chain before recording "X has no Y" (that one cost twice — see [03-devices-and-interaction.md](03-devices-and-interaction.md)); **and read Epic's documentation first** — see the entry below for what assuming there was none cost.

## ⚠ Seventh entry, 7–8 Sep 2026 — the stale doc comment, and it was our own agent that fell for it

Context: getting a player to *speak* to an LLM persona in a museum experiment (full account in [02a-llm-personas-and-conversations.md](02a-llm-personas-and-conversations.md)). The agent established **by measurement** that a voice channel is required and that nothing auto-creates one. Then it read `AddChatChannel`'s doc comment — *"fails if the `MaxSize` of the channel's `agent_group` is undefined"* — found `MaxSize` nowhere else in any digest, and recorded **"in-session conversation is BLOCKED on v42.10"** with a five-step argument. Steps 1–3 were measured and true. Step 4 was reasoned from the comment and never compiled.

The human refused the conclusion — *"there has to be a solution"*. Retested: build the channel, register it, it works. The `MaxSize` sentence is simply stale. Working recipe found the same night; Epic's own docs turned out to ship the exact pattern on the *second* of two pages, while the *first* page — the one that had been read — says nothing about how a player talks to the NPC.

**Why this is a distinct taxon.** Strikes one to six are inventions. This is the opposite: a faithful reading of a real source. That makes it *harder* to catch with the existing countermeasure — grepping the digest returns the wrong answer, confidently. The only thing that catches it is the empirical test the AI skipped because the source "already answered" the question.

**Two rules earned:**

1. **A doc comment is a claim, not evidence. If the route can be settled by a compile, compile it before writing "blocked."** Same rule as the `bEnablePersonaDevice` descriptor key: try the switch, don't infer from its name.
2. **Read Epic's docs first, then verify against the digest.** Both humans and agent assumed there was no documentation worth checking. There was; it would have reached the result in a fraction of the time. The docs were also *wrong* in one sentence — so neither source is authoritative alone, and the docs are still the faster starting point.

⭐ **Running score: six inventions by Epic's assistant, one over-faithful reading by ours. The human's stubbornness was the correction in both kinds.**
