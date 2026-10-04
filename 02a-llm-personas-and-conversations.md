---
verified-on: UEFN v42.10 – v42.30
last-reviewed-against: UEFN v42.30 (4 Oct 2026)
---

# LLM Personas & Conversations

Platform knowledge for Epic's LLM persona system — `persona_component`, `ai_session`, the Persona Modifier, the Prompt Editor, voices, the prompt size ceiling, and what is and isn't reachable in a live session. Everything here is dated. Old claims stay visible with the date and mechanism of their correction.

ℹ️ *Provenance: split out of [02-npcs-and-ai.md](02-npcs-and-ai.md) on 7 Sep 2026, the day it was written — the LLM findings turned out to be substantial and largely separate from `npc_behavior`, animation presets and spawner work. All dated entries preserved.*

First source project: a Fortnite museum experiment on UEFN **v42.10** / `++Fortnite+Release-42.10-CL-57819926`.

## ⭐ The API surface, read from the digest (7 Sep 2026)

> **Provenance and confidence.** Everything in this section comes from **reading the v42.10 digests**, not from running anything. Signatures, gating and lock status are **digest-verified**; behaviour is **runtime-untested**. Marked per the rule in [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md) — this is what the API *says*, not what it *does*.

### ⛔ The persona API is NOT in the Fortnite digest — it is in the UnrealEngine digest

The string `persona` **does not appear anywhere in `Fortnite.digest.verse`** (two incidental hits: "personality animations" on the Sidekick, and "personalized on/off state"). The LLM conversation system lives in:

- **`/UnrealEngine.com/Conversations`** — module `Conversations` in `UnrealEngine.digest.verse`

⚠️ **The trap:** `Fortnite.digest.verse` *does* contain a `conversation_device` (line ~7103) with `InitiateConversation`, `ShowConversation`, `OnConversationEvent` and so on. **That is the old Conversation Graph device** — pre-written branching dialogue, not the LLM. Grepping "conversation" in the Fortnite digest finds the wrong system and finds it convincingly. Grep the **UnrealEngine** digest for `persona`.

⭐ **Rule earned: the Fortnite digest is not the whole API.** Scene Graph components and the AI/conversation layer live under `/UnrealEngine.com/`. Grep all three digests, not just Fortnite.

### The shape of it

- **`persona_component`** — `class<final_super>(component)`. A **Scene Graph component**, consistent with Epic's docs saying the Persona Modifier adds one automatically.
- **`ai_session`** — `class<unique>`, reached via `Persona.GetAISession()`. This is the state object: history, pinned entries, registered capabilities.
- **`voice_model`** — `class<abstract><epic_internal>`. **You cannot define a voice**; you pick from the shipped subclasses.

### ⭐ `ai_session.Prompt` — structured output is a direct Verse call, no voice involved

```verse
Prompt<native><public>((local:)Prompt:message, response_type:type)<suspends>:result(response_type, ai_error)
```

Gated `@available {MinUploadedAtFNVersion := 4100}` (v41.00+). The digest comment: *"The response is parsed into response_type, which must be a struct containing only: numeric types, messages, booleans, enums, agents, or structs/arrays of those types. Field names and type names guide the model on how to fill out the structure."*

⭐ **This prompts the session and returns a typed struct without a player, a microphone, or a voice channel.** Field *names* are part of the prompt — the struct is documentation the model reads. Worth knowing before treating structured output as an advanced, late-stage feature; it may be the *easiest* way to exercise a persona, not the hardest.

### ⭐ `RegisterAction` — the model can call capabilities you register

```verse
RegisterAction<native><public>(Definition:prompt_binding_definition, Required:logic, output_type:type, Callback:type {_(:output_type):void})<transacts>:cancelable
```

Gated `@available {MinUploadedAtFNVersion := 4120}` (v41.20+). `prompt_binding_definition` is a struct of `Name` and `Description` — and the digest is explicit that both are **shown to the model**: *"Human-readable name the AI prompt sees when deciding which capability to call"* / *"Describes when/why the AI prompt should pick this capability."*

Digest description: *"Registers a fire-and-forget side-output the model can emit alongside any response, during prompts and goals. When Required is true, the model must emit this every response. Returns a cancelable that unregisters the action."*

⭐ **This is tool-calling, and none of Epic's public blog material we had recorded mentions it.** The `Required:logic` flag is notable — you can force a structured side-channel on *every* response.

### Session history persists and is runtime-compacted

Digest: *"Manages AI session state: conversation history, pinned entries, and registered capabilities… History is automatically compacted by the runtime. Completed goals are summarized into history."* Plus an explicit `ClearHistory()<transacts>`.

⚠️ **This refines the widely-repeated "the LLM session clears after each round."** There is a first-class session object with persisting, self-compacting history and a manual clear. What the round boundary actually does to it is **untested** — but "it just resets" is not what the API describes.

### ⛔ A voice channel is mandatory for anything spoken

Both `PromptToTalk` overloads take a `Channel:voice_channel(member_info)` and are `<decides>` — the digest says they **fail** *"if the interrupt rule prevents the prompt being submitted or the persona isn't in a channel."*

```verse
(Player:player).SetConversationTarget((local:)Persona:persona_component, Channel:voice_channel(member_info))<transacts>:void
(Player:player).GetConversationTarget()<decides><reads>:tuple(persona_component, voice_channel(...))
(Player:player).ClearConversationTarget()<transacts>:void
```

- `SetConversationTarget` is what binds a player's **prompt hotkey** to a specific persona — and the digest warns *"The persona must be in the voice channel with the player in order for the player to be able to successfully prompt."*
- `voice_channel` lives in `/Verse.org/Chat`; `(Entity:entity).GetVoiceChannels()<reads>:[]voice_channel(...)`. Channels cap at **80 members**; membership can also be restricted by parental controls or user settings.
- ⭐ **So "which persona am I talking to" is explicit state you manage**, not proximity magic. Multiple personas in one space needs deliberate target-swapping.
- ⭐⭐ **Conversation history propagates across a shared channel.** From `GetAISession`: *"When the persona is in a voice channel, every session in that channel will automatically propagate conversation history entries to all sessions found in the channel."* Two personas sharing a channel pool each other's conversations — so scoped-knowledge guides need **one channel each**.

### Moderation is observable — and substitution is visible

- `StartSayEvent:listenable(tuple(?agent, cancelable))` — fires when the persona **begins** speaking, *"before moderation resolves"*. Carries a `cancelable` to stop it talking.
- `CommitSayEvent:listenable(tuple(?agent, message))` — fires *"when the spoken message is finalized (after moderation)"*, carrying *"the actual message delivered — either the original or the canned replacement."*
- Also `StartHearEvent:listenable(agent)`, `StopHearEvent:listenable(agent)`, `StopSayEvent:listenable(logic)` (the logic is true when interrupted).

⭐ **`CommitSayEvent` is the honest source of what the NPC actually said.** If moderation swaps in a canned line, that is where you see it. Useful for any logging or transcript feature.

### Typed errors — including a character limit

`ai_error` (carries `Message`) with four subclasses:

- `ai_timeout_error`
- `ai_throttled_error` — Epic's announced rate limiting, already typed
- `ai_moderated_error`
- ⭐ **`ai_character_limit_error`**

Surfaced via `PromptFailureEvent:listenable(ai_error)` and as the error half of `Prompt`'s `result`.

### Interruption

`persona_interruption_rule` — `enum<open>`: `IgnoreUntilFinished`, `InterruptOnPromptStart`, `InterruptOnResponseStart`. Settable per-persona (`var PromptInterruptionRule`) **and** overridable per-call via the second `PromptToTalk` overload.

## Voices

`Personality` and `Voice` are `var<private>` with `SetPersonality(NewPersonality:message)<transacts><decides>` and `SetVoice(NewVoice:voice_model)<transacts><decides>` — **both changeable at runtime**, both `<decides>` so both can fail.

### The `voice_model` classes — 35 total, 19 usable, 16 `epic_internal`

| Voice | Descriptor |
|---|---|
| `terns_voice` | Narrator, Low, Warm |
| `moxie_voice` | Narrator, Med, Warm |
| `elmira_voice` | Refined, Med, Warm |
| `guild_voice` | Refined, Med, Warm |
| `clover_swift_voice` | Cinematic, Med, Warm |
| `brute_gunner_voice` | Cinematic, Med, Natural |
| `magnus_voice` | Glamorous, Med, Natural |
| `nezumi_voice` | Glamorous, Low, Natural |
| `helsie_midnight_voice` | Glamorous, Animated, Warm |
| `aura_voice` | Tough, Low, Natural |
| `halley_voice` | Tough, Med, Natural |
| `chase_voice` | Comic, High, Natural |
| `orin_voice` | Comic, High, Natural |
| `field_commander_voice` | Comic, Med, Natural |
| `battle_gamer_mae_voice` | Whimsical, Med, Warm |
| `lexa_hexbringer_voice` | Whimsical, High, Warm |
| `munitions_master_voice` | Whimsical, High, Warm |
| `sunspot_voice` | Whimsical, High, Warm |
| `revolt_voice` | *(no descriptor in digest)* |

**Locked, `epic_internal` (16):** `brite_bomber`, `cuddle_team_leader`, `fishstick`, `haylee_skye`, `hope`, `jonesy`, `kor`, `mancake`, `mecha_team_leader`, `midas`, `peely`, `raven`, `red_ruin_joni`, `the_imagined`, `tomatohead`, `ziggy`.

⚠️ **Correction to the commonly-cited "17 locked IP characters":** the digest ships **16** locked voice models. **`triggerfish_voice` does not exist in the digest at all** — locked or otherwise. The "17" figure is about *characters* in Epic's blog copy; the *voice* count is 16. Note also that `hope_voice`, `jonesy_voice`, `midas_voice`, `raven_voice`, `tomatohead_voice`, `mecha_team_leader_voice` and `cuddle_team_leader_voice` carry **no descriptor comment** — only some voices are documented in-digest.

⭐ **Picking a locked-sounding voice legally:** `kor_voice` (Narrator, Low, Warm) is locked, but **`terns_voice` carries the identical descriptor** and is usable. When a locked voice is the one you want, search the descriptors for a twin before compromising.

### There is a SECOND table, and it is the useful one (7 Sep 2026)

- `DT_LLMPersonaVoices` — 51 rows. ElevenLabs stock names (Alice, Laura, Callum…) plus obfuscated codenames (`BashfulApple002`, `PumpkinLoaf001`…). **This is not the whole list**, and searching it alone makes the Fortnite character voices look unavailable.
- ⭐ `DT_LLMPersonaVoicePreviews` — **77 rows**, carrying the Fortnite character voices under their real names: `aura_voice`, `terns_voice`, `moxie_voice`, `kor_voice`, `elmira_voice` … each with a `voiceId` and a preview `SoundWave`.

The Persona Device's `voice` field takes **`ElevenLabsVoices:<voiceId>`**. `ElevenLabsVoices:aura_voice` was accepted and round-tripped. So the `voice_model` classes used by the Character Definition and the string IDs used by the device **do** cover the same voices — two naming schemes over one set, not two different sets.

## ⛔⭐ THE PROMPT SIZE LIMIT IS 10,000 CHARACTERS (found the hard way, 7 Sep 2026)

**`LLMSession.SystemInstructionCharLimit` = 10000.** Verified by exceeding it and then reading the CVar. This is the single most load-bearing number for any persona work, and it is **not in Epic's documentation** — it is a console variable.

**How it presents.** A 15,013-char personality prompt wrote to the asset **without complaint** and saved fine. The failure comes at *session open*, in the Prompt Editor:

- UI banner: `Assembled system instruction exceeded limit`
- Response Details: *"The character limit for the text assembled from the prompt providor sources exceeds the LLM instruction limit."* (Epic's typo)
- Output log: `LLMSessionLog: Error: LLMSession Resonance conversation - OpenSession failed: Assembled system instruction exceeded limit`

⭐ **The asset layer imposes no limit — only the session does.** So an over-long prompt looks completely healthy until someone tries to talk to it. There is no editor warning, no validation error, no truncation. Check the length yourself.

⚠️ **"Assembled" is the important word.** The budget covers your authored text **plus Epic's own moderation and restriction facts**, concatenated in at session open. Epic's share could **not** be measured: the rows in `DT_ModerationPromptFacts` (`FNEModeration.SystemPrompt`, `FNEModeration.ResponseRestrictions`) are `Type: Template` with an empty `ValueString` — they resolve at runtime, not statically. **So the usable authored budget is 10,000 minus an unknown overhead.** Leave a real buffer.

**Measured on v42.10:**

| Authored chars | Result |
|---|---|
| 789 | ✅ opens |
| **8,786** | ✅ opens (a full guide fact sheet, ~1,200 buffer) |
| 15,013 | ⛔ `OpenSession failed` |

⛔ **Do not "fix" this by raising the CVar.** It is writable in the editor, but raising it locally would at best move the failure to the backend, and it is exactly the class of change that passes in PIE and fails on publish. Treat 10,000 as real and architect around it.

⭐ **The architectural consequence:** one persona cannot hold a large knowledge base. Depth comes from **more personas**, each with a narrow scope, or from injecting context at runtime (`ai_session.Prompt` / `RegisterAction`) rather than authoring it.

### The persona CVars worth knowing (v42.10)

| CVar | Value | Note |
|---|---|---|
| `LLMSession.SystemInstructionCharLimit` | **10000** | the hard ceiling, assembled |
| `LLMSession.StructFieldDescriptionCharLimit` | **2000** | budget for structured-output field descriptions — matters for `RegisterAction` and structured output |
| `AIPrompt.PlayerDialogue.MaximumLength` | 15 | ⚠️ **probably seconds** of player speech, not chars — inferred from the touch pair below, not confirmed |
| `AIPrompt.PlayerDialogue.TouchUIMaximumLength` | 7.5 | half the above on touch |
| `LLMFacts.PersonalityPromptName` | `PersonalityPrompt` | confirms the single authored fact key |
| `PersonaPrompt.ResonanceConfigurationID` | `persona-default` | ⭐ **"Resonance" is Epic's internal name for the conversation service** — it appears in log lines. Useful search term. |
| `AIPrompt.Persona.PlayerFocusMaxDistance` | 1000 | cm |
| `LLMSession.IsLLMSessionAllowed` | true | |

Search them with `EditorAppToolset.SearchCVars` on "llmsession", "aiprompt", "persona" (see [05-editor-and-tooling.md](05-editor-and-tooling.md)).

## Authoring — the Character Definition side

### Only ONE authored fact field exists (7 Sep 2026)

`DT_LLMFactTypes` — the registry defining which authored fact fields exist — has **exactly one row**: `PersonalityPrompt` (label "Personality", *"Defines the personality"*, knowledge type `Persona`).

⚠️ **There is no separate "Knowledge (Facts) Prompt" field**, contrary to how Epic's blog material reads. The Persona Modifier carries:

- **`personaFacts`** — one key, `LLMFactTypes:PersonalityPrompt` — the box you fill
- **`sessionFacts`** — `knowledgeType: Context`, empty — the runtime lane, fed by Verse

So the whole knowledge base goes into one box, against one 10,000-char ceiling.

ℹ️ Key-name inconsistency to know about: the **Character Definition** serialises the fact as `LLMFactTypes:PersonalityPrompt`, while the **Persona Device** serialises the same fact as `LLMFactTypes:SpeechPattern`. They are the same underlying entry — the registry row's localisation key is literally `SpeechPattern_FactProperty`. Write whichever key the target object already uses.

### ⚠️ The Prompt Editor caches its copy — and can clobber the asset (7 Sep 2026)

**The Character Definition editor picks up external changes live; the Prompt Editor Tool does not.** Writing `personaFacts` over MCP while both were open updated the Character Definition immediately, but the PET kept showing the old text until it was closed and reopened.

⛔ **This makes the PET a clobber risk.** A PET tab holding stale text will write that stale copy back over the asset if it saves. **Close and reopen the PET after any external change** to the persona.

### ⭐ Reading `Personality` tells you which persona is live

`persona_component.Personality` is `var<private>` but **publicly readable**. Printing it is the cheapest way to prove which persona an NPC is actually running.

Used to settle a real question: after a Persona Device's `bindDefaultPersonaToSpawnedNPC` fired against an NPC whose Character Definition also carries a Persona Modifier, the NPC reported **the Character Definition's** personality text, not the device's. The CD's persona wins, or the device's bind never lands — either way, **the CD is where the character should be authored.**

## Behaviour of the model itself

### Moderation fires on benign content (observed 7 Sep 2026)

Running a museum-guide question through **Bulk Response ×5**, one of the five came back as `The response has been Moderated`. The content was a polite refusal to discuss a non-existent Fortnite season — nothing sensitive.

⭐ **So the moderation false-positive rate is non-zero even for wholesome content**, and it is per-response, not per-prompt. Two consequences: don't judge a persona's behaviour from a single response, and for anything shipped, wire `CommitSayEvent` rather than assuming the model's output reached the player.

ℹ️ Also seen during Bulk Response, editor-side only: `Ensure condition failed: Attempted to retrieve FAppTime on a thread where there is no inherited time context`. An engine threading ensure from the async bulk path. Non-fatal, no effect on results, recorded so the next person doesn't chase it.

### ⭐ An empty persona is not neutral — it is an assistant

Epic's stock "Alice" persona ships with a **blank** fact map and, spoken to in-session, presents as a **UEFN helper assistant** — offering to help with the project. So the base system instruction Epic assembles carries assistant framing, and that framing is part of the text consuming the 10,000-char budget.

### ⭐ Authored-prompt craft: don't call anything a "signature line"

A guide persona's identity section said *"Signature line: The Island remembers everything. I just help translate."* The model appended it to **almost every response**, including one-line redirects, which flattened the character and wasted output.

**The fix is to specify frequency, not just content:**

> You have one signature line: *…* Use it **sparingly** — as a greeting to a new visitor, or to close a long conversation. **Never end two answers in a row with it, and never append it to a routine reply.** Most of your answers should simply end when the fact is delivered. Vary how you close; do not use a catchphrase as punctuation.

⭐ **General rule: any trait stated without a frequency will be applied at maximum frequency.** "Signature", "catchphrase" and "always says" are instructions to repeat. Cost here was ~315 chars against a 10,000 budget — cheap insurance.

### ⭐ Recalled-but-wrong beats invented — and needs a different fix

Asked "who is the Dark Voyager?", an unfactualised persona described *"a mysterious pilot lost to the void… suit designed to withstand the crushing pressure of space"* — the **Chapter 1 cosmetic**, not the Chapter 7 villain. It was not hallucinating; it was recalling something real and wrong.

⭐ **A generic "never invent" guardrail cannot catch that.** It took a **named counter-fact**: *"He is not an astronaut or a space pilot. Ignore any other Dark Voyager you may know of."* → **Rule: for any name that meant something else in an earlier era of the game, write the disambiguation explicitly.**

Also worth copying: a behaviour that must not fail should be stated **twice** — once as a rule and once as a dialogue example. The "send them to another wing" redirect was stated both ways and went 5/5 on bulk testing, having failed outright without them.

## ✅⭐⭐ SOLVED — in-session conversation WORKS, with no Persona Device (8 Sep 2026)

**A player can hold a spoken conversation with an authored persona on v42.10, using only a Character Definition, a Verse behaviour and an NPC Spawner.** Confirmed live: prompt available, the guide answered in her own voice and personality, and correctly declined a question about a season that doesn't exist.

⚠️ **The "BLOCKED" verdict recorded earlier the same evening was WRONG.** It is struck through below rather than deleted, because how it was wrong is the useful part. **The mechanism of the correction: it was reasoned from a doc comment and never compiled.** `AddChatChannel`'s documentation says it *"fails if the `MaxSize` of the channel's `agent_group` is undefined"*, `MaxSize` appears in no digest, so the route was declared closed. **The sentence is stale. Build the channel and it registers fine.** The human refused to accept the conclusion, which is the only reason it was retested.

⭐ **The rule this earns, which is the same rule the `bEnablePersonaDevice` entry earns: a doc comment is a claim, not evidence. If a route can be settled by a compile, compile it before writing it off.**

### ✅ The working recipe

1. **Character Definition** — Custom type, Persona Modifier with facts + voice, `CharacterModifier_VerseBehavior` pointing at your behaviour class. A UI modifier for `displayName` is optional but gives the nameplate.
2. **In the behaviour**, define a member-info class and build a channel:

```verse
guide_member_info := class(has_voice_member_info):
    var CanBroadcast<override>:logic = true

GuideChannelName<localizes>() : message = "Museum"

Group   := agent_group(guide_member_info){}
Channel := voice_channel(guide_member_info){Name := GuideChannelName(), Group := Group}
```

3. **Register the channel on the NPC's entity:** `E.AddChatChannel(Channel)`
4. ⭐ **Add the NPC's OWN agent to the group:** `if (SelfAgent := GetAgent[]) { Group.AddMember(SelfAgent, guide_member_info{}) }`
5. **Per player, exactly once:** `Group.AddMember(P, guide_member_info{})`, `PlayerEntity.AddChatChannel(Channel)`, `P.SetConversationTarget(Persona, Channel)`
6. Place the NPC Spawner with the Character Definition. Voice Chat must be on in the client.

Required `using`s beyond the obvious: `/Verse.org/AgentGroup`, `/Verse.org/Chat`, `/Fortnite.com/Characters`.

### ⛔ The two bugs that made it look impossible

Both produced the *same* symptom — **the prompt renders but reads "unavailable"** — which is easy to misread as "the feature is gated".

**1. Re-registering the channel on a timer leaks one channel per tick.** A join loop running every 2 s drove the NPC's channel count past **216**. No persona can resolve which channel it is on. **Join each player exactly once** and keep a `[player]logic` map of who is done. Note `set Map[Key] = Value` is failable and must sit in an `if` condition.

**2. ⭐ The NPC must be a MEMBER OF ITS OWN GROUP.** This was the last blocker and the least obvious. Registering the channel on the NPC's *entity* is **not** the same as the NPC's *agent* being a member of the `agent_group` that defines who is in the channel. With member count 1 (the player only) the prompt appears and the NPC ignores you completely. Adding `GetAgent[]` to the group takes it to 2 and everything works.

⭐ **Diagnostic that finds both: log `Group.GetMemberMap().Length` and `Entity.GetVoiceChannels().Length` on a change-only basis.** Healthy steady state is **member count 2** (NPC + player) and **channel count 2** (the explicit `AddChatChannel` plus one `SetConversationTarget` appears to add implicitly). Climbing numbers mean the leak; member count 1 means the NPC is not in its own channel.

### ✅ Publishability caveat CLEARED (8 Sep 2026)

The recipe was first proven with all Scene Graph Experimental Features enabled, raising the worry that constructing a `<final>` subclass of `<internal>` `chat_channel` only compiled because experimental mode suppresses those errors. **Tested: flags OFF, rebuilt, compiles clean and works.** The recipe is **production-viable and does not require experimental mode.**

⭐ Corroborating evidence found the same night: **Epic's own documentation ships this exact pattern** (see below), and Epic would not document a sample that requires experimental flags.

### 📘 Epic DOES document this — the page to find first

[Add an NPC to a Conversation in UEFN](https://dev.epicgames.com/documentation/fortnite/add-an-npc-to-a-conversation-in-unreal-editor-for-fortnite) carries a ~90-line sample doing exactly what was reverse-engineered here: `agent_group` + `voice_channel` built in Verse, the NPC added to its own group, `AddChatChannel`, `SetConversationTarget`. **No Persona Device.**

⚠️ **We never found it because the neighbouring page, [Create a Persona](https://dev.epicgames.com/documentation/fortnite/create-a-persona-in-unreal-editor-for-fortnite), documents the Character Definition and Prompt Editor and says nothing about how a player actually speaks to the NPC.** The setup is split across two pages and only the second one matters for getting a conversation running.

⭐ **Rule earned: read Epic's docs FIRST, then verify against the digest.** Both halves are needed — the docs contain the working recipe, *and* the `MaxSize` sentence that wrongly closed this whole investigation for an evening. Neither source is authoritative alone.

**Where Epic's sample is better than a first hand-rolled version:**

- **One-shot in `OnBegin`**, no polling loop — no channel leak by construction.
- **`OnEnd` tears down** with `ClearConversationTarget()` per player.

**Where Epic's sample is worse, and worth not copying blindly:**

- Everything hangs off one `if` block requiring `Players[0]` to exist **at NPC spawn**. If the NPC spawns before any player is in the playspace, the block fails and **nobody is ever added**; **late joiners are never added at all**. A spawner delay can hide this by winning the race.
- `team_agent_group` is declared and never used — dead code.
- ⚠️ `Players[0].GetSimulationEntity[]` — in the v42.10 digests `GetSimulationEntity` is defined on `entity`, `creative_device` and `creative_object`, **not on `player`**. May not compile as published.
- Their `team_member_info` is an empty `class(has_voice_member_info){}` — fine, since `CanBroadcast` has a default. Overriding it to `true` explicitly is optional.

**Best build = their structure, our robustness:** one-shot setup, NPC in its own group, a `[player]logic` guard so late joiners are handled exactly once, plus their `OnEnd` teardown.

### ⚠️ The interaction prompt is GLOBAL, not proximity-gated (8 Sep 2026)

Once `SetConversationTarget` is called, **the hold-to-talk prompt is available anywhere in the level**, at any distance from the NPC.

- `hearingRange` on the Persona Modifier (default 3000 = 30 m) governs **audio falloff**, not whether a conversation can be started.
- `SetConversationTarget` is **global per-player state** — one target at a time, no spatial component.

⭐ **Implication for any experience with more than one persona:** every NPC is permanently talkable from anywhere, and **the last `SetConversationTarget` call silently wins.** A room-per-guide design will not work without explicit gating.

**The fix is `ClearConversationTarget()`** — set the target on approach, clear it on departure, so the prompt follows the visitor. Proximity must be driven by your own code (trigger volume or distance check); the persona system provides none.

### 📊 Free measurement: response latency

`StartHearEvent` → `CommitSayEvent` timestamps from the first working session: **8.2 s, 8.2 s, 10.1 s**. That includes the player's speaking time, so it is an upper bound — subscribe to **`StopHearEvent`** instead for true think-time.

### ⛔ You cannot log what the persona said

`CommitSayEvent` fires reliably, but printing the `message` yields `[Redacted] -1931540415` — **Verse redacts `message` content in the output log.** You can detect and time responses; you cannot capture the text. Transcripts still need screenshots.

---

## ~~⛔⭐ IN-SESSION CONVERSATION IS BLOCKED ON v42.10~~ — CORRECTED, see above (recorded 7 Sep 2026, overturned the same night)

**~~You can author a persona completely, test it fully in the Prompt Editor, and have no legitimate way for a player to speak to it in a session.~~** Kept for the record because the reasoning is instructive: steps 1–2 below were established by **measurement** and remain true; step 4 was assumed from documentation and was **false**.

1. ✅ A persona on a Character Definition works. `persona_component` attaches, the facts load, the Prompt Editor converses with it.
2. ⛔ For a player to *talk* to it in-session, the persona and the player must share a **voice channel** — `PromptToTalk` and `SetConversationTarget` both require one.
3. ⛔ **Nothing auto-creates that channel.** Verified with a polling probe on the NPC's own entity: `GetVoiceChannels().Length` is 0 at `OnBegin` and stays 0 for the life of the session, including after a Persona Device binds to the NPC on spawn.
4. ⛔ ~~**The Verse route to build one is a dead end.** `voice_channel` needs an `agent_group`, and `AddChatChannel`'s own doc says it *"fails if the `MaxSize` of the channel's `agent_group` is undefined"* — but `MaxSize` appears exactly once across all three v42.10 digests: in that sentence. It is not a field on `agent_group`, so it cannot be set from Verse.~~ **FALSE — the doc sentence is stale; the channel builds fine.**
5. ⛔ The only thing that also creates the interaction UI is the **Persona Device**, and it is gated off (below).

### The Persona Device gate

`/CRD_AIPrompt/Device_AIPrompt.Device_AIPrompt_C` — `itemName` **"Persona Device"**, *"Create a new persona to interact with, powered by a LLM."*

Creative tags: `Device`, **`DeviceCreativeHidden`**, **`DeviceUEFNExperimental`**, `DeviceUEFNReady`, `NewThisSeason`, `RecentlyAdded`.

- ⛔ **Not in the Content Browser or Place Actors** in v42.10.
- ⛔ **Not in Project Settings** — neither Experimental Access nor Beta Access lists it (checked visually; Beta Access shows only Python Editor Scripting, Scene Graph System, Custom Items and Inventory, UEFN MCP Toolsets).
- 🔍 A descriptor key exists: `dataSets.experimental.personaDevice.bEnablePersonaDevice`. Present-and-**false** in some projects, **absent** in others.
- ⛔ **Setting it to `true` is INERT.** Tested 7 Sep: added the key, restarted UEFN. The key **survives** the restart (so v42.10 still parses it) but the device stays unavailable and the disallowed-reference error still fires. The switch behind it is gone.

⭐ **The nuance worth keeping:** the persona **API** is production — `persona_component`, `ai_session.Prompt`, `RegisterAction`, `SetConversationTarget` are plain public Verse with no experimental gate. It is specifically the **device** that is hidden. *(In hindsight, with the Verse route proven to work, Epic steering creators off the device and toward Character Definitions + Verse reads as deliberate rather than as a hole.)*

ℹ️ **Timeline, for context on how fast this is moving:** announced as testing-only, made publishable around **16 July 2026** (Epic blog post; v41.30, 30 July, as the shipping version), and by **v42.10 in September** the device is hidden again. Do not assume any finding here survives a version bump.

### ⛔ Placing the device anyway makes the project UNPLAYABLE

Instantiating the Blueprint class directly (MCP `SceneTools.add_to_scene_from_asset` on `/CRD_AIPrompt/Device_AIPrompt.Device_AIPrompt_C`) **does** place a working device. Do not do it:

```
UEFNValidation: Error: Disallowed reference to /CRD_AIPrompt/Device_AIPrompt.Device_AIPrompt_C
UEFNValidation: Error: Disallowed reference to /Script/AIPrompt.LLMPersonaConversationComponentHolder
LogValkyrie: Error: FlowStep_RunLocalValidation(): Failed request to validate project source data
LogValkyrie: Unable to play: Validation failed
```

⚠️⚠️ **The failure is delayed, and that is the trap.** From the moment it is placed, every save logs `AssetCheck: Error: … illegally references: /CRD_AIPrompt/Device_AIPrompt` — **and the project still plays.** It kept playing for roughly two hours and several sessions. Then a later launch needed upload validation and the project became unplayable with no new change to explain it.

⭐ **Rule earned: an `illegally references` / `Disallowed reference` error is a stop sign, not a warning.** Do not test around it. The gap between "the error appears" and "the thing breaks" is long enough to make the eventual failure look unrelated to its cause.

### ⭐ Live edit bypasses upload validation

The device genuinely worked once — a full spoken conversation with Epic's default "Alice" persona, prompt UI, hold-to-talk, the lot. That was only possible because the actor was **live-pushed into an already-running session**. Launching a *new* session runs upload validation, which fails.

⭐ **So a live-pushed change can run in a session even when the project cannot pass validation.** Useful (fast iteration) and dangerous (it proves nothing about validity). **"It worked in the session" is not evidence the project is valid** — and there is no committable revision of such a state to return to.

### GameplayEventFunction bindings — where they live, and how to write them

⭐ **The subscription is authored on the FUNCTION side, not the event side.** A device event (e.g. the NPC Spawner's `On Spawned`) shows "0 Array elements" with **no ⊕** — that is a read-out, not an input. You add the subscription on the *function that should listen*, and it names the event it wants.

Serialised form:

```
(DefaultHandlerFunctions=(),Instance=(EventSubscriptions=((
  Object="/<level path>:PersistentLevel.Device_CharacterSpawner_C_UAID_…",
  EventDescriptor=(MemberName="On Spawned",MemberGuid=<guid>,bSelfContext=True)))))
```

⚠️ **Over MCP the two directions behave differently:**

- **Adding** works by writing that Unreal text form to the property via `ObjectTools.set_properties` — copy a known-good subscription authored in the picker rather than inventing the `MemberReference` shape.
- **Clearing does NOT work** with the "empty" text form `(DefaultHandlerFunctions=(),Instance=())` — `set_properties` returns `true` and nothing changes. To clear, write the **JSON** shape instead: `{"eventSubscriptions": []}`. That works.

### The Persona Device's own properties (for whenever it returns)

- `defaultPersona` — an `LLMPersona` struct the device owns: `name`, `bShowName`, `voice`, `hearingRange` (3000 default = 30 m), `volume`, `color`, `personaFacts`.
- `addPlayersToPromptListOnGameStart` — **true** by default; adds players to the persona's prompt list at game start.
- `addPlayerToDefaultPersonaPromptList` / `addAllPlayersToDefaultPersonaPromptList` — *"Adds the … Player to the Conversation so they can communicate to the persona in chat/voice."* Epic's own tooltip warns about ordering: *"Ensure the instigator is added to the Conversation and removed from other prompt lists BEFORE adding them to a new prompt list."*
- `bindDefaultPersonaToSpawnedNPC` — the bridge to an NPC Spawner's `On Spawned`.
- ℹ️ Observed: an **unbound** device's prompt is available anywhere in the level, not gated to any NPC. Proximity would have to come from binding or from managing prompt lists.

## ⭐ Structured Output — how it is authored and exported (8 Sep 2026)

Authored in the **Prompt Editor → Structured Output tab** (UI only; the definitions are not reachable over MCP). The tab has an **export** that writes a real Verse file to the project, named `<persona>_<structname>.verse`.

**What the export actually contains** — more than expected, and worth reading before hand-writing any of it:

```verse
using { /UnrealEngine.com/Conversations }

guide_module := module:

    PersonalityPrompt<public><localizes> : message = "…the entire fact sheet…"

    understood_era<public> := struct:

        @ai_description("opens the barrier if true")
        openBarrier<public> : logic = false

    guide_persona_component<public> := class(persona_component):
        block:
            if (SetPersonality[PersonalityPrompt]) {}
            set PromptInterruptionRule = persona_interruption_rule.IgnoreUntilFinished
            if (SetVoice[aura_voice{}]) {}
```

Three things it gives you: the **whole personality prompt as a `message` literal**, the **response struct**, and a **self-configuring `persona_component` subclass**. So an entire persona can live in version-controlled Verse rather than only inside an asset.

### ⭐⭐ `@ai_description` is an INSTRUCTION, not a comment

```verse
@ai_description("opens the barrier if true")
openBarrier<public> : logic = false
```

**This attribute is sent to the model.** It is the mechanism behind the digest's *"field names and type names guide the model on how to fill out the structure"*, and it is what `LLMSession.StructFieldDescriptionCharLimit = 2000` budgets for.

⚠️ **It reads like a code comment and is easy to write as one.** The first instinct on reading it was that it documented the field for humans.

⭐ **Craft rule: describe the JUDGEMENT, not the MECHANISM.** *"opens the barrier if true"* asks the model to decide whether to open a barrier — something it has no basis to assess, since it does not know what a barrier is or when opening one is appropriate. Ask instead for the thing it can genuinely judge from the conversation, and let your code own the consequence:

> *"Set true only when the visitor has shown they understand Chapter 7 — for example naming the Zero Point Shards, Geno, or the Seven in their own words. Set false if they have only greeted you or asked one simple question."*

The same applies to **field names**, which are also prompt surface: `visitorUnderstandsChapter7` steers better than `openBarrier`.

### ⚠️ Save the Prompt Editor when closing, even after exporting

Closing the PET after defining a struct prompts to save. **Save it.** The export and the save do different jobs — the `.verse` file is the type your code compiles against; the persona asset holds the definition. Declining keeps the file while the persona forgets the struct.

## ⭐ Proximity gating — the fix for the global prompt (8 Sep 2026, working)

The always-on prompt problem is solved by driving `SetConversationTarget` / `ClearConversationTarget` from distance. Confirmed in play: the prompt appears and disappears with approach and departure.

**Two radii, not one:**

```verse
TalkRadius    500.0   # cm - set the conversation target
ReleaseRadius 700.0   # cm - clear it
```

⭐ **The gap is hysteresis and it is not optional.** With a single boundary, a visitor standing on it flickers the prompt on and off every poll.

⭐ **Use `DistanceXY`, not `Distance`.** Ignoring height means a guide on a plinth, or a visitor on a step, still measures as standing with her rather than metres away vertically.

⚠️ **Type trap:** `fort_character.GetTransform()` returns a `/UnrealEngine.com/Temporary/SpatialMath` transform, so the distance helpers must come from that module — needs `using { /UnrealEngine.com/Temporary/SpatialMath }`. This is the same duplicate-`vector3` collision recorded for `Focus` in [02-npcs-and-ai.md](02-npcs-and-ai.md).

## ⭐ Prompt craft: a restriction with no positive counterpart over-applies

A guide persona's identity said only:

> **You only know Chapter 7.** For any other chapter, send the visitor to that wing's guide and do not answer the question.

Asked *"what is Chapter 7 about?"* — her actual subject — **she refused, saying she was only there to discuss Chapter 7.** The phrase "Chapter 7" appeared in the sheet exclusively next to a refusal, and the dialogue example reinforced it, so the nearest matching pattern to any Chapter 7 question was the decline.

**The fix is a matched pair — state the permission as explicitly as the prohibition:**

> - **Chapter 7 is YOUR subject. Answer questions about it fully and gladly** — including plain ones like "what is Chapter 7 about?", which are exactly what you are here for. **Never deflect a question about your own chapter.**
> - **Redirect ONLY for OTHER chapters.** If a visitor asks about Chapters 1 to 6, send them to that wing's guide and do not answer.

Naming **Chapters 1 to 6** explicitly also helps — "any other chapter" left the model to infer what counted as *other*. Cost 233 chars; redirects still work, verified.

⭐ **General rule: any scope restriction needs an explicit positive half, or the model applies the restriction to the scope it was meant to protect.**

## ⛔ There is NO runtime fact-injection API (8 Sep 2026)

Asked whether runtime-injected context counts against the 10,000-character assembled ceiling. **The question has a prior question in front of it: on v42.10 you cannot inject context facts at all.**

- ⛔ **`sessionFacts` is read-only on the asset.** `ObjectTools.set_properties` refuses it: *"the following properties could not be set: sessionFacts"*. Its `factMap` is `{}` and cannot be authored.
- ⛔ **`ai_session` has exactly three members** — `ClearHistory()`, `Prompt(Prompt, response_type)`, `RegisterAction(...)`. **No `AddFact`, no `AddContext`, no pin API.**

⚠️ **Documented-but-unexposed, again.** `Prompt`'s own doc says it sends *"the session's full context (personality, pinned facts, history)"* — but nothing in any digest can pin a fact. Same shape as the `MaxSize` sentence that wrongly closed the conversation investigation for an evening: **Epic's documentation describes a system slightly ahead of what v42.10 exposes.** Treat any doc-only capability as unproven until compiled.

### What this means for runtime context

Only two runtime channels exist, and neither is a "fact":

| Channel | Budget it draws on |
|---|---|
| `ai_session.Prompt(message, type)` | a **conversation turn** — history is auto-compacted by the runtime, so probably *not* the 10,000 instruction ceiling (untested) |
| `RegisterAction` name + `@ai_description` | `LLMSession.StructFieldDescriptionCharLimit` = **2000**, a separate budget |

⭐ **So the authored sheet's remaining headroom is probably NOT also a "runtime context budget"** — but not because runtime context is free. Because runtime *facts* do not exist, and the alternatives draw on different budgets.

### Revised test, for whenever it matters

The obvious test — *"add a 1,500-char session fact and see if the session opens"* — **cannot be run**; there is no way to author one. The meaningful replacements are:

1. Send a deliberately large `Prompt` message on an already-open session. If it returns `ai_character_limit_error`, prompts have their own ceiling; if it answers, conversation turns are outside the instruction budget.
2. Register several `RegisterAction` definitions with long `@ai_description` text and find where 2,000 bites — per-field or per-session.

Both need only the Prompt Editor, no session build.

### 🔍 Runtime context — the two candidate channels, and what to test

**All hypotheses; nothing here is compiled or run yet.**

**Channel A — `ai_session.Prompt` as a context carrier.** Viable but not silent:

- It is a **real model call** returning a real typed response. Every context injection costs a round trip.
- **It enters history, and history is auto-compacted.** A fact sent five turns ago may be summarised into something useless. **Re-send on each approach; never rely on persistence.**
- **The response also enters history.** If the struct or wording invites the persona to say something, a later spoken reply may reference an answer the player never heard. Mitigation to test: a minimal struct plus a prompt phrased as stage direction — *"note: the visitor is now at the Shard exhibit; acknowledge nothing yet"*.
- ⚠️ **It is NOT gated by the interruption rule.** `PromptToTalk` is `<transacts><decides>` and fails when `persona_interruption_rule` forbids it; **`Prompt` is only `<suspends>`, with no `<decides>`.** So a context prompt fired mid-answer is not refused or deferred — it just runs. **Two concurrent calls on one session with no documented ordering.** Watch for interleaving corruption, not just a stutter.

**Channel B — `RegisterAction` descriptions as standing context.** Probably the better shape:

- The `Name` and `Description` are *"provided to the AI in the prompt"* and are what the model *"sees when deciding which capability to call"* — so **the description reaches the model whether or not the action is ever called.**
- Draws on `StructFieldDescriptionCharLimit` (2000), **not** the 10,000 instruction ceiling.
- **Creates no conversation turn**, so nothing to compact away.
- ⭐ **Use `Required := false` for a pure context carrier.** `Required := true` forces the model to emit that struct on *every* response — pointless output and latency when only the description is wanted. Reserve `true` for the actual judgement action, where a verdict every turn is the point.
- ❓ **The open question: can a description be updated by unregistering and re-registering?** The `cancelable` return suggests that is the intended update path. If so, a manager rewriting a "current location" action on approach gives standing context with no history pollution. **Unproven — compile it before believing it.**

### ⭐ The pattern, stated both ways

Three v42.10 capabilities are described in documentation but absent from the build: `agent_group.MaxSize`, `bEnablePersonaDevice`, and pinned facts.

**The pattern points in both directions, which is why the rule is worded as it is:**

- **`MaxSize`** was a doc claim that **wrongly closed a route that works** — a false negative that cost an evening.
- **Pinned facts** is a doc claim that **wrongly opens a route that does not exist** — a false positive that would have cost the next one.

⭐ **"Unproven until it compiles" covers both**, which is the reason to prefer it over "trust the docs" or "distrust the docs". Apply it to Epic's documentation exactly as [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md) applies it to AI-suggested APIs.

## ✅ Structured output drives gameplay — the door works (9 Sep 2026)

`RegisterAction` → structured verdict → `barrier_device.AddToIgnoreList` → that visitor walks through. Confirmed live: demonstrate understanding of Chapter 7 to the Chapter 7 guide and the door opens; a different era's guide, asked the same question, correctly declines and the door stays shut.

⭐ **`RegisterAction` DOES fire during ordinary voice conversation.** No `Prompt` call is needed to "activate" it — the model emits the struct alongside its spoken replies.

### ⚠️ The logging trap that cost an hour — log BOTH outcomes

The verdict callback originally logged only the `true` case:

```verse
OnVerdict(Verdict):
    if (Verdict.visitorUnderstandsChapter7?):
        …log and unlock…
    # no else branch
```

A healthy stream of `false` judgements therefore produced **zero log lines**, which was read as *"the action never fired"* — leading to a confident but wrong conclusion that `RegisterAction` does not participate in voice conversation, and to building an entire replacement mechanism for a problem that did not exist.

⭐ **Rule: any boolean judgement must log both branches.** Silence is not evidence of absence; it is evidence of a missing `else`. Same failure shape as the `MaxSize` episode — concluding from absence of evidence — arrived at from the opposite direction.

### ⛔ Out-of-band `ai_session.Prompt` is MODERATED — this revises the "Channel A" note above

The replacement mechanism was a `Prompt` call after each reply, asking for the verdict out of band, phrased as stage direction:

> *"INTERNAL ASSESSMENT ONLY — this is not the visitor speaking, and you must not reply to it or mention it…"*

**Result: 11 of 14 calls returned `The response has been Moderated`** (one `Unknown Error`, two succeeded). Worse, **the canned refusal surfaced on the player's screen** as a brief *"I won't do that"* banner mid-session.

⭐ **Why: a prompt instructing the persona to disregard the conversation frame and not reply reads as prompt injection — exactly what Epic's moderation exists to catch.** The phrasing intended to keep the assessment invisible is the phrasing most likely to be rejected.

⚠️ **This materially revises the runtime-context design recorded above.** `Prompt` was filed as "viable but not silent — costs a turn, enters compacted history, may leak into spoken replies." Add to that: **it is likely to be moderated outright if phrased as an aside, and its failures are visible to the player.** For anything the player should not see, `RegisterAction` is not merely cleaner — **it is the only channel that works.**

### The working shape

```verse
Persona.GetAISession().RegisterAction(
    prompt_binding_definition{ Name := …, Description := … },
    true,                          # Required - judge on every response
    understood_era,                # the exported struct
    OnVerdict)                     # callback: struct only, NO agent
```

- ⚠️ **The callback receives no agent.** Pair it with `StartHearEvent`, which does, and treat the most recent speaker as the subject.
- `Required := true` gives a verdict every response, which is what a continuous judgement wants.
- Guard with an `[agent]logic` map so each visitor is unlocked once, not on every subsequent reply.

⭐ **`barrier_device.AddToIgnoreList` unlocks per visitor**, so two people can be at different stages of the same exhibit without blocking each other — better suited to an open space than a literal door.

### ✅ Answered 1 Oct 2026 — the persona UI shows "unavailable" from spawn

> **Answered by Test 2 in the v42.30 section below:** the widget follows **group membership**; the button hint follows the **conversation target**. Register the channel once with the guide only, and add/remove visitors from the *group* on approach and departure. The "plain channel membership" hypothesis below was right — but the fix is `AddMember`/`RemoveMember`, not `AddChatChannel`/`RemoveChatChannel`. ✅ **Confirmed with two guides** in the applied section further down.

At spawn the talk UI renders with a guide's name and the state **"unavailable"**, before the player has approached anyone. It resolves on first approach and behaves correctly thereafter.

**Eliminated by testing:**

- ❌ *No conversation target set* — calling `ClearConversationTarget()` at join to establish a cleared state made no difference.
- ❌ *`hearingRange` too generous* — reducing it from 3000 to 900 on both personas made no difference.

**Remaining hypothesis: plain channel membership.** The player is added to every persona's `voice_channel` at join regardless of distance, and "unavailable" is simply the UI state for *in a persona's channel with no active conversation*. Fix to try: move `AddChatChannel` out of the join step into the proximity gate, paired with `RemoveChatChannel` on departure — with care, since asymmetric add/remove is what caused the 216-channel leak.

---

## ⭐ v42.30 — Epic's Conversations template, and two open questions settled (1 Oct 2026)

> **Provenance and confidence.** Source: **Epic's Conversations template**, the car-dealer haggling demo released with UEFN **v42.30** (`++Fortnite+Release-42.30-CL-58557680`). The digests, the template's Verse, its Character Definition and its NPC Spawner were read over MCP. **Tests 1 and 2 were compiled and run in a live session.** The test harness was removed afterwards and Epic's `persona_behavior.verse` restored, so the template is untouched.

### What changed in the API since v42.10

- ⭐ **`ai_session` and `persona_component` are unchanged.** Still `ClearHistory` / `Prompt` / `RegisterAction`; still no way to pin a fact. **Nothing in v42.20 or v42.30 makes conversations easier at the API level.**
- ✅ **The stale `MaxSize` sentence is gone.** `AddChatChannel` now reads *"Fails if the channel's agent_group already has more members than the channel type's limit"* (voice 80, text 100) and returns `result(void, add_channel_error)`; `RemoveChatChannel` mirrors it. **Confirms the 8 Sep correction** — and see [08-trusting-ai-on-uefn.md](08-trusting-ai-on-uefn.md) for what that stale sentence cost.
- ⭐ **New on `voice_channel` in v42.20** (`MinUploadedAtFNVersion := 4220`): `IsMemberSpeaking(Agent)`, `BeginBroadcastEvent()` and `EndBroadcastEvent()` (both `listenable(agent)`), `CanBroadcastChangeEvent()`. A cleaner way to know *who* is talking than pairing `StartHearEvent` with "most recent speaker" — relevant because the `RegisterAction` callback receives no agent. **Digest-verified, not yet run.**
- 🔍 **Persona Modifier fields missing from the v42.10 notes above:** `bOverrideBoundPersonaData` (*"author this character's voice and personality here. Leave either one blank to keep the one it ships with"*), `bOverrideCosmeticFacts` (*"If false, facts will be appended to the default ones of the NPC"*), `characterPersonaAssetBindings`, `bEnableFacialAnimations`, `attenuation`. Facts now sit in a `characterPrompt` sub-object (`PersonaPrompt`) holding `personaFacts` / `sessionFacts`. **Unknown whether these are new or were missed on v42.10.** The cosmetic-facts wording suggests some cosmetics ship with default persona facts.

### How Epic's template is built

- **The Character Definition is bare** — Custom type, default behaviour, **no Persona Modifier**.
- **Everything conversational sits on the NPC Spawner:** a **Persona Modifier** added under the spawner's extra NPC Character Modifiers (base prompt ≈ 2,000 chars: buyer role, currency, seven pricing rules), and `persona_behavior` as the spawner's Behavior Script override.
  - ⚠️ So *"the CD is where the character should be authored"* (7 Sep, above) is **one valid pattern, not the only one.** Spawner-side authoring lets one plain CD serve many personas.
- **Per-spawn randomisation:** `OnBegin` picks one of five customer types and a budget and appends them to the base prompt with `SetPersonality`.
- **Channel:** `agent_group` + `voice_channel` with NPC and player, `AddChatChannel` on `NPCEntity.GetSimulationEntity[]`, then `SetConversationTarget`.
- **Two `RegisterAction` bindings with `Required := false`:** "agreed a final price" (`logic` + `int`) and "told to leave".
- **NPC speaks first:** `PromptToTalk` with a stage direction — *"You just entered the office. Say hello to the Car Salesman…"*
- **Consequences deferred to `StopSayEvent`:** the action callbacks only set flags; score, fade and despawn run once the NPC finishes its sentence.
- **Fresh session per customer:** `DespawnAll` plus infinite respawn, so each customer is a new `persona_component` with a clean history.
- **Live captions:** `CommitSayEvent` → Verse fields on a UMG widget, cleared on `StopSayEvent`.

**Worth copying:** deferring consequences to `StopSayEvent`; NPC-speaks-first; fresh spawn = fresh session; captions. ⭐ **`message` is redacted in the log but renders fine in UMG** — the player can see a transcript even though you cannot log one.

**Not worth copying:** `GetPlayers()[0]` throughout (single player, no late joiners — the same flaw as Epic's docs sample); no `ClearConversationTarget` in `OnEnd`; the `AddChatChannel` result and `PromptFailureEvent` ignored; `team_agent_group` declared and unused (also in the docs sample).

⭐ **Moderation contrast, and it sharpens the 9 Sep rule.** The template's stage direction asks the NPC to *speak*, and passes. The out-of-band `Prompt` above asked it to *assess silently and not reply*, and was moderated 11 times in 14. **Rule: a stage direction that asks for speech passes; one that asks the model to hide something reads as injection.**

### ✅⭐ Test 1 — `SetPersonality` works mid-conversation

**Setup:** the personality ends with *"Your lucky number is 13."* After the greeting reaches `StopSayEvent`, `ai_session.Prompt("What is your lucky number?")` with a `{LuckyNumber:int}` struct returns **13**. `SetPersonality` then swaps that line for *"…7381. This replaces any lucky number you had before."* → `Prompt` again → **7381**. Asked out loud → **7381** in voice.

- ⭐ **A personality change reaches a live session on both the `Prompt` path and the voice path, and overrides an answer already in history** — it had said 13 seconds earlier.
- Both plainly worded `Prompt` calls returned without moderation.
- ⚠️ **This revises "There is NO runtime fact-injection API" above.** Facts still cannot be injected, but **`SetPersonality` is a working runtime context channel.** It *is* the authored prompt, so it draws on the 10,000-char assembled ceiling: **keep the base sheet short enough to leave room for whatever gets swapped in.**
- **Not tested:** cost or rate limits of rewriting it many times in one conversation.

### ✅⭐ Test 2 — what the talk UI shows, and the "unavailable" bug

Three runs. **A:** channel on the simulation entity; **B:** on the NPC entity — both with `SetConversationTarget` delayed 20 s. **C:** a simulated proximity cycle.

**Where the channel is registered makes no difference.** Simulation entity and NPC entity both succeed, report identical counts and converse normally.

**The talk UI has three independent layers:**

| State | Widget | Button hint |
|---|---|---|
| Not in the NPC's channel | none | none |
| In channel, NPC speaking | shows the NPC talking | no — press rejected (`IgnoreUntilFinished`) |
| In channel, no conversation target | "Talk Hold" | no — press ignored |
| In channel + target set | "Talk Hold" | **yes** — press works |

- ⭐ **Channel membership shows the widget. The conversation target adds the hint and makes the button work. The interruption rule decides whether a press is accepted while the NPC speaks.**
- ⭐ **`PromptToTalk` works before any target is set** — the NPC can greet a player who is merely in its channel.

**Run C — the proximity pattern, proven.** Channel registered **once at spawn with only the NPC in the group**. Then on a timer: 15 s away → `AddMember(Player)` + `SetConversationTarget` + greeting → 40 s → `ClearConversationTarget` + `RemoveMember(Player)` → 15 s → re-add.

| Phase | Group members | Channels | Observed |
|---|---|---|---|
| C1 away | 1 | 2 | no widget at all |
| C2 approach | 2 | 2 | greeting; press rejected while it spoke; then a normal exchange |
| C3 leave | 1 | 2 | widget vanished |
| C4 return | 2 | 2 | greeted again, waiting for input |

Two full cycles across two sessions; the channel count never moved. ⭐ **Group membership is the toggle, not channel registration** — which sidesteps the `AddChatChannel` churn behind the 216-channel leak entirely.

⚠️ **Correction to the channel-count diagnostic (8 Sep, above).** The baseline **channel count is 2 even with the NPC alone in its group and no target set**, so the second channel is *not* added by `SetConversationTarget`. A steady 2 is still healthy and a climbing count still means a leak — only the explanation was wrong.

### Revised recipe for multiple personas (supersedes the 8 Sep per-player join)

1. **At persona spawn:** build `agent_group` + `voice_channel`, add **the NPC only**, `AddChatChannel` once.
2. **On approach** (inside `TalkRadius`): `Group.AddMember(Visitor)`, `Visitor.SetConversationTarget(Persona, Channel)`, optionally `PromptToTalk` a greeting.
3. **On departure** (beyond `ReleaseRadius`): `Visitor.ClearConversationTarget()`, `Group.RemoveMember(Visitor)`.
4. **For local context:** `SetPersonality` with the base sheet plus a situation paragraph.
5. **Act on `RegisterAction` verdicts in `StopSayEvent`**, not in the callback.
6. **Show captions** from `CommitSayEvent`.

❓ **Open:** does a visitor's conversation history stay in the persona's session after `RemoveMember`? And with several visitors at one persona, they share one session — fine for a guided tour, but untested.

---

## ✅⭐ Applied: two personas, greetings and live captions (1–4 Oct 2026)

> **Provenance and confidence.** The v42.30 recipe above, applied to a two-guide museum build on UEFN v42.30 and play-tested in live sessions across two commits. Everything below was **compiled and run**; timings are measured, not estimated. Tooling traps hit along the way are in [05-editor-and-tooling.md](05-editor-and-tooling.md).

### ✅ The group-membership fix holds with two personas

- Visitors join a guide's group on approach and leave on departure; the guide's channel is registered once at spawn. **No widget away from both guides, the right guide's widget and hint at each, and no "unavailable" anywhere.** The two-persona question left open in Test 2 is closed.
- ⭐ **Clear the conversation target only if it is still yours.** `ClearConversationTarget()` clears *whatever* the player targets. Guard it: `if (Current := P.GetConversationTarget[], Current(0) = Persona)`. This works because `component` is `<unique>`, so personas compare with `=`.
- ⚠️ **Correction to the channel count, again:** `GetVoiceChannels().Length` counts **the whole island** — one baseline channel plus one per persona — whichever entity you ask. **3** with two guides; 2 in the one-NPC template. Steady = healthy; climbing = leak.

### ✅ Greeting on approach, then "welcome back"

- On arrival, after `SetConversationTarget`: `PromptToTalk` with *"A visitor has just walked up to you. Greet them in one or two sentences, introduce yourself, and ask what they would like to know."* A `[player]logic` map per persona switches later approaches to *"…welcome them back in one short sentence…"*. Walking between guides no longer replays introductions.
- **Never moderated** across a dozen greetings — consistent with the rule that stage directions asking for *speech* pass.
- The model already knows the **player's name** from session context; welcome-backs are personal with no extra work.
- Under `IgnoreUntilFinished`, `PromptToTalk` **fails rather than queues** if the persona is mid-answer to someone else. Log it; don't retry blindly.

### ⭐ Captions — how Epic does it, and what the platform allows

- **`CommitSayEvent` fires exactly once per reply**, however long. There is no per-sentence event, and Verse cannot split a `message`, so **true line-by-line caption sync is impossible.**
- **Epic's template does not page captions either** (checked in the asset and in play): its widget is an Overlay + background Image + auto-wrapping text block that **grows with the reply** (3–6 lines observed) and hides on `StopSayEvent`.
- ⛔ **The HUD Message device is the wrong tool:** fixed-size placements cut long replies off. Replace it with a custom widget.
- ✅ **A caption widget that works (`WB_Caption`):** Canvas → Overlay anchored bottom-centre, auto-size, rounded dark background, text wrapping at 1,000 px so the box grows taller rather than wider. Two Verse fields — `Caption` (message) and `CaptionVisible` (bool, bound to Visibility via **To Visibility (Boolean)**). One widget per visitor, added with `GetPlayerUI[P].AddWidget` on first use. Longest reply observed: 4 lines. *(Authoring traps: [09-custom-uis.md](09-custom-uis.md); MCP traps: [05-editor-and-tooling.md](05-editor-and-tooling.md).)*
- **Hold after speech:** captions hide **2.5 s after `StopSayEvent`**, are superseded by any newer line (a per-player generation counter), and clear immediately when the visitor walks away. **Without the hold, a one-line welcome-back committed a second before the persona stopped and flashed past unread.**
- ℹ️ **The brief one-line box on approach is Fortnite's own talk widget**, not yours (lighter tint, square corners). It appears on joining a persona's group. Nothing to fix.

### 📊 Measured: think time and caption lag (2 Oct 2026, 15 replies)

| Reply | Think time (prompt → voice) | Caption lag (voice → text) |
|---|---|---|
| Greetings / welcome back (3) | 1.5–1.6 s | **2.5–3.2 s** |
| Short-sheet persona answers (3) | 1.8 s | **3.3–3.6 s** |
| Long-sheet persona answers (9) | 1.8–2.1 s | **3.5–6.2 s** (avg 4.6) |

Measured in Verse with `GetSimulationElapsedTime()`: prompt = `StopHearEvent` (or the greeting call), voice = `StartSayEvent`, text = `CommitSayEvent`.

- **Think time is fast and flat** — the model is not the wait.
- ⭐ **Caption lag grows with reply length.** Shortest lines lag least; the longest question produced the longest lag. **Best explanation: voice streams from the first generated sentence, while the text is released only after the whole reply is generated and moderated.** That also explains why spoken replies start so quickly. **Inferred** — reply length cannot be measured directly, because the text is redacted in the log.
- **Nothing on your side can shorten it** — the caption is set in the frame the text arrives. Only a different API (streamed or per-sentence text) would.

### ✅⭐ The speaker label closes the perceived gap

On `StartSayEvent` the caption shows **"`<Name>`: ..."** immediately; on `CommitSayEvent` it becomes **"`<Name>`: `<reply>`"**. The reported effect: the caption gap stops registering as a gap at all, and switching between personas feels seamless.

- Built with `Join(array{SpeakerLabel(GuideName), Said}, LabelGap())` — **`Join` works on `message`s.**
- `GuideName` is an `@editable string` on the shared `npc_behavior`, set per Character Definition. ⭐ **Plain values are editable on a CD-hosted behaviour; only level-actor references are not** — which is exactly the asset/instance boundary in [02-npcs-and-ai.md](02-npcs-and-ai.md), seen from the permitted side.

### Small things worth keeping

- **Log both branches of a structured verdict.** Adding the `false` log immediately showed a persona unconvinced by a first answer and convinced by the second — and earlier showed five `true` verdicts going nowhere because **the manager device was missing from the level.** (The 9 Sep logging trap, earning its keep twice.)
- `AddChatChannel` and `player_ui.AddWidget` are **`no_rollback`**: call them outside any `if` condition or `<decides>` function, then test the result. *(The `no_rollback` rule in [01-verse-language-and-compiler.md](01-verse-language-and-compiler.md).)*

---

## ✅ Scaling to six personas (4 Oct 2026)

> **Provenance.** The same build on v42.30, extended over MCP and play-tested. Builds on the applied section above.

### How a new persona is made — no new code

- **Duplicate an existing Character Definition** (`AssetTools.duplicate`). Every copy keeps the shared Verse behaviour, so greetings, captions, proximity and the structured verdict work immediately.
- **Fact sheet** → `CharacterModifierPersona_C_0.CharacterPrompt` → `personaFacts.factMap["LLMFactTypes:PersonalityPrompt"]`, loaded verbatim and read back to verify. Sheets ran ≈ **8,000–9,000 chars** and all sessions opened — close to the 10,000 ceiling, with the buffer the ceiling section above recommends.
- **Voice** → `voiceModel` = `/CRD_AIPrompt/_Verse/VNI/AIPrompt.<voice>`, chosen from the **descriptor** to match each sheet's personality (Glamorous/Animated, Cinematic, Refined, Whimsical). ⭐ The descriptor table near the top of this file is the right tool for this — pick on descriptor, not on name.
- **Nameplate text** → `CharacterModifier_UI_C_0.displayName` / `shortDescription`.
- **Spawner** → `PID_Device_AISpawner_Character`, then `NPCCharacterDefinitionComponent.nPCCharacterDefinition`. Place personas **far enough apart that no two talk/release zones overlap** (15 m worked).
- ⚠️ **Still editor-only:** the `@editable` `GuideName` on each CD — MCP cannot reach the behaviour instance (see [05-editor-and-tooling.md](05-editor-and-tooling.md)).

### Observations

- **Switching between six personas is clean — the group-membership pattern scales.**
- One persona answered a question squarely inside its own subject with a canned *"I'm not here to talk about that."* **Bulk Response ×5 in the Prompt Editor found nothing**, so it is filed as a one-off, most likely a moderation substitution — that sheet carries the most violent wording (*shot*, *obliterated*). **Softer wording is the fix if it recurs.** Compare the benign-content moderation note above: the false-positive rate is non-zero and per-response.
- ⚠️ **Don't identify personas by position in logs.** Tagging log lines with the NPC's X coordinate worked for two personas; a third was placed at the same X as another. **Log a name field instead.**

### ⚠️ NPC nameplates are unreliable on v42.30 — despite correct settings

All six Character Definitions: `displayName` set, `showNamePlate` **Always**, `showIconMinimap` **Always**. Spawners carry no overrides beyond the CD itself. Results across launches:

| Run | Change | Nameplates seen |
|---|---|---|
| 1 | Island Settings `nameplateDisplayMode` = Default | some; gone after a push |
| 2 | clean launch, Default | none |
| 3 | set to **Always over MCP** (saved; file confirmed) | three of six |
| 4 | editor restart, clean launch, still Always | none |
| 5 | Default → Always **toggled in the editor UI**, compile + push | two of six |
| 6 | one CD amended and **saved manually**, push | **all but one** — including a CD that was not touched |

- **Not deterministic.** The same configuration gave different subsets.
- **Edits made in the editor UI followed by a push correlate with improvement; the MCP change alone did not hold across a restart.** That is the same incomplete-MCP-write shape recorded in [05-editor-and-tooling.md](05-editor-and-tooling.md).
- **Ruled out:** `maxTrackersOnHUD` (governs tracker devices only), spawner property overrides, distance (`bLimitNamePlateMaxDistance` false), focus (`Never`), line of sight (`AlwaysShow`).
- **Unproven hypothesis:** an interaction with the persona / conversation UI, since every affected NPC carries a persona. **Untested with a persona-less NPC** — that is the one experiment that would settle it.
- ⭐ **Decision: do not rely on NPC nameplates for anything players need.** Use **signs or plaques beside each persona** (name + subject) and a **`map_indicator_device`** per persona for the map ([03-devices-and-interaction.md](03-devices-and-interaction.md)). Leave Island Settings on Always so any nameplates that do appear are a bonus.
