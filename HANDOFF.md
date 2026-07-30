# Handoff — casehub-iot

**Head commit (project):** ca69181 — feat(#63): IoTAiResolutionAgent — LLM resolution agent with polling, claim, execution, escalation, and timeout sweep

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

1. File ras issue for full Ganglion Uni→blocking migration (detect/compact/close) — trailing from last session
2. #74 (RBAC, tenancy filtering for MCP) — unblocked (life#60 closed)
3. #46 (evaluate webapp extraction to app tier) — independent

## What Was Done (2026-07-30)

**Branch:** `issue-63-llm-resolution-agent` — closed, landed as `ca69181`.
**Covers:** #63.

- **#63 IoTAiResolutionAgent** — LLM agent that claims cases from `iot-ai-resolution`
  queue view, loads CBR suggestions, calls Claude via engine's langchain4j `Agent`
  infrastructure, risk-classifies each planned action via `IoTActionRiskClassifier`,
  executes autonomous actions via `DeviceCommandWorkerFunction`, or escalates to
  `iot-operator-assisted` with full `AiEscalationContext`.
- Features: semaphore-bounded concurrent LLM calls, transient vs deterministic
  retry classification, status guard against timeout sweep races, partial worker
  failure tracking with stop-on-first-failure escalation.
- Adversarial design review: 9 rounds, 18 issues, 13 verified fixes.

## Cross-Repo Changes

- **engine:** `findByView()` added to `CaseQueueService` (on `issue-810` branch).
  Stale Uni→blocking references fixed in queue module's `CaseLabelEvaluator` and tests.
  `casehub-platform-view` + `casehub-platform-view-inmem` added to engine parent BOM.
- **life:** #60 closed — RBAC foundation shipped. Unblocks iot#74.

## What's Left

- File ras issue for full Ganglion Uni→blocking migration · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #74 | RBAC, tenancy filtering, principal propagation for MCP | M | Med | Unblocked — life#60 closed |
| #77 | WebSocket/SSE streaming state changes via MCP | M | High | Future protocol |
| #46 | Evaluate webapp extraction to app tier | L | High | Independent |
| #81 | Queue listing REST endpoints | M | Med | Display queue entries with triage metadata |
| #82 | Re-routing on context changes / CBR re-evaluation | M | High | Engine PER_EVALUATION support |

## Key References

- Spec: `specs/issue-63-llm-resolution-agent/2026-07-29-llm-resolution-agent-design.md`
- Review: `~/adr/casehub-iot/llm-resolution-agent-20260729-235357/tracker.md`
