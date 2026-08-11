---
layout: post
title: "The Metrics That Weren't Quite Right"
date: 2026-08-06
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub IoT]
tags: [micrometer, observability, metrics, ai-resolution]
---

The AI resolution agent (#63) shipped without instrumentation. That was deliberate — scope boundaries deferred metrics to a separate issue. But a resolution agent with no visibility into its own behaviour is a liability. You can't see LLM latency, you can't see escalation rates, you can't see whether CBR similarity actually correlates with successful resolution. The agent works, but you're flying blind.

This was the first Micrometer introduction in the entire IoT repo. No existing patterns to follow, no prior art in the codebase. I brought Claude in for the design, and the design review is where the interesting things happened.

The initial spec defined `entries.processed` as a counter for "entries reaching a terminal state" — nine outcome tags, clean taxonomy. Claude's review caught that three of those outcomes aren't terminal states at all. A claim contention skip means another virtual thread won the race — the entry is alive and being processed elsewhere. Counting it alongside genuine outcomes corrupts the throughput metric. We split it out as a separate `claim.contention` counter.

The subtler catch was the double-counting race. The timeout sweep and the processing virtual thread can both act on the same entry when a timeout fires between the LLM call completing and action execution beginning. The sweep records `outcome=timeout`, the status guard records `outcome=status-guard-abort` — same entry, two counter increments. There's no clean fix without adding coordination overhead that's worse than the problem. We documented it explicitly: dashboard builders can't naively sum outcomes.

Timer boundaries needed precision too. The LLM call timer had to start *after* semaphore acquisition, not before — otherwise you're measuring contention wait time, not provider latency. The entry processing timer had to start after successful claim, not at method entry — sub-millisecond claim failures would pollute the distribution.

The token consumption question turned out to be a dead end for now. The engine's `Agent.execute()` calls langchain4j's `ChatModel`, gets back a `ChatResponse` with full token usage data, then discards it — only the parsed JSON response survives into the `WorkerResult`. There's no way to get token counts without an engine API change. I filed engine#872 for that.

The implementation itself was straightforward once the spec was right. `MeterRegistry` injected into the agent, `Timer.Sample` instances at the natural instrumentation points, `AtomicInteger` backing the queue pending gauge so Prometheus scrapes don't hit the database. A `@Readiness` health check reports whether the agent is enabled, both queue views resolved, and the semaphore is alive — null-guarded for the window between health endpoint availability and `@PostConstruct` completion.

Every existing test got metric assertions via `SimpleMeterRegistry` — Micrometer's in-memory test registry that records real values instead of requiring mock interaction verification. The test for each outcome path now confirms the right counter was incremented with the right tags.

Two follow-up issues filed: engine#872 for token usage exposure, and iot#89 for Grafana dashboards. The metrics exist but are invisible to operators without dashboards — measuring things nobody can see is only marginally better than not measuring at all.
