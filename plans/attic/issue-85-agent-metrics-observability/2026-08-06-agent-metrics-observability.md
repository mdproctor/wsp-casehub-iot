# Agent Metrics and Observability Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #85 — Agent performance metrics and observability
**Issue group:** #85

**Goal:** Add Micrometer metrics and MicroProfile Health instrumentation
to `IoTAiResolutionAgent` — LLM call latency, success/escalation rates,
resolution quality correlated with CBR similarity bands, and a readiness
health check.

**Architecture:** Inline instrumentation in `IoTAiResolutionAgent` with
`MeterRegistry` injection. Timers, counters, and gauges recorded at
natural instrumentation points. Separate `@Readiness` health check CDI
bean that delegates to agent state via package-private methods.

**Tech Stack:** Quarkus Micrometer (Prometheus registry), MicroProfile
Health (SmallRye), Micrometer `SimpleMeterRegistry` for tests.

## Global Constraints

- First Micrometer/Health introduction in the repo — no existing patterns
  to follow except `BridgeReadinessCheck` for health checks
- All metric names use prefix `casehub.iot.ai.resolution` with Micrometer
  dot-separated lowercase convention
- `IoTAiResolutionAgent` tests use reflection-based injection with
  Mockito — no CDI container
- Token consumption metrics deferred — requires engine `Agent` API change

---

### Task 1: Add Micrometer and Health dependencies

**Files:**
- Modify: `webapp/pom.xml`

**Interfaces:**
- Consumes: nothing
- Produces: `quarkus-micrometer-registry-prometheus` and
  `quarkus-smallrye-health` on the webapp classpath

- [ ] **Step 1: Add dependencies to webapp/pom.xml**

Add after the existing `quarkus-scheduler` dependency:

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

Both are managed by the Quarkus BOM — no version element needed.

- [ ] **Step 2: Verify build compiles**

Run: `mvn -pl webapp compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add webapp/pom.xml
git commit -m "feat(#85): add quarkus-micrometer-registry-prometheus and smallrye-health to webapp"
```

---

### Task 2: Instrument IoTAiResolutionAgent with metrics

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java`

**Interfaces:**
- Consumes: `MeterRegistry` (from Micrometer, now on classpath from Task 1)
- Produces: package-private `boolean isReady()` and
  `Map<String, Object> healthData()` methods (consumed by Task 3)

This is the largest task — it instruments the agent and extends all
existing tests with metric assertions. Broken into sub-steps by method.

#### 2a: Add fields and init() registration

- [ ] **Step 1: Add MeterRegistry injection and metric fields**

Add to `IoTAiResolutionAgent`:

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import java.util.concurrent.atomic.AtomicInteger;

@Inject MeterRegistry registry;

private Timer pollDurationTimer;
private Timer llmCallDurationTimer;
private Timer entryDurationTimer;
private Timer actionExecutionDurationTimer;
private final AtomicInteger pendingCount = new AtomicInteger(0);
```

- [ ] **Step 2: Register gauges and create timers in init()**

Add at the end of `init()`, after `deviceCommandFn` construction:

```java
pollDurationTimer = Timer.builder("casehub.iot.ai.resolution.poll.duration")
        .register(registry);
llmCallDurationTimer = Timer.builder("casehub.iot.ai.resolution.llm.call.duration")
        .register(registry);
entryDurationTimer = Timer.builder("casehub.iot.ai.resolution.entry.duration")
        .register(registry);
actionExecutionDurationTimer = Timer.builder("casehub.iot.ai.resolution.action.execution.duration")
        .register(registry);

Gauge.builder("casehub.iot.ai.resolution.semaphore.available",
              llmSemaphore, Semaphore::availablePermits)
     .register(registry);

Gauge.builder("casehub.iot.ai.resolution.queue.pending",
              pendingCount, AtomicInteger::get)
     .register(registry);
```

- [ ] **Step 3: Add package-private health methods**

Add to `IoTAiResolutionAgent`:

```java
boolean isReady() {
    return config.enabled()
            && aiResolutionViewId != null
            && operatorAssistedViewId != null;
}

Map<String, Object> healthData() {
    return Map.of(
            "enabled", config.enabled(),
            "aiResolutionViewResolved", aiResolutionViewId != null,
            "operatorAssistedViewResolved", operatorAssistedViewId != null,
            "semaphorePermits", llmSemaphore != null ? llmSemaphore.availablePermits() : 0);
}
```

- [ ] **Step 4: Verify build compiles**

Run: `mvn -pl webapp compile -q`
Expected: BUILD SUCCESS

#### 2b: Instrument poll() and processNewEntries()

- [ ] **Step 5: Wrap poll() with Timer.Sample**

Replace `poll()` method body:

```java
public void poll() {
    if (!config.enabled() || aiResolutionViewId == null || operatorAssistedViewId == null) {
        return;
    }
    Timer.Sample sample = Timer.start(registry);
    try {
        processNewEntries();
        sweepStaleEntries();
    } finally {
        sample.stop(pollDurationTimer);
    }
}
```

- [ ] **Step 6: Update processNewEntries() to record pending count**

Add `pendingCount.set(pending.size())` after the `findPending` call:

```java
private void processNewEntries() {
    List<CaseQueueEntry> pending = queueService.findPending(aiResolutionViewId, tenancyId);
    pendingCount.set(pending.size());
    for (CaseQueueEntry entry : pending) {
        virtualThreads.submit(() -> {
            try {
                processEntry(entry);
            } catch (Exception e) {
                LOG.errorf(e, "Unexpected error processing queue entry %s", entry.getId());
            }
        });
    }
}
```

#### 2c: Instrument processEntry() with try-finally

- [ ] **Step 7: Add CBR band derivation helper**

Add static method to `IoTAiResolutionAgent`:

```java
static String cbrBand(List<ResolutionSuggestion> suggestions, boolean cbrConfigPresent) {
    if (suggestions == null || !cbrConfigPresent) {
        return "unknown";
    }
    if (suggestions.isEmpty()) {
        return "none";
    }
    double max = suggestions.stream()
            .mapToDouble(ResolutionSuggestion::similarityScore)
            .max()
            .orElse(0.0);
    if (max >= 0.85) return "high";
    if (max >= 0.6) return "medium";
    return "low";
}
```

- [ ] **Step 8: Restructure processEntry() with try-finally and metric recording**

Replace the `processEntry` method. Key changes:
- Claim contention increments `claim.contention`, returns without timer
- `Timer.Sample` starts after successful claim
- `cbrBand` tracked as a local variable, defaulting to `"unknown"`
- Try-finally ensures timer stops and counter increments on all paths
- `executeActions` returns a String outcome (`"executed"` or `"partial-failure"`)

```java
private void processEntry(CaseQueueEntry pendingEntry) {
    CaseQueueEntry entry;
    try {
        entry = queueService.claim(pendingEntry.getId(), tenancyId, config.agentId());
    } catch (IllegalStateException e) {
        LOG.debugf("Entry %s already claimed — skipping", pendingEntry.getId());
        registry.counter("casehub.iot.ai.resolution.claim.contention").increment();
        return;
    }

    Timer.Sample sample = Timer.start(registry);
    String outcome = "error";
    String cbrBand = "unknown";
    boolean cbrConfigPresent = false;

    try {
        UUID caseId = entry.getCaseId();
        CaseInstance instance = caseCache.get(caseId);
        if (instance == null) {
            LOG.warnf("Case %s not found in cache — releasing entry", caseId);
            outcome = "case-not-found";
            return;
        }

        String caseType = instance.getCaseMetaModel().getName();
        var defOpt = definitionRegistry.findByName(caseType);
        if (defOpt.isEmpty()) {
            escalateWithReason(entry, "Case definition not found: " + caseType, instance, List.of());
            outcome = "case-not-found";
            return;
        }

        CbrConfig cbrConfig = defOpt.get().getCbrConfig();
        cbrConfigPresent = cbrConfig != null;
        List<ResolutionSuggestion> suggestions;
        if (cbrConfigPresent) {
            var features = extractFeatures(instance);
            suggestions = retrievalService.retrieve(cbrConfig, features, tenancyId);
        } else {
            suggestions = List.of();
        }
        cbrBand = cbrBand(suggestions, cbrConfigPresent);

        writePreLlmContext(instance, suggestions);

        AiResolutionPlan plan = callLlmWithRetry(entry, instance, suggestions);
        if (plan == null) {
            outcome = "llm-error";
            return;
        }

        if (plan.decision() == Decision.ESCALATE) {
            escalateWithReason(entry, plan.escalationReason(), instance, suggestions);
            outcome = "llm-escalated";
            return;
        }

        if (!riskCheckPasses(plan, entry, instance, caseType, suggestions)) {
            outcome = "risk-gate";
            return;
        }

        if (!statusGuardPasses(entry)) {
            LOG.infof("Entry %s was moved by timeout sweep — aborting execution", entry.getId());
            outcome = "status-guard-abort";
            return;
        }

        outcome = executeActions(plan, entry, instance, suggestions);
    } finally {
        sample.stop(Timer.builder("casehub.iot.ai.resolution.entry.duration")
                .tag("outcome", outcome)
                .register(registry));
        registry.counter("casehub.iot.ai.resolution.entries.processed",
                "outcome", outcome, "cbr.band", cbrBand).increment();
    }
}
```

- [ ] **Step 9: Change executeActions() return type to String**

Modify `executeActions` to return the outcome string instead of void:

```java
private String executeActions(AiResolutionPlan plan, CaseQueueEntry entry,
                              CaseInstance instance, List<ResolutionSuggestion> suggestions) {
    // ... existing body unchanged until the end ...

    if (allSucceeded) {
        instance.getCaseContext().set("aiResolutionResults", results);
        LOG.infof("AI resolution succeeded for case %s — %d actions executed",
                instance.getUuid(), results.size());
        return "executed";
    } else {
        instance.getCaseContext().set("aiResolutionResults", results);
        updateEscalationContext(instance, "Partial worker failure",
                suggestions, plan.reasoning(), plan.actions(), results);
        queueService.escalate(entry.getId(), tenancyId, operatorAssistedViewId);
        LOG.warnf("Partial failure for case %s — %d/%d actions executed, escalating",
                instance.getUuid(), results.stream().filter(ExecutedActionResult::succeeded).count(),
                plan.actions().size());
        return "partial-failure";
    }
}
```

#### 2d: Instrument callLlmWithRetry()

- [ ] **Step 10: Add LLM call timer and retry counter**

Inside the retry loop, add timer after `acquire()` and record per attempt.
Add retry counter increment before backoff sleep:

```java
private AiResolutionPlan callLlmWithRetry(CaseQueueEntry entry, CaseInstance instance,
                                           List<ResolutionSuggestion> suggestions) {
    String prompt = AiResolutionPromptBuilder.build(
            extractFeatures(instance), suggestions, AUTONOMOUS_ACTIONS);

    for (int attempt = 0; attempt <= MAX_RETRIES; attempt++) {
        try {
            llmSemaphore.acquire();
            Timer.Sample llmSample = Timer.start(registry);
            try {
                WorkerResult<Map<String, Object>> result =
                        llmAgent.execute(Map.of("prompt", prompt));
                llmSample.stop(Timer.builder("casehub.iot.ai.resolution.llm.call.duration")
                        .tag("outcome", "success").register(registry));
                return objectMapper.convertValue(result.output(), AiResolutionPlan.class);
            } catch (Exception e) {
                llmSample.stop(Timer.builder("casehub.iot.ai.resolution.llm.call.duration")
                        .tag("outcome", "error").register(registry));
                throw e;
            } finally {
                llmSemaphore.release();
            }
        } catch (AgentException e) {
            escalateWithReason(entry, "LLM error: " + e.getMessage(), instance, suggestions);
            return null;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            escalateWithReason(entry, "Interrupted waiting for LLM permit", instance, suggestions);
            return null;
        } catch (Exception e) {
            if (isTransient(e) && attempt < MAX_RETRIES) {
                LOG.warnf("Transient LLM failure (attempt %d/%d): %s",
                        attempt + 1, MAX_RETRIES + 1, e.getMessage());
                registry.counter("casehub.iot.ai.resolution.llm.retries").increment();
                try {
                    Thread.sleep(RETRY_DELAYS_MS[attempt]);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    escalateWithReason(entry, "Interrupted during retry backoff", instance, suggestions);
                    return null;
                }
            } else {
                escalateWithReason(entry, "LLM failure after retries: " + e.getMessage(),
                        instance, suggestions);
                return null;
            }
        }
    }
    return null;
}
```

#### 2e: Instrument executeActions() and sweepStaleEntries()

- [ ] **Step 11: Add action execution timer and counter to executeActions()**

Wrap action loop with timer and add per-action counter:

```java
private String executeActions(AiResolutionPlan plan, CaseQueueEntry entry,
                              CaseInstance instance, List<ResolutionSuggestion> suggestions) {
    List<ExecutedActionResult> results = new ArrayList<>();
    boolean allSucceeded = true;

    Timer.Sample actionSample = Timer.start(registry);
    for (PlannedActionSpec spec : plan.actions()) {
        Map<String, Object> input = Map.of(
                "targetDeviceId", spec.targetDeviceId(),
                "action", spec.actionType(),
                "parameters", spec.parameters());

        WorkerResult<Map<String, Object>> workerResult = deviceCommandFn.apply(input);

        if (workerResult.outcome() instanceof io.casehub.worker.api.WorkerOutcome.Failed<?> failed) {
            results.add(new ExecutedActionResult(spec, false, failed.reason()));
            registry.counter("casehub.iot.ai.resolution.actions.executed",
                    "succeeded", "false").increment();
            allSucceeded = false;
            break;
        } else {
            String workerOutcome = workerResult.output() != null
                    ? workerResult.output().toString() : "SUCCESS";
            results.add(new ExecutedActionResult(spec, true, workerOutcome));
            registry.counter("casehub.iot.ai.resolution.actions.executed",
                    "succeeded", "true").increment();
        }
    }

    String executionOutcome = allSucceeded ? "success" : "partial-failure";
    actionSample.stop(Timer.builder("casehub.iot.ai.resolution.action.execution.duration")
            .tag("outcome", executionOutcome).register(registry));

    if (allSucceeded) {
        instance.getCaseContext().set("aiResolutionResults", results);
        LOG.infof("AI resolution succeeded for case %s — %d actions executed",
                instance.getUuid(), results.size());
        return "executed";
    } else {
        instance.getCaseContext().set("aiResolutionResults", results);
        updateEscalationContext(instance, "Partial worker failure",
                suggestions, plan.reasoning(), plan.actions(), results);
        queueService.escalate(entry.getId(), tenancyId, operatorAssistedViewId);
        LOG.warnf("Partial failure for case %s — %d/%d actions executed, escalating",
                instance.getUuid(), results.stream().filter(ExecutedActionResult::succeeded).count(),
                plan.actions().size());
        return "partial-failure";
    }
}
```

- [ ] **Step 12: Add timeout counter to sweepStaleEntries()**

Add counter increment before each escalation:

```java
private void sweepStaleEntries() {
    List<CaseQueueEntry> all = queueService.findByView(aiResolutionViewId, tenancyId);
    Instant threshold = Instant.now().minusSeconds(config.timeoutSeconds());
    for (CaseQueueEntry entry : all) {
        if (entry.getStatus() == QueueEntryStatus.CLAIMED
                && entry.getClaimedAt() != null
                && entry.getClaimedAt().isBefore(threshold)) {
            LOG.warnf("Timeout sweep: escalating stale entry %s (claimed at %s)",
                    entry.getId(), entry.getClaimedAt());
            registry.counter("casehub.iot.ai.resolution.entries.processed",
                    "outcome", "timeout", "cbr.band", "unknown").increment();
            queueService.escalate(entry.getId(), tenancyId, operatorAssistedViewId);
        }
    }
}
```

- [ ] **Step 13: Verify build compiles**

Run: `mvn -pl webapp compile -q`
Expected: BUILD SUCCESS

#### 2f: Extend existing tests with metric assertions

- [ ] **Step 14: Add SimpleMeterRegistry to test setUp()**

Add to `IoTAiResolutionAgentTest`:

```java
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;

// As field:
private SimpleMeterRegistry meterRegistry;

// In setUp(), after existing inject() calls:
meterRegistry = new SimpleMeterRegistry();
inject(agent, "registry", meterRegistry);
```

- [ ] **Step 15: Add metric assertion helpers**

Add to `IoTAiResolutionAgentTest`:

```java
private double counterValue(String name, String... tags) {
    Counter counter = meterRegistry.find(name).tags(tags).counter();
    return counter != null ? counter.count() : 0.0;
}

private long timerCount(String name, String... tags) {
    Timer timer = meterRegistry.find(name).tags(tags).timer();
    return timer != null ? timer.count() : 0;
}
```

Add imports:

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Timer;
```

- [ ] **Step 16: Add metric assertions to happyPath test**

Add at the end of `happyPath_claimsExecutesAndWritesResults`:

```java
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "executed", "cbr.band", "none")).isEqualTo(1.0);
assertThat(counterValue("casehub.iot.ai.resolution.actions.executed",
        "succeeded", "true")).isGreaterThanOrEqualTo(1.0);
assertThat(timerCount("casehub.iot.ai.resolution.llm.call.duration",
        "outcome", "success")).isEqualTo(1);
assertThat(timerCount("casehub.iot.ai.resolution.poll.duration")).isEqualTo(1);
```

Add import:

```java
import static org.assertj.core.api.Assertions.assertThat;
```

Add AssertJ dependency to `webapp/pom.xml` test scope if not present:

```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 17: Add metric assertions to remaining tests**

`llmDecidesToEscalate`:
```java
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "llm-escalated", "cbr.band", "none")).isEqualTo(1.0);
```

`riskGateTriggersEscalation`:
```java
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "risk-gate", "cbr.band", "none")).isEqualTo(1.0);
```

`timeoutSweep_escalatesStaleClaimedEntries`:
```java
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "timeout", "cbr.band", "unknown")).isEqualTo(1.0);
```

`statusGuard_abortsWhenEntryMoved`:
```java
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "status-guard-abort", "cbr.band", "none")).isEqualTo(1.0);
```

`partialWorkerFailure_escalatesWithExecutedRecord`:
```java
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "partial-failure", "cbr.band", "none")).isEqualTo(1.0);
assertThat(counterValue("casehub.iot.ai.resolution.actions.executed",
        "succeeded", "false")).isEqualTo(1.0);
assertThat(counterValue("casehub.iot.ai.resolution.actions.executed",
        "succeeded", "true")).isEqualTo(1.0);
```

`llmDeterministicFailure_escalatesImmediately`:
```java
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "llm-error", "cbr.band", "none")).isEqualTo(1.0);
assertThat(timerCount("casehub.iot.ai.resolution.llm.call.duration",
        "outcome", "error")).isEqualTo(1);
```

`llmTransientFailure_retriesThenEscalates`:
```java
assertThat(counterValue("casehub.iot.ai.resolution.llm.retries")).isEqualTo(2.0);
assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
        "outcome", "llm-error", "cbr.band", "none")).isEqualTo(1.0);
```

- [ ] **Step 18: Add claim contention test**

Add new test method:

```java
@Test
void claimContention_incrementsSeparateCounter() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId);

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.claim(entry.getId(), TENANCY, "iot-ai-agent"))
            .thenThrow(new IllegalStateException("Already claimed"));
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());

    agent.poll();

    assertThat(counterValue("casehub.iot.ai.resolution.claim.contention")).isEqualTo(1.0);
    assertThat(counterValue("casehub.iot.ai.resolution.entries.processed",
            "outcome", "claim-failed")).isEqualTo(0.0);
}
```

- [ ] **Step 19: Run all tests**

Run: `mvn -pl webapp test -Dtest=IoTAiResolutionAgentTest -q`
Expected: All tests PASS

- [ ] **Step 20: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java
git add webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java
git add webapp/pom.xml
git commit -m "feat(#85): instrument IoTAiResolutionAgent with Micrometer metrics

Refs #85

Adds timers (poll, LLM call, entry processing, action execution),
counters (entries processed by outcome+CBR band, claim contention,
LLM retries, actions executed), and gauges (semaphore permits,
queue pending). All existing tests extended with metric assertions."
```

---

### Task 3: Add IoTAiResolutionReadinessCheck

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionReadinessCheck.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionReadinessCheckTest.java`

**Interfaces:**
- Consumes: `IoTAiResolutionAgent.isReady()` and
  `IoTAiResolutionAgent.healthData()` (package-private, from Task 2)
- Produces: `/q/health/ready` endpoint data for `ai-resolution-agent`

- [ ] **Step 1: Write the health check test**

Create `IoTAiResolutionReadinessCheckTest.java`:

```java
package io.casehub.iot.webapp.app.resolution;

import org.eclipse.microprofile.health.HealthCheckResponse;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class IoTAiResolutionReadinessCheckTest {

    @Test
    void up_whenAgentIsReady() {
        IoTAiResolutionAgent agent = mock(IoTAiResolutionAgent.class);
        when(agent.isReady()).thenReturn(true);
        when(agent.healthData()).thenReturn(Map.of(
                "enabled", true,
                "aiResolutionViewResolved", true,
                "operatorAssistedViewResolved", true,
                "semaphorePermits", 3));

        IoTAiResolutionReadinessCheck check = new IoTAiResolutionReadinessCheck(agent);
        HealthCheckResponse response = check.call();

        assertThat(response.getStatus()).isEqualTo(HealthCheckResponse.Status.UP);
        assertThat(response.getName()).isEqualTo("ai-resolution-agent");
        assertThat(response.getData().get().get("semaphorePermits")).isEqualTo(3L);
    }

    @Test
    void down_whenAgentNotReady() {
        IoTAiResolutionAgent agent = mock(IoTAiResolutionAgent.class);
        when(agent.isReady()).thenReturn(false);
        when(agent.healthData()).thenReturn(Map.of(
                "enabled", false,
                "aiResolutionViewResolved", false,
                "operatorAssistedViewResolved", false,
                "semaphorePermits", 0));

        IoTAiResolutionReadinessCheck check = new IoTAiResolutionReadinessCheck(agent);
        HealthCheckResponse response = check.call();

        assertThat(response.getStatus()).isEqualTo(HealthCheckResponse.Status.DOWN);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl webapp test -Dtest=IoTAiResolutionReadinessCheckTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Write the health check implementation**

Create `IoTAiResolutionReadinessCheck.java`:

```java
package io.casehub.iot.webapp.app.resolution;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.health.HealthCheck;
import org.eclipse.microprofile.health.HealthCheckResponse;
import org.eclipse.microprofile.health.HealthCheckResponseBuilder;
import org.eclipse.microprofile.health.Readiness;

import java.util.Map;

@Readiness
@ApplicationScoped
public class IoTAiResolutionReadinessCheck implements HealthCheck {

    private final IoTAiResolutionAgent agent;

    @Inject
    public IoTAiResolutionReadinessCheck(IoTAiResolutionAgent agent) {
        this.agent = agent;
    }

    @Override
    public HealthCheckResponse call() {
        HealthCheckResponseBuilder builder = HealthCheckResponse.named("ai-resolution-agent")
                .status(agent.isReady());
        Map<String, Object> data = agent.healthData();
        data.forEach((key, value) -> {
            if (value instanceof Boolean b) {
                builder.withData(key, b);
            } else if (value instanceof Long l) {
                builder.withData(key, l);
            } else if (value instanceof Integer i) {
                builder.withData(key, (long) i);
            } else {
                builder.withData(key, String.valueOf(value));
            }
        });
        return builder.build();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -pl webapp test -Dtest=IoTAiResolutionReadinessCheckTest -q`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn -pl webapp test -q`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionReadinessCheck.java
git add webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionReadinessCheckTest.java
git commit -m "feat(#85): add IoTAiResolutionReadinessCheck health check

Refs #85

@Readiness check reports UP when agent is enabled, views resolved,
and LLM agent initialized. Null-guards semaphore for pre-init calls."
```

---

### Task 4: Full build verification and consumer guide update

**Files:**
- Modify: `docs/guides/consumer-guide.md` (add metrics/health endpoint documentation)

**Interfaces:**
- Consumes: all prior tasks
- Produces: updated documentation

- [ ] **Step 1: Run full Maven build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 2: Update consumer guide with metrics endpoint documentation**

Add a section to `docs/guides/consumer-guide.md` documenting the new
`/q/metrics` and `/q/health` endpoints for the webapp module. Include
the metric names and their meaning.

- [ ] **Step 3: File follow-up issues**

File two GitHub issues per §6 scope boundaries:

1. **Token consumption metrics** on `casehubio/engine`:
   "feat: expose token usage from Agent.execute() — enrich WorkerResult
   with ChatResponse.tokenUsage() data (inputTokenCount, outputTokenCount).
   Required by casehubio/iot#85 for LLM cost monitoring."

2. **Grafana dashboards** on `casehubio/iot`:
   "feat: Grafana dashboard for AI resolution agent metrics — visualise
   LLM latency, success/escalation rates, CBR correlation, queue depth.
   Metrics exposed by #85, dashboards needed to make them visible to
   operators."

- [ ] **Step 4: Commit**

```bash
git add docs/guides/consumer-guide.md
git commit -m "docs(#85): add metrics and health endpoint documentation to consumer guide

Refs #85"
```
