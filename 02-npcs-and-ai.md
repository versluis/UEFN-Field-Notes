---
verified-on: UEFN v41.10 – v42.10
last-reviewed-against: UEFN v42.10 (3 Sep 2026)
---

# NPCs & AI

The `npc_behavior` sandbox, animation presets, `Focus`, spawners, flocks, and the Irwin/Burt discovery. Dates on each entry. Project context: these findings come from a sheep-petting game ("the sheep" below), but the mechanisms are general.

## The npc_behavior Sandbox (the big one)

- ⚠️ **CORRECTED 26 Aug 2026 — this entry was wrong as written.** The old claim was: *`npc_behavior` cannot reference level-placed devices via `@editable`.* **It compiles.** Adding `@editable MyButton : button_device = button_device{}` to an `npc_behavior` file on UEFN v42 built with zero errors and zero warnings. The declaration is legal.
  - **What was still open** was *wiring*, not declaring. `@editable` fields on `npc_behavior` surface on the **Character Definition asset**, not the NPC Spawner — and an asset may not be able to point at a level-**placed** device instance. That is the likely origin of the original belief: the field appears somewhere it can't be usefully assigned, and "cannot be wired" gets recorded as "cannot be referenced".
  - **Do not repeat the old claim.** Say precisely which half is established: declaration ✅, wiring ❓. **RESOLVED 27 Aug 2026 — the answer is NO, and the reason matters.** Three parts: (1) ✅ declaration compiles; (2) ✅ the field surfaces, at `CD_Sheep.CD_Sheep:CharacterModifier_VerseBehavior_C_2.sheep_behavior_0` as `myButton` — note that is **two levels down**, on the *NPCBehaviorScript instance*, not on the behavior modifier, which exposes only `canSwim` and `nPCBehaviorScript`; (3) ❌ it **cannot be assigned a level-placed device** — `set_properties` fails, and the placed button actor exposes 453 properties with **zero** containing the string "Verse", so there is no Verse-layer object to bind to.

⭐ **The mechanism, which is the real finding:** the Character Definition is an **asset**; the `@editable` field lives inside that asset; **an asset cannot hold a reference to a level actor instance.** That is a fundamental Unreal rule, not a Verse or UEFN quirk. **The "npc_behavior sandbox" is the asset/instance boundary** — nothing in the Verse language forbids the reference, the asset simply cannot hold one.

**What it rules IN:** anything resolving the device at **runtime** instead of author time is untouched — tag discovery from inside the behavior, spawner events keyed on agent, manager-owned proximity. The original conclusion was right in effect, wrong in mechanism — and the mechanism is what tells you which doors are still open. ✅ **CONFIRMED AT THE UI LEVEL 27 Aug 2026.** The Button actor was selected in the level via `SelectActors` (verified with `GetSelectedActors`), then the picker was used by hand. The `@editable` row **is** visible in the Character Definition's Details panel; the BROWSE list offers **only the World**, never an actor; **"Use Selected" does nothing** and the field stays `None`; and the Button's own Details panel has no matching row either. **The soft-object-path theory does not apply here** — UEFN does not offer that escape hatch for this property. The asset/instance boundary holds. This is settled; do not re-litigate it.

- `@editable` fields on `npc_behavior` surface in the **Character Definition asset**, not on the NPC Spawner device.
- `GetPlayspace()` is unavailable inside `npc_behavior`. Correct chain: `GetEntity[]` → `Entity.GetPlayspaceForEntity[]` (needs `using { /Fortnite.com/Playspaces }`).
- Getting the character: `GetAgent[]` → `Agent.GetFortCharacter[]` — **not** the `fort_character[Agent]` cast.

## ⭐ Custom NPCs ride on Epic's wildlife system (found 27 Aug 2026)

Reading a custom NPC Character Definition's properties over MCP turned up something undocumented:

- `skeletalMesh` = **`/Irwin/AI/Base/Burt_Mammal_Invis`** — an **invisible** base mammal mesh, *not* the project's visible mesh
- `animationBP` = **`/Irwin/AI/Prey/Burt/Animations/Burt_AnimBP`** — Epic's prey-mammal Animation Blueprint
- `animPreset` = the project's Basic Locomotion Animation Preset (5 slots: idle, moveForward, moveBackward, moveLeft, moveRight)
- `characterBlueprint` = the project's Actor Blueprint — which is where the *visible* mesh actually lives

🔍 **"Irwin" is Epic's internal wildlife system; "Burt" is its prey mammal.** Fortnite's chickens run on the same machinery — which is why they have a full state machine (walk, idle, attack, eat, feather VFX, sounds) for free from one Details panel.

⚠️ **The Burt hypothesis was interesting but WRONG as a cause of the animation bug — see below.** The animation preset is the system actually driving the visible mesh, and it works fine.

**Strategic note:** the wildlife spawner delivers fully-formed NPC behaviour — locomotion, animation, interaction, audio — from a built-in device with Details-panel options (count, type, team), and swapping the type changes mesh, behaviour, animation *and* interaction together. Whatever backs it is far more capable than what you can hand-build today. Worth asking how much of it you can ride rather than reimplement.

## ✅ SOLVED 27 Aug 2026 — a months-long "animation bug" was one wrong property

The animation preset's **`moveForward` slot was assigned the idle clip instead of the walk clip.** The other three movement slots were correct all along.

The NPC walks forward → the preset plays `moveForward` → which contained the idle clip. Fixed via `ObjectTools.set_properties` on the CDO (`Default__<PresetClass>_C`) and **confirmed live in play: the NPC now transitions from idle to walking.**

**⭐ Why it survived three project versions.** `moveForward` is the *only* slot the NPC ever exercises — `NavigateTo` always drives it forward. Backward, left and right were correctly configured and **never once fired**. So the single broken slot was the single slot always in use, and the three working ones were invisible. It presented as a total animation failure while being a one-property setup slip (idle assigned first, rows duplicated down, three corrected and one missed).

**⭐ What it cost.** `GetPlayAnimationController` — engine-broken at the time — was **never the blocker for locomotion animation.** The commit that "ruined everything" and parked the project was solving a problem that did not need Verse at all. **The preset already handles locomotion.**

**⭐ The rule earned:** *check the animation preset's slot assignments before writing a line of Verse animation code, and read every slot — not just the one you think is wrong.* A correct-looking preset with one bad row is indistinguishable from a broken engine API until you actually read the values.

- ~~**Known follow-up:** movement speed outruns the walk cycle (foot-skating).~~ ✅ **DONE (28 Aug 2026).** Fixed the honest way, via a `CharacterModifier_Movement` on the Character Definition rather than fudging `playRate`: **walkSpeed 2.0 → 1.0 m/s, runSpeed 4.1 → 1.5, acceleration/deceleration 20.5 → 8.0.** Foot-skating resolved, confirmed in play. ⭐ **Note the units: those speeds are METRES per second**, not cm — the modifier's defaults (2 / 4.1 / 5.5) are human-scale figures.
- *(Superseded to-dos, kept for the record: the CDO path worked — `get_properties` on a Blueprint class fails; use `Default__<Class>_C`, which is how the wrong slot was found and fixed. A planned chicken-comparison test became unnecessary once the preset slot explained everything.)*

## Engine bugs (not your code) — and their expiry dates

- **`GetPlayAnimationController` was non-functional** for custom animation playback. Broken since ~v40.30, confirmed still broken at v41.10. ~~Do not burn session time on it; check release notes each UEFN update.~~

✅ **FIXED ON v42 — CONFIRMED WORKING 28 Aug 2026. This entry is retired.** Live test: seven consecutive cycles, `GetPlayAnimationController[]` succeeded **every** time (never once the failure branch), and `PlayAndAwait` completed cleanly ~3.3 s later on each. Random selection across four clips observed varying correctly. **Verse-driven animation playback is available again.**

⭐ **The lesson to carry forward:** this API was recorded as broken for roughly two engine versions, was blamed for parking a project, and kept a design goal marked *blocked* for months — and **nobody re-tested it after an engine jump.** An "engine-blocked" note is a claim with an expiry date. Re-test them on every major version bump; it cost ten minutes here and returned a capability the project had written off.

- ⚠️ **A cautionary provenance tale (flagged 26 Aug 2026):** the claim *"`npc_behavior` cannot reference level-placed devices via `@editable`"* — the premise an entire build spec rested on — **had no code artifact behind it anywhere in project history.** No commit ever attempted a device reference. The constraint is widely reported in the UEFN community and turned out true in effect, but **the project had never proved it.** Five minutes of testing settled it (see the Sandbox section above). A version number or a "known constraint" that isn't tied to a verified artifact is a label, not a finding.

## ⭐ The blocking-call-in-a-polled-loop trap (30 Aug 2026 — bit twice in one night)

**A `<suspends>` call placed ahead of a time-sensitive branch in a polled loop silently gates everything after it.** Symptoms look like flakiness, not like blocking:

- A full `PlayAndAwait` idle (~3.3 s) + settle sleep ran before the loop could reach the interaction-request branch → "the prompt appears 1 time in 5."
- A blocking `Focus` call ran before the request → prompt delayed by the turn animation — and for a player the NPC cannot perceive (approached from behind), **`Focus` never completes**, so the branch behind it never ran at all.

**Rules:** (1) in a loop that polls for player-facing state, nothing `<suspends>` goes in front of the responsive branch — `spawn{}` the cosmetic work instead; (2) **any awaited NPC action that can fail to complete gets a `race` against a `Sleep` timeout**, or every trigger leaks a permanently-parked task — invisible at 5 NPCs, a mystery at 50; (3) check whether an animation-preset layer already covers the visual gap the skipped call leaves.

- Related: **cleared `@editable` array slots are default-constructed device references** — a reference to nothing that grants and teleports into the void. Trim the array; never leave cleared entries.

## NPC actions — Focus (✅ WORKS, once the cadence is right — 29 Aug 2026)

⭐ **Rejected, then rescued by changing *when* it is called rather than *what* it does.** Called every 0.25 s in a pause loop, the NPC re-aimed continuously and spun on the spot as the player circled it — which read as broken. Called **once**, on the transition into "a player has arrived", it reads as a genuine reaction. **The mechanic was never wrong; the cadence was.**

**Pattern:** a `var HasNoticedPlayer : logic` on the behavior class, set on arrival and reset in the navigation branch, so each fresh approach earns a fresh look and each NPC tracks its own state independently.

⭐ **Emergent bonus nobody coded: `Focus` respects the NPC's field of view.** A pure-distance proximity check with no angle test still produced directional behaviour in play: the NPC only notices a player approaching **from roughly 90° either side** — approach from directly behind and it does not react at all. That directionality comes from the NPC's own perception system. **The `CharacterModifier_Perception` sight settings appear to gate `Focus` targets**, meaning the awareness system does useful work without being explicitly wired. Remember this before hand-rolling any field-of-view logic.

**Turn rate judged "just right" unmodified** — no tuning needed.

### Caveats that still stand

- **`Focus` rotates the whole body** — no head-look or aim offset. Acceptable as a single turn; unacceptable as continuous tracking, which is the whole lesson above.

### Original write-up, kept for the record

- ❌ **`npc_actions_component.Focus` rotates the WHOLE BODY — there is no head-look or aim offset.** On a grazing quadruped the entire mesh spins on Z as the player circles it, which reads badly. Fine for a biped turning to face you; wrong for an animal with its head down.
- Two overloads: `Focus(Location, ?LockFocus)` and `Focus(Target:entity, ?LockFocus)`. ⚠️ **The location overload takes a `/Verse.org/SpatialMath` vector3 — NOT the `/UnrealEngine.com/Temporary/SpatialMath` vector3 used in device code.** Use the **entity** overload to sidestep that collision; `fort_character` exposes `GetEntity[]`, so a player's entity is easy to reach.
- **If revisited:** call it **once** when the player first arrives, not every loop tick. Continuous re-focus is what produced the spinning.
- ⚠️ **`entity` as a named type requires `using { /Verse.org/SceneGraph }`.** `GetEntity[]` works without it — the import is only needed when you *write the type out* (e.g. `?entity` in a signature). The compiler's error names the missing using, which is a nice touch.

## Flocks, spawners and coordination (29 Aug 2026)

- ⭐ **One spawner with a raised count beats multiple spawners.** `Device_CharacterSpawner` has `spawnCount` and `totalSpawnLimit` (both default **1**) plus `spawnRadius` (**metres** — set ~10 so NPCs don't spawn stacked). This is how Epic's own wildlife spawner works. One place to change group size, and `npc_spawner_device.GetAgents()` hands you the whole group.
- **Useful `npc_spawner_device` API:** `SpawnedEvent:listenable(agent)` (fires once per spawn), `GetAgents():[]agent`, `SpawnAt(Position, ?Rotation)<suspends>:?agent`, `Spawn()`, `DespawnAll(?agent)`.
- No direct NPC despawn/destroy API. Working pattern: `NPC.Damage(100000.0)` — death triggers despawn. Spawner-side event: `npc_spawner_device.EliminatedEvent.Await()`. (A cleaner lifecycle exists at the spawner level: `spawnOnEnabled` plus `despawnAIsWhenDisabled`, so `Enable()`/`Disable()` become spawn/despawn for a whole area.)
- `NavigateToRandomLocation` does not compile in current UEFN — use manual polar-to-cartesian offset math for random wander destinations.
- ⭐ **Per-NPC association needs a manager — there is no way for NPCs to coordinate between themselves.** `FindCreativeObjectsWithTag` returns *every* tagged object, so N NPCs would all claim the same device. And **a device cannot be tagged at runtime**: only `entity` implements `has_tags`; `creative_object_interface` implements `positional` instead, so the "tag it as claimed" trick is impossible.
  - **Working pattern:** a `creative_device` manager holds `@editable []button_device`, hands them out one at a time, and counts. The NPC finds the manager by tag, casts to the custom class, and calls it. **The NPC keeps owning its own interaction; the manager only does what a lone NPC provably cannot — allocate and count.**
- ⚠️ **Simultaneous claims — partially derisked, not closed.** Originally: one spawner staggers its batch ~3 s apart, so claims serialise with room to spare — spawn timing doing the work, not the code. *(Update, Sep 2026: a two-spawner test on the pooled-device architecture came back clean — ~300 ms interleave, no double-claims, no missed counts.)* ⚠️ **Still not fully closed:** 300 ms is ten times tighter than the single-spawner stagger and survived cleanly, but it is **not same-frame**. Two spawners firing in one tick, or `SpawnAt` placing a whole group at once, remains untested. **Observed clean at observed timings ≠ proven safe.**
