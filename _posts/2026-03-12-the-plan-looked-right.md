---
layout: post
title: "The Plan Looked Right"
date: 2026-03-12
series: "Building With AI"
---

I hadn't shipped personal code in over a decade. Then AI coding tools closed the gap between my engineering judgment and my ability to execute, and I built seven projects in ten days — a macOS task manager, a markdown reader, a Firefox extension, two iOS apps, an OmniFocus plugin, and a Git reimplementation in Rust. Five languages, five platforms.

I wrote structured postmortems for each one. The same root cause appeared in every single project.

---

On the OmniFocus plugin, the AI-generated implementation plan included an API surface table — 11 rows specifying exactly which call to use for each feature. Method signatures. Return types. It looked like a senior engineer's implementation document.

Five of the eleven entries were wrong. A 45% error rate on the most consequential part of the specification.

| Plan specified | What actually works |
|---|---|
| `this.plugIn.preferences` | `new Preferences()` at load time only |
| `prefs["key"] = value` | `prefs.write("key", value)` |
| `app.openURL(url)` | `URL.fromString(str).open()` |
| `task.dropped` | `task.taskStatus === Task.Status.Dropped` |
| `Actions/` + `Lib/` directories | Everything in `Resources/` |

Thirteen fix commits out of thirty-seven total. A 6.5:1 fix-to-feature ratio. A 10-minute spike — paste five lines into the OmniFocus console — would have caught all five before any feature code was written.

This wasn't an isolated incident. Across all seven projects:

A Firefox extension plan committed to `script.src` for injection and `setInterval` for intercepting a library initialization. Both wrong — one is an async timing issue, the other a fundamental constraint of the JavaScript event loop. The fix required a 140-line architectural rewrite post-merge.

A Swift plan committed to `createPDF(configuration:)`. Doesn't support pagination. Four fix commits and a full API pivot.

A Rust plan used `canonicalize()` as a security check on paths that don't exist yet. The check silently failed. The fallback proceeded without validation. A security bypass hiding behind a reasonable-looking API choice.

A SwiftUI plan assumed `@FocusState` sets accessibility focus on macOS. It doesn't. Three commits to diagnose a silent failure. The same plan assumed `#Predicate` expressions that compile will run. Nine categories of runtime crash.

Every one of these was discoverable in under ten minutes with a throwaway prototype. None required the full application to be built first.

---

The pattern has three properties.

The documentation says what the API *should* do. It doesn't say what happens at the boundary — cached pages, synchronous webpack blocks, non-existent paths, accessibility edge cases, layout engine feedback loops.

The plan stated each choice as a fact, not a hypothesis. The specificity created false confidence. It *looked* authoritative, so it wasn't questioned.

And every bug was discoverable with a trivial spike — but no spike was run, because the plan felt finished.

---

If you've been away from code for a while, you're more susceptible to this than someone who never left. Your engineering judgment is intact — you can evaluate whether architecture makes sense. What's degraded is your muscle memory for which specific APIs actually behave as documented. The plan feels right because the reasoning is sound. You don't have the recent scar tissue that says "actually, `canonicalize()` fails on non-existent paths."

AI makes this worse, not better. The planning phase is where these tools excel — they generate confident, well-structured plans with specific API selections. And here's the paradox: the more detailed the plan, the more it costs when the assumptions are wrong. A vague plan that says "inject a script into the page" gets validated empirically. A detailed plan that says "use `<script src>` for async loading since the map mounts after React hydration" gets implemented as specified — and the wrong assumption survives until post-merge.

My macOS app plan went through three deepening passes with eight research agents. Each pass added specificity to assumptions that turned out to be wrong. The deepening compounded speculation, not knowledge.

---

The discipline I arrived at is simple enough to be boring: nothing in the plan gets stated as fact until someone has seen it work. Every API choice is "approach TBD after spike" until a throwaway prototype confirms it. Twenty lines. Ten minutes. Run it.

I went from a 45% error rate on project one to zero new platform surprises on the later features of project five. Not because I memorized the APIs — but because I stopped trusting plans that hadn't been tested.

Coming back to code after a decade, the hardest thing to relearn wasn't any language or framework. It was the discipline of running something before believing it works.

---

*Part of a series on building software with AI. Patterns drawn from real commit histories, bug counts, and structured postmortems across Swift, JavaScript, Rust, and OmniJS.*
