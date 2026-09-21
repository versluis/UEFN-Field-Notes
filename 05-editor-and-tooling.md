---
verified-on: UEFN v41.10 – v42.20
last-reviewed-against: UEFN v42.20 (20 Sep 2026)
---

# Editor & Tooling

CVar search, the MCP toolsets and their Python source, agent skills, editor hygiene, VS Code setup, and the coordinate mapping.

## Editor & project hygiene

- **All asset file operations (create, delete, rename, move) happen inside UEFN.** VS Code and Explorer are safe for editing file *contents* only. Violating this split broke module references and killed an entire project version.
- **MCP file operations are *inside* UEFN and are therefore sanctioned** (clarified 26 Aug 2026). `VerseToolset.Move` / `Delete` / `CreateDirectory` run in the editor's own process and keep module references intact. The rule above bans **VS Code and Explorer**, not tooling that goes through the editor.
- Validate assets exist in `Assets.digest.verse` **before** writing code that references them.
- UEFN 5.8+: the Fortnite device panel lives in the **Content Drawer → Fortnite** section (the top menu was removed).
- The Meadow Island template ships with **no nav mesh** — one workaround is a wildlife spawner placed to force nav mesh generation. ⚠️ **The locality half of this theory was disproven 30 Aug 2026:** NPCs in a second area navigate with **no wildlife spawner anywhere nearby**, so whatever the spawner does, it is not a local effect. The decisive test — delete the spawner entirely and see — is deferred (deleting it also deletes the wildlife it spawns).

## Login stall on first launch after an update

**v42.00 field note (26 Aug 2026):** first launch after the update stalled at "90% — Logging in.." with no error. Known long-running issue class, actively reported on v42; documented pattern is stall → crash/kill → relaunch works instantly. Mitigations that helped others: run as administrator; launch UEFN with Fortnite fully closed; check [status.epicgames.com](https://status.epicgames.com). *Resolution verified working:* **(1)** Start Fortnite once — just to the main menu, logged in, no play needed. **(2)** Close *everything*: Fortnite, UEFN, and the Epic launcher itself. **(3)** Task Manager check: no residual Fortnite/UEFN/Epic/BattlEye processes lurking. **(4)** Restart the launcher, then UEFN → loads through to project selection. The Fortnite-first step provisions the profile; the full teardown clears whatever stale session was jamming the 90% login. Running as admin was not required in the end.

## VS Code setup

- **First open in VS Code must come from UEFN's toolbar button (left of "Compile Verse").** That first open installs and enables the Verse extension and the revision-control extension. Opening the project folder directly in VS Code beforehand gives plain text with no highlighting or diagnostics and no explanation. Neither extension is on the VS Code marketplace — they ship inside the UEFN install (v42, 29 Aug 2026, clean-machine setup).
- ⭐ **The Verse build server is a toggle in VS Code (v42, found 29 Aug 2026).** The Unreal Engine icon at VS Code's top right is a button, not decoration. Click → grid icon = build server active; hover shows last result; click again = build; red badge = built with errors; results land in the Problems panel. **UEFN must be running with the project open** — VS Code is talking to the editor, not compiling on its own. Equivalent to UEFN's Push Verse Only (F6) / Refresh Session (F5).

## ⭐ The MCP toolsets are Python, and the source is ON DISK (found 6 Sep 2026)

Find the UEFN install via the running process path (it is not necessarily in Program Files). The MCP toolsets whose names are lower_snake_case (`editor_toolset.toolsets.object.ObjectTools`) are **Python modules you can read**:

`<UEFN install>\Engine\Plugins\Experimental\Toolsets\EditorToolset\Content\Python\editor_toolset\`

- `toolsets/` — actor, asset, blueprint, primitive, scene, material, static_mesh, texture, data_table… **the real signatures**, e.g. `ActorTools.add_component`, `set_actor_transform`, `get_components`, built on `unreal.SubobjectDataSubsystem`.
- `tests/` — working usage examples for each toolset. **Better than guessing, and better than asking an AI.**
- ⚠️ CamelCase toolsets (`ValkyrieToolset.EntityToolset`, `DeviceToolset`, `VerseToolset`) are **not** in this Python tree — presumably C++. **Scene Graph entity creation is therefore NOT known to be Python-scriptable**; use MCP `EntityToolset` for that.
- **UEFN has a Python console** (bottom bar, beside Output Log) for running scripts by hand.

### 🤖 Epic ships AGENT SKILLS inside the editor

`editor_toolset/skills/*.py` define `unreal.AgentSkill` subclasses whose payload is **instructions written for an AI agent**, decorated `@agent_skill`. Example — `placing_actors.py`: *"Decide whether a newly placed actor should snap to the ground from what it represents. Props, furniture, characters, vehicles, rocks, foliage should snap… lights, cameras, audio emitters, triggers and volumes belong at the requested location, not on the ground."* Others: `blueprint_basics`, `material_basics`, `default_outdoor_lighting`, `diffing`, `source_control`, `unreal_skill_best_practices`. **Epic is building agent-facing documentation into the editor itself** — worth reading before hand-rolling behaviour Epic has already specified.

## ⭐ CVar search — the discovery tool for editor behaviours with no visible UI (6 Sep 2026)

`EditorAppToolset.SearchCVars` (param is **`name`**, not `searchTerm`) greps every console variable's *name and help text*. Many UEFN behaviours that look like unchangeable editor policy turn out to be a named, documented CVar. **Search here before assuming a behaviour can't be turned off** — and before asking an AI, which will invent a plugin menu.

- ✅ **Fortnite building actors snapping to the grid on placement:** `BuildingActor.SuppressSnapToGrid` — *"When true snap to grid will be turned off."* Default `false`; set `true` to place structure pieces freely.
- ⚠️ **CVars are editor-session state, not project settings** — expect them to reset on editor restart, and re-set after any UEFN relaunch. Nothing in the project records them.
- ⚠️ Applies to **all** building actors at once, not per-actor.
- Other CVars spotted in that same search: `VI.ActorSnap` (snap to other actors, off by default), `Editor.Gizmo.TemporarySnapDisableAvailable` (hold-key to temporarily disable snapping, off by default), `modeling.EnableVolumeSnapping`.

## UEFN MCP (beta — shipped Aug 2026)

⭐ **THE MCP SURFACE IS NOW A THREE-TOOL GATEWAY — read this before anything below (verified on v42.20, 20 Sep 2026).** `tools/list` no longer returns the toolset tools directly. It returns exactly **three**:

- `list_toolsets` — names + descriptions, takes no arguments.
- `describe_toolset(toolset_name)` — returns JSON with every tool name, description and **input schema** in that toolset.
- `call_tool(tool_name, arguments, toolset_name?)` — how you now invoke everything the entries below describe. Omit `toolset_name` to call a top-level tool.

**Nothing was removed — only the addressing changed.** Read `ValkyrieToolset.VerseToolset.BuildAll` throughout this file as `call_tool(toolset_name: "ValkyrieToolset.VerseToolset", tool_name: "BuildAll")`. ⚠️ **The `list_toolsets` diagnostic below still holds, one level down:** no `ValkyrieToolset.*` entries in its output = the plugins are off. ⚠️ **Which version this landed in is NOT established** — all that is verified is that it is true on v42.20. Do not cite a date for the change.

- ⭐ **`describe_toolset` is now the authority on signatures, and it is live.** Stop guessing parameter names from the Python tree, from this file, or from an AI — ask the editor. Worth one call before first use of any tool. The error messages are unusually good too: a missing required parameter returns **the full input schema alongside the params you sent**. Found immediately this way — `ListFiles` requires **both** `path` and `bRecursive`, and `bRecursive` is **not** optional.
- **30 toolsets on v42.20, against the 12 recorded in Aug 2026:** the five `ValkyrieToolset.*` (`VerseToolset`, `DeviceToolset`, `EntityToolset`, `SessionToolset`, `ValkyriePythonToolset`), `EditorToolset.EditorAppToolset` + `LogsToolset`, **thirteen** `editor_toolset.toolsets.*` (the August list plus `curve_table`, `material_instance`, `programmatic`, `skeletal_mesh`), four `NiagaraToolsets.*`, `PhysicsToolsets.PhysicsAssetToolset`, `GameplayTagsToolset`, and **four UI-authoring toolsets that did not exist in the August enumeration** — see below.
- **`ValkyrieToolset.VerseToolset` re-enumerated 20 Sep 2026:** `WriteFile, Replace, ReadFile, Move, ListFiles, Grep, Delete, CreateDirectory, Copy, BuildAll` — **unchanged** from the August list. The write → compile → read-diagnostics loop still stands as described below.
- **The Verse root is the project's own package, not `/Game`.** `ListFiles(path: "", bRecursive: false)` returns the project package plus the read-only digests `/Verse.org`, `/UnrealEngine.com`, `/Fortnite.com` and the `…/Assets` package. Build every path from what the listing returns rather than assuming a root.

- **Enabling it properly — this gated a whole session.** Enabling the *server* is not enough. Three things under **Project Settings**:
  1. **Python Editor Scripting Tools**
  2. **UEFN MCP Toolset**
  3. *Custom Items and Inventory* (optional)

  Plus **Auto Start Server** under Editor Preferences → Model Context Protocol. Server at `http://127.0.0.1:8000/mcp`. Official docs: [dev.epicgames.com/documentation/fortnite/uefn-mcp](https://dev.epicgames.com/documentation/fortnite/uefn-mcp)
- ⚠️ **The failure mode to recognise.** With only the server enabled, MCP **connects successfully** and reports 12 generic Unreal toolsets (EditorApp, Logs, Niagara, UMG, MVVM, Physics…) and **none of the UEFN ones**. It presents as a working connection with a disappointing feature set, *not* as a misconfiguration — which is why it cost a session. **Diagnostic: if `list_toolsets` shows no `ValkyrieToolset.*` entries, the plugins are off.** Enabling them required **no editor restart**; the toolsets appeared live.
- **Client side:** Claude Code reads `.mcp.json` from the directory it is *launched from*, **not** the project root.
- **The toolsets that matter** (enumerated 26 Aug 2026):
  - `ValkyrieToolset.VerseToolset` — ReadFile, WriteFile, Replace, Grep, ListFiles, Move, Copy, Delete, CreateDirectory, **BuildAll**. BuildAll returns *structured diagnostics*: severity, error code, message, file path, line/character span. **This is a complete write → compile → read-error loop with no human step in it.**
  - `ValkyrieToolset.DeviceToolset` — ListDeviceAssets, PlaceDevice (with transform), Get/Set/ListDeviceProperties, Add/Remove/ListEventBindings, GetBindingOptions.
  - `ValkyrieToolset.SessionToolset`, `EntityToolset` (Scene Graph), `ValkyriePythonToolset`.
  - `editor_toolset.*` — **ObjectTools**, **SceneTools** (place/remove actors, OFPA asset-path lookup, data layers), ActorTools, AssetTools, plus mesh/material/texture/data-table tools.
  - `EditorToolset.EditorAppToolset` — **`CaptureViewport`** renders an annotated world-space grid with labelled actors **and returns structured data alongside the image** (actor name, class, exact world position, distance). This is how an agent sees the level. `CaptureEditorImage` grabs the whole editor as the human sees it — use it when a Details panel is the question.
  - `EditorToolset.LogsToolset` — **`GetLogEntries`, with category filter and regex. This is how an agent reads Verse `Print` output.** The primary debugging channel during experiments.
- ⚠️ **`ListDeviceProperties` rejects built-in Blueprint devices.** It expects a `/Script/VerseDevices.ScriptDevice`; passing an NPC Spawner's actor path fails with *"is not valid ScriptDevice"*. **Unsolved as of 26 Aug 2026** for placed built-in device instances. *(Partial route found 27 Aug: asset-side properties read AND write via `ObjectTools` on the CDO path `Default__<Class>_C` — the trick that read a Character Definition's wiring and fixed an animation preset; see [02-npcs-and-ai.md](02-npcs-and-ai.md). Whether any path reaches a level-placed built-in device instance remains unsolved as written.)* ✅ **SOLVED 14 Sep 2026: `ObjectTools` reads a level-placed built-in device's Details-panel options directly.** Pass the actor path (`/<Project>/<Level>.<Level>:PersistentLevel.<ActorName>`) to `get_properties`. ⭐ **The property names are the panel labels with only the first letter lowercased, spaces kept:** `"damage Spawner After Spawn"`, `"limit Spawned Creatures"`, `"number of Creatures"`. Discover them with `list_properties` — ~90 KB of output, so grep it for names followed by `{"type"` rather than reading it. It returns live, *unsaved* editor values (the OFPA file on disk still showed the old value). `set_properties` presumably takes the same names — **untested**.
- `CaptureViewport`'s `captureTransform` captures from an arbitrary pose **without moving the human's viewport camera** — verified. Safe for an agent to look around freely. `SetShowFlag`, by contrast, **does** change what the human sees on screen; restore it afterwards.
- The `Navigation` show flag produced no nav-mesh visualisation in the UEFN viewport. **Inconclusive** — most likely the flag isn't wired, rather than evidence of a missing nav mesh. Do not cite this either way.
- Known issue: tool calls can cause **editor hitching**. Do not confuse an MCP hitch with unrelated machine instability; check Event Viewer before blaming either. *(v42.10 release notes promise "batch multiple tool calls into a single script" — see [00-version-watch.md](00-version-watch.md).)*
- Verse file operations via MCP are confined to the project (sandboxed).
- Keep agent changes small and reviewable; ask for a plan before multi-step execution (Epic's own MCP guidance, independently arrived at here too).

### ⭐ UI-authoring toolsets over MCP (found 20 Sep 2026)

Four toolsets aimed squarely at UMG/Verse UI work, **none of them in the August enumeration**. Signatures below came from `describe_toolset`, so they are the editor's own, not recalled:

- **`UMGToolSet.UMGToolSet`** (21 tools) — `CreateWidgetBlueprint`, `AddWidget`, `RemoveWidget`, `MoveWidget`, `RenameWidget`, `WrapWidgets`, `ReplaceWidgetWithTemplate` / `WithNamedSlot` / `WithChild`, `GetWidgets`, `GetWidgetDescription`, `GetWidgetClassInfo`, `GetWidgetTreeDepth`, `ListWidgetClasses`, `ListWidgetBlueprints`, `GetNamedSlots`, `SetNamedSlotContent`, `AddUIComponent`, `RemoveUIComponent`, `MoveUIComponent`, `CompileWidgetBlueprint`. Returns `{"refPath": "…"}` pointers you can pass straight to `ObjectTools` or back into the toolset.
- ⚠️ **Epic's own warning, quoted from the toolset description — a SILENT failure mode.** For every widget and slot the toolset returns: call `ObjectTools.list_properties(widget)` **first**, then `get_properties` / `set_properties` with those exact names. *"Property names vary per widget class and CANNOT be guessed… Skipping step 1 causes set_properties to silently fail or set wrong properties."* Same class of trap as the built-in-device property names above — and here it fails **without an error**, which is worse.
- **`VerseFieldsToolset.VerseFieldsToolset`** (6 tools) — `AddVerseField`, `EditVerseField`, `RemoveVerseField`, `DuplicateVerseField`, `ListVerseFields`, **`BindWidgetPropertyToVerseField`**. This is **the Verse↔UMG data bridge, and it is scriptable** — the piece that makes a widget property read a Verse field's value.
- **`MVVMToolset.MVVMToolset`** — authors ViewModel Blueprints derived from `UMVVMViewModelBase`, attaches them to a Widget Blueprint, creates property-to-property view bindings (with conversion functions where types differ) and event bindings from widget multicast delegates, plus a repair call that regenerates binding graphs from stored binding data.
- **`WidgetAnimationToolset.WidgetAnimationToolset`** — `UWidgetAnimation` authoring. ⭐ Its description notes `UWidgetAnimation` extends `UMovieSceneSequence`, so **every SequencerTools / SequencerKeyframingTools / SequencerConditionTools method that takes a sequence, track, section or channel works on widget animations unmodified.** Use this toolset for the UMG-specific lifecycle and binding work, the Sequencer toolsets for tracks and keyframes.

*Worked recipes built on these are in [09-custom-uis.md](09-custom-uis.md).*

### ⭐ Debugging without MCP, and MCP client config (14–20 Sep 2026)

- ⭐ **Verse `Print` output is also in the editor log FILE** — no MCP needed: `%LOCALAPPDATA%\UnrealEditorFortnite\Saved\Logs\UnrealEditorFortnite.log`, lines `LogVerse: : <text>`. Rounds are bracketed by `MinigameStateChanged: EFortMinigameState::InProgress` / `PostGameEnd`; pushes by `[Push Verse Changes] operation successful`; compile errors by `VerseBuild: Error:`. Older editor sessions rotate to `UnrealEditorFortnite-backup-<timestamp>.log`. ⭐ **Its real value is history:** grepping every round of a day is what disproved a wrong diagnosis (see [02-npcs-and-ai.md](02-npcs-and-ai.md) and [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md)). A live `GetLogEntries` read shows one round; the file shows the pattern.
- **Placed-device settings are partly readable offline** from the OFPA file under `Content\__ExternalActors__\<Level>\…\*.uasset`: booleans, floats and strings appear as printable text right after the Details-panel label (e.g. `Damage Spawner After Spawn` → `False`). **Enum and int values do not** — they are name-table references. Fallback only; this cannot write.
- ⚠️ **The UEFN MCP server can come up long after the editor does.** Log line: `LogModelContextProtocol: Starting MCP server on port 8000 at URL path '/mcp'` — observed ~50 minutes after launch. An agent session started before that line has no UEFN tools, and **MCP tools load only at session start**. ✅ **WORKAROUND FOUND 20 Sep 2026 — a restart is NOT actually required.** The server is plain **streamable-HTTP MCP**, so an agent with shell access can drive it directly and bypass its own dead MCP client: `POST http://127.0.0.1:8000/mcp` with `Accept: application/json, text/event-stream`, send `initialize`, read the **`Mcp-Session-Id`** response header, then pass that header on every subsequent `tools/call`. Verified on a session whose UEFN client had already failed at startup: `list_toolsets`, `describe_toolset`, `ListFiles` and **`BuildAll` (returned `[]`, project clean)** all worked. ⚠️ **Caveats:** results arrive as raw JSON you must parse yourself, every call costs a shell hop, and the session id lives only as long as the editor does.
- ⚠️ **`claude mcp add` without `-s` is LOCAL scope** — keyed to a folder path, and it did not load after an editor restart when the workspace path's drive-letter case differed (**suspected, not proven**). ✅ **Fix: user scope** — `claude mcp add -s user --transport http uefn http://127.0.0.1:8000/mcp`. It loads in every project; in non-UEFN sessions it just shows as failed, with no other effect.
- ⭐ **Compile probe: verify code *before* a human types it (14 Sep 2026).** When the human types the Verse themselves to learn it, a draft that doesn't compile costs them a round of typing. Recipe: `WriteFile` a temporary `zz_probe_<name>.verse` with the draft, **class names renamed** (`zz_…`) so they don't collide with live classes → `BuildAll` → ⚠️ **then plant a deliberate error and build again.** A zero-diagnostic result alone doesn't prove the new file was compiled; seeing `Unknown identifier` reported against the probe's own path does → `Delete` the probe → `BuildAll` once more to confirm the project is clean. All in-editor, so it is sanctioned under the file-operations rule above. Verified this way: an `@editable []my_row` array of a `class<concrete>` with `@editable` fields; a `<transacts><decides>` method used as a `for` filter; `Mod[A, B] = 0` as a failable check.
- **Reading a Verse device's `@editable` values over MCP (14 Sep 2026):** `DeviceToolset.GetDeviceProperties` works on a *Verse* device (a ScriptDevice). Field names are lowerCamel. An `@editable []my_class` array comes back as `refPath`s to sub-objects. ⚠️ **The sub-objects' own fields cannot be read:** `ObjectTools.list_properties` lists them but `get_properties` fails with *"the following properties could not be read"*, and `ActorLabel` is unreadable on devices too. **Fallback:** the owning device's OFPA `.uasset` lists which devices the rows reference, and names only the fields that differ from their defaults, but not the values.
- **`ValkyrieToolset.SessionToolset` (enumerated 14 Sep 2026):** `StartSession`, `StopSession`, `StartGame`, `StopGame`, **`PushChanges(bVerseOnly)`**, `GetSessionStatus`, `GetGameState`, **`GetClientLogEntries(pattern, maxResults, startLine)`**. Only the status calls tested so far. With `BuildAll`, this is the whole loop — edit → build → push → start game → read `Print` output — with the human needed only to play. ⚠️ `GetClientLogEntries` reads a **local** Fortnite client's log only, so it is no help for console clients.

## ⛔ → ✅ Mixed-platform playtest sessions hang at "waiting on client platforms" (15–16 Sep 2026, v42.10 — FIXED in v42.20)

**Symptom:** with a second player on a console in the party, UEFN sits at *Launching session* forever, with no error, whoever invites whom and whoever leads the party.

**What the log shows** (`LogValkyrieActivityTracker`): upload, matchmaking, beacon connection and the **server** cook all complete; only the **client-platform cook** never finishes. On a working solo launch that stage clears in 0.15–0.5 s.

- ✅ **The editor's `404 unknown session id` is NORMAL, not the fault.** Every *Play* looks up the *previous* session ID first and gets a 404 before creating a new one, including launches that went on to work fine. Don't chase it.
- ⭐ **"waiting on client platforms" = waiting for the connected client.** When the console's Join fails, the client never becomes ready and the editor waits forever with no error. **The editor log cannot see *why*** — the answer is on the console side.
- ⚠️ **A console client stage can genuinely take minutes.** One successful last-gen console launch spent **2 min 14 s** in that stage against 0.15–0.5 s for PC. **Don't cancel a console launch before ~3 minutes** — but waiting alone did not fix the failures.
- **Rule of thumb:** after any failed or cancelled launch involving a console, restart **both** the editor and the console's Fortnite before trying again. Stale session state on either side makes the next launch hang silently — a stale session on the console is what produces *"session ID is invalid"* on Join.
- ✅ **IDENTIFIED AND FIXED BY EPIC — it is a v42.10 platform bug, not a setup mistake.** Epic's thread [*Live Edit Sessions not joinable as of 42.10*](https://forums.unrealengine.com/t/live-edit-sessions-not-joinable-as-of-42-10/2763059): console players cannot join a Live Edit session hosted by a PC user. Epic staff: *"This is scheduled to be addressed in v42.20"*; the issue was closed as **Fixed** and appears on the [v42.20 known-issues list](https://forums.unrealengine.com/t/v42-20-known-issues-and-pre-release-updates/2775799). ~~**No workaround exists — wait for v42.20, then re-test.**~~ ✅ **RESOLVED ON v42.20 — RETESTED AND WORKING (21 Sep 2026).** The co-player session was tried again after the update and it works: the console player joins normally. Epic's fix holds. *(Specifics not captured yet — which console(s), whether the multi-minute client-platform wait is also gone, and whether the restart-both-sides ritual is still needed. The resolved verdict does not depend on them.)*
- ⭐ **The lesson, and it cost roughly twenty attempts across two consoles and a brand-new empty project: check Epic's known-issues and pre-release threads BEFORE spending sessions on a connection problem.** Everything proven locally was already stated outright in that thread. Workarounds tried and failed: launch-order changes, swapping party leader, full restarts both sides, a fresh project, two different console generations.
- **Other workarounds, if it ever recurs:** (1) launch the session solo **first**, return to the lobby, *then* invite; (2) the timing trick — the second account presses **Join the instant** the first shows *connecting*; (3) fall back to a private version and island code, below.

## ⭐ A PRIVATE VERSION is the reliable way to playtest co-op (17 Sep 2026)

**Proven:** with Live Edit sessions broken for console players on v42.10, a full co-op round was played via a **private version island code** entered in Fortnite's normal search field. No edit session involved, so the session bug does not apply.

**How it is made:** in UEFN, connect a session, then **Project → memory calculation**. That uploads the project and produces a **Private Version** code to share with teammates ([Epic docs](https://dev.epicgames.com/documentation/fortnite/publishing-projects-in-unreal-editor-for-fortnite)). Players enter the code in Fortnite's Discover/search field and land in a private lobby. The project stays **unpublished** until a release is created in the Creator Portal.

- ⚠️ **Creator Portal "Memory Usage: Action Needed"** blocks *publishing a release* until a memory calculation has been run for that private version (*"Publishing is not permitted for Private Versions that have not been memory checked"*). The private code is still playable meanwhile.
- ⚠️ **The "Verse Validation Errors" dialog appeared during this flow even though every Verse edit had gone through the editor**, not from outside. So "external edits cause it" is **not the whole story** — it also shows up around session refreshes and the publish/memory-calculation flow. Treatment is unchanged: `BuildAll` to confirm the code is fine, then **Rebuild Verse And Reload Map**, never *Continue*, never save in that state. *(See [01-verse-language-and-compiler.md](01-verse-language-and-compiler.md).)*

## Coordinates & rotation — LUF vs XYZ

- **LUF vs XYZ coordinate translation — exact mapping derived 27 Aug 2026.** The toolsets speak XYZ; UEFN's Details panel displays **Left-Up-Forward**. Measured by placing a device at a known XYZ and reading the panel:

| Sent (toolset XYZ) | Shown (UEFN panel LUF) |
|---|---|
| `x: 8300` | Forward = **8300** |
| `y: -11400` | Left = **11400** |
| `z: 2688` | Up = **2688** |

**So the panel shows `(L, U, F) = (−Y, Z, X)`.** Note the **sign flip on Y** — that is the one that will catch you, because it silently mirrors a position rather than obviously breaking it. ✅ **Officially confirmed 6 Sep 2026:** `/Verse.org/SpatialMath`'s `vector3` declares fields `Left`/`Up`/`Forward` with Epic's own comments — *"The Left (was -Y) component"*, *"The Up (was Z) component"* — the derived mapping in Epic's words. (The `/UnrealEngine.com/Temporary/SpatialMath` vector3 used in device code remains X/Y/Z; the two types stay incompatible.) Consistent with standard Unreal axes (+X forward, +Y right, +Z up; Left = −Y), so **the toolset is not wrong and needs no correction factor — the panel is simply a different presentation of the same world position.** An agent-placed actor lands where the XYZ says it will.
- ✅ **ROTATION DERIVED 6 Sep 2026 for ENTITIES — it is 1:1, no offset.** A prefab rotated by hand *−90° about the up axis* in the editor reads back from the toolset as exactly **`yaw: -90`**; its untouched neighbour reads **`yaw: 0`**. **`yaw` (about Z) IS the editor's horizontal turn**, same number, same sign — an editor gesture and an MCP value are directly interchangeable. ⭐ **Entity transforms round-trip exactly** (send −90 → read −90), which makes entities far more predictable for programmatic placement than the device path. ⚠️ The old warning still stands for **devices**: a `button_device` placed at `0/0/0` *displayed* `180° / −90° / −180°` in its Details panel. Whether that is a device quirk or a panel-display transform is still unconfirmed — entities are the safe path either way.
- ⭐ **Worked example — programmatic fence run (6 Sep 2026):** a fence prefab tiles seamlessly at **410 cm spacing on X with `yaw: -90`**, `SetEntityTransform` to re-space an existing run, `CreateEntity` to extend it. Six panels = 20.5 m laid in one message.
- *(superseded)* ~~Rotation is also transformed, and the mapping is NOT yet derived. Sending `pitch/yaw/roll = 0/0/0` displayed as `180° / −90° / −180°`. Until someone works this out, sanity-check agent-set rotations in the viewport~~ — and be especially careful near hand-rolled polar-to-cartesian math, which computes in raw XYZ.
