---
verified-on: UEFN v42.10
last-reviewed-against: UEFN v42.10 (6 Sep 2026)
---

# Imports & Validation

Asset packs, the validation gate, and publishing.

## Importing UE marketplace asset packs (first done 6 Sep 2026 — a UE 5.5.4 environment pack)

**UE packs are never UEFN-ready — but ⚠️ CORRECTED within the hour (6 Sep 2026): commit-time validation is a REPORT, not a gate.** A revision submitted successfully (491 files) with ~60 validation errors still standing, logged during that very submit. A greyed Submit button on the first attempt was the **empty Description field**, misread as validation blocking. The hard gate for invalid content is *presumed* to be **publish** — not yet observed. Validation errors at commit are your to-do list, not your jailer. Source UE version is irrelevant; restricted *content classes* are the issue. Triage order, by leverage:

1. **Delete the pack's demo/showcase levels first** (inside UEFN) — they carry unsupported landscape assets (`LandscapeLayerInfoObject`) and produce all the "Node Unknown / Illegal property override" warnings. Epic wants extra levels gone before publish anyway.
2. **Delete `FoliageType_InstancedStaticMesh` assets** — unsupported class, no fix exists. The referenced meshes survive.
3. **Blueprints: treat pack BPs as dead weight — delete the folder and use the static meshes.** *(Hardened same day, after sanitizing was tried.)* **Sanitize Blueprint** clears the restricted-content error (graphs stripped) but a second error survives it: *"contains an exposed non-private instance of asset class Blueprint"* (`FortValidator_FortExposedAssets`) — the Blueprint **class asset itself** appears unshippable in UEFN content, sanitized or not. Every arrangement a pack BP provides can be rebuilt from its `SM_` pieces, which validate clean.
4. **Bulk one-click fixes:** static meshes need *Low/Medium Minimum LOD For Quality* set (Switch/mobile scalability); textures need **right-click → Conform Texture**. Both batch across selections.
5. **Hard limits have no auto-fix:** LOD0 > 30,000 vertices must be genuinely reduced by an artist.
6. **Master materials containing `MaterialFunction` instances** are rejected with no offered fix — the expensive case, since the pack's material instances parent to them. Budget a rebuild-and-reparent in UEFN if resave doesn't clear it.
7. The Submit dialog *also* requires a non-empty Description — a separate blocker from validation.

⭐ **Do pack imports on a revision-control branch.** Validation failure then costs nothing: fix at leisure or abandon the branch.

⭐ **Where the gate actually sits (corrected the same day it was first written):** LORE runs full validation at every submit and *reports* — but **commits anyway**. So WIP checkpoints during a long asset fix ARE available ("commit my progress, finish tomorrow" works). The authoritative server-side copy may hold invalid content, but that content presumably cannot pass the **publish** gate (unverified — the `UEFNValidation` error class is assumed publish-blocking and has not been tested against a real publish). ⚠️ Practical rule: the per-submit error list is a live to-do; don't let it normalise — what commits today still has to publish someday.

- ⚠️ Imported asset packs also trigger repeated **"make assets private"** churns in later sessions — one observed blocking a session launch. Budget editor time for it after any pack import.
