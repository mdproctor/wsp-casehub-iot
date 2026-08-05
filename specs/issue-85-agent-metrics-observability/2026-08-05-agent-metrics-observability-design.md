# Agent Performance Metrics and Observability

**Date:** 2026-08-05
**Issue:** casehubio/iot#85
**Parent spec:** `specs/issue-63-llm-resolution-agent/2026-07-29-llm-resolution-agent-design.md` §9

## Summary

Instrumentation for `IoTAiResolutionAgent`: LLM call latency, success/escalation
rates, token consumption (if the engine Agent API exposes it), and resolution
quality metrics correlated with CBR similarity bands. First introduction of
Micrometer and MicroProfile Health into the IoT webapp module.

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
| `poll.duration` | End-to-end `poll()` cycle (new entries + sweep) | — |
| `llm.call.duration` | Single LLM call latency (per attempt, not including retries) | `outcome`: success, error |
| `entry.duration` | Full entry processing time (claim → final outcome) | `outcome`: (see §3) |
| `action.execution.duration` | All actions in a plan executed sequentially | `outcome`: success, partial-failure |

### Counters

| Metric | What it counts | Tags |
|--------|---------------|------|
| `entries.processed` | Entries reaching a terminal state | `outcome`: executed, llm-escalated, risk-gate, timeout, partial-failure, llm-error, claim-failed, case-not-found, status-guard-abort; `cbr.band`: high, medium, low, none |
| `llm.retries` | Transient retry attempts (not the final attempt) | — |
| `actions.executed` | Individual device command executions | `succeeded`: true, false |

### Gauges

| Metric | What it reports |
|--------|----------------|
| `semaphore.available` | Available LLM permits (of `maxConcurrentLlmCalls`) |
| `queue.pending` | PENDING entries found in the last poll cycle |

### Token metrics (conditional)

If `Agent.execute()` or `WorkerResult` exposes token usage (input/output counts):

| Metric | Type |
|--------|------|
| `llm.tokens.input` | Distribution summary |
| `llm.tokens.output` | Distribution summary |

If the API doesn't surface token data, omit these and file a follow-up issue on
the engine repo.

### CBR similarity band derivation

Derived from the highest similarity score in `List<ResolutionSuggestion>`:
- `high`: max similarity >= 0.85
- `medium`: max similarity 0.6–0.85
- `low`: max similarity < 0.6
- `none`: empty suggestions list
- `unknown`: CBR context not available (timeout sweep, claim-failed, case-not-found)

The `cbr.band` tag is always present on `entries.processed` for tag consistency.
Early exits before CBR retrieval use `unknown`.

---

## §3 Instrumentation Points

### `poll()` (line 116)

Wrap the entire method body with `Timer.Sample`. Record `queue.pending` gauge
from `pending.size()` inside `processNewEntries()`.

### `processEntry()` (line 137)

Start `Timer.Sample` at entry. Stop at every exit point with the appropriate
outcome tag. Increment `entries.processed` counter with both `outcome` and
`cbr.band` tags. CBR band computed after suggestions are retrieved.

Exit points and outcome tags:
- Claim `IllegalStateException` → `claim-failed`
- Case not in cache → `case-not-found`
- Case definition not found → `case-not-found`
- `callLlmWithRetry` returns null → `llm-error`
- LLM decides ESCALATE → `llm-escalated`
- Risk check fails → `risk-gate`
- Status guard fails → `status-guard-abort`
- `executeActions` completes → `executed` or `partial-failure`

### `callLlmWithRetry()` (line 193)

Each `llmAgent.execute()` call individually timed with `llm.call.duration`.
Tagged `outcome=success` or `outcome=error`.

Retry attempts increment `llm.retries` counter.

Token extraction: after `result.output()`, check if the result map contains
token usage keys. If present, record to distribution summaries.

### `executeActions()` (line 262)

Wrap with `action.execution.duration` timer. Each `deviceCommandFn.apply()`
increments `actions.executed` with `succeeded=true` or `succeeded=false`.

### `sweepStaleEntries()` (line 301)

Each escalation increments `entries.processed` with `outcome=timeout`.

### Semaphore gauge

Registered once in `init()` after semaphore creation:
```java
Gauge.builder("casehub.iot.ai.resolution.semaphore.available",
              llmSemaphore, Semaphore::availablePermits)
     .register(registry);
```

---

## §4 Health Check

`IoTAiResolutionReadinessCheck` — `@Readiness @ApplicationScoped` bean following
the `BridgeReadinessCheck` pattern.

Reports **UP** when all conditions are met:
1. `config.enabled()` is true
2. `aiResolutionViewId` is non-null (view resolved at startup)
3. `operatorAssistedViewId` is non-null

Reports **DOWN** with diagnostic data when any condition fails.

Data fields exposed:
- `enabled` — boolean
- `aiResolutionViewResolved` — boolean
- `operatorAssistedViewResolved` — boolean
- `semaphorePermits` — int (current available)

The health check injects `IoTAiResolutionAgent` and calls package-private
methods `isReady()` and `healthData()`. The agent owns its state; the health
check translates it to `HealthCheckResponse`.

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
- `llmTransientFailure_retriesThenEscalates` → `llm.retries` = 2,
  `entries.processed{outcome=llm-error}` = 1
- `partialWorkerFailure` → `entries.processed{outcome=partial-failure}` = 1,
  `actions.executed{succeeded=false}` = 1

### New test

`IoTAiResolutionReadinessCheckTest` — verify UP when agent is fully initialized,
DOWN when disabled or views not resolved.

### Not tested here

Prometheus scrape endpoint (`/q/metrics`) — Quarkus infrastructure.

---

## §6 Scope Boundaries

**In scope:**
- `quarkus-micrometer-registry-prometheus` and `quarkus-smallrye-health` dependencies
- Inline Micrometer instrumentation in `IoTAiResolutionAgent`
- `IoTAiResolutionReadinessCheck` health check
- Token metrics if Agent API exposes them
- CBR similarity band tagging on outcome counters
- Test coverage via `SimpleMeterRegistry`

**Out of scope:**
- Grafana dashboards or alerting rules
- Metrics for other modules (bridge, MCP, providers)
- Custom Prometheus recording rules
- Token metrics if Agent API doesn't expose them (follow-up engine issue)
