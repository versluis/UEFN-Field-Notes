---
verified-on: UEFN v41.10 – v42.10
last-reviewed-against: UEFN v42.10 (3 Sep 2026)
---

# Version watch

Dated version notices and per-update reviews. Read new release notes against every file in this repo before assuming an entry still holds.

> ⚠️ **VERSION NOTICE (26 Aug 2026):** everything in these notes tagged v41.10 or earlier was verified on **UEFN v41.10**. The editor has since jumped to **v42.00 / Unreal Engine 6.0.0** — a major engine transition. Old entries stay (old-version knowledge remains valid *for that version*), but nothing is assumed true on v42 until re-tested. v42 verdicts are added as dated addenda, never overwrites.

> ⚠️ **VERSION NOTICE (3 Sep 2026): v42.00 → v42.10.** Release notes: [42.10 Fortnite Ecosystem Updates and Release Notes](https://dev.epicgames.com/documentation/fortnite/42-10-fortnite-ecosystem-updates-and-release-notes)
> 📚 **For any future update, start here:** [What's New in Unreal Editor for Fortnite](https://dev.epicgames.com/documentation/fortnite/whats-new-in-unreal-editor-for-fortnite) — the running index. Entries go to **forum posts first**, and the deeper documentation is linked from inside those posts.

**v42.10 reviewed against the field notes 3 Sep 2026.** ✅ **Nothing changed for us in:** `GetPlayAnimationController`, `FindCreativeObjectsWithTag`, `button_device`, `npc_behavior`, `tracker_device`, `map_indicator_device` — the entries stand. ✅ **No revision control changes at all** — no LORE/URC/branch/merge mentions, so the branch findings carry over. ✅ **Scene Graph got only** a looping-skeletal-animation fix and `EaseOut` re-exposed as experimental — **`mesh_component` is still `epic_internal`**, so the Scene Graph interactable blocker is unchanged. Five items *do* touch active work:

- ⚠️ **Asset redirection widened.** *"Non-private assets and Verse-compiled code assets now automatically place redirectors when moved/renamed."* Moving or renaming a **Verse file** can now leave a redirector where v42.00 would not have. Redirectors **merge across branches intact** (see [06-lore-revision-control.md](06-lore-revision-control.md)), so *Fix Up Redirectors* before merging matters more than it did.
- ⛔ **`@editable` `concrete_subtype` fields must now contain `<concrete>` types only** — non-compliance is a **validation error on republish**. Grep your project for `concrete_subtype` before the next publish; this is a silent republish-blocker, not a compile error.
- 🔍 **`InteractMapping` + an Interact action (Experimental).** Gives the **keybind event only** — no prompt, no targeting — so it is **not** a replacement for a pooled interaction device. Possible angle on per-NPC association: subscribe to Interact, then do your own nearest-NPC pick, no N buttons needed. **Hypothesis only — grep the digest first**, per [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md).
- ✅ **UEFN MCP: "batch multiple tool calls into a single script."** Aimed squarely at the per-call **editor hitching** recorded in [05-editor-and-tooling.md](05-editor-and-tooling.md).
- ✅ **Stability:** load-time regression fixed for maps with a **large number of Scene Graph entities**, and **player-reference GC hitches eliminated**. Both change what a scale test measures — re-run on v42.10; v42.00 numbers are suspect.

## 📡 Epic roadmap watch — annotations (5 Sep 2026)

From Epic's living UEFN roadmap (screenshots reviewed Q3 2026 → Q2 2027):

- **`material` is `epic_internal` — may become obsolete Q1 2027.** *Verse for Materials* is on the roadmap at high confidence. Two-prop swaps and duplicated-Blueprint/CD colour variants are workarounds for a constraint Epic intends to remove. Don't re-derive them past that point — re-check the digest first.
- **The islands-and-levels constraint (no shared state across islands) — may weaken Q4 2026.** *Cross Island Shared Persistence* is in progress at medium confidence (most-upvoted roadmap item). "Areas within one island" remains the right call today; revisit if multi-island progression ever matters.
- **Scene Graph NPC timing:** *AI Navigation in Scene Graph* and *AI Perception in Scene Graph* (both Q4 2026, experimental) are the missing organs for a Scene Graph-authored NPC; *Interactive Scene Graph UI* Q1 2027 (low confidence). A from-scratch Scene Graph NPC project realistically starts after these land, not before.
- **New Verse VM in progress (high confidence):** expect compile/runtime behaviour shifts on version-update days — check the digest and [01-verse-language-and-compiler.md](01-verse-language-and-compiler.md) against reality after every UEFN update.

## Hardware notes

Findings from the machines these notes were produced on; generalizable parts marked.

- 🔍 **HP Z840: known UEFN crash class — Kernel Power events during live sessions**, suspected Easy Anti-Cheat + TPM 1.2 interaction on older workstation platforms. Partially mitigated via BIOS power-management changes (June 2026); not fully resolved. Long agent-driven play sessions are exposure. Worth knowing if you run UEFN on pre-TPM-2.0 hardware.
- ✅ **The playtest pattern that sidesteps it:** UEFN alone runs on the workstation, changes push to Epic's servers, and a console (PS5) joins the live session as the client on a second screen. **The Fortnite client never runs on the dev machine.**
- **Lenovo P15 (RTX A5000 16GB, 64GB RAM):** clean UEFN v42 install; UEFN and a local Fortnite playtest on the same machine ran cool. Install footprint: ~60GB Fortnite + ~20GB UEFN; the launcher demanded 143GB free. Tip: disable Realtime Viewport while playtesting locally. Full setup write-up to appear on [versluis.com](https://www.versluis.com).
