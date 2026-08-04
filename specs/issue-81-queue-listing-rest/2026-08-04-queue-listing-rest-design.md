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

| Parameter | Type   | Default | Values |
|-----------|--------|---------|--------|
| `view`    | String | both    | `ai-resolution`, `operator-assisted` |
| `status`  | String | all     | `PENDING`, `CLAIMED` |

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

Device/situation fields are extracted from the case's working context
(`CaseInstance.getCaseContext().getOrDefault("working", Map.of())`).
Null when the case is no longer in the cache (edge case — active cases
are architecturally guaranteed to be cached).

### QueueEntryDetail (detail)

```java
record QueueEntryDetail(
    QueueEntrySummary entry,
    Map<String, Object> workingContext,
    List<ResolutionSuggestion> suggestions,
    AiResolutionPlan resolutionPlan,
    AiEscalationContext escalationContext,
    List<ExecutedActionResult> executionResults
)
```

Reuses existing records from `webapp-api`: `ResolutionSuggestion`,
`AiResolutionPlan`, `AiEscalationContext`, `ExecutedActionResult`.
No new data types needed.

---

## §4 Implementation — ResolutionQueueResource

### View resolution

Same pattern as `IoTAiResolutionAgent.init()`: resolve view IDs from
`SubjectViewStore.findByTenancy()` at `@PostConstruct`. Cache
`aiResolutionViewId` and `operatorAssistedViewId`.

### List endpoint flow

1. Determine which view(s) to query based on `?view` filter (default: both)
2. `CaseQueueService.findByView(viewId, tenancyId)` per view
3. Optional status filter applied in-memory
4. For each entry: `CaseInstanceCache.get(caseId)` → extract device/situation
   identity from working context
5. Return `List<QueueEntrySummary>`

### Detail endpoint flow

1. Query both views via `CaseQueueService.findByView()`, find entry by ID
2. `CaseInstanceCache.get(caseId)` → full working context
3. Load CBR suggestions via `IoTCbrRetrievalService.retrieve()`
4. Read `aiResolutionResults` and `aiEscalationContext` from case context
5. Return `QueueEntryDetail`

### Dependencies (all already available in webapp)

```java
@Inject CaseQueueService queueService;
@Inject CaseInstanceCache caseCache;
@Inject CaseDefinitionRegistry definitionRegistry;
@Inject SubjectViewStore viewStore;
@Inject IoTCbrRetrievalService retrievalService;
@Inject CurrentPrincipal principal;
```

No new engine modifications required. All services used by the existing
`IoTAiResolutionAgent` and `CaseResource`.

---

## §5 Edge Cases

| Scenario | Behaviour |
|----------|-----------|
| View not configured at startup | Endpoint returns empty list (view ID is null) |
| Case not in cache | Entry in list with null device/situation fields |
| Entry not found (detail) | 404 |
| No AI state yet (PENDING entry) | `resolutionPlan`, `escalationContext`, `executionResults` are null |
| Escalated entry | `previousViewName` populated, `escalationContext` present |

### Performance — cache lookup per entry

`CaseInstanceCache` is an in-memory map. Each `.get(caseId)` is ~100ns.
The queue is not bounded by the AI agent's semaphore (which only limits
concurrent LLM calls) — under LLM outage or agent-disabled conditions,
queue depth grows with case arrival rate. But even at hundreds of entries,
total cache lookup time is sub-millisecond. JSON serialization dominates.

Data availability is architecturally guaranteed: cases in the queue are
active (unresolved), and active cases remain in the cache. The null
fallback covers the narrow race where a case completes between the queue
query and the cache lookup.

---

## §6 Testing Strategy

### webapp (integration, CDI) — ResolutionQueueResourceTest

| # | Test | Verifies |
|---|------|----------|
| 1 | List — both views combined | Entries from both views returned with correct viewName |
| 2 | List — view filter | `?view=ai-resolution` returns only AI queue entries |
| 3 | List — status filter | `?status=PENDING` excludes CLAIMED entries |
| 4 | List — enrichment | Each entry carries deviceId, deviceClass, roomType, situationId |
| 5 | List — case not in cache | Entry returned with null device/situation fields |
| 6 | List — view not configured | Empty list, no error |
| 7 | Detail — happy path | Full enrichment: CBR suggestions, AI plan, escalation context, results |
| 8 | Detail — entry not found | 404 |
| 9 | Detail — no AI state | PENDING entry with null resolution/escalation fields |
| 10 | Detail — escalated entry | escalationContext populated, previousViewName set |
| 11 | Security | Unauthenticated → 401; wrong role → 403 |

### Not tested here

CBR retrieval (covered by `IoTCbrRetrievalService` tests), risk
classification (covered by `IoTActionRiskClassifierTest`), queue
operations (covered by engine's `CaseQueueServiceTest`).

---

## §7 Scope Boundaries

**In scope:**
- `ResolutionQueueResource` with list and detail endpoints
- `QueueEntrySummary` and `QueueEntryDetail` response records
- View resolution, tenancy filtering, query parameters
- Tests for all paths

**Out of scope:**
- Queue entry mutation (claim/release/escalate) — already on `CaseQueueService`
- SSE/WebSocket streaming of queue changes (#77)
- Agent metrics/observability (#85)
- TypeScript page consuming these endpoints (separate issue)
