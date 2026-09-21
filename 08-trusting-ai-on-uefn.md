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
- ⭐ **Denied capability — the false negative** (20 Sep 2026, and the most expensive shape yet): asserting that a real, shipping feature *does not exist*. ⚠️ **It never fails loudly.** An invented API won't compile and you learn in seconds; a denied capability sends you off to build a laborious workaround that **compiles, runs and ships** — so nothing ever tells you it was wrong. **The tell: you are being told to do something effortful to achieve something ordinary.** Detail below.

**Countermeasures that work:** grep the digest (`Verse.digest.verse`, `Assets.digest.verse`, `Fortnite.digest.verse`); read the toolset Python source and its `tests/` ([05-editor-and-tooling.md](05-editor-and-tooling.md)); `SearchCVars` before assuming an editor behaviour has no switch; walk the `:= class(parent)` inheritance chain before recording "X has no Y" (that one cost twice — see [03-devices-and-interaction.md](03-devices-and-interaction.md)); **and read Epic's documentation first** — see the entry below for what assuming there was none cost.

## ⚠ Seventh entry, 7–8 Sep 2026 — the stale doc comment, and it was our own agent that fell for it

Context: getting a player to *speak* to an LLM persona in a museum experiment (full account in [02a-llm-personas-and-conversations.md](02a-llm-personas-and-conversations.md)). The agent established **by measurement** that a voice channel is required and that nothing auto-creates one. Then it read `AddChatChannel`'s doc comment — *"fails if the `MaxSize` of the channel's `agent_group` is undefined"* — found `MaxSize` nowhere else in any digest, and recorded **"in-session conversation is BLOCKED on v42.10"** with a five-step argument. Steps 1–3 were measured and true. Step 4 was reasoned from the comment and never compiled.

The human refused the conclusion — *"there has to be a solution"*. Retested: build the channel, register it, it works. The `MaxSize` sentence is simply stale. Working recipe found the same night; Epic's own docs turned out to ship the exact pattern on the *second* of two pages, while the *first* page — the one that had been read — says nothing about how a player talks to the NPC.

**Why this is a distinct taxon.** Strikes one to six are inventions. This is the opposite: a faithful reading of a real source. That makes it *harder* to catch with the existing countermeasure — grepping the digest returns the wrong answer, confidently. The only thing that catches it is the empirical test the AI skipped because the source "already answered" the question.

**Two rules earned:**

1. **A doc comment is a claim, not evidence. If the route can be settled by a compile, compile it before writing "blocked."** Same rule as the `bEnablePersonaDevice` descriptor key: try the switch, don't infer from its name.
2. **Read Epic's docs first, then verify against the digest.** Both humans and agent assumed there was no documentation worth checking. There was; it would have reached the result in a fraction of the time. The docs were also *wrong* in one sentence — so neither source is authoritative alone, and the docs are still the faster starting point.

⭐ **Running score: six inventions by Epic's assistant, one over-faithful reading by ours. The human's stubbornness was the correction in both kinds.**


## ⚠ Eighth entry, 14 Sep 2026 — a mechanism fitted to one data point (ours)

Question: *why does the creature spawner's elimination log stop after 2 kills?* The agent read the Verse (correct), the spawner's settings from its actor file (correct), and **the most recent round** of the log (2 events, then silence). It assembled a clean story from real, documented settings — *self-damage on spawn + a lifetime spawn limit + an invisible spawner = the device quietly destroys itself and its event dies with it* — and presented it as the likely cause. The setting was turned off. **Zero events.**

Re-reading the **whole day's** log instead of the last round settled it in one grep: four zero-event rounds had happened *before* the setting was ever touched. The event was never reliable, and a controlled test proved it (see [02-npcs-and-ai.md](02-npcs-and-ai.md)).

**Why this is a distinct taxon.** No invention — every setting was real and every docs quote accurate — and no stale source. The failure was **sampling**: a coherent mechanism fitted to the one observation that happened to be nearest, when the history that would have falsified it sat in the same file. It also *explained the specific number* (2), which made it feel more confirmed than it was. **A story that fits the detail is not the same as a story that survives the other rounds.**

**Rules earned:**

1. **Before explaining a symptom, check how consistently it happens.** "Stops after 2" was really "0–2, varying" — a different bug.
2. **Prefer the instrumented run to the theory.** Logging both routes side by side in one round answered in a single playtest what settings-reading could not.

⭐ **Running score: six inventions by Epic's assistant, two failures by ours — one over-faithful reading, one under-sampled one. Both caught by testing, not by reasoning harder.**

## ⚠ Ninth entry, 20 Sep 2026 — Epic's assistant DENIED two real features

Beginning Verse UI work from a snippet Epic's in-editor assistant produced, and asked about updating a score on screen, it maintained — repeatedly — that:

- ❌ *"everything is immutable and we cannot edit any values on any widget once they're created"*
- ❌ *"widget bindings don't exist in UEFN"*

**Both false, and both disproven the same day by building the thing.**

- ✅ **`text_base` (`/UnrealEngine.com/Temporary/UI`), which `text_block` inherits from, declares `SetText`, `SetTextColor`, `SetTextSize`, `SetTextOpacity`** and their getters. A `text_block` already on screen updates **in place**. ⭐ **What *is* immutable is `DefaultText`**, documented in the digest as *"Used only during initialization of the widget and not modified by SetText"* — **a real constraint on one family of fields, generalised into a false constraint on everything.** The seed of truth is what makes it persuasive.
- ✅ **MVVM widget bindings not only exist, they are how Epic's own `UserInterfaces_Demo` sample is built.** There is a **View Bindings** panel in the widget editor, Verse fields on Epic's own widgets, and **two entire MCP toolsets** dedicated to authoring them (`MVVMToolset`, `VerseFieldsToolset`). This is not an obscure corner — it is the headline feature of the sample project shipped to teach UEFN UI.

**The workaround it taught:** tear the widget down and rebuild it from scratch on every update. ⚠️ **That code WORKS.** It shipped and behaved correctly on screen. It is simply several times the work, rebuilds an entire widget tree per value change, and installs a wrong model of the platform in the reader's head.

**Why this is a distinct taxon, and the worst one for cost.** Strikes one to six **invent** things — the code doesn't compile and you find out in seconds. The seventh (ours) was a faithful reading of a **stale** source. This one **denies a real capability**, and the resulting workaround compiles, runs, and passes playtest. **There is no failure signal at any point.** You ship it, it works, and you never look again — the error propagates into every UI you build afterwards.

**Rule earned:**

⭐ **When an assistant says the platform CAN'T do something ordinary, disbelieve it harder than when it says it can.** A hallucinated API is self-correcting; the compiler is the countermeasure and it fires immediately. A denied capability is **self-confirming** — the workaround's success reads as evidence the denial was right.

- **The tell: being told to do something laborious to achieve something routine.** "Rebuild the whole widget to change one number" should have read as an alarm, not as an answer.
- **The countermeasure: grep the digest for the thing you would EXPECT to exist** — search `SetText`, not "can widgets be updated". Then **walk the `:= class(parent)` chain**, because `text_block` is `class<final>(text_base)` and every setter lives on the parent. *(The same inheritance-chain rule that already cost twice on `button_device` — see [03-devices-and-interaction.md](03-devices-and-interaction.md).)*
- Full working recipes for both paths are in [09-custom-uis.md](09-custom-uis.md).

⭐ **Running score: nine failures logged — seven by Epic's assistant (six inventions, one denial) and two by ours (one over-faithful reading, one under-sampled). Every single one was caught by building the thing, never by reasoning harder about it.**
