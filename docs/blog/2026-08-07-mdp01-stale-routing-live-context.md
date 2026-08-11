---
layout: post
title: "When the Case Base Says One Thing but the Sensors Say Another"
date: 2026-08-07
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [cbr, ai-resolution, event-driven, design]
series: issue-82-cbr-rerouting-context
---

The AI resolution agent routes cases based on CBR similarity — high confidence goes to the LLM, medium to an operator with suggestions, low to unassisted. The problem: those stats are computed once, at case start, and never re-evaluated. If the case's working context changes while it's sitting in the AI queue — new sensor readings arrive, the situation evolves — the original routing decision is stale.

The fix needed to detect context changes on queued cases, re-run CBR retrieval, and re-route when the similarity band drops. Three trigger mechanisms were on the table: the agent's existing poll cycle, a CDI event observer, or a hybrid.

The engine already fires a `CaseContextUpdatedEvent` on every working layer change. It has zero observers — nobody consumes it. It was designed for exactly this kind of consumer-side extension. The observer filters by layer ("working" only), checks whether the case is in the AI resolution queue (one query), and debounces at 30 seconds per case. If the re-evaluated CBR band has dropped below the AI threshold, it calls `queueService.escalate()` — the same API the agent itself uses for timeout sweeps.

Event-driven beat polling for one decisive reason: PENDING entries. A case sitting in the AI queue waiting to be claimed can be re-routed before the agent ever touches it. No wasted LLM call, no semaphore permit burned, no status guard abort. The poll-based approach would have added up to 10 seconds of stale routing — and during that window, the agent might claim the case, spend 20 seconds on an LLM call, then discover via the status guard that the entry was moved. The event observer short-circuits that entirely.

Re-routing is downward only. Cases can drop from AI resolution to operator queues, but never the reverse. The argument for upward re-routing (promoting a medium-confidence case to AI when similarity improves) doesn't survive contact with operator workflows — an operator who has started reading a case shouldn't have it pulled away mid-review. If the case base matures enough for more cases to qualify for AI resolution, new cases start with higher confidence. In-flight promotion adds complexity for a scenario that rarely materialises during a single case's lifetime.

The race condition between the observer and the agent is handled by the same mechanism that handles the timeout sweep race: the agent's status guard checks whether the entry is still in the AI view before executing device commands. Observer moves the entry, agent detects it's gone, aborts cleanly. No coordination, no shared state — both actors interact through queue state.

The implementation turned out to be the least interesting part. One class, nine tests, same patterns as the agent. The design reasoning took longer than the code.
