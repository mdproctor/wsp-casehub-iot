# Agent Performance Metrics and Observability

**Date:** 2026-08-05
**Issue:** casehubio/iot#85
**Parent spec:** `specs/issue-63-llm-resolution-agent/2026-07-29-llm-resolution-agent-design.md` §9

## Summary

Instrumentation for `IoTAiResolutionAgent`: LLM call latency, success/escalation
rates, and resolution quality metrics correlated with CBR similarity bands. First
introduction of Micrometer and MicroProfile Health into the IoT webapp module.

Token consumption metrics require an engine change to `Agent` (see §2) — deferred
to a follow-up issue on the engine repo.

---

## §1 Dependencies

Two Quarkus extensions added to `webapp/pom.xml` (managed by Quarkus BOM, no
version needed):

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-health</artifactId>
</dependency>
```

`quarkus-micrometer-registry-prometheus` pulls in Micrometer core and auto-exposes
`/q/metrics` in Prometheus format. `quarkus-smallrye-health` enables `/q/health`
with MicroProfile Health.

First introduction of both extensions in this repo. Other modules can add them
independently later.

---

## §2 Metric Inventory

All metrics use the prefix `casehub.iot.ai.resolution`. Micrometer naming
convention (dots, lowercase).

### Timers

| Metric | What it measures | Tags |
|--------|-----------------|------|
| `poll.duration` | Synchronous poll dispatch and sweep duration (async entry processing excluded — see §3) | — |
| `llm.call.duration` | Single `llmAgent.execute()` call latency (excludes semaphore wait — see §3) | `outcome`: success, error |
| `entry.duration` | Entry processing time from successful claim to final outcome (excludes claim contention — see §3) | `outcome`: (see §3) |
| `action.execution.duration` | All actions in a plan executed sequentially | `outcome`: success, partial-failure |

### Counters

| Metric | What it counts | Tags |
|--------|---------------|------|
| `entries.processed` | Exit points in `processEntry()` where this agent completed or abandoned processing after claiming. Does not include claim contention (see `claim.contention`). | `outcome`: executed, llm-escalated, risk-gate, timeout, partial-failure, llm-error, case-not-found, status-guard-abort, error; `cbr.band`: high, medium, low, none, unknown |
| `claim.contention` | Claim attempts that lost the race (`IllegalStateException` from `queueService.claim()`) — normal concurrent behaviour, not a failure | — |
| `llm.retries` | Transient retry attempts (not the final attempt) | — |
| `actions.executed` | Individual device command executions | `succeeded`: true, false |

**Double-counting note:** When the timeout sweep escalates a stale entry and the
processing virtual thread subsequently hits the status guard, both `timeout` and
`status-guard-abort` increment for the same entry. Dashboard builders should not
naively sum all outcomes to compute "total entries processed" — `status-guard-abort`
is a secondary signal, not a distinct entry.

### Gauges

| Metric | What it reports | Pattern |
|--------|-----------------|---------|
| `semaphore.available` | Available LLM permits (of `maxConcurrentLlmCalls`) | Lambda gauge sampling `Semaphore::availablePermits` — registered once in `init()` |
| `queue.pending` | PENDING entries found in the last poll cycle | `AtomicInteger` field updated in `processNewEntries()`, lambda gauge reads the field — no DB query on Prometheus scrape |

### Token metrics — deferred (engine change required)

`Agent.execute()` calls `model.chat(request)` which returns a langchain4j
`ChatResponse` with `tokenUsage()` (input/output counts). But `execute()` only
extracts `response.aiMessage().text()`, parses it as JSON, and returns it as a
`Map<String, Object>`. The `ChatResponse` — and its token usage — is discarded.

Token usage is API-level metadata, not part of the response content. It will never
appear in the output map.

**Required engine change:** Enrich `WorkerResult` with token metadata, or add a
`ChatResponse` accessor to the `Agent` API. File a follow-up issue on
`casehubio/engine` with this specific requirement.

When the engine change lands, add:

| Metric | Type |
|--------|------|
| `llm.tokens.input` | Distribution summary |
| `llm.tokens.output` | Distribution summary |

### CBR similarity band derivation

Derived from the highest similarity score in `List<ResolutionSuggestion>`:
- `high`: max similarity >= 0.85
- `medium`: max similarity 0.6–0.85
- `low`: max similarity < 0.6
- `none`: suggestions list is empty AND `cbrConfig` was non-null (CBR queried, no matches)
- `unknown`: CBR context not available — covers: `cbrConfig == null` (case type has
  no CBR), timeout sweep, case-not-found, and any unhandled exception before CBR retrieval

The `cbr.band` tag is always present on `entries.processed` for tag consistency.

---

## §3 Instrumentation Points

### `poll()` (line 116)

Wrap the entire method body with `Timer.Sample` for `poll.duration`. This captures
synchronous work only: `findPending()` query + task submission + `sweepStaleEntries()`.
Actual entry processing happens asynchronously on virtual threads — `entry.duration`
is the complementary metric for that.

Update the `AtomicInteger` backing `queue.pending` from `pending.size()` inside
`processNewEntries()`.

### `processEntry()` (line 137)

**Try-finally wrapper:** The entire method body (after claim) is wrapped in a
try-finally that guarantees the `entry.duration` timer is stopped and
`entries.processed` is incremented even on unhandled exceptions. On unhandled
exception, the outcome is `error` and `cbr.band` is `unknown` (CBR context may
not have been retrieved yet).

```
claim entry (line 140)
  → IllegalStateException → increment claim.contention, return (no timer, no entries.processed)

Timer.Sample starts HERE (after successful claim)
cbrBand = "unknown"  // default until CBR retrieval completes
try {
    ... load case, retrieve CBR, call LLM, execute actions ...
    // cbrBand updated after suggestions are retrieved
finally {
    sample.stop(entryDurationTimer.tag("outcome", outcome))
    entries.processed.increment(outcome, cbrBand)
}
```

Exit points and outcome tags (all inside the try-finally):
- Case not in cache → `case-not-found`
- Case definition not found → `case-not-found`
- `callLlmWithRetry` returns null → `llm-error`
- LLM decides ESCALATE → `llm-escalated`
- Risk check fails → `risk-gate`
- Status guard fails → `status-guard-abort`
- `executeActions` completes → `executed` or `partial-failure`
- Any unhandled exception → `error`

### `callLlmWithRetry()` (line 193)

**Timer placement:** `Timer.Sample` is created **after** successful
`llmSemaphore.acquire()`, inside the inner try block. The timer measures only
`llmAgent.execute()` — not semaphore wait time. The sample is stopped in the
inner finally block (before `release()`) on success, or in catch blocks on error.

Each attempt records `llm.call.duration` independently:
- Successful execute → `outcome=success`
- Exception from execute → `outcome=error` (recorded before retry sleep)

`InterruptedException` from `acquire()` bypasses the timer entirely (no LLM call
happened — acceptable).

Retry attempts increment `llm.retries` counter.

### `executeActions()` (line 262)

Wrap with `action.execution.duration` timer. Each `deviceCommandFn.apply()`
increments `actions.executed` with `succeeded=true` or `succeeded=false`.

### `sweepStaleEntries()` (line 301)

Each escalation increments `entries.processed` with `outcome=timeout` and
`cbr.band=unknown` (CBR context is not available at sweep time).

### Semaphore gauge

Registered once in `init()` after semaphore creation:
```java
Gauge.builder("casehub.iot.ai.resolution.semaphore.available",
              llmSemaphore, Semaphore::availablePermits)
     .register(registry);
```

### Queue pending gauge

`AtomicInteger pendingCount` field, updated in `processNewEntries()`:
```java
// In init():
Gauge.builder("casehub.iot.ai.resolution.queue.pending",
              pendingCount, AtomicInteger::get)
     .register(registry);

// In processNewEntries():
List<CaseQueueEntry> pending = queueService.findPending(...);
pendingCount.set(pending.size());
```

---

## §4 Health Check

`IoTAiResolutionReadinessCheck` — `@Readiness @ApplicationScoped` bean using
MicroProfile Health.

Reports **UP** when all conditions are met:
1. `config.enabled()` is true
2. `aiResolutionViewId` is non-null (view resolved at startup)
3. `operatorAssistedViewId` is non-null

Reports **DOWN** with diagnostic data when any condition fails.

Data fields exposed:
- `enabled` — boolean
- `aiResolutionViewResolved` — boolean
- `operatorAssistedViewResolved` — boolean
- `semaphorePermits` — int (current available, or 0 if agent not yet initialized)

The health check injects `IoTAiResolutionAgent` and calls package-private
methods `isReady()` and `healthData()`. The agent owns its state; the health
check translates it to `HealthCheckResponse`.

**Null safety:** `healthData()` null-guards `llmSemaphore` (created in
`@PostConstruct`). If the health endpoint is hit before `init()` completes,
`semaphorePermits` reports 0 and `isReady()` returns false (view IDs are null).

---

## §5 Testing Strategy

### Unit tests (extend existing `IoTAiResolutionAgentTest`)

Inject a real `SimpleMeterRegistry` (Micrometer's in-memory test registry) via
the existing reflection pattern. Assert on actual metric values rather than mock
interactions.

New assertions on existing tests (no new test methods for most cases):

- `happyPath` → `entries.processed{outcome=executed}` = 1,
  `actions.executed{succeeded=true}` >= 1, `llm.call.duration` has 1 recording
- `llmDecidesToEscalate` → `entries.processed{outcome=llm-escalated}` = 1
- `riskGateTriggersEscalation` → `entries.processed{outcome=risk-gate}` = 1
- `timeoutSweep` → `entries.processed{outcome=timeout}` = 1
- `statusGuard_abortsWhenEntryMoved` → `entries.processed{outcome=status-guard-abort}` = 1
- `llmDeterministicFailure_escalatesImmediately` →
  `entries.processed{outcome=llm-error}` = 1, `llm.call.duration{outcome=error}` has 1 recording
- `llmTransientFailure_retriesThenEscalates` → `llm.retries` = 2,
  `entries.processed{outcome=llm-error}` = 1
- `partialWorkerFailure` → `entries.processed{outcome=partial-failure}` = 1,
  `actions.executed{succeeded=false}` = 1

### New tests

- `IoTAiResolutionReadinessCheckTest` — verify UP when agent is fully initialized,
  DOWN when disabled or views not resolved. Verify `semaphorePermits` returns 0
  before `init()`.
- `claimContention_incrementsSeparateCounter` — verify `claim.contention` is
  incremented (not `entries.processed`) when claim throws `IllegalStateException`.

### Not tested here

Prometheus scrape endpoint (`/q/metrics`) — Quarkus infrastructure.

---

## §6 Scope Boundaries

**In scope:**
- `quarkus-micrometer-registry-prometheus` and `quarkus-smallrye-health` dependencies
- Inline Micrometer instrumentation in `IoTAiResolutionAgent`
- `IoTAiResolutionReadinessCheck` health check
- CBR similarity band tagging on outcome counters
- Test coverage via `SimpleMeterRegistry`

**Out of scope (with tracking):**
- Token consumption metrics — requires engine change to `Agent` API. File follow-up
  on `casehubio/engine`.
- Grafana dashboards and alerting rules — metrics are invisible to operators without
  dashboards. File follow-up issue on this repo.

**Out of scope (no action needed):**
- Metrics for other modules (bridge, MCP, providers)
- Custom Prometheus recording rules
