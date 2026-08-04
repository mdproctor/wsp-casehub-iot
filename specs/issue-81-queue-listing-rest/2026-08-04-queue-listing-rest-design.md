# Queue Listing REST Endpoints — AI Resolution Views

**Date:** 2026-08-04
**Issue:** casehubio/iot#81
**Parent spec:** `specs/issue-63-llm-resolution-agent/2026-07-29-llm-resolution-agent-design.md` §9

## Summary

REST endpoints for displaying AI resolution queue entries with triage metadata.
Covers both `iot-ai-resolution` (pending/in-progress AI work) and
`iot-operator-assisted` (escalated to human) views. List + detail pattern:
the list endpoint returns enriched summaries; the detail endpoint loads full
CBR suggestions, AI resolution plan, and escalation context.

---

## §1 Endpoints

```
GET  /api/resolution/queue            → list entries across both views
GET  /api/resolution/queue/{entryId}  → full triage detail for one entry
```

### Query filters (list)

| Parameter | Type   | Default          | Values |
|-----------|--------|------------------|--------|
| `view`    | String | both             | `ai-resolution`, `operator-assisted` |
| `status`  | String | `PENDING,CLAIMED` | `PENDING`, `CLAIMED`, `REVOKED` |

Default excludes REVOKED entries (resolved cases). Explicit `?status=REVOKED`
can be passed to see them. When `?status=PENDING` is passed, the implementation
uses `CaseQueueService.findPending(viewId, tenancyId)` directly instead of
filtering in-memory.

### Security

`@RolesAllowed("iot-viewer")` on both endpoints. Tenancy filtering via
`CurrentPrincipal.tenancyId()` on all `CaseQueueService` calls.

---

## §2 Module Placement

| Module | Component | Purpose |
|--------|-----------|---------|
| `webapp-api` (Tier 1) | `QueueEntrySummary`, `QueueEntryDetail` | Pure data records. No CDI, no JPA |
| `webapp` | `ResolutionQueueResource` | REST resource with `@Path("/api/resolution/queue")` |

Response records go in `io.casehub.iot.webapp.resolution` alongside
the existing `AiResolutionPlan`, `AiEscalationContext`, `ExecutedActionResult`.

---

## §3 Response Records

### QueueEntrySummary (list)

```java
record QueueEntrySummary(
    UUID entryId,
    UUID caseId,
    String caseType,
    String viewName,
    QueueEntryStatus status,
    String assignedTo,
    Instant createdAt,
    Instant claimedAt,
    Instant escalatedAt,
    String previousViewName,
    String deviceId,
    String deviceClass,
    String roomType,
    String situationId
)
```

Enough for the list view to show: "Thermostat in Living Room — CLAIMED
by iot-ai-agent 30s ago" without loading CBR or AI state.

All case-derived fields (`caseType`, `deviceId`, `deviceClass`, `roomType`,
`situationId`) are extracted from the case's working context
(`CaseInstance.getCaseContext().getOrDefault("working", Map.of())`).
All are null when the case is no longer in the cache. Active cases are
architecturally guaranteed to be cached; null covers the narrow race
where a case completes between the queue query and the cache lookup.

`viewName` resolution: `CaseQueueService.escalate()` sets `viewName = null`
on the target entry. The resource resolves viewName from the cached view ID
→ name mapping (built from `SubjectViewStore.findByTenancy()` at startup).

### QueueEntryDetail (detail)

```java
record QueueEntryDetail(
    QueueEntrySummary entry,
    Map<String, Object> workingContext,
    List<ResolutionSuggestion> suggestions,
    AiEscalationContext escalationContext,
    List<ExecutedActionResult> executionResults
)
```

`AiResolutionPlan` is not stored in the case context — it's ephemeral within
`IoTAiResolutionAgent.processEntry()`. For escalated cases,
`AiEscalationContext.partialPlan` captures the planned actions. For successful
cases, `executionResults` (from `aiResolutionResults` context key) shows what
was executed. No separate plan field needed.

Reuses existing records from `webapp-api`: `ResolutionSuggestion`,
`AiEscalationContext`, `ExecutedActionResult`. No new data types needed.

---

## §4 Implementation — ResolutionQueueResource

### View resolution

Same pattern as `IoTAiResolutionAgent.init()`: resolve view IDs from
`SubjectViewStore.findByTenancy()` at `@PostConstruct`. Cache
`aiResolutionViewId`, `operatorAssistedViewId`, and a `Map<UUID, String>`
of viewId → viewName for resolving null viewName on escalated entries.

### List endpoint flow

1. Determine which view(s) to query based on `?view` filter (default: both)
2. Per view:
   - If `?status=PENDING`: use `CaseQueueService.findPending(viewId, tenancyId)`
   - Otherwise: use `CaseQueueService.findByView(viewId, tenancyId)`
3. Filter: exclude REVOKED unless explicitly requested; apply status filter
4. For each entry: `CaseInstanceCache.get(caseId)` → snapshot working context
   once; extract device/situation identity from that snapshot
5. Resolve viewName from cached viewId → name mapping when entry's viewName is null
6. Return `List<QueueEntrySummary>`

### Detail endpoint flow

1. `CaseQueueEntryStore.findById(entryId)` → verify
   `entry.getTenancyId().equals(principal.tenancyId())`; 404 if not found
   or tenancy mismatch
2. `CaseInstanceCache.get(caseId)` → snapshot working context once
3. CBR suggestions: `CaseDefinitionRegistry.findByName(caseType)` →
   `getCbrConfig()` → `extractFeatures(workingContext)` →
   `IoTCbrRetrievalService.retrieve(cbrConfig, features, tenancyId)`.
   If cbrConfig is null → empty suggestions list
4. Read `aiResolutionResults` (as `List<ExecutedActionResult>`) and
   `aiEscalationContext` (as `AiEscalationContext`) from case context
5. Resolve viewName from cached mapping
6. Return `QueueEntryDetail`

### Dependencies (all already available in webapp)

```java
@Inject CaseQueueService queueService;
@Inject CaseInstanceCache caseCache;
@Inject CaseDefinitionRegistry definitionRegistry;
@Inject SubjectViewStore viewStore;
@Inject IoTCbrRetrievalService retrievalService;
@Inject CurrentPrincipal principal;
```

The detail endpoint also needs `CaseQueueEntryStore` (the SPI behind
`CaseQueueService`) for `findById()`. This is already a CDI bean in
the webapp runtime.

No other engine modifications required.

---

## §5 Edge Cases

| Scenario | Behaviour |
|----------|-----------|
| View not configured at startup | Endpoint returns empty list (view ID is null) |
| Case not in cache | All case-derived fields null (caseType, device*, situationId) |
| Entry not found (detail) | 404 |
| Tenancy mismatch (detail) | 404 (same as not found — no information leak) |
| No AI state yet (PENDING entry) | `resolutionPlan`, `escalationContext`, `executionResults` are null |
| Escalated entry | `previousViewName` populated, `escalationContext` present |
| viewName null on entry | Resolved from cached viewId → name mapping |
| REVOKED entries | Excluded by default; visible with explicit `?status=REVOKED` |

### Performance — cache lookup per entry

`CaseInstanceCache` is an in-memory map. Each `.get(caseId)` is ~100ns.
The queue is not bounded by the AI agent's semaphore (which only limits
concurrent LLM calls) — under LLM outage or agent-disabled conditions,
queue depth grows with case arrival rate. But even at hundreds of entries,
total cache lookup time is sub-millisecond. JSON serialization dominates.

### Concurrency — reads during AI agent processing

The REST endpoint reads `CaseInstance` while `IoTAiResolutionAgent` may
concurrently write context keys (`aiResolutionResults`, `aiEscalationContext`).
`CaseContext` uses `ConcurrentHashMap` — individual reads are atomic. The
endpoint snapshots the working context once per entry and uses that snapshot
for both display fields and CBR feature extraction. Stale reads between
requests are acceptable for a dashboard view.

---

## §6 Testing Strategy

### webapp (integration, CDI) — ResolutionQueueResourceTest

| # | Test | Verifies |
|---|------|----------|
| 1 | List — both views combined | Entries from both views returned with correct viewName |
| 2 | List — view filter | `?view=ai-resolution` returns only AI queue entries |
| 3 | List — status filter | `?status=PENDING` excludes CLAIMED; default excludes REVOKED |
| 4 | List — enrichment | Each entry carries deviceId, deviceClass, roomType, situationId |
| 5 | List — case not in cache | Entry returned with null case-derived fields |
| 6 | List — view not configured | Empty list, no error |
| 7 | List — viewName null (escalated) | viewName resolved from cached mapping |
| 8 | Detail — happy path | Full enrichment: CBR suggestions, AI plan, escalation context, results |
| 9 | Detail — entry not found | 404 |
| 10 | Detail — tenancy mismatch | 404 (no information leak) |
| 11 | Detail — no AI state | PENDING entry with null resolution/escalation fields |
| 12 | Detail — no CBR config | Empty suggestions list |
| 13 | Detail — escalated entry | escalationContext populated, previousViewName set |
| 14 | Security | Unauthenticated → 401; wrong role → 403 |

### Not tested here

CBR retrieval (covered by `IoTCbrRetrievalService` tests), risk
classification (covered by `IoTActionRiskClassifierTest`), queue
operations (covered by engine's `CaseQueueServiceTest`).

---

## §7 Scope Boundaries

**In scope:**
- `ResolutionQueueResource` with list and detail endpoints
- `QueueEntrySummary` and `QueueEntryDetail` response records
- View resolution (including null viewName on escalated entries)
- Tenancy filtering with explicit verification on detail endpoint
- REVOKED status handling (excluded by default)
- Tests for all paths

**Out of scope:**
- Queue entry mutation (claim/release/escalate) — already on `CaseQueueService`
- SSE/WebSocket streaming of queue changes (#77)
- Agent metrics/observability (#85)
- TypeScript page consuming these endpoints (separate issue)
- Pagination (defer until production scale warrants it)
- Shared view resolution helper (refactor — both this resource and
  `IoTAiResolutionAgent` resolve views the same way)

---

## §8 Review Resolutions

Design review (light, 2026-08-04) — issues resolved:

| # | Issue | Resolution |
|---|-------|------------|
| 1 | REVOKED status ignored | Default filter excludes REVOKED; explicit `?status=REVOKED` available |
| 2 | viewName null after escalation | Resolve from cached viewId → name mapping |
| 3 | Detail O(N) scan | Use `CaseQueueEntryStore.findById()` + tenancy verification |
| 4 | No pagination | Deferred — queue depth is bounded by active case volume |
| 5 | CBR retrieval underspecified | Full chain specified: definition → cbrConfig → features → retrieve |
| 6 | caseType also nullable | All case-derived fields documented as nullable |
| 7 | Concurrent reads | Snapshot once per entry; ConcurrentHashMap guarantees atomic reads |
| 8 | Response record placement | Keep in `resolution` package — domain data, not generic REST DTOs |
| 9 | Duplicated view resolution | Deferred — refactor, not required for this issue |
| 10 | Use `findPending()` directly | Accepted — use when `?status=PENDING` |
