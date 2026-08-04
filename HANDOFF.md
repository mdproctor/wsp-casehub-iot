# Handoff — casehub-iot

**Head commit (project):** 2e2c916 — feat(#74): RBAC, tenancy filtering, and principal propagation for MCP tools
**Date:** 2026-08-04

---

## Immediate Next Step

Pick up #60 (CBR temporal recency weighting) — next on the critical path. Run `/work` to begin.

## Delivery Epics

### Epic A: MCP Production Readiness

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #74 | RBAC, tenancy filtering, principal propagation | M | Med | **Done** — landed as 2e2c916 | ~~1~~ |
| #77 | WebSocket/SSE streaming state changes | M | High | Parked — MCP protocol spec | — |

### Epic B: CBR Completion (#48 parent)
#49 (infra) and #50 (suggestion) already closed.

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #60 | Temporal recency weighting | M | Med | **Next** | 2 |
| #61 | Situation-level suggestion surfacing | M | Med | Ready | 3 |
| #82 | Re-routing on context changes | M | High | Ready (no engine dep) | 6 |

#60 ∥ #61 — independent, can run in parallel.

### Epic C: AI Resolution Agent v2

| # | Description | Scale | Complexity | Status | Seq |
|---|-------------|-------|------------|--------|-----|
| #81 | Queue listing REST endpoints | M | Med | Ready | 4 |
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
 ✅     B      B      C      C      B      C
```

**Parked:** #77 (MCP protocol), #42 (production scale), #84 (future)

## Cross-Repo

**Blocking** (we owe):
- engine#834 — Ganglion Uni→blocking migration · M · Med

## What Was Done (2026-08-04)

**Branch:** `issue-74-rbac-tenancy-mcp` — closed, landed as `2e2c916`.
**Covers:** #74.

- **IoTRoles** constants class in `casehub-iot-api` (VIEWER, OPERATOR, ADMIN)
- **McpIdentityContext** — `Instance<CurrentPrincipal>` with three-guard scope check, graceful fallback to config tenancy + `"mcp-agent"` for unsecured hosts (bridge)
- **@RolesAllowed** on all 4 MCP tools (VIEWER for reads, OPERATOR for commands)
- **Tenancy filtering** — `findByTenancyId()` for device list, `findById(deviceId, tenancyId)` default method for single-device ops, `findHistory(deviceId, tenancyId, ...)` for data-layer history isolation
- **actorId propagation** — `dispatchedBy` uses authenticated caller identity
- **IoTCommandAuditEvent tenancyId** — audit trail includes tenant for post-deprovisioning correlation
- **DeviceResource fixes** — dispatchedBy bug (was tenancyId, now actorId), dead null check in filterByTenancy removed
- **Webapp migration** — 10x `@RolesAllowed` string literals → `IoTRoles` constants
- **Design review** — 4-dimension adversarial ($37, 41 issues across coherence/structure/robustness/cross-cutting)
- **New issues filed:** #86 (cross-tenant admin MCP), #87 (deprovisioned device history regression), #88 (webapp integration tests)
- Consumer and contributor guides synced with RBAC/tenancy changes

## What's Left

- Pre-existing: `webapp-api` tests broken by neocortex SNAPSHOT `PlanTrace` constructor change · S · Low
