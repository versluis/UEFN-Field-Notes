---
verified-on: UEFN v42.00 – v42.10
last-reviewed-against: UEFN v42.10 (3 Sep 2026)
---

# Scene Graph & Prefabs

Entities, components, the one-component rule, the working prefab recipe, and what's closed and why.

## Scene Graph — availability (checked 28 Aug 2026)

- ✅ **`interactable_component` and `basic_interactable_component` are NOT experimental.** They live in the standard `/Verse.org/SceneGraph` module in `Verse.digest.verse`. **Scene Graph System** ticks on under **Project Settings → Beta Access**, alongside Python Editor Scripting and Custom Items and Inventory.
- ⛔ **Do NOT tick "Scene Graph Experimental Features".** That list is a *different* set (`ability_effect_component`, `camera_director_component`, `fort_projectile_component`, `orthographic_camera_component`…) and enabling it **"will disable all Verse Error/Warnings for experimental API usage"** — which silently destroys the value of `BuildAll` diagnostics. It also makes the project **unpublishable**, possibly irreversibly.
- ⚠️ *"Disabling Scene Graph will not remove any content added while enabled."* The toggle is not a clean undo. **Revision control is the escape hatch, not the checkbox.**
- **What `basic_interactable_component` offers** (all `@editable`, no Verse logic required): `SuccessLimit` (interactions stop after N successes), `Cooldown` / `CooldownPerAgent`, `InteractableDuration` (hold-to-interact), `CanInteractMessage` / `CannotInteractMessage`, plus `StartedEvent` / `SucceededEvent` / `CanceledEvent` and `Enable`/`Disable` (*"disabled components do not provide interaction prompts"*).
- **It attaches to an entity**, and `entity.AddComponents([]component)` is public — with `npc_behavior.GetEntity[]` proven, an NPC can be handed its own interaction at runtime. **No actor, no physics body, nothing to teleport.**
- ⚠️ **Conflict flagged 29 Aug 2026, before testing:** Epic's *Interactable Components* doc states the `interactable_component` **needs to be attached to a `mesh_component` to work.** *(Resolved 29 Aug — the mesh requirement turned out to be the whole story; see the CLOSED verdict below.)*
- ⚠️ **Community report (Feb 2025, older UEFN):** spawning an entity from Verse and adding a transform + interactable component at runtime **crashed the server**. Age and version make it a weak signal, but: **commit before the first AddComponents attempt**, and expect the first failure mode to be a dead session rather than a compile error.

## ⛔ Scene Graph interactables are CLOSED for Actor-geometry NPCs (settled 29 Aug 2026)

**The chain, and where it breaks** (tested on an NPC built via the Character Definition pipeline):

1. ✅ The NPC **is** a Scene Graph entity — 10 components, including `npc_actions_component` and `npc_awareness_component`
2. ✅ `basic_interactable_component` can be constructed and added to it at runtime
3. ❌ But `interactable_component` **has no shape of its own** — nothing defines *where* you interact
4. ❌ Scene Graph has **no collider components**; geometry comes only from `mesh_component`
5. ❌ `mesh_component` is **`epic_internal`** — you cannot construct one from Verse
6. ❌ A Character-Definition NPC's geometry is **Actor-side (its Blueprint)**, invisible to Scene Graph

So the component attaches to an entity that has nothing to interact with, and there is no way to give it any. **This is a real access boundary, not a missing `using`.**

⭐ **The correct split, stated for the record:** Scene Graph's **NPC brain** is reachable today (`Focus`, `SeeTargetEvent`, `Idle`, `NavigateTo` with a result). Scene Graph's **interaction system** is not, and cannot be, on an Actor-geometry NPC — it requires authoring the NPC as a Scene Graph entity from the start, which per Epic's roadmap ([00-version-watch.md](00-version-watch.md)) is a Q4 2026+ proposition. **Until then, a device-based prompt (pooled) is the only prompt available on such NPCs. Do not re-test this without a version change.**

## Reusable multi-mesh constructs — what UEFN actually offers (charted 6 Sep 2026, at the cost of four AI fictions)

The question "how do I make a prefab-like fence section?" has one working answer today and two closed doors:

- ✅ **`Group` (Ctrl+G) + duplicate — the tool that exists.** Move/duplicate/select as one. NOT linked — edits don't propagate to copies. Sufficient for set dressing.
- ⛔ **Level Instances are NOT in UEFN** — despite being real in *both* neighbours: UE5 has the right-click → Create Level Instance flow, Fortnite **Creative** has a Level Instance *Device*; UEFN has neither (standing feature request on the forums). The right-click menu simply has no Level section. **A feature that is real-everywhere-but-here is prime AI-hallucination bait** (see [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md)).
- ⛔ **Marketplace Blueprints** — unshippable (see [07-imports-and-validation.md](07-imports-and-validation.md)).
- ✅ **THE WORKING PREFAB RECIPE — proven end-to-end 6 Sep 2026 (human authoring, agent stamping):**
  1. **Place Actors → Entity** (empty), twice: a parent and a child. **Entity-onto-entity drag in the Outliner parents fine** — it is *actor*-onto-entity that red-icons ("actor cannot be placed on this target"): two different object worlds.
  2. On each child: **Details → + Add Component → `mesh_component`**, pick the mesh. The editor adds `epic_internal` components freely — only Verse construction is barred.
  3. ⚠️ **The mesh choice is permanent** — once a `mesh_component` has its mesh, it cannot be changed (unlike a Blueprint component's mesh slot). Wrong mesh = delete the child, make a new one.
  4. Duplicate children to build the assembly; **adjust the pivot (middle-mouse-drag) and save it as the offset** so instances tile.
  5. **Save As Prefab** on the parent → registers as a class (`ListEntityClasses` shows it `bIsPrefab: true` beside Epic's own).
  6. **Instances stamp programmatically:** MCP `EntityToolset.CreateEntity` with the prefab class + world transform placed working copies at exact spacing — perimeter fencing becomes a loop, not an afternoon.
  - ⚠️ **Prefab instances are NOT cheap at scale — first performance signal, 6 Sep 2026.** Fencing two 68×105 m areas took ~172 fence-prefab instances (each containing several `mesh_component` children, so ~700–1000 meshes). **UEFN visibly struggled during bulk duplication.** Mitigations, cheapest first: (1) **nest** — make a prefab of a whole finished run/area and duplicate *that* (one edge → prefab → rotate+duplicate → whole area → duplicate; 3 gestures instead of 129 placements); (2) fence only player-visible edges; (3) reduce panel count with a longer section prefab. Note the flip side: **the editor's *content* budget stayed at 1%** after all 172, because instances reuse the same meshes — content/disk budget and runtime memory are different budgets.
- ⭐ **Editing the prefab asset updates every placed instance.** Fence too tall? Edit the prefab once; all 172 follow. This is the linked-editing behaviour Blueprints and Level Instances could not provide here — and the reason to prefer prefabs over duplicated raw meshes for anything repeated.
- ⭐ **Division of labour that actually works:** MCP tooling for *precise* placement (exact spacing, derived offsets, verified transforms); the **editor** for *bulk repetition* (select → duplicate → move by a known offset). 43 panels placed by tool established the pattern and the corner; the remaining 129 were three editor gestures. Neither side should grind at the other's strength.
- ℹ️ Imported-mesh material slots keep their source-DCC names — one imported pack's are Italian (*Viola* = purple, alongside Sporco/Erba/GranoChiaro). Not a bug, an author.
- 🔶 **Scene Graph Entity Prefab Definition — the Content-Browser-first variant, half-charted:** the ECS rule holds (**one component of a type per entity** — the Add/Duplicate menus grey out after one `mesh_component`), so multi-mesh = **child entities**, each with one mesh. The prefab editor's mesh picker works; **the UI for adding child entities inside a prefab is still unfound** — the open edge. Once authored, prefabs register in `ListEntityClasses` with `bIsPrefab: true` and are stampable via MCP `CreateEntity`. ⛔ **Building the hierarchy from Verse is impossible**: `static_mesh_component` (an AI fiction — see [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md)) doesn't exist, and the real `mesh_component` constructor is `epic_internal`.
- Epic's own NPC prefabs (`Custom_NPC_Prefab_Entity`, the Irwin family: Burt, Grandma, Nug, Robert, Smackie) confirm the pattern is load-bearing — the NPC spawner has been stamping a prefab all along.
