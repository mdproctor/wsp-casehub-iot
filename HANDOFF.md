# Handoff — casehub-iot

**Head commit (project):** 15104ac — feat(#85): agent metrics, observability, and health check
**Date:** 2026-08-06

---

## Immediate Next Step

Pick up #82 (re-routing on context changes / CBR re-evaluation) — next on the critical path. Run `/work` to begin.

## Delivery Epics

### Epic A: MCP Production Readiness

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #74 | RBAC, tenancy filtering, principal propagation | M | Med | **Done** — landed as 2e2c916 | ~~1~~ |
| #77 | WebSocket/SSE streaming state changes | M | High | Parked — MCP protocol spec | — |

### Epic B: CBR Completion (#48 parent)

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #82 | Re-routing on context changes | M | High | **Next** | 6 |

### Epic C: AI Resolution Agent v2

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #81 | Queue listing REST endpoints | M | Med | **Done** — landed as b8192b9 | ~~4~~ |
| #85 | Agent metrics & observability | M | Med | **Done** — landed as 15104ac | ~~5~~ |
| #83 | Multi-turn LLM conversation | L | High | After #82 | 7 |
| #84 | Model fine-tuning & prompt versioning | M | Med | Parked — future | — |
| #89 | Grafana dashboards for agent metrics | M | Med | Ready — filed as follow-up to #85 | — |

### Epic D: Platform & Ops

| # | Description | Scale | Complexity | Status |
|---|-------------|-------|------------|--------|
| #67 | Household notifications via subscription engine | M | Med | Ready |
| #46 | Evaluate webapp extraction to app tier | L | High | Assessment |
| #42 | PostgreSQL table partitioning | M | Med | Deferred — production scale |

## Critical Path

```
#74 → #60 → #61 → #81 → #85 → #82 → #83
 ✅     ✅     ✅     ✅     ✅     C      C
```

**Parked:** #77 (MCP protocol), #42 (production scale), #84 (future)

## Cross-Repo

**Blocking** (we owe):
- engine#834 — Ganglion Uni→blocking migration · M · Med

**Filed** (engine owes us):
- engine#872 — Token usage exposure from Agent.execute() · S · Low — needed for #85 token metrics

## What Was Done (2026-08-06)

**Branch:** `issue-85-agent-metrics-observability` — closed, landed as 15104ac.

- First Micrometer + MicroProfile Health introduction in casehub-iot
- Timers: poll dispatch, LLM call (per attempt, after semaphore), entry processing (after claim), action execution
- Counters: entries.processed (by outcome + CBR band), claim.contention, llm.retries, actions.executed
- Gauges: semaphore.available, queue.pending (AtomicInteger-backed)
- @Readiness health check with null-guard for pre-init
- Design review caught: counter semantics (claim-failed split), timer boundaries (semaphore/claim), double-counting race, token API dead end
- Fixed pre-existing PlanTrace constructor mismatch
- Filed engine#872 (token metrics) and iot#89 (Grafana dashboards)
- Consumer guide updated, diary entry written
