# UEFN Field Notes

Field-tested knowledge of **UEFN, Verse, Scene Graph and LORE** — written by AI agents during real development sessions, corrected by observation, and structured for retrieval by humans and agents alike.

This is a living record, not a tutorial. It documents what actually happens when you build on Unreal Editor for Fortnite today: the extraordinarily capable built-in floor, the largely undocumented custom layer above it, and the exact places where the two meet. Much of what is recorded here exists in no official documentation.

## Who writes this

These notes come out of active projects (currently *Sheep Petting Simulator* and a Fortnite museum experiment) built by a mixed human/agent team: a human developer and designer (Jay), a second human designer/playtester (Julia), a chat-based AI design partner (Anthropic's Claude), and an agentic AI implementer (Claude Code) connected to the UEFN editor over MCP. Model versions change over time and are recorded in commit messages rather than here. Findings are verified in the editor, in the compiler, in the logs, or live in play — and marked accordingly (see the legend below).

The working copy lives in a private Notion workspace; this repository is its regularly synced public mirror. Corrections and additions land upstream first and flow here at session close.

## Why it reads the way it does

**The authority of these notes comes from showing their own workings.** Wrong claims are not deleted — they stay visible, struck through or marked corrected, with the date and the *mechanism* of the correction. "We believed X on the 26th, disproved it on the 28th, and here is why the error was convincing" is worth more than a confident restatement of X-corrected, because it tells the next reader (human or agent) what the failure looked like from the inside.

For the same reason, hedges are load-bearing. "Observed clean at observed timings" is not the same claim as "proven safe", and the notes say which one they mean.

## The legend

Every substantive claim carries a verdict marker, a date, and the version it was observed on.

| Marker | Meaning |
|---|---|
| ✅ | **Verified** — observed working, with the evidence type stated |
| ❌ | **Disproven** — a claim (often our own earlier one) shown to be false |
| ⚠️ | **Caution** — partially verified, version-sensitive, or a known trap |
| ⛔ | **Closed** — a door that is genuinely shut; do not re-test without a version change |
| ⭐ | **Rule earned** — a working practice paid for with lost time |
| 🔍 | **Undocumented discovery** — real behaviour with no official documentation |
| 🔶 | **Half-charted** — partially explored; the open edge is stated |
| ℹ️ | **Context** — background that aids interpretation |

**Evidence types**, stated with or near the marker: *compiler-proven* (BuildAll / digest), *live-playtest* (observed in a running session), *log-verified* (read from editor or LORE logs), *docs-cited* (Epic documentation or release notes), *hypothesis* (reasoned but untested). A claim without an evidence type should be read as hypothesis.

## Version discipline

UEFN moves fast and findings expire. Two conventions keep the notes honest:

1. **Every finding is stamped with the version it was observed on.** Old-version knowledge is never deleted — it remains valid *for that version*, and has repeatedly proven useful after regressions and when comparing behaviour across updates.
2. **Every file carries a `last reviewed against` line in its front matter.** This records the most recent engine version someone has read the file against. "Proven on v41.10, reviewed against v42.10, still stands" and "proven on v41.10, never re-checked" are different claims, and the front matter is what distinguishes them. An "engine-blocked" note is a claim with an expiry date — one of this project's costliest lessons was leaving one unre-tested across an engine jump.

Current engine baseline: **UEFN v42.10 / Unreal Engine 6.0** (September 2026).

## Structure

| File | Covers |
|---|---|
| `00-version-watch.md` | Dated version notices, per-update reviews, Epic roadmap watch |
| `01-verse-language-and-compiler.md` | Effects, `no_rollback`, failable conjunctions, syntax traps, reading compiler output |
| `02-npcs-and-ai.md` | The `npc_behavior` sandbox, animation presets, `Focus`, spawners, flocks, Epic's wildlife system |
| `02a-llm-personas-and-conversations.md` | Epic's LLM persona system: `persona_component`, `ai_session`, voices, the 10,000-char prompt ceiling, the in-session conversation recipe, prompt craft |
| `03-devices-and-interaction.md` | The runtime-device constraint, buttons on moving NPCs, tag discovery, pooling, HUD/map devices |
| `04-scene-graph-and-prefabs.md` | Entities, the one-component rule, the working prefab recipe, what is closed and why |
| `05-editor-and-tooling.md` | CVar search, the MCP toolsets and their Python source, agent skills, coordinate mapping |
| `06-lore-revision-control.md` | Branches, three-way merges, file locking, binary conflicts, log forensics |
| `06a-lore-branching-and-merging.md` | The forensics in full: three silent merge failures diagnosed from the logs, the personal-backup-branch model, conflict paths, the multi-user lock lifecycle |
| `07-imports-and-validation.md` | Marketplace asset packs, the validation gate, publishing |
| `08-trusting-ai-on-uefn.md` | Scored AI failure modes (including Epic's own assistant) and countermeasures |

A **letter suffix** marks a sub-topic that outgrew its parent file and was split out: `02a` belongs to `02`, `06a` to `06`. The parent keeps the working rules and links down; the child carries the depth. Splitting this way keeps related files adjacent and never renumbers what already exists.

## For agents

If you are an AI agent consuming these notes:

- Filter by marker to separate what is proven from what is assumed. ✅ with an evidence type is safe to build on *for the stamped version*; everything else is input, not ground truth.
- Check the file's `last reviewed against` line before relying on any finding older than the current engine version.
- Treat every API claim — including the ones in these notes — as a hypothesis to be confirmed against the digest files (`Verse.digest.verse`, `Assets.digest.verse`, `Fortnite.digest.verse`) before building on it. That rule is itself a finding here (`08-trusting-ai-on-uefn.md`), and it applies to us too.

## Contributing and disputing

Findings here are meant to be falsifiable. If you can disprove one, that is a contribution: open an issue or PR with the evidence — a digest line, compiler output, a log excerpt, or a described repro with the engine version. Corrections are added as dated addenda rather than silent edits, in keeping with the conventions above.

## License

MIT — see [LICENSE](LICENSE), chosen for maximum reuse including automated agent reuse. Settled before any external contributions, deliberately: relicensing is easy now and legally awkward after outside PRs are merged.

---

*Maintained by the Sheep Petting Simulator team. The working copy lives in Notion; this mirror syncs at session close. Article-length write-ups drawing on these notes are linked from the relevant files as they are published.*
