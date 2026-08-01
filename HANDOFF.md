# Handoff — casehub-iot

**Head commit (project):** ca69181 — feat(#63): IoTAiResolutionAgent — LLM resolution agent with polling, claim, execution, escalation, and timeout sweep

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up #74 (RBAC, tenancy filtering for MCP) — now unblocked since life#60 closed. Run `/work` to start.

## What Was Done (2026-08-01)

- Filed engine#834 — Ganglion Uni→blocking migration (detect/compact/close). Clears trailing obligation.
- Confirmed life#60 closed — #74 unblocked.
- Confirmed engine already supports `PER_EVALUATION` CBR timing — #82 has no engine dependency.

## Cross-Repo

**Blocking** (we owe):
- engine#834 — Ganglion Uni→blocking migration (gates IoT ganglia updates) · M · Med

## What's Left

- IoT ganglia must update after engine#834 lands · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #74 | RBAC, tenancy filtering, principal propagation for MCP | M | Med | **Next up** — unblocked (life#60 closed) |
| #81 | Queue listing REST endpoints | M | Med | Display queue entries with triage metadata |
| #82 | Re-routing on context changes / CBR re-evaluation | M | High | No engine dep — PER_EVALUATION already implemented |
| #77 | WebSocket/SSE streaming state changes via MCP | M | High | Future protocol |
| #46 | Evaluate webapp extraction to app tier | L | High | Independent |

## Key References

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*
