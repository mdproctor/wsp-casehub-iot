---
layout: post
title: "Multi-Turn Conversations and the CI Archaeology That Got Us There"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [ai-resolution, multi-turn, agent-session, mcp, ci-fix, cbr, design-review]
series: issue-83-multi-turn-llm-conversation
---

The session started with a CI fix and ended with a working multi-turn LLM conversation agent. The path between them was more interesting than either destination.

## The invisible reactor gap

IoT CI had been failing with `cannot find symbol` on `CaseQueueService.findByView()` — a method that clearly existed in the engine source. Local builds passed every time. The problem turned out to be an archaeology exercise: `casehub-engine-queue` existed as a directory with a valid pom, but was never added to the engine's `<modules>` list. It had been deployed to GitHub Packages once, manually, back in July. Every CI build since then silently skipped it. Local builds worked because `mvn install` puts current code into `~/.m2` regardless of reactor membership. The disconnect was invisible until someone added a new method and a downstream consumer tried to call it.

The fix was one line in the parent pom. Getting there required fixing checkstyle violations across four modules, a `CompoundTask` constructor mismatch that predated my change, and a `PreferenceProvider` injection gap in a test that hadn't been run in CI for weeks. Three commits to engine, each surfacing another layer of drift.

## Closing the CBR arc

With the CI fix green, I closed epic #48 — Case-Based Reasoning for IoT. All five children shipped: infrastructure (#49), resolution suggestion (#50), work item prediction (#51), false-positive suppression (#52), and context-change re-routing (#82). I also closed #89 (Grafana dashboards) — the user pointed out that casehub-pages should own metrics visualization as a reusable capability, not a per-app Grafana stack. Filed casehub-pages#293 instead.

## The design pivot on #83

I came into multi-turn LLM conversation (#83) assuming we'd build on langchain4j's `AiServices` for tool-use conversations. The user stopped me: "scan blocks and engine — we've already built chat turn based stuff." They were right. The platform already had `AgentProvider.openSession()` returning an `AgentSession` with multi-turn `query()`, streaming `AgentEvent`s, and MCP server attachment. The blocks library had supervisor, loop, and debate patterns using `AgentProvider`. The infrastructure I was about to design from scratch already existed.

This changed the design from "build a conversation layer" to "wire the existing one." The IoT MCP tools (`iot_get_devices`, `iot_get_state`, `iot_get_history`) give the LLM direct device access during its reasoning. The conversation loop is a simple for-loop with `MultiTurnResolutionState` tracking turns and termination — the `AgentSession` handles conversation memory and tool dispatch internally.

## What the design review caught

Three independent reviewers all flagged the same issue: auto mode's original design started with single-shot `Agent.execute()` and escalated to a session on `CONTINUE`. But the session has no memory of turn 1 — the context is lost in the handoff. The fix: auto mode always opens a session. Simple cases resolve on turn 1 and close immediately; the session overhead is negligible compared to the LLM call cost. Context continuity matters more than saving one `openSession()` call.

The reviewers also caught that the blocks `LoopBuilder` API doesn't support the imperative `step/terminateWhen` pattern I'd pseudocoded — it's an orchestration framework with routing and decomposition, not a while-loop wrapper. The actual API uses `exitCondition(Predicate)` and `maxIterations(int)`. I'd been designing against an API that didn't exist.

## What the agent can do now

In `auto` mode, the agent opens a session, sends the case context and CBR suggestions as the first turn, and waits. The LLM can call MCP tools to query device state, read sensor history, check what devices are in the room. It responds with `CONTINUE` (need more information), `RESOLVED` (here's my plan), or `ESCALATE` (too complex for automation). Each turn is collected from the `Multi<AgentEvent>` stream — text deltas assembled into a response, tool calls matched to their results, token counts tallied from `InvocationComplete`. The whole conversation persists as a `ConversationTranscript` on the case, so operators see the full reasoning chain when a case escalates.

The safety invariant holds: the LLM can read device state freely during the conversation, but actual device commands execute after the conversation concludes, through the existing risk classification pipeline. The MCP server exposed to the session is read-only.

## Looking forward

The conversation infrastructure is IoT-specific today — `AgentEventCollector`, `MultiTurnResolutionState`, the session wiring — but the abstractions are portable. When another CaseHub app needs multi-turn LLM resolution, these promote to `engine-api` or `platform`. The `AgentProvider` + `AgentSession` layer is already platform-level; what's missing is a reusable conversation orchestration layer between "open a session" and "domain-specific resolution logic." That's the extraction target for a future issue.

#83 is the last major piece in Epic C (AI Resolution Agent v2). What started as a single-shot LLM call bolted onto the queue system is now a conversational agent with information gathering, iterative refinement, full observability, and a safety model that keeps humans in the loop for anything the risk classifier flags.
