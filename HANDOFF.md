# Handoff — casehub-iot

**Head commit (project):** b8192b9 — feat(#81): queue listing REST endpoints for AI resolution views
**Date:** 2026-08-05

---

## Immediate Next Step

Pick up #85 (agent metrics & observability) — next on the critical path. Run `/work` to begin.

## Delivery Epics

### Epic A: MCP Production Readiness

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #74 | RBAC, tenancy filtering, principal propagation | M | Med | **Done** — landed as 2e2c916 | ~~1~~ |
| #77 | WebSocket/SSE streaming state changes | M | High | Parked — MCP protocol spec | — |

### Epic B: CBR Completion (#48 parent)
#49 (infra), #50 (suggestion), #60 (recency), #61 (surfacing) all closed.

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #82 | Re-routing on context changes | M | High | Ready (no engine dep) | 6 |

### Epic C: AI Resolution Agent v2

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #81 | Queue listing REST endpoints | M | Med | **Done** — landed as b8192b9 | ~~4~~ |
| #85 | Agent metrics & observability | M | Med | **Next** | 5 |
| #83 | Multi-turn LLM conversation | L | High | After #85 | 7 |
| #84 | Model fine-tuning & prompt versioning | M | Med | Parked — future | — |

### Epic D: Platform & Ops

| # | Description | Scale | Complexity | Status |
|---|-------------|-------|------------|--------|
| #67 | Household notifications via subscription engine | M | Med | Ready |
| #46 | Evaluate webapp extraction to app tier | L | High | Assessment |
| #42 | PostgreSQL table partitioning | M | Med | Deferred — production scale |

## Critical Path

```
#74 → #60 → #61 → #81 → #85 → #82 → #83
 ✅     ✅     ✅     ✅     C      B      C
```

**Parked:** #77 (MCP protocol), #42 (production scale), #84 (future)

## Cross-Repo

**Blocking** (we owe):
- engine#834 — Ganglion Uni→blocking migration · M · Med

## What Was Done (2026-08-05)

**Branch:** `issue-81-queue-listing-rest` — closed, landed as b8192b9.

- `GET /api/resolution/queue` — list entries across `iot-ai-resolution` and `iot-operator-assisted` views, enriched with device context
- `GET /api/resolution/queue/{entryId}` — detail with CBR suggestions and escalation context
- `QueueEntrySummary` and `QueueEntryDetail` response records in webapp-api
- Design review caught: viewName null after escalation, REVOKED exclusion, O(N) detail scan → findById
- Self-review caught: `AiResolutionPlan` is ephemeral (never stored in context) — removed from detail response
- Fixed pre-existing PlanTrace and CbrRetentionPolicy constructor mismatches
- Consumer guide updated with new endpoint documentation

## What's Left

- Pre-existing: `webapp-api` tests broken by neocortex SNAPSHOT `PlanTrace` constructor change · S · Low
