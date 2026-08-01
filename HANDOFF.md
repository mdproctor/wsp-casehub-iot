# Handoff — casehub-iot

**Head commit (project):** ca69181 — feat(#63): IoTAiResolutionAgent
**Date:** 2026-08-01

---

## Immediate Next Step

Start #74 (RBAC/tenancy for MCP) — foundational security, unblocked, on the critical path. Run `/work` to begin.

## Delivery Epics

### Epic A: MCP Production Readiness
Security and real-time capability for MCP tools.

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #74 | RBAC, tenancy filtering, principal propagation | M | Med | **Ready** | 1 |
| #77 | WebSocket/SSE streaming state changes | M | High | Parked — MCP protocol spec | — |

### Epic B: CBR Completion (#48 parent)
Complete the CBR feedback loop. #49 (infra) and #50 (suggestion) already closed.

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #60 | Temporal recency weighting | M | Med | Ready | 2 |
| #61 | Situation-level suggestion surfacing | M | Med | Ready | 3 |
| #82 | Re-routing on context changes | M | High | Ready (no engine dep) | 6 |

#60 ∥ #61 — independent, can run in parallel. #82 benefits from #60/#61 done first.

### Epic C: AI Resolution Agent v2
Mature the LLM agent shipped in #63.

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #81 | Queue listing REST endpoints | M | Med | Ready | 4 |
| #85 | Agent metrics & observability | M | Med | Ready | 5 |
| #83 | Multi-turn LLM conversation | L | High | After #85 | 7 |
| #84 | Model fine-tuning & prompt versioning | M | Med | Parked — future | — |

Ordering: #81 (ops visibility) → #85 (instrument) → #83 (big feature, measured).

### Epic D: Platform & Ops
Independent items — filler between blocked states.

| # | Description | Scale | Complexity | Status |
|---|-------------|-------|------------|--------|
| #67 | Household notifications via subscription engine | M | Med | Ready |
| #46 | Evaluate webapp extraction to app tier | L | High | Assessment |
| #42 | PostgreSQL table partitioning | M | Med | Deferred — production scale |

## Critical Path

```
#74 → #60 → #61 → #81 → #85 → #82 → #83
 A      B      B      C      C      B      C
```

**Parallel lanes** (work-slots or separate sessions):
- A (#74) ∥ B (#60) — completely independent
- B (#61) ∥ C (#81) — independent
- D (#67, #46) as filler when blocked

**Parked:** #77 (MCP protocol), #42 (production scale), #84 (future)

## Cross-Repo

**Blocking** (we owe):
- engine#834 — Ganglion Uni→blocking migration · M · Med (IoT ganglia update after it lands)

## What Was Done (2026-08-01)

- Filed engine#834 — Ganglion Uni→blocking migration. Clears trailing obligation.
- Confirmed life#60 closed — #74 unblocked.
- Confirmed PER_EVALUATION already implemented in engine — #82 has no engine dep.
- Organised 13 open issues into 4 delivery epics with critical path.
- Created GitHub labels: `epic-a:mcp-production`, `epic-b:cbr-completion`, `epic-c:ai-resolution-v2`, `epic-d:platform-ops`.
