# Handoff — casehub-iot

**Head commit (project):** 2e2c916 — feat(#74): RBAC, tenancy filtering, and principal propagation for MCP tools
**Date:** 2026-08-04

---

## Immediate Next Step

Pick up #81 (queue listing REST endpoints) — next on the critical path. Run `/work` to begin.

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
| #81 | Queue listing REST endpoints | M | Med | **Next** | 4 |
| #85 | Agent metrics & observability | M | Med | Ready | 5 |
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
 ✅     ✅     ✅     C      C      B      C
```

**Parked:** #77 (MCP protocol), #42 (production scale), #84 (future)

## Cross-Repo

**Blocking** (we owe):
- engine#834 — Ganglion Uni→blocking migration · M · Med

## What Was Done (2026-08-04)

**Branch:** `issue-60-cbr-recency-surfacing` — closed (issues only, no code changes).
**Covers:** #60, #61.

- Both features were already implemented on main under earlier branches (commits `357928e` for #60 and `2b57e1b` for #61, referencing original spec issue numbers #64, #65)
- Closed #60 and #61 to formalise issue tracking
- Recovered 4 orphaned specs from closed branches to workspace main

## What's Left

- Pre-existing: `webapp-api` tests broken by neocortex SNAPSHOT `PlanTrace` constructor change · S · Low
