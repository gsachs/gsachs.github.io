---
layout: post
title: "Spike Before You Specify: The 45% Error Rate on Platform API Assumptions in AI-Assisted Development"
subtitle: "What happens when you let an AI plan your implementation before running a single line of code"
date: 2026-03-11
series: "Building With AI"
---

I hadn't shipped personal code in over a decade. Then AI coding tools closed the gap between my engineering judgment and my ability to execute, and I built seven projects in ten days — a macOS GTD task manager, a markdown reader, a Firefox browser extension, two iOS apps, an OmniFocus plugin, and a Git reimplementation in Rust. Five languages, five platforms. I wrote structured postmortems for each one. The same root cause appeared in every single project.

**The AI's training data contains API documentation, not API behavior. The gap between the two is where every post-merge bug lives.**

## The Number That Changed How I Work

On my OmniFocus plugin project (ReviewLinter), the implementation plan included an "OmniJS API Surface" table — 11 rows, each specifying exactly which API call to use for each feature. It looked authoritative. It was detailed. It had method signatures and expected return types.

Five of the eleven entries were wrong. A 45% error rate on the most consequential part of the specification.

Here's what the plan said versus what actually worked:

| Plan specified | What actually works |
|---|---|
| `this.plugIn.preferences` | `new Preferences()` at load time only |
| `prefs["key"] = value` | `prefs.write("key", value)` |
| `app.openURL(url)` | `URL.fromString(str).open()` |
| `task.dropped` | `task.taskStatus === Task.Status.Dropped` |
| `Actions/` + `Lib/` directories | Everything in `Resources/` |

Each wrong call required a fix commit plus investigation time. The project ended with 13 fix commits out of 37 total — a fix-to-feature ratio of 6.5:1. More than a third of all commits existed solely to correct API assumptions that were stated as facts in the plan.

A 10-minute spike — paste five lines into the OmniFocus Automation console — would have caught all five.

## This Wasn't an Isolated Incident

The same pattern repeated across every project, every language, every platform.

**Firefox extension (BusAlert) — JavaScript:**
The plan committed to `<script src>` for injecting a page script and `setInterval` for polling a global variable. Both were wrong.

`script.src` creates an async fetch task. On cached/fast pages, React had already constructed the Mapbox map before the external script finished loading. The fix: `script.textContent`, which executes in the current task.

`setInterval` cannot interpose between two statements in the same synchronous block. Webpack bundles that assign `window.mapboxgl` and call `new mapboxgl.Map()` in the same block give no timer callback a chance to fire between them. The fix: `Object.defineProperty` on `window`, which traps the setter synchronously. This is a fundamental JavaScript runtime constraint, not an obscure edge case — but you only discover it by running code against a real page.

These two bugs required 160+ lines of corrective changes post-merge, including a full architectural rewrite of the most critical file in the project. The commit message for that rewrite? "bugfix." The most architecturally significant change had the least informative record.

**macOS GTD app (Krama) — Swift/SwiftUI:**
Three platform assumptions turned into expensive surprises:

`@FocusState` sets NSWindow first-responder but does NOT set `AXFocused = YES` on the accessibility element. This caused every UI test using `element.typeText()` to fail silently. Three commits to diagnose.

SwiftData's `#Predicate` macro compiles expressions that crash at runtime — nine distinct categories of them. `lowercased()`, optional force-unwrap, optional to-many traversal, enum cases, `Date()` in closures. Each one compiles cleanly. Each one crashes. The project ended up with a nine-item gotcha catalog that exists because every item was hit in production code.

`onDisappear` fires after the window title updates on macOS `NavigationSplitView`. This made navigate-away an unreliable synchronization point and required a dedicated solution document and multiple test flake investigations to diagnose.

**macOS markdown reader — Swift:**
The plan committed to `createPDF(configuration:)` for PDF export. The API doesn't support paginated output. This discovery, mid-build, cost four fix commits and a full pivot to `NSPrintOperation`. The plan had specified the wrong API with enough confidence that the implementation was built around it before the limitation surfaced.

**Git in Rust (vrit) — Rust:**
`std::fs::canonicalize()` returns an error on non-existent paths. The code used it as a path traversal security check during checkout — but during checkout, the target file doesn't exist yet. The security check silently failed, and the fallback (`unwrap_or_else`) proceeded without validation. A security bypass caused by an API precondition that isn't enforced by the type system.

**iOS habit tracker (Abhyasa) — Swift/SwiftUI:**
`aspectRatio(1, .fit)` inside a `LazyVGrid` triggers infinite layout recursion — child asks for width to compute height, container asks for height to compute layout, forever. `.swipeActions` is scoped to `List > ForEach` and silently does nothing elsewhere. The `#Unique` macro requires iOS 18+, not iOS 17+ as assumed. SwiftUI's Picker requires exact type matching between the selection binding and tag types — `Binding<T>` and `Binding<Optional<T>>` are different types, and mismatch causes silent failure with no error.

## The Pattern

Every one of these bugs shares three properties:

1. **The API documentation (or Claude's training data) says what the API *should* do.** It doesn't say what happens at the boundary — cached pages, synchronous webpack blocks, non-existent paths, macOS-specific accessibility behavior, layout engine feedback loops.

2. **The plan stated the API choice as a fact, not a hypothesis.** There was no "approach TBD after spike" anywhere. The specificity of the plan created false confidence. It *looked* authoritative, so nobody questioned it.

3. **The bug was discoverable in under 10 minutes with a running prototype.** Every single one. Paste five lines into a console. Inject a 20-line script into a real page. Build a one-view app with a single `@FocusState` binding. Run `canonicalize()` on a path that doesn't exist. None of these required the full application to be built first.

## Why AI Makes This Worse, Not Better

With traditional development, you write the code yourself and hit the API boundary immediately. The feedback loop is tight: write, run, discover the behavior, adjust.

With AI-assisted development, the planning phase is where the AI excels — it generates detailed, confident, well-structured plans with specific API choices. The output looks like a senior engineer's implementation document. And if you've been away from the code long enough, you're especially susceptible to this. Your engineering judgment is intact — you can read a plan and evaluate whether the architecture makes sense. What's degraded is your muscle memory for *which specific APIs actually behave as documented*. The plan feels right because the reasoning is sound. The API choices feel right because they match the documentation. You don't have the recent scar tissue that says "actually, `canonicalize()` fails on non-existent paths" or "actually, `@FocusState` doesn't set AXFocused on macOS."

So you build features on top of the plan before validating the foundation.

My Firefox extension plan was *deepened* — run through a second analysis pass that added more detail. My macOS app plan went through *three* deepening passes with eight research agents. Each pass added specificity to assumptions that turned out to be wrong. The deepening compounded speculation, not knowledge.

Here's the paradox: the more detailed the AI-generated plan, the more it costs when the assumptions are wrong. A vague plan that says "inject a script into the page" gets validated empirically. A detailed plan that says "use `<script src>` for async loading since the map mounts after React hydration" gets implemented as specified — and the assumption survives until the feature is built and failing in production.

## The Fix Is Embarrassingly Simple

**Phase 0 of every project is a throwaway spike.**

Before any feature code, before any plan deepening, write the smallest possible test of your platform assumptions. The acceptance criteria:

- Can I do the one thing the entire project depends on?
- Does the API behave the way the plan assumes?
- What happens at the boundary the documentation doesn't cover?

For the Firefox extension, Phase 0 would have been: inject a script, capture a reference to the map object, post a message back. Twenty lines. Run it on the actual page. If `script.src` loads too late, you discover it in 5 minutes instead of post-merge.

For the macOS app, Phase 0 would have been: create one SwiftData model, one `@Query`, one `#Predicate` that filters by a computed property. Run it. Hit the first runtime crash. Find workaround 1 of 9 before the architecture depends on there being zero.

For the OmniFocus plugin, Phase 0 would have been: paste five API calls into the Automation console. Watch three of them fail. Update the plan with what actually works.

**The plan should say "approach TBD after spike" for every platform API the developer hasn't called in this session.** Not "TBD after research." Not "TBD after reading the docs." After spike — meaning after running code in the target environment and observing the behavior.

## What Changes in Your CLAUDE.md

If you're using AI coding tools with a project constitution (CLAUDE.md, rules files, system prompts), encode this as a non-negotiable rule:

> **Spike before specifying platform APIs.** If Claude names a specific API it hasn't called in this session, mark it "approach TBD after spike" in the plan. Validate in Phase 0 before writing any feature code. Plans for underdocumented platforms should specify *what* to achieve, not *which API to call*.

And this companion rule, which I learned the hard way:

> **Never deepen a plan more than once.** Deepening adds detail to unvalidated guesses. Each pass compounds speculation, not knowledge. One plan pass, then build.

## The Metric That Matters

After adopting this approach, my later projects had measurably different trajectories. The iOS habit tracker (Abhyasa) absorbed its platform gotchas during the MVP phase, then shipped three post-MVP features (garden tier system, ambient environment, nature glyph calendar) with zero new platform surprises. The structural lessons transferred — once you've spiked the platform, subsequent features build on validated ground.

The broader arc: I went from a 45% API error rate on project one to zero new platform surprises on the post-MVP features of project five. Not because I memorized the APIs — but because I stopped trusting plans that hadn't been tested. Coming back to code after a decade, the hardest thing to relearn wasn't any language or framework. It was the discipline of running something before believing it works.

The question I now ask before every project: **What's the one platform behavior this entire project depends on — and have I seen it work with my own eyes?**

If the answer is no, the plan isn't ready. The spike is.

---

*This article is part of a series on what I learned building seven projects in ten days with Claude Code. Each project produced a structured postmortem and learn report. The patterns described here are drawn from real commit histories, real bug counts, and real time costs across Swift, JavaScript, Rust, and OmniJS.*
