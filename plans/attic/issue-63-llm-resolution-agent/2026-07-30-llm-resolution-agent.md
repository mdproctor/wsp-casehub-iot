# IoTAiResolutionAgent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #63 — feat: LLM resolution agent — IoTAiResolutionAgent
**Issue group:** #63

**Goal:** Build an LLM agent that claims cases from the `iot-ai-resolution`
queue view, loads CBR suggestions as context, calls the LLM to produce a
resolution plan, risk-classifies each action, and either executes directly
via worker functions or escalates to the operator-assisted queue.

**Architecture:** `@Scheduled` polling via `CaseQueueService.findPending()`,
LLM calls via the engine's `Agent` infrastructure (langchain4j), direct
worker function execution for Autonomous actions, escalation via
`CaseQueueService.escalate()`. Pure data records in `webapp-api` (Tier 1),
CDI wiring in `webapp`.

**Tech Stack:** Java 21, Quarkus 3.x, langchain4j (via engine `Agent`),
CDI, Mockito, AssertJ, JUnit 5

## Global Constraints

- All `webapp-api` types are CDI-free (Tier 1). No `@Inject`, no
  `@ApplicationScoped`, no Jakarta EE annotations.
- All config properties use `@WithDefault` — no SmallRye startup validation
  failures.
- Tenancy ID via `@ConfigProperty(name = "casehub.iot.tenancy-id")` — the
  platform-wide single tenancy property (not per-module config mapping).
- `casehub-engine-queue` and `casehub-platform-view` dependencies already
  in webapp pom.xml and parent BOM — no pom changes needed.
- Engine change: add `findByView(UUID, String)` to `CaseQueueService` in
  `casehub-engine` repo. Must be committed to engine first and
  `mvn install`'d before webapp integration tests.
- Tests use reflection injection for CDI fields (matching existing pattern
  in `IoTQueueViewInitializerTest`).

---

### Task 1: Engine — add `findByView()` to CaseQueueService

Cross-repo change in `casehub-engine`. Exposes the unfiltered entry list
through the service layer so the timeout sweep can find CLAIMED entries.

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/engine/queue/src/main/java/io/casehub/engine/queue/service/CaseQueueService.java`
- Test: `/Users/mdproctor/claude/casehub/engine/queue/src/test/java/io/casehub/engine/queue/service/CaseQueueServiceTest.java`

**Interfaces:**
- Consumes: `CaseQueueEntryStore.findByView(UUID viewId, String tenancyId)` (already exists)
- Produces: `CaseQueueService.findByView(UUID viewId, String tenancyId): List<CaseQueueEntry>` — returns all entries regardless of status

- [ ] **Step 1: Write the failing test**

```java
@Test
void findByView_returnsAllStatuses() {
    UUID viewId = UUID.randomUUID();
    String tenancyId = "test-tenant";

    CaseQueueEntry pending = new CaseQueueEntry(
        UUID.randomUUID(), UUID.randomUUID(), tenancyId,
        viewId, "test-view", QueueEntryStatus.PENDING, Instant.now());
    CaseQueueEntry claimed = new CaseQueueEntry(
        UUID.randomUUID(), UUID.randomUUID(), tenancyId,
        viewId, "test-view", QueueEntryStatus.CLAIMED, Instant.now());
    claimed.setAssignedTo("agent");
    claimed.setClaimedAt(Instant.now().minusSeconds(600));

    when(store.findByView(viewId, tenancyId)).thenReturn(List.of(pending, claimed));

    List<CaseQueueEntry> result = service.findByView(viewId, tenancyId);

    assertThat(result).hasSize(2);
    assertThat(result).extracting(CaseQueueEntry::getStatus)
        .containsExactlyInAnyOrder(QueueEntryStatus.PENDING, QueueEntryStatus.CLAIMED);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /Users/mdproctor/claude/casehub/engine/queue/pom.xml test -pl . -Dtest=CaseQueueServiceTest#findByView_returnsAllStatuses`
Expected: FAIL — method `findByView` does not exist on `CaseQueueService`

- [ ] **Step 3: Implement findByView**

Add to `CaseQueueService` using `ide_insert_member`:

```java
public List<CaseQueueEntry> findByView(UUID viewId, String tenancyId) {
    return store.findByView(viewId, tenancyId);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f /Users/mdproctor/claude/casehub/engine/queue/pom.xml test -pl . -Dtest=CaseQueueServiceTest#findByView_returnsAllStatuses`
Expected: PASS

- [ ] **Step 5: Install engine to local maven**

Run: `mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml install -Dmaven.test.skip=true`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add queue/src/main/java/io/casehub/engine/queue/service/CaseQueueService.java queue/src/test/java/io/casehub/engine/queue/service/CaseQueueServiceTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#63): add findByView to CaseQueueService — exposes all entries regardless of status"
```

---

### Task 2: Data records — AiResolutionPlan, PlannedActionSpec, ExecutedActionResult, AiEscalationContext

Pure data records in `webapp-api` (Tier 1). No CDI.

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/AiResolutionPlan.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/PlannedActionSpec.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ExecutedActionResult.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/AiEscalationContext.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/Decision.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/resolution/AiResolutionPlanTest.java`

**Interfaces:**
- Consumes: `ResolutionSuggestion` (from `io.casehub.iot.webapp.cbr`)
- Produces: `AiResolutionPlan`, `PlannedActionSpec`, `ExecutedActionResult`, `AiEscalationContext`, `Decision` — consumed by Tasks 3, 4, 5

- [ ] **Step 1: Create Decision enum**

```java
package io.casehub.iot.webapp.resolution;

public enum Decision {
    EXECUTE,
    ESCALATE
}
```

- [ ] **Step 2: Create PlannedActionSpec record**

```java
package io.casehub.iot.webapp.resolution;

import java.util.Map;
import java.util.Objects;

public record PlannedActionSpec(
        String actionType,
        String targetDeviceId,
        Map<String, Object> parameters,
        String rationale
) {
    public PlannedActionSpec {
        Objects.requireNonNull(actionType, "actionType must not be null");
        Objects.requireNonNull(targetDeviceId, "targetDeviceId must not be null");
        parameters = parameters != null ? Map.copyOf(parameters) : Map.of();
    }
}
```

- [ ] **Step 3: Create AiResolutionPlan record**

```java
package io.casehub.iot.webapp.resolution;

import java.util.List;
import java.util.Objects;

public record AiResolutionPlan(
        Decision decision,
        String reasoning,
        List<PlannedActionSpec> actions,
        String escalationReason
) {
    public AiResolutionPlan {
        Objects.requireNonNull(decision, "decision must not be null");
        Objects.requireNonNull(reasoning, "reasoning must not be null");
        actions = actions != null ? List.copyOf(actions) : List.of();
    }
}
```

- [ ] **Step 4: Create ExecutedActionResult record**

```java
package io.casehub.iot.webapp.resolution;

import java.util.Objects;

public record ExecutedActionResult(
        PlannedActionSpec action,
        boolean succeeded,
        String outcome
) {
    public ExecutedActionResult {
        Objects.requireNonNull(action, "action must not be null");
        Objects.requireNonNull(outcome, "outcome must not be null");
    }
}
```

- [ ] **Step 5: Create AiEscalationContext record**

```java
package io.casehub.iot.webapp.resolution;

import io.casehub.iot.webapp.cbr.ResolutionSuggestion;

import java.util.List;

public record AiEscalationContext(
        String reason,
        List<ResolutionSuggestion> consideredSuggestions,
        String partialAnalysis,
        List<PlannedActionSpec> partialPlan,
        List<ExecutedActionResult> executedActions
) {}
```

- [ ] **Step 6: Write AiResolutionPlanTest — JSON round-trip**

```java
package io.casehub.iot.webapp.resolution;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AiResolutionPlanTest {

    private static final ObjectMapper mapper = new ObjectMapper();

    @Test
    void deserializesExecutePlan() throws Exception {
        String json = """
            {
              "decision": "EXECUTE",
              "reasoning": "High similarity match with past filter replacement case",
              "actions": [{
                "actionType": "SET_TEMPERATURE",
                "targetDeviceId": "thermo-001",
                "parameters": {"target": 22},
                "rationale": "Reset to normal operating temperature"
              }],
              "escalationReason": null
            }
            """;

        AiResolutionPlan plan = mapper.readValue(json, AiResolutionPlan.class);

        assertThat(plan.decision()).isEqualTo(Decision.EXECUTE);
        assertThat(plan.actions()).hasSize(1);
        assertThat(plan.actions().get(0).actionType()).isEqualTo("SET_TEMPERATURE");
        assertThat(plan.actions().get(0).targetDeviceId()).isEqualTo("thermo-001");
        assertThat(plan.escalationReason()).isNull();
    }

    @Test
    void deserializesEscalatePlan() throws Exception {
        String json = """
            {
              "decision": "ESCALATE",
              "reasoning": "Context differs significantly from past cases",
              "actions": [],
              "escalationReason": "No matching resolution pattern for multi-zone failure"
            }
            """;

        AiResolutionPlan plan = mapper.readValue(json, AiResolutionPlan.class);

        assertThat(plan.decision()).isEqualTo(Decision.ESCALATE);
        assertThat(plan.actions()).isEmpty();
        assertThat(plan.escalationReason()).isNotNull();
    }

    @Test
    void rejectsNullDecision() {
        assertThatThrownBy(() -> new AiResolutionPlan(null, "reason", List.of(), null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void rejectsNullReasoning() {
        assertThatThrownBy(() -> new AiResolutionPlan(Decision.EXECUTE, null, List.of(), null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullActionsDefaultsToEmptyList() {
        AiResolutionPlan plan = new AiResolutionPlan(Decision.ESCALATE, "reason", null, "escalate");
        assertThat(plan.actions()).isEmpty();
    }
}
```

- [ ] **Step 7: Run tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/webapp-api/pom.xml test -Dtest=AiResolutionPlanTest`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ webapp-api/src/test/java/io/casehub/iot/webapp/resolution/
git commit -m "feat(#63): data records for AI resolution — AiResolutionPlan, PlannedActionSpec, ExecutedActionResult, AiEscalationContext"
```

---

### Task 3: AiResolutionPromptBuilder

Builds the LLM prompt string from case context, CBR suggestions, and
available action types. Pure computation — Tier 1, no CDI.

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/AiResolutionPromptBuilder.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/resolution/AiResolutionPromptBuilderTest.java`

**Interfaces:**
- Consumes: `ResolutionSuggestion` (from `cbr` package), `Map<String, Object>` (case context from working layer)
- Produces: `AiResolutionPromptBuilder.build(Map<String, Object> caseContext, List<ResolutionSuggestion> suggestions, Set<String> availableActions): String`

- [ ] **Step 1: Write the failing test — basic prompt construction**

```java
package io.casehub.iot.webapp.resolution;

import io.casehub.iot.webapp.cbr.ResolutionSuggestion;
import io.casehub.neocortex.memory.cbr.PlanTrace;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class AiResolutionPromptBuilderTest {

    @Test
    void buildIncludesCaseContextDeviceClassAndRoomType() {
        Map<String, Object> context = Map.of(
            "deviceClass", "thermostat",
            "roomType", "living_room",
            "eventDescription", "Temperature rising above threshold"
        );
        List<ResolutionSuggestion> suggestions = List.of();
        Set<String> actions = Set.of("SET_TEMPERATURE");

        String prompt = AiResolutionPromptBuilder.build(context, suggestions, actions);

        assertThat(prompt).contains("thermostat");
        assertThat(prompt).contains("living_room");
        assertThat(prompt).contains("Temperature rising above threshold");
        assertThat(prompt).contains("SET_TEMPERATURE");
    }

    @Test
    void buildIncludesSuggestionDetails() {
        Map<String, Object> context = Map.of("deviceClass", "thermostat");
        ResolutionSuggestion suggestion = new ResolutionSuggestion(
            "past-case-1", 0.92, "Sustained temperature rise",
            "Replaced blocked HVAC filter", "RESOLVED", 0.95,
            Map.of("deviceClass", "thermostat"),
            Map.of("deviceClass", 1.0),
            List.of(new PlanTrace("check-filter", "device-control",
                "set-temperature", "SUCCESS", 1, Map.of("target", 22)))
        );

        String prompt = AiResolutionPromptBuilder.build(
            context, List.of(suggestion), Set.of("SET_TEMPERATURE"));

        assertThat(prompt).contains("0.92");
        assertThat(prompt).contains("Replaced blocked HVAC filter");
        assertThat(prompt).contains("RESOLVED");
        assertThat(prompt).contains("set-temperature");
    }

    @Test
    void buildHandlesEmptySuggestions() {
        Map<String, Object> context = Map.of("deviceClass", "thermostat");

        String prompt = AiResolutionPromptBuilder.build(
            context, List.of(), Set.of("SET_TEMPERATURE"));

        assertThat(prompt).contains("No similar past cases found");
    }

    @Test
    void buildHandlesEmptyAvailableActions() {
        Map<String, Object> context = Map.of("deviceClass", "thermostat");

        String prompt = AiResolutionPromptBuilder.build(
            context, List.of(), Set.of());

        assertThat(prompt).contains("No autonomous actions available");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/webapp-api/pom.xml test -Dtest=AiResolutionPromptBuilderTest`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement AiResolutionPromptBuilder**

```java
package io.casehub.iot.webapp.resolution;

import io.casehub.iot.webapp.cbr.ResolutionSuggestion;
import io.casehub.neocortex.memory.cbr.PlanTrace;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public final class AiResolutionPromptBuilder {

    private AiResolutionPromptBuilder() {}

    public static String build(Map<String, Object> caseContext,
                               List<ResolutionSuggestion> suggestions,
                               Set<String> availableActions) {
        var sb = new StringBuilder();

        sb.append("## Current Situation\n\n");
        caseContext.forEach((key, value) ->
            sb.append("- **").append(key).append(":** ").append(value).append("\n"));

        sb.append("\n## Past Resolutions\n\n");
        if (suggestions.isEmpty()) {
            sb.append("No similar past cases found.\n");
        } else {
            for (int i = 0; i < suggestions.size(); i++) {
                var s = suggestions.get(i);
                sb.append("### Match ").append(i + 1)
                  .append(" (similarity: ").append(s.similarityScore()).append(")\n");
                sb.append("- **Problem:** ").append(s.problem()).append("\n");
                sb.append("- **Solution:** ").append(s.solution()).append("\n");
                sb.append("- **Outcome:** ").append(s.outcome()).append("\n");
                if (s.confidence() != null) {
                    sb.append("- **Confidence:** ").append(s.confidence()).append("\n");
                }
                if (!s.planSteps().isEmpty()) {
                    sb.append("- **Plan steps:**\n");
                    for (PlanTrace step : s.planSteps()) {
                        sb.append("  - ").append(step.workerName())
                          .append(" (").append(step.capabilityName()).append(")")
                          .append(" → ").append(step.stepOutcome()).append("\n");
                    }
                }
                sb.append("\n");
            }
        }

        sb.append("## Available Actions\n\n");
        if (availableActions.isEmpty()) {
            sb.append("No autonomous actions available. You must ESCALATE.\n");
        } else {
            sb.append("You may use these action types: ");
            sb.append(availableActions.stream().sorted().collect(Collectors.joining(", ")));
            sb.append("\n");
        }

        sb.append("\n## Instructions\n\n");
        sb.append("Decide whether to EXECUTE a resolution plan or ESCALATE to a human operator.\n");
        sb.append("If you have high confidence based on past resolutions, produce a plan.\n");
        sb.append("If the situation is novel or risky, ESCALATE.\n");

        return sb.toString();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/webapp-api/pom.xml test -Dtest=AiResolutionPromptBuilderTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add webapp-api/src/main/java/io/casehub/iot/webapp/resolution/AiResolutionPromptBuilder.java webapp-api/src/test/java/io/casehub/iot/webapp/resolution/AiResolutionPromptBuilderTest.java
git commit -m "feat(#63): AiResolutionPromptBuilder — constructs LLM prompt from case context, CBR suggestions, and available actions"
```

---

### Task 4: IoTAiResolutionConfig

`@ConfigMapping` interface for the agent configuration.

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionConfig.java`

**Interfaces:**
- Consumes: nothing
- Produces: `IoTAiResolutionConfig` — injected by `IoTAiResolutionAgent` (Task 5)

- [ ] **Step 1: Create IoTAiResolutionConfig**

```java
package io.casehub.iot.webapp.app.resolution;

import io.casehub.api.model.ai.ModelType;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;

import java.time.Duration;

@ConfigMapping(prefix = "casehub.iot.ai-resolution")
public interface IoTAiResolutionConfig {

    @WithDefault("true")
    boolean enabled();

    @WithName("poll-interval")
    @WithDefault("10s")
    Duration pollInterval();

    @WithName("timeout-seconds")
    @WithDefault("300")
    int timeoutSeconds();

    @WithName("model-type")
    @WithDefault("ANTHROPIC")
    ModelType modelType();

    @WithName("agent-id")
    @WithDefault("iot-ai-agent")
    String agentId();

    @WithName("max-concurrent-llm-calls")
    @WithDefault("3")
    int maxConcurrentLlmCalls();
}
```

- [ ] **Step 2: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionConfig.java
git commit -m "feat(#63): IoTAiResolutionConfig — @ConfigMapping for AI resolution agent"
```

---

### Task 5: IoTAiResolutionAgent — core agent with full test suite

The CDI bean that polls, claims, calls LLM, risk-classifies, executes, and
escalates. This is the main deliverable.

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java`

**Interfaces:**
- Consumes:
  - `IoTAiResolutionConfig` (Task 4)
  - `AiResolutionPlan`, `PlannedActionSpec`, `ExecutedActionResult`, `AiEscalationContext`, `Decision` (Task 2)
  - `AiResolutionPromptBuilder.build()` (Task 3)
  - `CaseQueueService.findPending(UUID, String): List<CaseQueueEntry>` (engine)
  - `CaseQueueService.findByView(UUID, String): List<CaseQueueEntry>` (Task 1)
  - `CaseQueueService.claim(UUID, String, String): CaseQueueEntry` (engine)
  - `CaseQueueService.escalate(UUID, String, UUID): CaseQueueEntry` (engine)
  - `CaseQueueEntryStore.findById(UUID): Optional<CaseQueueEntry>` (engine)
  - `CaseInstanceCache.get(UUID): CaseInstance` (engine)
  - `IoTCbrRetrievalService.retrieve(CbrConfig, Map, String): List<ResolutionSuggestion>` (existing)
  - `IoTActionRiskClassifier.classify(PlannedAction, ClassificationContext): RiskDecision` (existing)
  - `DeviceCommandWorkerFunction.apply(Map): WorkerResult<Map>` (existing)
  - `Agent.execute(Map): WorkerResult<Map>` (engine langchain4j)
  - `SubjectViewStore.findByTenancy(String): List<SubjectViewSpec>` (platform)
  - `CaseDefinitionRegistry.findByName(String): Optional<CaseDefinition>` (engine)
- Produces: `IoTAiResolutionAgent` — standalone, no downstream consumers

- [ ] **Step 1: Write test scaffolding and helper**

```java
package io.casehub.iot.webapp.app.resolution;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.context.CaseContext;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ai.Agent;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.ClassificationContext;
import io.casehub.api.spi.RiskDecision;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.engine.queue.model.CaseQueueEntry;
import io.casehub.engine.queue.model.QueueEntryStatus;
import io.casehub.engine.queue.service.CaseQueueService;
import io.casehub.engine.queue.spi.CaseQueueEntryStore;
import io.casehub.iot.webapp.cbr.IoTCbrRetrievalService;
import io.casehub.iot.webapp.cbr.ResolutionSuggestion;
import io.casehub.iot.webapp.resolution.AiResolutionPlan;
import io.casehub.iot.webapp.resolution.Decision;
import io.casehub.iot.webapp.resolution.PlannedActionSpec;
import io.casehub.platform.api.view.SubjectViewSpec;
import io.casehub.platform.api.view.SubjectViewStore;
import io.casehub.worker.api.PlannedAction;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Field;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class IoTAiResolutionAgentTest {

    private static final String TENANCY = "test-tenant";
    private static final UUID AI_VIEW_ID = UUID.randomUUID();
    private static final UUID OPERATOR_VIEW_ID = UUID.randomUUID();
    private static final ObjectMapper objectMapper = new ObjectMapper();

    private CaseQueueService queueService;
    private CaseQueueEntryStore entryStore;
    private CaseInstanceCache caseCache;
    private IoTCbrRetrievalService retrievalService;
    private ActionRiskClassifier riskClassifier;
    private CaseDefinitionRegistry definitionRegistry;
    private SubjectViewStore viewStore;
    private Agent llmAgent;
    private Function<Map<String, Object>, WorkerResult<Map<String, Object>>> deviceCommandFn;
    private IoTAiResolutionAgent agent;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() throws Exception {
        queueService = mock(CaseQueueService.class);
        entryStore = mock(CaseQueueEntryStore.class);
        caseCache = mock(CaseInstanceCache.class);
        retrievalService = mock(IoTCbrRetrievalService.class);
        riskClassifier = mock(ActionRiskClassifier.class);
        definitionRegistry = mock(CaseDefinitionRegistry.class);
        viewStore = mock(SubjectViewStore.class);
        llmAgent = mock(Agent.class);
        deviceCommandFn = mock(Function.class);

        IoTAiResolutionConfig config = mock(IoTAiResolutionConfig.class);
        when(config.enabled()).thenReturn(true);
        when(config.timeoutSeconds()).thenReturn(300);
        when(config.agentId()).thenReturn("iot-ai-agent");
        when(config.maxConcurrentLlmCalls()).thenReturn(3);

        when(viewStore.findByTenancy(TENANCY)).thenReturn(List.of(
            new SubjectViewSpec(AI_VIEW_ID, "iot-ai-resolution", TENANCY,
                "iot-triage:ai-resolution", null, "enqueuedAt", "ASC", null, Instant.now()),
            new SubjectViewSpec(OPERATOR_VIEW_ID, "iot-operator-assisted", TENANCY,
                "iot-triage:operator-assisted", null, "enqueuedAt", "ASC", null, Instant.now())
        ));

        agent = new IoTAiResolutionAgent();
        inject(agent, "queueService", queueService);
        inject(agent, "entryStore", entryStore);
        inject(agent, "caseCache", caseCache);
        inject(agent, "retrievalService", retrievalService);
        inject(agent, "riskClassifier", riskClassifier);
        inject(agent, "definitionRegistry", definitionRegistry);
        inject(agent, "viewStore", viewStore);
        inject(agent, "config", config);
        inject(agent, "tenancyId", TENANCY);
        inject(agent, "llmAgent", llmAgent);
        inject(agent, "deviceCommandFn", deviceCommandFn);
        inject(agent, "objectMapper", objectMapper);

        agent.init();
    }

    private static void inject(Object target, String fieldName, Object value) throws Exception {
        Field f = target.getClass().getDeclaredField(fieldName);
        f.setAccessible(true);
        f.set(target, value);
    }

    private CaseQueueEntry pendingEntry(UUID caseId) {
        return new CaseQueueEntry(UUID.randomUUID(), caseId, TENANCY,
            AI_VIEW_ID, "iot-ai-resolution", QueueEntryStatus.PENDING, Instant.now());
    }

    private CaseQueueEntry claimedEntry(UUID caseId, UUID entryId, Instant claimedAt) {
        CaseQueueEntry e = new CaseQueueEntry(entryId, caseId, TENANCY,
            AI_VIEW_ID, "iot-ai-resolution", QueueEntryStatus.CLAIMED, Instant.now());
        e.setAssignedTo("iot-ai-agent");
        e.setClaimedAt(claimedAt);
        return e;
    }

    private CaseInstance caseInstance(UUID caseId, String caseType) {
        CaseInstance instance = new CaseInstance();
        instance.setUuid(caseId);
        CaseMetaModel meta = new CaseMetaModel();
        meta.setName(caseType);
        instance.setCaseMetaModel(meta);
        CaseContext ctx = mock(CaseContext.class);
        when(ctx.getOrDefault(any(), any())).thenReturn(Map.of());
        instance.setCaseContext(ctx);
        return instance;
    }

    private WorkerResult<Map<String, Object>> llmExecuteResult() {
        Map<String, Object> planMap = Map.of(
            "decision", "EXECUTE",
            "reasoning", "High similarity match",
            "actions", List.of(Map.of(
                "actionType", "SET_TEMPERATURE",
                "targetDeviceId", "thermo-001",
                "parameters", Map.of("target", 22),
                "rationale", "Reset temperature"
            )),
            "escalationReason", ""
        );
        return WorkerResult.of(planMap);
    }

    private WorkerResult<Map<String, Object>> llmEscalateResult() {
        Map<String, Object> planMap = Map.of(
            "decision", "ESCALATE",
            "reasoning", "Novel situation",
            "actions", List.of(),
            "escalationReason", "No matching pattern"
        );
        return WorkerResult.of(planMap);
    }
}
```

- [ ] **Step 2: Write happy path test**

```java
@Test
void happyPath_claimsExecutesAndWritesResults() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId);
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly");

    CaseDefinition def = mock(CaseDefinition.class);
    CbrConfig cbrConfig = CbrConfig.builder().domain("iot").caseType("hvac-anomaly").build();
    when(def.getCbrConfig()).thenReturn(cbrConfig);

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.claim(entry.getId(), TENANCY, "iot-ai-agent")).thenReturn(entry);
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));
    when(retrievalService.retrieve(any(), any(), eq(TENANCY))).thenReturn(List.of());
    when(llmAgent.execute(any())).thenReturn(llmExecuteResult());
    when(riskClassifier.classify(any(), any())).thenReturn(new RiskDecision.Autonomous());
    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(entry));
    when(deviceCommandFn.apply(any())).thenReturn(WorkerResult.of(Map.of("result", "SUCCESS")));

    // Timeout sweep returns empty — no stale claims
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());

    agent.poll();

    verify(queueService).claim(entry.getId(), TENANCY, "iot-ai-agent");
    verify(llmAgent).execute(any());
    verify(deviceCommandFn).apply(any());
    verify(instance.getCaseContext()).set(eq("aiResolutionResults"), any());
}
```

- [ ] **Step 3: Write escalation test — LLM decides ESCALATE**

```java
@Test
void llmDecidesToEscalate_movesEntryToOperatorAssisted() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId);
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly");

    CaseDefinition def = mock(CaseDefinition.class);
    when(def.getCbrConfig()).thenReturn(CbrConfig.builder().domain("iot").caseType("hvac-anomaly").build());

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.claim(entry.getId(), TENANCY, "iot-ai-agent")).thenReturn(entry);
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));
    when(retrievalService.retrieve(any(), any(), eq(TENANCY))).thenReturn(List.of());
    when(llmAgent.execute(any())).thenReturn(llmEscalateResult());
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());

    agent.poll();

    verify(queueService).escalate(entry.getId(), TENANCY, OPERATOR_VIEW_ID);
    verify(deviceCommandFn, never()).apply(any());
}
```

- [ ] **Step 4: Write risk gate test**

```java
@Test
void riskGateTriggersEscalation_whenAnyActionGateRequired() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId);
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly");

    CaseDefinition def = mock(CaseDefinition.class);
    when(def.getCbrConfig()).thenReturn(CbrConfig.builder().domain("iot").caseType("hvac-anomaly").build());

    // LLM returns EXECUTE with a LOCK action
    Map<String, Object> planMap = Map.of(
        "decision", "EXECUTE",
        "reasoning", "Lock the door",
        "actions", List.of(Map.of(
            "actionType", "LOCK",
            "targetDeviceId", "lock-001",
            "parameters", Map.of(),
            "rationale", "Secure entry"
        )),
        "escalationReason", ""
    );

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.claim(entry.getId(), TENANCY, "iot-ai-agent")).thenReturn(entry);
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));
    when(retrievalService.retrieve(any(), any(), eq(TENANCY))).thenReturn(List.of());
    when(llmAgent.execute(any())).thenReturn(WorkerResult.of(planMap));
    when(riskClassifier.classify(any(), any()))
        .thenReturn(new RiskDecision.GateRequired("Lock requires approval", true, null, null, null, null));
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());

    agent.poll();

    verify(queueService).escalate(entry.getId(), TENANCY, OPERATOR_VIEW_ID);
    verify(deviceCommandFn, never()).apply(any());
}
```

- [ ] **Step 5: Write timeout sweep test**

```java
@Test
void timeoutSweep_escalatesStaleClaimedEntries() {
    UUID caseId = UUID.randomUUID();
    UUID entryId = UUID.randomUUID();
    Instant staleClaimTime = Instant.now().minusSeconds(600); // well past 300s timeout
    CaseQueueEntry staleEntry = claimedEntry(caseId, entryId, staleClaimTime);

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of());
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of(staleEntry));

    agent.poll();

    verify(queueService).escalate(entryId, TENANCY, OPERATOR_VIEW_ID);
}
```

- [ ] **Step 6: Write status guard race condition test**

```java
@Test
void statusGuard_abortsWhenEntryEscalatedBetweenLlmAndExecution() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId);
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly");

    CaseDefinition def = mock(CaseDefinition.class);
    when(def.getCbrConfig()).thenReturn(CbrConfig.builder().domain("iot").caseType("hvac-anomaly").build());

    // Entry was escalated by timeout sweep between LLM and execution
    CaseQueueEntry movedEntry = new CaseQueueEntry(entry.getId(), caseId, TENANCY,
        OPERATOR_VIEW_ID, "iot-operator-assisted", QueueEntryStatus.PENDING, Instant.now());

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.claim(entry.getId(), TENANCY, "iot-ai-agent")).thenReturn(entry);
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));
    when(retrievalService.retrieve(any(), any(), eq(TENANCY))).thenReturn(List.of());
    when(llmAgent.execute(any())).thenReturn(llmExecuteResult());
    when(riskClassifier.classify(any(), any())).thenReturn(new RiskDecision.Autonomous());
    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(movedEntry));
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());

    agent.poll();

    verify(deviceCommandFn, never()).apply(any());
    verify(queueService, never()).escalate(any(), any(), any());
}
```

- [ ] **Step 7: Write partial worker failure test**

```java
@Test
void partialWorkerFailure_escalatesWithExecutedActionRecord() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId);
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly");

    CaseDefinition def = mock(CaseDefinition.class);
    when(def.getCbrConfig()).thenReturn(CbrConfig.builder().domain("iot").caseType("hvac-anomaly").build());

    // Two-action plan
    Map<String, Object> planMap = Map.of(
        "decision", "EXECUTE",
        "reasoning", "Two-step fix",
        "actions", List.of(
            Map.of("actionType", "TURN_OFF", "targetDeviceId", "dev-1",
                   "parameters", Map.of(), "rationale", "Power cycle"),
            Map.of("actionType", "SET_TEMPERATURE", "targetDeviceId", "dev-2",
                   "parameters", Map.of("target", 20), "rationale", "Reset")
        ),
        "escalationReason", ""
    );

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.claim(entry.getId(), TENANCY, "iot-ai-agent")).thenReturn(entry);
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));
    when(retrievalService.retrieve(any(), any(), eq(TENANCY))).thenReturn(List.of());
    when(llmAgent.execute(any())).thenReturn(WorkerResult.of(planMap));
    when(riskClassifier.classify(any(), any())).thenReturn(new RiskDecision.Autonomous());
    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(entry));
    // First action succeeds, second fails
    when(deviceCommandFn.apply(any()))
        .thenReturn(WorkerResult.of(Map.of("result", "SUCCESS")))
        .thenReturn(WorkerResult.failed("Device unreachable"));
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());

    agent.poll();

    verify(queueService).escalate(entry.getId(), TENANCY, OPERATOR_VIEW_ID);
    verify(deviceCommandFn, times(2)).apply(any());
    verify(instance.getCaseContext()).set(eq("aiEscalationContext"), any());
}
```

- [ ] **Step 8: Write disabled test**

```java
@Test
void disabled_noProcessing() throws Exception {
    IoTAiResolutionConfig disabledConfig = mock(IoTAiResolutionConfig.class);
    when(disabledConfig.enabled()).thenReturn(false);
    inject(agent, "config", disabledConfig);

    agent.poll();

    verifyNoInteractions(queueService);
}
```

- [ ] **Step 9: Run all tests to verify they fail**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/webapp/pom.xml test -Dtest=IoTAiResolutionAgentTest`
Expected: FAIL — `IoTAiResolutionAgent` does not exist

- [ ] **Step 10: Implement IoTAiResolutionAgent**

Create `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java`.

The implementation must:
1. `@PostConstruct init()` — resolve view IDs from `SubjectViewStore`, build semaphore
2. `poll()` — guarded by `config.enabled()`. Calls `processNewEntries()` then `sweepStaleEntries()`
3. `processNewEntries()` — `findPending`, submit each to virtual thread with semaphore
4. `processEntry(CaseQueueEntry)` — claim, load case, retrieve CBR, write pre-LLM context, call LLM, risk-classify, status-guard, execute/escalate
5. `sweepStaleEntries()` — `findByView`, filter CLAIMED past timeout, escalate each
6. `resolveWorker(String actionType)` — switch on action type

Key implementation details:
- `@Inject @VirtualThreads ExecutorService virtualThreads` for offloading
- `Semaphore llmSemaphore` initialized from `config.maxConcurrentLlmCalls()`
- Retry logic: catch `AgentException` → escalate immediately; catch other `Exception` → classify as transient, retry up to 2x with backoff
- Status guard: re-read entry from `entryStore.findById()` before execution, verify `status == CLAIMED && viewId == aiResolutionViewId`
- Write `AiEscalationContext` to `caseContext.set("aiEscalationContext", ...)` pre-LLM
- Write results to `caseContext.set("aiResolutionResults", ...)` on success

- [ ] **Step 11: Run all tests**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/webapp/pom.xml test -Dtest=IoTAiResolutionAgentTest`
Expected: PASS — all 7 tests green

- [ ] **Step 12: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java
git commit -m "feat(#63): IoTAiResolutionAgent — LLM resolution agent with polling, claim, execution, escalation, and timeout sweep"
```

---

### Task 6: Full build verification

**Files:** None — verification only.

- [ ] **Step 1: Run full webapp-api test suite**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/webapp-api/pom.xml test`
Expected: All existing + new tests PASS

- [ ] **Step 2: Run full webapp test suite**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/webapp/pom.xml test`
Expected: All existing + new tests PASS

- [ ] **Step 3: Run full project build**

Run: `mvn -f /Users/mdproctor/claude/casehub/iot/pom.xml install`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run IDE diagnostics on new files**

Check all new files for compilation errors via `ide_diagnostics`.

---
