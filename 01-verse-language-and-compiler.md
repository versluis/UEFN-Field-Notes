---
verified-on: UEFN v41.10 – v42.10
last-reviewed-against: UEFN v42.10 (3 Sep 2026)
---

# Verse Language & Compiler

Effects, syntax traps, and compiler behaviour. Hard-won from real project versions (June 2026 onward); dates on each entry.

## Syntax & structure

- `@editable` must be on **its own line** above the field.
- Block comments are `<# #>` (or indented `<#>`), **not** `#( #)`.
- Verse indentation is syntactically significant (Python-style); a single space off produces confusing errors.
- `void` functions ending in a trailing `if`/`else` cause persistent, misleading compile errors. ⚠️ **Narrowed 29 Aug 2026: a bare trailing `if` with NO `else` is fine.** A production function ending in an unguarded `if` compiles clean. Only the `if`/`else` form is affected — stop adding defensive trailing `Print`/`Sleep(0.0)` filler to functions that merely end in an `if`.
- Animation asset fields use optional type syntax: `@editable WalkAnim : ?animation_sequence = false`, unwrapped before use. Needs `using { /Verse.org/Assets }` for the `animation_sequence` type itself.

## ⚠️ The failable-conjunction trap (the one that parked a project)

**Never pair a failable call you *need* with a failable call you merely *want* in the same `if` condition.** Comma-separated failables in Verse are a **conjunction** — if any one fails, the entire block is skipped. This is the bug behind a commit message reading *"this commit basically ruined everything"* (29 Jun 2026), diagnosed only two months later (26 Aug 2026) by diffing commits:

```
# BROKEN — GetPlayAnimationController[] was engine-broken at the time, so it always failed,
# so NavigateTo (inside the block) NEVER RAN. The NPC froze.
if (Anim := Character.GetPlayAnimationController[], Navigatable := Character.GetNavigatable[]):
    Navigatable.NavigateTo(Target)
```

**The symptom looks nothing like the cause.** "NPC stands still" reads as a navigation failure; the actual fault was an animation call in the same conditional. An entire evening was lost to this, and the project was parked over it.

**Rule: nest the optional capability *inside* the essential one, so the essential path always survives.**

```
# CORRECT — navigation happens whether or not animation is available
if (Navigatable := Character.GetNavigatable[]):
    # ... nav math, Target := MakeNavigationTarget(...)
    if (Anim := Character.GetPlayAnimationController[], Walk := WalkAnim?):
        race:
            block:
                loop:
                    Anim.PlayAndAwait(Walk)
            block:
                Navigatable.NavigateTo(Target)
    else:
        Navigatable.NavigateTo(Target)
```

The `race` idiom above is the **correct** pattern for "animate while navigating" — arrival cancels the animation loop. It was right in the original commit and stays right; only the conditional coupling was wrong.

## ⭐ The `no_rollback` trap — this has now bitten twice, learn it once

Any function that mutates a `var` acquires the `no_rollback` effect, and **a `no_rollback` function cannot be called from inside an `if` condition** (a failable context). Symptom: *"This invocation calls a function that has the 'no_rollback' effect, which is not allowed by its context."*

- **Fix: hoist the call into a local, then branch on the local.** `X := Thing.Mutator()` then `if (Y := X?):` — rather than `if (Y := Thing.Mutator()?):`.
- Alternatively make the function pure (no `var`) using a `for` comprehension.

## Access boundaries

- ⭐ **Why `@editable` is mandatory for assets, not a stylistic choice (established 28 Aug 2026).** Referencing project animation assets *directly* from inside an `npc_behavior` class **cannot work**, for two separate reasons the compiler reports together:
  1. `Invalid access of internal module (…)Anims` — the generated asset sub-modules are **internal**, not `<public>`, so a sibling module cannot import them.
  2. `Invalid access of scoped{…} data … from control scope …` — the assets are **scoped to the project root**, and a class body is a nested control scope outside it.

  **So `@editable ?animation_sequence` fields, assigned on the Character Definition asset, are the sanctioned route.**
- `material` is `epic_internal` — cannot be an `@editable` default. Workaround: two-prop swap (hide one, show the other). *(Roadmap note in [00-version-watch.md](00-version-watch.md): Verse for Materials may obsolete this Q1 2027.)*
- ✅ **`npc_behavior` can cast a discovered object to a CUSTOM Verse device class**, not just built-ins like `button_device`. Proven 29 Aug 2026: `my_manager_class[Obj]` after `FindCreativeObjectsWithTag`. **This is what makes an NPC→manager architecture possible** — a sandboxed NPC can call methods on your own devices. End the function with a non-conditional expression, or move the conditional into a separate `suspends` function invoked via `spawn`.

## Reading compiler output

- ⚠️ **Errors cascade — read the earliest one first (v42, 29 Aug 2026).** One stray token after a class declaration (`my_behavior := class(npc_behavior): xxx`) produced 12 errors in the file, including `OnBegin` "could not find a parent function to override" and "Unknown identifier `FindCreativeObjectsWithTag`" / "`GetAgent`" on untouched lines — because the class body no longer parsed. **Those downstream errors are symptoms, not faults.** Fix the lowest-line error, rebuild, then look again. Compare the `FindCreativeObjectsWithTag` missing-import trap ([03-devices-and-interaction.md](03-devices-and-interaction.md)): the same error text can mean three different things.
- ⚠️ **Digest noise in VS Code's Problems panel is not a build failure (v42, 29 Aug 2026).** A freshly opened project shows ~100+ errors, all in `Fortnite.digest.verse` under Built-in Digests, all *"Non-abstract class inherits abstract function `Cancel` from `cancelable`"*. The build server still reports success. Use the funnel icon to filter to the active file; only errors in your own `.verse` files count. Do not spend time on the digest.
