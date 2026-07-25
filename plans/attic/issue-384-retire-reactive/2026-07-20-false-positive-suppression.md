# False-Positive Suppression via CBR Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #52 — feat: false-positive suppression via CBR
**Issue group:** #52

**Goal:** Build a CBR-based feedback loop from operator dismissals back to the situation trigger pipeline. When a new situation fires and matches a historically-dismissed pattern, auto-downgrade severity or suppress the notification using a graduated three-tier model.

**Architecture:** Extends the ras `RasTriggerPolicy` SPI with `PolicyDecision` (metadata on decisions) and `TriggerDecision.SUPPRESS`. IoT implements a custom `IoTSuppressionTriggerPolicy` that queries the CBR case base for similar dismissed patterns before returning TRIGGER. The system applies graduated suppression: annotate (tier 1), demote to notify-only (tier 2), or full suppress (tier 3). Operator dismissals and case false-positives feed back as CBR evidence. Safety-critical situations are never suppressed.

**Tech Stack:** Java 21, Quarkus CDI, neocortex CBR (`CbrCaseMemoryStore`, `FeatureVectorCbrCase`, `CbrFeatureSchema`), casehub-ras-api (`RasTriggerPolicy`, `SituationEvaluator`), Flyway (PostgreSQL with JSONB)

**Cross-repo:** Tasks 1–2 modify `casehubio/ras` (api + runtime). A ras issue must be created first. After ras changes are installed to local Maven, Tasks 3–7 implement the IoT feature.

## Global Constraints

- `webapp-api` is Tier 1 — pure Java, no CDI annotations, no Quarkus runtime dependencies
- `webapp` is the CDI wiring module — `@ApplicationScoped`, `@ConfigMapping`, producers, JPA entities
- Safety-critical situations (caseName in `IoTSafetyCaseTypes.SAFETY_CASE_TYPES` or situationId in `SAFETY_SITUATION_IDS`) are never suppressed
- CBR case type partition: `"iot-dismissal:" + situationId` — hard partition by situation type
- CBR case outcomes: `"dismissed"`, `"actioned"`, `"override-actioned"`
- System suppression decisions never enter the CBR case base — only operator evidence
- Suppression config prefix: `casehub.iot.suppression`
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing
- ras changes require a dedicated issue in `casehubio/ras`
- Flyway migration version: V503 (iot-webapp datasource)
- `DefaultRasTriggerPolicy` is instantiated as a plain object by IoT's policy (not CDI-injected)

---

### Task 0: Prerequisites — ras issue + workspace setup

**Files:**
- None (setup only)

- [ ] **Step 1: Create ras issue**

```bash
gh issue create --repo casehubio/ras \
  --title "feat: PolicyDecision metadata + TriggerDecision.SUPPRESS + SituationChangeEvent enhancements" \
  --body "..."
```

Issue body covers: `PolicyDecision` record, `SuppressionMetadataKeys`, `TriggerDecision.SUPPRESS`, `SituationChangeEvent` metadata field + `SUPPRESSED`/`DISMISSED` change types, `RasTriggerPolicy` return type change.

- [ ] **Step 2: Create ras branch**

```bash
git -C /Users/mdproctor/claude/casehub/ras checkout main
git -C /Users/mdproctor/claude/casehub/ras pull
git -C /Users/mdproctor/claude/casehub/ras checkout -b issue-N-policy-decision-suppress
```

- [ ] **Step 3: Open IntelliJ workspace**

```
ide_open_workspace(modules: ["/Users/mdproctor/claude/casehub/ras", "/Users/mdproctor/claude/casehub/iot"])
```

Use the returned project_path for all subsequent `mcp__intellij-index__*` calls.

---

### Task 1: ras-api — PolicyDecision, SUPPRESS, SituationChangeEvent metadata

All changes in `casehub-ras-api`. This task introduces the new API types that downstream consumers (IoT's suppression policy) will implement against.

**Files:**
- Create: `ras/api/src/main/java/io/casehub/ras/api/PolicyDecision.java`
- Create: `ras/api/src/main/java/io/casehub/ras/api/SuppressionMetadataKeys.java`
- Modify: `ras/api/src/main/java/io/casehub/ras/api/TriggerDecision.java` — add SUPPRESS
- Modify: `ras/api/src/main/java/io/casehub/ras/api/SituationChangeEvent.java` — add metadata field, SUPPRESSED/DISMISSED to ChangeType
- Modify: `ras/api/src/main/java/io/casehub/ras/api/RasTriggerPolicy.java` — return type change
- Test: `ras/api/src/test/java/io/casehub/ras/api/PolicyDecisionTest.java`
- Test: `ras/api/src/test/java/io/casehub/ras/api/SituationChangeEventTest.java` (update existing)

**Interfaces:**
- Produces:
  - `PolicyDecision(TriggerDecision decision, Map<String, Object> metadata)` — record
  - `PolicyDecision(TriggerDecision decision)` — convenience constructor, `metadata = Map.of()`
  - `SuppressionMetadataKeys.TIER`, `.DISMISSAL_RATE`, `.MATCH_COUNT`, `.AVERAGE_SIMILARITY`
  - `TriggerDecision.SUPPRESS` — new enum constant
  - `SituationChangeEvent.ChangeType.SUPPRESSED`, `.DISMISSED` — new enum constants
  - `SituationChangeEvent(tenancyId, situationId, correlationKey, changeType, context, metadata)` — 6-arg constructor
  - `RasTriggerPolicy.evaluate()` returns `Uni<PolicyDecision>` (was `Uni<TriggerDecision>`)

- [ ] **Step 1: Write failing test for PolicyDecision**

Create `PolicyDecisionTest.java`:

```java
package io.casehub.ras.api;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class PolicyDecisionTest {

    @Test
    void convenienceConstructor_emptyMetadata() {
        var pd = new PolicyDecision(TriggerDecision.TRIGGER);
        assertThat(pd.decision()).isEqualTo(TriggerDecision.TRIGGER);
        assertThat(pd.metadata()).isEmpty();
    }

    @Test
    void fullConstructor_preservesMetadata() {
        var meta = Map.<String, Object>of("key", "value");
        var pd = new PolicyDecision(TriggerDecision.SUPPRESS, meta);
        assertThat(pd.decision()).isEqualTo(TriggerDecision.SUPPRESS);
        assertThat(pd.metadata()).containsEntry("key", "value");
    }

    @Test
    void nullDecision_throws() {
        assertThatNullPointerException()
                .isThrownBy(() -> new PolicyDecision(null, Map.of()));
    }

    @Test
    void nullMetadata_throws() {
        assertThatNullPointerException()
                .isThrownBy(() -> new PolicyDecision(TriggerDecision.TRIGGER, null));
    }
}
```

- [ ] **Step 2: Run test to verify it fails** (PolicyDecision does not exist yet)

```bash
mvn -C /Users/mdproctor/claude/casehub/ras/api test -pl api -Dtest=PolicyDecisionTest -DfailIfNoTests=false
```

- [ ] **Step 3: Create PolicyDecision record**

```java
package io.casehub.ras.api;

import java.util.Map;
import java.util.Objects;

public record PolicyDecision(TriggerDecision decision, Map<String, Object> metadata) {
    public PolicyDecision {
        Objects.requireNonNull(decision, "decision");
        Objects.requireNonNull(metadata, "metadata");
        metadata = Map.copyOf(metadata);
    }

    public PolicyDecision(TriggerDecision decision) {
        this(decision, Map.of());
    }
}
```

- [ ] **Step 4: Create SuppressionMetadataKeys**

```java
package io.casehub.ras.api;

public final class SuppressionMetadataKeys {
    public static final String TIER = "suppression.tier";
    public static final String DISMISSAL_RATE = "suppression.dismissalRate";
    public static final String MATCH_COUNT = "suppression.matchCount";
    public static final String AVERAGE_SIMILARITY = "suppression.averageSimilarity";
    private SuppressionMetadataKeys() {}
}
```

- [ ] **Step 5: Add SUPPRESS to TriggerDecision**

Add `SUPPRESS` after `RESOLVE` in the enum.

- [ ] **Step 6: Update SituationChangeEvent — add metadata + SUPPRESSED + DISMISSED**

Modify the record to add 6-arg constructor with `Map<String, Object> metadata`:

```java
public record SituationChangeEvent(
        String tenancyId, String situationId, String correlationKey,
        ChangeType changeType, SituationContext context,
        Map<String, Object> metadata) {

    public SituationChangeEvent {
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(situationId, "situationId");
        Objects.requireNonNull(correlationKey, "correlationKey");
        Objects.requireNonNull(changeType, "changeType");
        Objects.requireNonNull(context, "context");
        Objects.requireNonNull(metadata, "metadata");
    }

    public SituationChangeEvent(String tenancyId, String situationId,
            String correlationKey, ChangeType changeType, SituationContext context) {
        this(tenancyId, situationId, correlationKey, changeType, context, Map.of());
    }

    public enum ChangeType {
        TRIGGERED,
        RESOLVED,
        DISCARDED,
        SUPPRESSED,
        DISMISSED
    }
}
```

- [ ] **Step 7: Update RasTriggerPolicy return type**

```java
public interface RasTriggerPolicy {
    Uni<PolicyDecision> evaluate(SituationContext context, SituationDefinition definition);
}
```

- [ ] **Step 8: Update SituationChangeEventTest**

Add tests for: 6-arg constructor with metadata, SUPPRESSED/DISMISSED change types, null metadata rejection, backward-compat 5-arg constructor defaults metadata to `Map.of()`.

- [ ] **Step 9: Run all api tests**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/ras/api/pom.xml test
```

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add api/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat: PolicyDecision, SUPPRESS, SituationChangeEvent metadata (ras#N)"
```

---

### Task 2: ras runtime — SituationEvaluator SUPPRESS + metadata, DefaultRasTriggerPolicy, RasMetrics

Updates ras runtime to handle the new API types. The `SituationEvaluator` gets a SUPPRESS arm in its decision switch and metadata threading through all trigger paths. `DefaultRasTriggerPolicy` wraps returns in `PolicyDecision`.

**Files:**
- Modify: `ras/runtime/src/main/java/io/casehub/ras/runtime/SituationEvaluator.java`
- Modify: `ras/runtime/src/main/java/io/casehub/ras/runtime/DefaultRasTriggerPolicy.java`
- Modify: `ras/runtime/src/main/java/io/casehub/ras/runtime/RasMetrics.java`
- Modify: `ras/runtime/src/test/java/io/casehub/ras/runtime/DefaultRasTriggerPolicyTest.java`
- Modify: `ras/runtime/src/test/java/io/casehub/ras/runtime/SituationEvaluatorTest.java`
- Modify: `ras/runtime/src/test/java/io/casehub/ras/runtime/RasMetricsTest.java`

**Interfaces:**
- Consumes: `PolicyDecision`, `TriggerDecision.SUPPRESS`, `SituationChangeEvent.ChangeType.SUPPRESSED`
- Produces: Updated `SituationEvaluator` that handles SUPPRESS and merges metadata into `baseCaseData`

- [ ] **Step 1: Write failing test for SUPPRESS handling in SituationEvaluator**

Add to `SituationEvaluatorTest.java`:

```java
@Test
void suppressDecisionRemovesContextAndFiresSuppressedEvent() {
    var ganglion = new MockGanglion("g1", Set.of("temp.reading"),
            FixedDetectionResult.detected("g1", 0.9));
    var def = new SituationDefinition("sit-1", Set.of("temp.reading"),
            Duration.ofMinutes(5), null, new ChainMode.Or(Set.of("g1")),
            new TriggerAction.CreateCase(TRIGGER_CONFIG), null);

    // Policy that always returns SUPPRESS
    var suppressPolicy = new RasTriggerPolicy() {
        @Override
        public Uni<PolicyDecision> evaluate(SituationContext ctx, SituationDefinition d) {
            return Uni.createFrom().item(new PolicyDecision(TriggerDecision.SUPPRESS,
                    Map.of("suppression.tier", "full", "suppression.dismissalRate", 0.92)));
        }
    };

    var registry = new SituationDefinitionRegistry(
            List.of(() -> List.of(new SituationRegistration(def))), List.of(ganglion));
    initMetrics(registry);
    evaluator = new SituationEvaluator(store, suppressPolicy, caseTrigger, registry,
            3, changeEvent, metrics);

    evaluator.evaluate(event("temp.reading", T1), def, "key-1", "tenant-a");

    assertThat(caseTrigger.firedCases()).isEmpty();
    assertThat(store.find("sit-1", "key-1", "tenant-a").await().indefinitely()).isEmpty();
    assertThat(changeEvent.events()).hasSize(1);
    assertThat(changeEvent.events().get(0).changeType())
            .isEqualTo(SituationChangeEvent.ChangeType.SUPPRESSED);
    assertThat(changeEvent.events().get(0).metadata())
            .containsEntry("suppression.tier", "full");
}
```

- [ ] **Step 2: Write failing test for metadata merge into baseCaseData**

```java
@Test
void triggerWithMetadataMergesIntoCaseData() {
    var ganglion = new MockGanglion("g1", Set.of("temp.reading"),
            FixedDetectionResult.detected("g1", 0.9));
    var configWithData = new CaseTriggerConfig("ns", "case", "1.0",
            Map.of("existing", "value"));
    var def = new SituationDefinition("sit-1", Set.of("temp.reading"),
            Duration.ofMinutes(5), null, new ChainMode.Or(Set.of("g1")),
            new TriggerAction.CreateCase(configWithData), null);

    var annotatePolicy = new RasTriggerPolicy() {
        @Override
        public Uni<PolicyDecision> evaluate(SituationContext ctx, SituationDefinition d) {
            return Uni.createFrom().item(new PolicyDecision(TriggerDecision.TRIGGER,
                    Map.of("suppression.tier", "annotate", "suppression.dismissalRate", 0.45)));
        }
    };

    var registry = new SituationDefinitionRegistry(
            List.of(() -> List.of(new SituationRegistration(def))), List.of(ganglion));
    initMetrics(registry);
    evaluator = new SituationEvaluator(store, annotatePolicy, caseTrigger, registry,
            3, changeEvent, metrics);

    evaluator.evaluate(event("temp.reading", T1), def, "key-1", "tenant-a");

    assertThat(caseTrigger.firedCases()).hasSize(1);
    var firedConfig = caseTrigger.firedCases().get(0).config();
    assertThat(firedConfig.baseCaseData()).containsEntry("existing", "value");
    assertThat(firedConfig.baseCaseData()).containsEntry("suppression.tier", "annotate");
    assertThat(firedConfig.baseCaseData()).containsEntry("suppression.dismissalRate", 0.45);
}
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/ras/runtime/pom.xml test -Dtest=SituationEvaluatorTest -DfailIfNoTests=false
```

- [ ] **Step 4: Update DefaultRasTriggerPolicy — return PolicyDecision**

Change all return statements from `Uni.createFrom().item(TriggerDecision.X)` to `Uni.createFrom().item(new PolicyDecision(TriggerDecision.X))`.

- [ ] **Step 5: Update SituationEvaluator — handle PolicyDecision + SUPPRESS**

In `processEvent()`:
1. Change `TriggerDecision decision = triggerPolicy.evaluate(...)` to `PolicyDecision policyDecision = triggerPolicy.evaluate(...)`
2. Extract `policyDecision.decision()` for the switch
3. Thread `policyDecision.metadata()` into `executeDecision()`

In `executeDecision()`:
1. Add `Map<String, Object> policyMetadata` parameter
2. For TRIGGER/TRIGGER_AND_CONTINUE with CreateCase: merge metadata into `baseCaseData` before `caseTrigger.fire()`
3. For TRIGGER/TRIGGER_AND_CONTINUE with NotifyOnly: pass metadata on `SituationChangeEvent`
4. Add SUPPRESS case:
```java
case SUPPRESS:
    this.closeGanglia(definition, situationId, correlationKey, tenancyId);
    this.store.remove(situationId, correlationKey, tenancyId).await().indefinitely();
    this.changeEvent.fireAsync(new SituationChangeEvent(
            tenancyId, situationId, correlationKey,
            ChangeType.SUPPRESSED, context, policyMetadata));
    this.metrics.situationSuppressed(situationId, tenancyId);
    return true;
```

- [ ] **Step 6: Add RasMetrics.situationSuppressed()**

```java
public void situationSuppressed(String situationId, String tenancyId) {
    registry.counter("ras.engine.situations.suppressed",
            "situation_id", situationId, "tenancy_id", tenancyId).increment();
}
```

- [ ] **Step 7: Update DefaultRasTriggerPolicyTest — assert PolicyDecision return type**

Change all assertions from `assertThat(result).isEqualTo(TriggerDecision.X)` to `assertThat(result.decision()).isEqualTo(TriggerDecision.X)` and add `assertThat(result.metadata()).isEmpty()`.

- [ ] **Step 8: Update TestChangeEvent if needed**

The `TestChangeEvent` mock in SituationEvaluatorTest may need updating to handle the new `SituationChangeEvent` constructor. Check if it stores events and verify the metadata is accessible.

- [ ] **Step 9: Add RasMetrics test for suppression counter**

```java
@Test
void situationSuppressedIncrementsCounter() {
    metrics.situationSuppressed("sit-1", "tenant-a");
    metrics.situationSuppressed("sit-1", "tenant-a");

    assertThat(meterRegistry.counter("ras.engine.situations.suppressed",
            "situation_id", "sit-1", "tenancy_id", "tenant-a").count()).isEqualTo(2.0);
}
```

- [ ] **Step 10: Run all ras runtime tests**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/ras/runtime/pom.xml test
```

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ras add runtime/
git -C /Users/mdproctor/claude/casehub/ras commit -m "feat: SituationEvaluator SUPPRESS handling + metadata threading (ras#N)"
```

- [ ] **Step 12: Build and install ras to local Maven**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/ras/pom.xml install -DskipTests
```

---

### Task 3: IoT suppression domain — types, feature schema, SuppressionEvaluator

Foundation types and the core business logic for computing suppression assessments from CBR evidence.

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/risk/IoTSafetyCaseTypes.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/SuppressionTier.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/SuppressionAssessment.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/SuppressionConfig.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/SuppressionEvaluator.java`
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureSchemas.java` — add `situationDismissal()`
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/risk/IoTActionRiskClassifier.java` — use `IoTSafetyCaseTypes`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/SuppressionEvaluatorTest.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureSchemasTest.java` (update existing)

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.retrieveSimilar()`, `FeatureVectorCbrCase`, `CbrQuery`
- Produces:
  - `IoTSafetyCaseTypes.SAFETY_CASE_TYPES` — `Set<String>` (shared constant)
  - `IoTSafetyCaseTypes.SAFETY_SITUATION_IDS` — `Set<String>` (shared constant)
  - `SuppressionTier` — enum: `NONE`, `ANNOTATE`, `DEMOTE`, `SUPPRESS`
  - `SuppressionAssessment(tier, dismissalRate, totalCases, dismissedCases, averageSimilarity)` — record
  - `SuppressionConfig(fullThreshold, demotionThreshold, minCases, topK, minSimilarity)` — record
  - `SuppressionEvaluator.assess(situationId, features, tenantId)` → `SuppressionAssessment`
  - `IoTCbrFeatureSchemas.situationDismissal()` → `CbrFeatureSchema`

- [ ] **Step 1: Create IoTSafetyCaseTypes**

```java
package io.casehub.iot.webapp.risk;

import java.util.Set;

public final class IoTSafetyCaseTypes {
    public static final Set<String> SAFETY_CASE_TYPES = Set.of("safety-alert");
    public static final Set<String> SAFETY_SITUATION_IDS = Set.of(
            "smoke-detected", "co-detected", "water-leak-detected");
    private IoTSafetyCaseTypes() {}
}
```

- [ ] **Step 2: Refactor IoTActionRiskClassifier to use IoTSafetyCaseTypes**

Replace `private static final Set<String> SAFETY_CASE_TYPES = Set.of("safety-alert")` with reference to `IoTSafetyCaseTypes.SAFETY_CASE_TYPES`.

- [ ] **Step 3: Create SuppressionTier, SuppressionAssessment, SuppressionConfig**

```java
// SuppressionTier.java
package io.casehub.iot.webapp.cbr;
public enum SuppressionTier { NONE, ANNOTATE, DEMOTE, SUPPRESS }

// SuppressionAssessment.java
package io.casehub.iot.webapp.cbr;
public record SuppressionAssessment(
        SuppressionTier tier,
        double dismissalRate,
        int totalCases,
        int dismissedCases,
        double averageSimilarity) {}

// SuppressionConfig.java
package io.casehub.iot.webapp.cbr;
public record SuppressionConfig(
        double fullThreshold,
        double demotionThreshold,
        int minCases,
        int topK,
        double minSimilarity) {
    public SuppressionConfig {
        if (fullThreshold < demotionThreshold)
            throw new IllegalArgumentException("fullThreshold must be >= demotionThreshold");
        if (minCases < 1)
            throw new IllegalArgumentException("minCases must be >= 1");
    }
    public static SuppressionConfig defaults() {
        return new SuppressionConfig(0.9, 0.7, 5, 20, 0.5);
    }
}
```

- [ ] **Step 4: Add situationDismissal() to IoTCbrFeatureSchemas**

```java
public static CbrFeatureSchema situationDismissal() {
    var fields = new ArrayList<>(commonFields());
    fields.add(FeatureField.numeric("detectionConfidence", 0.0, 1.0,
            new SimilaritySpec.GaussianDecay(0.2)));
    return new CbrFeatureSchema("iot-dismissal", fields);
}
```

- [ ] **Step 5: Write failing tests for SuppressionEvaluator**

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class SuppressionEvaluatorTest {

    private CbrCaseMemoryStore store;
    private SuppressionEvaluator evaluator;

    @BeforeEach
    void setUp() {
        store = mock(CbrCaseMemoryStore.class);
        evaluator = new SuppressionEvaluator(store, SuppressionConfig.defaults());
    }

    @Test
    void assess_noSimilarCases_returnsNone() {
        when(store.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
                .thenReturn(List.of());
        var result = evaluator.assess("temp-threshold", features(), "t1");
        assertThat(result.tier()).isEqualTo(SuppressionTier.NONE);
        assertThat(result.totalCases()).isZero();
    }

    @Test
    void assess_belowMinCases_returnsNone() {
        when(store.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
                .thenReturn(dismissedCases(3, 1.0)); // 3 < minCases(5)
        var result = evaluator.assess("temp-threshold", features(), "t1");
        assertThat(result.tier()).isEqualTo(SuppressionTier.NONE);
    }

    @Test
    void assess_highDismissalRate_returnsSuppressTier() {
        when(store.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
                .thenReturn(mixedCases(10, 9, 0.8)); // 9/10 = 90% >= fullThreshold
        var result = evaluator.assess("temp-threshold", features(), "t1");
        assertThat(result.tier()).isEqualTo(SuppressionTier.SUPPRESS);
        assertThat(result.dismissalRate()).isCloseTo(0.9, within(0.01));
    }

    @Test
    void assess_moderateDismissalRate_returnsDemoteTier() {
        when(store.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
                .thenReturn(mixedCases(10, 8, 0.7)); // 8/10 = 80%, >= demotion(0.7) but < full(0.9)
        var result = evaluator.assess("temp-threshold", features(), "t1");
        assertThat(result.tier()).isEqualTo(SuppressionTier.DEMOTE);
    }

    @Test
    void assess_lowDismissalRate_returnsAnnotateTier() {
        when(store.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
                .thenReturn(mixedCases(10, 3, 0.6)); // 3/10 = 30%, > 0 but < demotion
        var result = evaluator.assess("temp-threshold", features(), "t1");
        assertThat(result.tier()).isEqualTo(SuppressionTier.ANNOTATE);
    }

    @Test
    void assess_zeroDismissals_returnsNone() {
        when(store.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
                .thenReturn(mixedCases(10, 0, 0.7)); // 0% dismissed
        var result = evaluator.assess("temp-threshold", features(), "t1");
        assertThat(result.tier()).isEqualTo(SuppressionTier.NONE);
    }

    @Test
    void assess_queriesWithCorrectCaseType() {
        when(store.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
                .thenReturn(List.of());
        evaluator.assess("motion-at-night", features(), "t1");
        var captor = org.mockito.ArgumentCaptor.forClass(CbrQuery.class);
        verify(store).retrieveSimilar(captor.capture(), eq(FeatureVectorCbrCase.class));
        assertThat(captor.getValue().caseType()).isEqualTo("iot-dismissal:motion-at-night");
    }

    @Test
    void constructor_nullStoreThrows() {
        assertThatNullPointerException()
                .isThrownBy(() -> new SuppressionEvaluator(null, SuppressionConfig.defaults()));
    }

    // Helper methods
    private Map<String, Object> features() {
        return Map.of("deviceClass", "thermostat", "roomType", "bedroom",
                "hourOfDay", 14.0, "dayType", "weekday", "season", "summer");
    }

    private List<ScoredCbrCase<FeatureVectorCbrCase>> dismissedCases(int count, double score) {
        return mixedCases(count, count, score);
    }

    private List<ScoredCbrCase<FeatureVectorCbrCase>> mixedCases(
            int total, int dismissed, double avgScore) {
        var results = new java.util.ArrayList<ScoredCbrCase<FeatureVectorCbrCase>>();
        for (int i = 0; i < total; i++) {
            String outcome = i < dismissed ? "dismissed" : "actioned";
            var cbrCase = new FeatureVectorCbrCase(
                    outcome, Map.of("deviceClass", FeatureValue.string("thermostat")));
            results.add(new ScoredCbrCase<>(cbrCase, "case-" + i, avgScore));
        }
        return results;
    }
}
```

- [ ] **Step 6: Implement SuppressionEvaluator**

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import java.util.*;

public class SuppressionEvaluator {
    private final CbrCaseMemoryStore store;
    private final SuppressionConfig config;

    public SuppressionEvaluator(CbrCaseMemoryStore store, SuppressionConfig config) {
        this.store = Objects.requireNonNull(store);
        this.config = Objects.requireNonNull(config);
    }

    public SuppressionAssessment assess(String situationId, Map<String, Object> features, String tenantId) {
        var featureMap = FeatureValue.toFeatureMap(features);
        var query = CbrQuery.of(tenantId, new MemoryDomain("iot"),
                io.casehub.platform.api.path.Path.root(),
                "iot-dismissal:" + situationId, featureMap, config.topK())
                .withMinSimilarity(config.minSimilarity())
                .withRetrievalMode(RetrievalMode.FEATURE_ONLY);

        List<ScoredCbrCase<FeatureVectorCbrCase>> results =
                store.retrieveSimilar(query, FeatureVectorCbrCase.class);

        if (results.isEmpty() || results.size() < config.minCases()) {
            return new SuppressionAssessment(SuppressionTier.NONE, 0.0,
                    results.size(), 0, 0.0);
        }

        int dismissed = 0;
        double scoreSum = 0.0;
        for (var scored : results) {
            if ("dismissed".equals(scored.cbrCase().outcome())) {
                dismissed++;
            }
            scoreSum += scored.score();
        }
        double rate = (double) dismissed / results.size();
        double avgSimilarity = scoreSum / results.size();

        SuppressionTier tier;
        if (dismissed == 0) {
            tier = SuppressionTier.NONE;
        } else if (rate >= config.fullThreshold()) {
            tier = SuppressionTier.SUPPRESS;
        } else if (rate >= config.demotionThreshold()) {
            tier = SuppressionTier.DEMOTE;
        } else {
            tier = SuppressionTier.ANNOTATE;
        }

        return new SuppressionAssessment(tier, rate, results.size(), dismissed, avgSimilarity);
    }
}
```

- [ ] **Step 7: Update IoTCbrFeatureSchemasTest — add situationDismissal test**

```java
@Test
void situationDismissal_schemaHasCommonFieldsPlusConfidence() {
    var schema = IoTCbrFeatureSchemas.situationDismissal();
    assertThat(schema.schemaId()).isEqualTo("iot-dismissal");
    var fieldNames = schema.fields().stream().map(FeatureField::name).toList();
    assertThat(fieldNames).contains("deviceClass", "roomType", "hourOfDay",
            "dayType", "season", "detectionConfidence");
}
```

- [ ] **Step 8: Run all webapp-api tests**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/iot/webapp-api/pom.xml test
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: suppression domain core — evaluator, types, feature schema (#52)"
```

---

### Task 4: IoTSuppressionTriggerPolicy

The custom `RasTriggerPolicy` implementation. Delegates to `DefaultRasTriggerPolicy` for chain-mode evaluation, then checks CBR for suppression before returning the decision.

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTSuppressionTriggerPolicy.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTSuppressionTriggerPolicyTest.java`

**Interfaces:**
- Consumes: `DefaultRasTriggerPolicy`, `SuppressionEvaluator`, `IoTSafetyCaseTypes`, `DeviceRegistry`, `IoTCbrFeatureExtractors`
- Produces: `IoTSuppressionTriggerPolicy implements RasTriggerPolicy`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.iot.api.DeviceRegistry;
import io.casehub.iot.webapp.risk.IoTSafetyCaseTypes;
import io.casehub.ras.api.*;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.*;
import java.util.*;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class IoTSuppressionTriggerPolicyTest {

    private SuppressionEvaluator suppressionEvaluator;
    private DeviceRegistry deviceRegistry;
    private IoTSuppressionTriggerPolicy policy;

    private static final CaseTriggerConfig NORMAL_CONFIG =
            new CaseTriggerConfig("ns", "hvac-anomaly", "1.0", Map.of());
    private static final CaseTriggerConfig SAFETY_CONFIG =
            new CaseTriggerConfig("ns", "safety-alert", "1.0", Map.of());

    @BeforeEach
    void setUp() {
        suppressionEvaluator = mock(SuppressionEvaluator.class);
        deviceRegistry = mock(DeviceRegistry.class);
        policy = new IoTSuppressionTriggerPolicy(suppressionEvaluator, deviceRegistry);
    }

    private SituationContext ctx(String situationId) {
        return SituationContext.initial(situationId, "device/thermostat-1", "t1",
                Instant.parse("2026-07-20T14:00:00Z"));
    }

    private SituationDefinition def(String situationId, CaseTriggerConfig config) {
        return new SituationDefinition(situationId, Set.of("temp.reading"),
                Duration.ofMinutes(5), null, new ChainMode.Or(Set.of("g1")),
                new TriggerAction.CreateCase(config), null);
    }

    @Test
    void continueAccumulating_passedThrough_noSuppressionCheck() {
        // Chain mode not satisfied → CONTINUE_ACCUMULATING passes through unchanged
        var context = ctx("sit-1");
        var definition = def("sit-1", NORMAL_CONFIG);
        // No detections → chain not satisfied → DefaultRasTriggerPolicy returns CONTINUE_ACCUMULATING

        var result = policy.evaluate(context, definition).await().indefinitely();
        assertThat(result.decision()).isEqualTo(TriggerDecision.CONTINUE_ACCUMULATING);
        verifyNoInteractions(suppressionEvaluator);
    }

    @Test
    void safetyCriticalCaseType_neverSuppressed() {
        var context = ctx("sit-1").withDetection(
                new DetectionResult("g1", 0.9, DetectionSignal.DETECTED, Map.of()),
                Instant.parse("2026-07-20T14:00:00Z"));
        var definition = def("sit-1", SAFETY_CONFIG);

        var result = policy.evaluate(context, definition).await().indefinitely();
        assertThat(result.decision()).isEqualTo(TriggerDecision.TRIGGER);
        verifyNoInteractions(suppressionEvaluator);
    }

    @Test
    void safetySituationId_neverSuppressed() {
        var context = ctx("smoke-detected").withDetection(
                new DetectionResult("g1", 0.9, DetectionSignal.DETECTED, Map.of()),
                Instant.parse("2026-07-20T14:00:00Z"));
        var definition = new SituationDefinition("smoke-detected", Set.of("smoke.alarm"),
                Duration.ofMinutes(5), null, new ChainMode.Or(Set.of("g1")),
                new TriggerAction.NotifyOnly(), null);

        var result = policy.evaluate(context, definition).await().indefinitely();
        assertThat(result.decision()).isEqualTo(TriggerDecision.TRIGGER);
        verifyNoInteractions(suppressionEvaluator);
    }

    @Test
    void triggerWithHighDismissalRate_returnsSuppressWithMetadata() {
        var context = ctx("sit-1").withDetection(
                new DetectionResult("g1", 0.9, DetectionSignal.DETECTED, Map.of()),
                Instant.parse("2026-07-20T14:00:00Z"));
        var definition = def("sit-1", NORMAL_CONFIG);

        when(suppressionEvaluator.assess(eq("sit-1"), any(), eq("t1")))
                .thenReturn(new SuppressionAssessment(
                        SuppressionTier.SUPPRESS, 0.92, 15, 14, 0.85));

        var result = policy.evaluate(context, definition).await().indefinitely();
        assertThat(result.decision()).isEqualTo(TriggerDecision.SUPPRESS);
        assertThat(result.metadata())
                .containsEntry(SuppressionMetadataKeys.TIER, "full")
                .containsEntry(SuppressionMetadataKeys.DISMISSAL_RATE, 0.92);
    }

    @Test
    void triggerWithModerateDismissalRate_returnsSuppressWithDemoteMetadata() {
        var context = ctx("sit-1").withDetection(
                new DetectionResult("g1", 0.9, DetectionSignal.DETECTED, Map.of()),
                Instant.parse("2026-07-20T14:00:00Z"));
        var definition = def("sit-1", NORMAL_CONFIG);

        when(suppressionEvaluator.assess(eq("sit-1"), any(), eq("t1")))
                .thenReturn(new SuppressionAssessment(
                        SuppressionTier.DEMOTE, 0.78, 12, 9, 0.72));

        var result = policy.evaluate(context, definition).await().indefinitely();
        assertThat(result.decision()).isEqualTo(TriggerDecision.SUPPRESS);
        assertThat(result.metadata())
                .containsEntry(SuppressionMetadataKeys.TIER, "demote");
    }

    @Test
    void triggerWithLowDismissalRate_returnsTriggerWithAnnotation() {
        var context = ctx("sit-1").withDetection(
                new DetectionResult("g1", 0.9, DetectionSignal.DETECTED, Map.of()),
                Instant.parse("2026-07-20T14:00:00Z"));
        var definition = def("sit-1", NORMAL_CONFIG);

        when(suppressionEvaluator.assess(eq("sit-1"), any(), eq("t1")))
                .thenReturn(new SuppressionAssessment(
                        SuppressionTier.ANNOTATE, 0.45, 10, 5, 0.65));

        var result = policy.evaluate(context, definition).await().indefinitely();
        assertThat(result.decision()).isEqualTo(TriggerDecision.TRIGGER);
        assertThat(result.metadata())
                .containsEntry(SuppressionMetadataKeys.TIER, "annotate")
                .containsEntry(SuppressionMetadataKeys.DISMISSAL_RATE, 0.45);
    }

    @Test
    void triggerWithNoHistory_returnsTriggerNoMetadata() {
        var context = ctx("sit-1").withDetection(
                new DetectionResult("g1", 0.9, DetectionSignal.DETECTED, Map.of()),
                Instant.parse("2026-07-20T14:00:00Z"));
        var definition = def("sit-1", NORMAL_CONFIG);

        when(suppressionEvaluator.assess(eq("sit-1"), any(), eq("t1")))
                .thenReturn(new SuppressionAssessment(
                        SuppressionTier.NONE, 0.0, 0, 0, 0.0));

        var result = policy.evaluate(context, definition).await().indefinitely();
        assertThat(result.decision()).isEqualTo(TriggerDecision.TRIGGER);
        assertThat(result.metadata()).isEmpty();
    }
}
```

- [ ] **Step 2: Implement IoTSuppressionTriggerPolicy**

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.iot.api.DeviceRegistry;
import io.casehub.iot.webapp.risk.IoTSafetyCaseTypes;
import io.casehub.ras.api.*;
import io.casehub.ras.runtime.DefaultRasTriggerPolicy;
import io.smallrye.mutiny.Uni;
import java.util.*;

public class IoTSuppressionTriggerPolicy implements RasTriggerPolicy {

    private final DefaultRasTriggerPolicy delegate = new DefaultRasTriggerPolicy();
    private final SuppressionEvaluator suppressionEvaluator;
    private final DeviceRegistry deviceRegistry;

    public IoTSuppressionTriggerPolicy(SuppressionEvaluator suppressionEvaluator,
                                        DeviceRegistry deviceRegistry) {
        this.suppressionEvaluator = Objects.requireNonNull(suppressionEvaluator);
        this.deviceRegistry = Objects.requireNonNull(deviceRegistry);
    }

    @Override
    public Uni<PolicyDecision> evaluate(SituationContext context, SituationDefinition definition) {
        return delegate.evaluate(context, definition).map(base -> {
            if (base.decision() != TriggerDecision.TRIGGER
                    && base.decision() != TriggerDecision.TRIGGER_AND_CONTINUE) {
                return base;
            }
            if (isSafetyCritical(definition)) {
                return base;
            }

            Map<String, Object> features = extractFeatures(context);
            SuppressionAssessment assessment = suppressionEvaluator.assess(
                    definition.situationId(), features, context.tenancyId());

            return switch (assessment.tier()) {
                case NONE -> base;
                case ANNOTATE -> new PolicyDecision(base.decision(), buildMetadata(assessment, "annotate"));
                case DEMOTE -> new PolicyDecision(TriggerDecision.SUPPRESS, buildMetadata(assessment, "demote"));
                case SUPPRESS -> new PolicyDecision(TriggerDecision.SUPPRESS, buildMetadata(assessment, "full"));
            };
        });
    }

    private boolean isSafetyCritical(SituationDefinition definition) {
        if (IoTSafetyCaseTypes.SAFETY_SITUATION_IDS.contains(definition.situationId())) {
            return true;
        }
        return definition.triggerAction() instanceof TriggerAction.CreateCase createCase
                && IoTSafetyCaseTypes.SAFETY_CASE_TYPES.contains(createCase.config().caseName());
    }

    private Map<String, Object> extractFeatures(SituationContext context) {
        var features = new LinkedHashMap<String, Object>();
        var device = deviceRegistry.findById(context.correlationKey());
        device.ifPresent(d -> {
            if (d.deviceClass() != null) features.put("deviceClass", d.deviceClass());
            if (d.location() != null) features.put("roomType", d.location());
        });
        IoTCbrFeatureExtractors.deriveTemporalFeatures(features, context.lastSignal());
        double maxConfidence = context.detections().stream()
                .filter(td -> td.result().signal().isAtLeast(DetectionSignal.WEAK))
                .mapToDouble(td -> td.result().confidence())
                .max().orElse(0.0);
        features.put("detectionConfidence", maxConfidence);
        return Map.copyOf(features);
    }

    private Map<String, Object> buildMetadata(SuppressionAssessment assessment, String tierLabel) {
        return Map.of(
                SuppressionMetadataKeys.TIER, tierLabel,
                SuppressionMetadataKeys.DISMISSAL_RATE, assessment.dismissalRate(),
                SuppressionMetadataKeys.MATCH_COUNT, assessment.totalCases(),
                SuppressionMetadataKeys.AVERAGE_SIMILARITY, assessment.averageSimilarity());
    }
}
```

- [ ] **Step 3: Run tests**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/iot/webapp-api/pom.xml test -Dtest=IoTSuppressionTriggerPolicyTest
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: IoTSuppressionTriggerPolicy — graduated CBR suppression (#52)"
```

---

### Task 5: DismissalRecorder

Records operator dismissals (situation-level and case-level) as CBR evidence.

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/DismissalRecorder.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/DismissalRecorderTest.java`

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.retain()`, `DeviceRegistry`, `SituationStore`, `IoTCbrFeatureExtractors`
- Produces: `DismissalRecorder.recordSituationDismissal(situationId, correlationKey, tenancyId, reason)` → records CBR case + removes situation

- [ ] **Step 1: Write failing tests**

Key tests:
- `recordDismissal_contextFound_recordsCbrCaseAndRemovesSituation`
- `recordDismissal_contextAbsent_recordsCbrCaseWithoutRemoval`
- `recordDismissal_featureExtraction_usesDeviceRegistryAndTemporalFeatures`
- `recordDismissal_detectionConfidence_excludesAntiSignals`
- `recordDismissal_deviceNotInRegistry_omitsDeviceFeatures`
- `recordCaseOutcome_falsePositive_recordsDismissed`
- `recordCaseOutcome_actioned_recordsActioned`

- [ ] **Step 2: Implement DismissalRecorder**

The recorder stores `FeatureVectorCbrCase` with caseType `"iot-dismissal:" + situationId`. Feature extraction follows the same pattern as `IoTSuppressionTriggerPolicy.extractFeatures()` — extract into a shared utility method.

- [ ] **Step 3: Run tests**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/iot/webapp-api/pom.xml test -Dtest=DismissalRecorderTest
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: DismissalRecorder — CBR evidence from operator dismissals (#52)"
```

---

### Task 6: Persistence, CDI observers, wiring

JPA entity, Flyway migration, CDI observers for SUPPRESSED and DISMISSED events, schema registration, and CDI producer for the policy.

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/SuppressionLogEntry.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/SuppressionLogObserver.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/DismissalGangliaObserver.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/IoTSuppressionTriggerPolicyProducer.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/SuppressionConfigMapping.java`
- Create: `webapp/src/main/resources/db/iot-webapp/migration/V503__create_iot_suppression_log.sql`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/IoTCbrSchemaRegistration.java` — add `situationDismissal()`

**Interfaces:**
- Consumes: `SituationChangeEvent(SUPPRESSED)`, `SituationChangeEvent(DISMISSED)`, `SuppressionEvaluator`, `DismissalRecorder`
- Produces: CDI-managed `IoTSuppressionTriggerPolicy`, `SuppressionLogEntry` JPA entity

- [ ] **Step 1: Create Flyway migration V503**

```sql
CREATE TABLE iot_suppression_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    situation_id    VARCHAR(255) NOT NULL,
    correlation_key VARCHAR(255) NOT NULL,
    tenancy_id      VARCHAR(255) NOT NULL,
    suppressed_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    tier            VARCHAR(20)  NOT NULL,
    dismissal_rate     DOUBLE PRECISION NOT NULL,
    matched_case_count INT             NOT NULL,
    average_similarity DOUBLE PRECISION NOT NULL,
    context_snapshot   JSONB,
    overridden      BOOLEAN      NOT NULL DEFAULT FALSE,
    overridden_at   TIMESTAMPTZ,
    overridden_by   VARCHAR(255)
);

CREATE INDEX idx_suppression_log_situation ON iot_suppression_log (situation_id, suppressed_at DESC);
CREATE INDEX idx_suppression_log_recent ON iot_suppression_log (suppressed_at DESC) WHERE NOT overridden;
```

- [ ] **Step 2: Create SuppressionLogEntry JPA entity**

```java
@Entity
@Table(name = "iot_suppression_log")
public class SuppressionLogEntry {
    @Id @GeneratedValue UUID id;
    String situationId;
    String correlationKey;
    String tenancyId;
    Instant suppressedAt;
    @Enumerated(EnumType.STRING) SuppressionTier tier;
    double dismissalRate;
    int matchedCaseCount;
    double averageSimilarity;
    @Column(columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON) SituationContext contextSnapshot;
    boolean overridden;
    Instant overriddenAt;
    String overriddenBy;
    // getters, setters, constructor
}
```

- [ ] **Step 3: Create SuppressionLogObserver**

CDI observer on `SituationChangeEvent(SUPPRESSED)` — persists `SuppressionLogEntry`.

- [ ] **Step 4: Create DismissalGangliaObserver**

CDI observer on `SituationChangeEvent(DISMISSED)` — closes ganglia via `SituationDefinitionRegistry`.

- [ ] **Step 5: Create SuppressionConfigMapping + IoTSuppressionTriggerPolicyProducer**

`@ConfigMapping(prefix = "casehub.iot.suppression")` for the config properties.
CDI producer wiring `IoTSuppressionTriggerPolicy` with `SuppressionEvaluator`, `DeviceRegistry`.

- [ ] **Step 6: Update IoTCbrSchemaRegistration**

Add `cbrStore.registerSchema(IoTCbrFeatureSchemas.situationDismissal());`

- [ ] **Step 7: Run webapp tests**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/iot/webapp/pom.xml test
```

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: suppression persistence, observers, CDI wiring (#52)"
```

---

### Task 7: REST endpoints

Dismiss, suppression history, override, and stats endpoints on `SituationResource`.

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SituationResource.java` — add 4 endpoints
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/rest/DismissRequest.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/rest/SuppressionHistoryResponse.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/rest/SuppressionStatsResponse.java`
- Test: REST endpoint tests

**Interfaces:**
- Consumes: `DismissalRecorder`, `SuppressionLogEntry` (via EntityManager), `SuppressionEvaluator`, `CaseTrigger`
- Produces:
  - `POST /api/situations/active/{correlationKey}/dismiss` → 204
  - `GET /api/situations/suppressions` → `List<SuppressionHistoryResponse>`
  - `POST /api/situations/suppressions/{id}/override` → 200 with case ID
  - `GET /api/situations/{situationId}/suppression-stats` → `SuppressionStatsResponse`

- [ ] **Step 1: Create request/response records in webapp-api**

```java
// DismissRequest.java
public record DismissRequest(String situationId, String reason) {}

// SuppressionHistoryResponse.java
public record SuppressionHistoryResponse(
        UUID id, String situationId, String correlationKey,
        String tier, double dismissalRate, int matchedCaseCount,
        Instant suppressedAt, boolean overridden) {}

// SuppressionStatsResponse.java
public record SuppressionStatsResponse(
        String situationId, int suppressedCount, int demotedCount,
        int overrideCount, double currentDismissalRate, boolean safetyCritical) {}
```

- [ ] **Step 2: Implement dismiss endpoint**

```java
@POST @Path("/active/{correlationKey}/dismiss")
public Response dismissSituation(@PathParam("correlationKey") String correlationKey,
                                  DismissRequest request) {
    dismissalRecorder.recordSituationDismissal(
            request.situationId(), correlationKey, tenancyId(), request.reason());
    return Response.noContent().build();
}
```

- [ ] **Step 3: Implement suppression history endpoint**

```java
@GET @Path("/suppressions")
public List<SuppressionHistoryResponse> listSuppressions(
        @QueryParam("situationId") String situationId,
        @QueryParam("since") Instant since,
        @QueryParam("includeOverridden") @DefaultValue("false") boolean includeOverridden) {
    // Query SuppressionLogEntry with filters
}
```

- [ ] **Step 4: Implement override endpoint**

```java
@POST @Path("/suppressions/{id}/override")
public Response overrideSuppression(@PathParam("id") UUID id) {
    // 1. Find SuppressionLogEntry
    // 2. Mark overridden
    // 3. Record CBR case outcome="override-actioned"
    // 4. Re-trigger via CaseTrigger using contextSnapshot
    // 5. Return case ID
}
```

- [ ] **Step 5: Implement suppression stats endpoint**

```java
@GET @Path("/{situationId}/suppression-stats")
public SuppressionStatsResponse getSuppressionStats(
        @PathParam("situationId") String situationId) {
    // Aggregate from iot_suppression_log + live CBR query for currentDismissalRate
}
```

- [ ] **Step 6: Run all tests**

```bash
mvn --batch-mode -f /Users/mdproctor/claude/casehub/iot install
```

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/ webapp/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: suppression REST endpoints — dismiss, history, override, stats (#52)"
```
