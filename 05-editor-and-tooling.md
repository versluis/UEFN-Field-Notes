---
verified-on: UEFN v41.10 – v42.10
last-reviewed-against: UEFN v42.10 (3 Sep 2026)
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
- ⚠️ **`ListDeviceProperties` rejects built-in Blueprint devices.** It expects a `/Script/VerseDevices.ScriptDevice`; passing an NPC Spawner's actor path fails with *"is not valid ScriptDevice"*. **Unsolved as of 26 Aug 2026** for placed built-in device instances. *(Partial route found 27 Aug: asset-side properties read AND write via `ObjectTools` on the CDO path `Default__<Class>_C` — the trick that read a Character Definition's wiring and fixed an animation preset; see [02-npcs-and-ai.md](02-npcs-and-ai.md). Whether any path reaches a level-placed built-in device instance remains unsolved as written.)*
- `CaptureViewport`'s `captureTransform` captures from an arbitrary pose **without moving the human's viewport camera** — verified. Safe for an agent to look around freely. `SetShowFlag`, by contrast, **does** change what the human sees on screen; restore it afterwards.
- The `Navigation` show flag produced no nav-mesh visualisation in the UEFN viewport. **Inconclusive** — most likely the flag isn't wired, rather than evidence of a missing nav mesh. Do not cite this either way.
- Known issue: tool calls can cause **editor hitching**. Do not confuse an MCP hitch with unrelated machine instability; check Event Viewer before blaming either. *(v42.10 release notes promise "batch multiple tool calls into a single script" — see [00-version-watch.md](00-version-watch.md).)*
- Verse file operations via MCP are confined to the project (sandboxed).
- Keep agent changes small and reviewable; ask for a plan before multi-step execution (Epic's own MCP guidance, independently arrived at here too).

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
