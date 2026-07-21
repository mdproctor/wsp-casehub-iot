# CBR-Aware Case Triage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #62 — feat: CBR-aware case triage — IoTCbrCaseQueueRoutingStrategy
**Issue group:** #62

**Goal:** Route cases to queue views based on CBR similarity confidence, using the existing label → SubjectView → CaseQueueEntry pipeline.

**Architecture:** Engine writes CBR summary stats (`cbrBestSimilarity`, `cbrMatchCount`, `cbrOutcomeConsistency`) to the case working context during `injectCbrExperiences()`. IoT `LabelRule`s evaluate these scalars to set triage labels. `SubjectViewSpec`s match the labels to route cases into queue views. Safety-critical case types bypass CBR — structural override per CaseHub class.

**Tech Stack:** Java 21, Quarkus CDI, casehub-platform-api (LabelRule, LabelAction, LambdaExpression, SubjectViewSpec), casehub-engine (CaseStartedEventHandler), casehub-engine-queue (CaseQueueEntry)

**Spec:** `docs/superpowers/specs/2026-07-21-cbr-case-triage-design.md`

## Global Constraints

- `webapp-api` is Tier 1 — no CDI annotations, no JPA, no Quarkus runtime
- `webapp` is the Quarkus runtime module — CDI, `@ConfigMapping`, `@Observes`
- Engine change is cross-repo (`casehub-engine/runtime`)
- `casehub-engine-queue` dependency is a new addition to `webapp/pom.xml`
- Single tenancy: `casehub.iot.tenancy-id` config property
- `LambdaExpression` has no static factory — use `new LambdaExpression<>(...)`
- `WritableLayerImpl.engineSet()` for context writes (not `MutableCaseContext`)

---

### Task 1: Engine — CBR summary stats in working context

**Files:**
- Modify: `casehub-engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandler.java`
- Test: `casehub-engine/runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandlerTest.java`

**Interfaces:**
- Consumes: `RetrievedExperience.similarityScore()`, `RetrievedExperience.outcome()`
- Produces: context keys `cbrBestSimilarity` (double), `cbrMatchCount` (int), `cbrOutcomeConsistency` (double) in the WORKING layer

- [ ] **Step 1: Write failing test — summary stats written when experiences present**

Add to `CaseStartedEventHandlerTest.java`:

```java
@Test
void cbrSummaryStats_written_when_experiences_present() {
    CaseDefinition definition =
        CaseDefinition.builder().namespace("test").name("stats-case").version("1.0").build();
    CbrConfig config = CbrConfig.builder()
        .feature("severity", ".severity")
        .domain("test-domain")
        .topK(5).minSimilarity(0.5).build();
    definition.setCbrConfig(config);

    CaseMetaModel metaModel = new CaseMetaModel();
    metaModel.setNamespace("test");
    metaModel.setName("stats-case");
    metaModel.setVersion("1.0");
    CaseInstance instance = createCaseInstance(metaModel);
    when(caseDefinitionRegistry.getCaseDefinition(metaModel)).thenReturn(definition);

    RetrievedExperience exp1 = new RetrievedExperience(
        "problem1", "solution1", "COMPLETED", 0.9, 0.85,
        Map.of(), List.of(), Map.of());
    RetrievedExperience exp2 = new RetrievedExperience(
        "problem2", "solution2", "COMPLETED", 0.8, 0.72,
        Map.of(), List.of(), Map.of());
    RetrievedExperience exp3 = new RetrievedExperience(
        "problem3", "solution3", "FAULTED", 0.7, 0.65,
        Map.of(), List.of(), Map.of());
    when(cbrRetrievalService.retrieve(eq(definition), eq(instance)))
        .thenReturn(List.of(exp1, exp2, exp3));

    handler.onCaseStarted(new CaseStartedEvent(instance)).await().indefinitely();

    var ctx = instance.getCaseContext().layer(ContextLayer.WORKING);
    assertThat(ctx.get("cbrBestSimilarity")).isEqualTo(0.85);
    assertThat(ctx.get("cbrMatchCount")).isEqualTo(3);
    // 2 COMPLETED out of 3 total = 0.6667
    assertThat((double) ctx.get("cbrOutcomeConsistency")).isCloseTo(0.6667, within(0.001));
}
```

Use `ide_insert_member` in `CaseStartedEventHandlerTest` with anchor `empty_experiences_not_written_to_context`, position `after`.

- [ ] **Step 2: Write failing test — empty experiences writes no stats**

```java
@Test
void cbrSummaryStats_not_written_when_experiences_empty() {
    CaseDefinition definition =
        CaseDefinition.builder().namespace("test").name("empty-stats-case").version("1.0").build();
    CbrConfig config = CbrConfig.builder()
        .feature("severity", ".severity")
        .domain("test-domain")
        .topK(5).minSimilarity(0.5).build();
    definition.setCbrConfig(config);

    CaseMetaModel metaModel = new CaseMetaModel();
    metaModel.setNamespace("test");
    metaModel.setName("empty-stats-case");
    metaModel.setVersion("1.0");
    CaseInstance instance = createCaseInstance(metaModel);
    when(caseDefinitionRegistry.getCaseDefinition(metaModel)).thenReturn(definition);
    when(cbrRetrievalService.retrieve(definition, instance)).thenReturn(List.of());

    handler.onCaseStarted(new CaseStartedEvent(instance)).await().indefinitely();

    var ctx = instance.getCaseContext().layer(ContextLayer.WORKING);
    assertThat(ctx.get("cbrBestSimilarity")).isNull();
    assertThat(ctx.get("cbrMatchCount")).isNull();
    assertThat(ctx.get("cbrOutcomeConsistency")).isNull();
}
```

- [ ] **Step 3: Write failing test — null outcomes produce 0.0 consistency**

```java
@Test
void cbrOutcomeConsistency_zero_when_all_outcomes_null() {
    CaseDefinition definition =
        CaseDefinition.builder().namespace("test").name("null-outcome-case").version("1.0").build();
    CbrConfig config = CbrConfig.builder()
        .feature("severity", ".severity")
        .domain("test-domain")
        .topK(5).minSimilarity(0.5).build();
    definition.setCbrConfig(config);

    CaseMetaModel metaModel = new CaseMetaModel();
    metaModel.setNamespace("test");
    metaModel.setName("null-outcome-case");
    metaModel.setVersion("1.0");
    CaseInstance instance = createCaseInstance(metaModel);
    when(caseDefinitionRegistry.getCaseDefinition(metaModel)).thenReturn(definition);

    RetrievedExperience exp1 = new RetrievedExperience(
        "problem1", "solution1", null, null, 0.90,
        Map.of(), List.of(), Map.of());
    RetrievedExperience exp2 = new RetrievedExperience(
        "problem2", "solution2", null, null, 0.80,
        Map.of(), List.of(), Map.of());
    when(cbrRetrievalService.retrieve(eq(definition), eq(instance)))
        .thenReturn(List.of(exp1, exp2));

    handler.onCaseStarted(new CaseStartedEvent(instance)).await().indefinitely();

    var ctx = instance.getCaseContext().layer(ContextLayer.WORKING);
    assertThat(ctx.get("cbrBestSimilarity")).isEqualTo(0.90);
    assertThat(ctx.get("cbrMatchCount")).isEqualTo(2);
    assertThat(ctx.get("cbrOutcomeConsistency")).isEqualTo(0.0);
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn --batch-mode -pl runtime test -Dtest=CaseStartedEventHandlerTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: 3 new tests FAIL — `cbrBestSimilarity` is null (not yet written)

- [ ] **Step 5: Implement — add summary stats to injectCbrExperiences**

In `CaseStartedEventHandler.injectCbrExperiences()`, after the existing `engineSet("cbrExperiences", serialised)` block (line 157), add summary stat writes. Also add the `computeOutcomeConsistency` private method.

Use `ide_replace_member` on `injectCbrExperiences` to replace the method body:

```java
CaseMetaModel metaModel = instance.getCaseMetaModel();
if (metaModel == null) {
    return Uni.createFrom().voidItem();
}
CaseDefinition definition = caseDefinitionRegistry.getCaseDefinition(metaModel);
if (definition == null || definition.getCbrConfig() == null) {
    return Uni.createFrom().voidItem();
}
List<RetrievedExperience> experiences = cbrRetrievalService.retrieve(definition, instance);
if (!experiences.isEmpty()) {
    List<Map<String, Object>> serialised =
        OBJECT_MAPPER.convertValue(
            experiences, new TypeReference<List<Map<String, Object>>>() {});
    MutableCaseContext mutableContext = (MutableCaseContext) instance.getCaseContext();
    WritableLayerImpl layer = (WritableLayerImpl) mutableContext.writableLayer(ContextLayer.WORKING);
    layer.engineSet("cbrExperiences", serialised);
    layer.engineSet("cbrBestSimilarity",
        experiences.stream()
            .mapToDouble(RetrievedExperience::similarityScore)
            .max().orElse(0.0));
    layer.engineSet("cbrMatchCount", experiences.size());
    layer.engineSet("cbrOutcomeConsistency", computeOutcomeConsistency(experiences));
}
return Uni.createFrom().voidItem();
```

Add the `computeOutcomeConsistency` method using `ide_insert_member` after `injectCbrExperiences`:

```java
private static double computeOutcomeConsistency(List<RetrievedExperience> experiences) {
    Map<String, Long> freq = experiences.stream()
        .map(RetrievedExperience::outcome)
        .filter(java.util.Objects::nonNull)
        .collect(java.util.stream.Collectors.groupingBy(
            java.util.function.Function.identity(),
            java.util.stream.Collectors.counting()));
    if (freq.isEmpty()) {
        return 0.0;
    }
    return (double) java.util.Collections.max(freq.values()) / experiences.size();
}
```

Add missing imports: `java.util.Collections`, `java.util.Objects`, `java.util.function.Function`, `java.util.stream.Collectors`.

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode -pl runtime test -Dtest=CaseStartedEventHandlerTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL tests PASS (existing + 3 new)

- [ ] **Step 7: Run full engine build**

Run: `mvn --batch-mode install -f /Users/mdproctor/claude/casehub/engine/pom.xml -DskipTests=false`
Expected: BUILD SUCCESS — updated engine SNAPSHOT in local Maven repo

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandler.java runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandlerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: write CBR summary stats to working context (casehubio/iot#62)"
```

---

### Task 2: IoT — IoTTriageLabelRules factory + tests

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTTriageLabelRules.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTTriageLabelRulesTest.java`

**Interfaces:**
- Consumes: `LabelRule` (`casehub-platform-api`), `LabelAction.Add`, `LambdaExpression`
- Produces: `IoTTriageLabelRules.cbrTriageRules(double aiMinSimilarity, double aiMinConsistency)` → `List<LabelRule>`. Labels: `iot-triage:ai-resolution`, `iot-triage:operator-assisted`, `iot-triage:operator-manual`

- [ ] **Step 1: Write failing test — high confidence routes to ai-resolution**

Create `IoTTriageLabelRulesTest.java`:

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.platform.api.label.LabelAction;
import io.casehub.platform.api.label.LabelRule;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class IoTTriageLabelRulesTest {

    private final List<LabelRule> rules = IoTTriageLabelRules.cbrTriageRules(0.85, 0.80);

    @Test
    void highConfidence_routesToAiResolution() {
        Map<String, Object> ctx = Map.of(
            "cbrBestSimilarity", 0.90,
            "cbrOutcomeConsistency", 0.85,
            "cbrMatchCount", 5);
        List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
        assertThat(actions).hasSize(1);
        assertThat(actions.get(0)).isInstanceOf(LabelAction.Add.class);
        assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:ai-resolution");
    }
}
```

Use `ide_create_file` for the test file.

- [ ] **Step 2: Write failing tests — medium, low, boundary cases**

Add tests using `ide_insert_member`:

```java
@Test
void mediumConfidence_routesToOperatorAssisted() {
    Map<String, Object> ctx = Map.of(
        "cbrBestSimilarity", 0.70,
        "cbrOutcomeConsistency", 0.60,
        "cbrMatchCount", 3);
    List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
    assertThat(actions).hasSize(1);
    assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:operator-assisted");
}

@Test
void lowConfidence_routesToOperatorManual() {
    Map<String, Object> ctx = Map.of(
        "cbrBestSimilarity", 0.30,
        "cbrOutcomeConsistency", 1.0,
        "cbrMatchCount", 1);
    List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
    assertThat(actions).hasSize(1);
    assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:operator-manual");
}

@Test
void noCbrData_routesToOperatorManual() {
    Map<String, Object> ctx = Map.of();
    List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
    assertThat(actions).hasSize(1);
    assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:operator-manual");
}

@Test
void atExactAiThreshold_routesToAiResolution() {
    Map<String, Object> ctx = Map.of(
        "cbrBestSimilarity", 0.85,
        "cbrOutcomeConsistency", 0.80,
        "cbrMatchCount", 4);
    List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
    assertThat(actions).hasSize(1);
    assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:ai-resolution");
}

@Test
void highSimilarity_lowConsistency_routesToOperatorAssisted() {
    Map<String, Object> ctx = Map.of(
        "cbrBestSimilarity", 0.90,
        "cbrOutcomeConsistency", 0.50,
        "cbrMatchCount", 4);
    List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
    assertThat(actions).hasSize(1);
    assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:operator-assisted");
}

@Test
void atMediumFloor_routesToOperatorAssisted() {
    Map<String, Object> ctx = Map.of(
        "cbrBestSimilarity", 0.5,
        "cbrOutcomeConsistency", 0.0,
        "cbrMatchCount", 2);
    List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
    assertThat(actions).hasSize(1);
    assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:operator-assisted");
}

@Test
void justBelowMediumFloor_routesToOperatorManual() {
    Map<String, Object> ctx = Map.of(
        "cbrBestSimilarity", 0.499,
        "cbrOutcomeConsistency", 1.0,
        "cbrMatchCount", 5);
    List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
    assertThat(actions).hasSize(1);
    assertThat(((LabelAction.Add) actions.get(0)).label()).isEqualTo("iot-triage:operator-manual");
}

@Test
void exactlyOneRuleFires_mutualExclusivity() {
    List<Map<String, Object>> contexts = List.of(
        Map.of("cbrBestSimilarity", 0.95, "cbrOutcomeConsistency", 0.90, "cbrMatchCount", 5),
        Map.of("cbrBestSimilarity", 0.70, "cbrOutcomeConsistency", 0.60, "cbrMatchCount", 3),
        Map.of("cbrBestSimilarity", 0.30, "cbrOutcomeConsistency", 1.0, "cbrMatchCount", 1),
        Map.of());
    for (Map<String, Object> ctx : contexts) {
        List<LabelAction> actions = LabelRule.evaluate(rules, ctx);
        assertThat(actions).as("context: " + ctx).hasSize(1);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode -pl webapp-api test -Dtest=IoTTriageLabelRulesTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: FAIL — `IoTTriageLabelRules` class not found

- [ ] **Step 4: Implement IoTTriageLabelRules**

Create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTTriageLabelRules.java` using `ide_create_file`:

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.platform.api.expression.LambdaExpression;
import io.casehub.platform.api.label.LabelAction;
import io.casehub.platform.api.label.LabelRule;

import java.util.List;
import java.util.Map;

public final class IoTTriageLabelRules {

    static final double MEDIUM_FLOOR_SIMILARITY = 0.5;

    private IoTTriageLabelRules() {}

    public static List<LabelRule> cbrTriageRules(
            double aiMinSimilarity, double aiMinConsistency) {
        return List.of(
            new LabelRule("cbr-high",
                new LambdaExpression<>(ctx -> {
                    double sim = doubleOr(ctx, "cbrBestSimilarity", 0.0);
                    double con = doubleOr(ctx, "cbrOutcomeConsistency", 0.0);
                    return sim >= aiMinSimilarity && con >= aiMinConsistency;
                }),
                List.of(new LabelAction.Add("iot-triage:ai-resolution"))),
            new LabelRule("cbr-medium",
                new LambdaExpression<>(ctx -> {
                    double sim = doubleOr(ctx, "cbrBestSimilarity", 0.0);
                    double con = doubleOr(ctx, "cbrOutcomeConsistency", 0.0);
                    return sim >= MEDIUM_FLOOR_SIMILARITY
                        && !(sim >= aiMinSimilarity && con >= aiMinConsistency);
                }),
                List.of(new LabelAction.Add("iot-triage:operator-assisted"))),
            new LabelRule("cbr-low-or-none",
                new LambdaExpression<>(ctx ->
                    doubleOr(ctx, "cbrBestSimilarity", 0.0) < MEDIUM_FLOOR_SIMILARITY),
                List.of(new LabelAction.Add("iot-triage:operator-manual")))
        );
    }

    @SuppressWarnings("unchecked")
    private static double doubleOr(Object ctxObj, String key, double def) {
        Map<String, Object> ctx = (Map<String, Object>) ctxObj;
        Object v = ctx.get(key);
        return v instanceof Number n ? n.doubleValue() : def;
    }
}
```

Note: `LambdaExpression<C, R>` is generic. The `LabelRule` constructor expects `CompiledExpression<Map<String, Object>, Boolean>`. The lambda receives `Object` (the raw `C` parameter), so `doubleOr` accepts `Object` and casts to `Map`. Check the actual `LabelRule.evaluate()` call chain to confirm the type passed to `eval()`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode -pl webapp-api test -Dtest=IoTTriageLabelRulesTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: ALL 9 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTTriageLabelRules.java webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTTriageLabelRulesTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: IoTTriageLabelRules — CBR-based queue routing labels (#62)"
```

---

### Task 3: IoT webapp — IoTTriageConfig + IoTQueueViewInitializer + CaseHub wiring

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/triage/IoTTriageConfig.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/triage/IoTQueueViewInitializer.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/HvacAnomalyCaseHub.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/GenericResponseCaseHub.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/SafetyAlertCaseHub.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/SecurityAlertCaseHub.java`
- Modify: `webapp/pom.xml` (add `casehub-engine-queue` dependency)
- Modify: `webapp/src/main/resources/application.properties`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/triage/IoTQueueViewInitializerTest.java`

**Interfaces:**
- Consumes: `IoTTriageLabelRules.cbrTriageRules(double, double)` from Task 2, `SubjectViewStore`, `SubjectViewOrchestrator`, `SubjectViewSpec` from `casehub-platform-api`/`casehub-platform-view`
- Produces: `IoTTriageConfig` interface (config mapping), `IoTQueueViewInitializer` (startup bean), 4 SubjectViewSpecs created at startup

- [ ] **Step 1: Add casehub-engine-queue dependency to webapp/pom.xml**

Add to the `<dependencies>` section of `webapp/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-queue</artifactId>
</dependency>
```

Version managed by parent BOM. Also add `casehub-platform-view` if not already present (needed for `SubjectViewOrchestrator`):

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-view</artifactId>
</dependency>
```

Check the parent pom `<dependencyManagement>` section to confirm both are declared. If not, add them there first.

- [ ] **Step 2: Create IoTTriageConfig**

Create `webapp/src/main/java/io/casehub/iot/webapp/app/triage/IoTTriageConfig.java` using `ide_create_file`:

```java
package io.casehub.iot.webapp.app.triage;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;

@ConfigMapping(prefix = "casehub.iot.triage")
public interface IoTTriageConfig {

    @WithName("ai-resolution.min-similarity")
    @WithDefault("0.85")
    double aiMinSimilarity();

    @WithName("ai-resolution.min-consistency")
    @WithDefault("0.80")
    double aiMinConsistency();
}
```

- [ ] **Step 3: Write failing test — IoTQueueViewInitializer creates views at startup**

Create `IoTQueueViewInitializerTest.java` using `ide_create_file`:

```java
package io.casehub.iot.webapp.app.triage;

import io.casehub.platform.api.view.SubjectViewSpec;
import io.casehub.platform.api.view.SubjectViewStore;
import io.casehub.platform.view.SubjectViewOrchestrator;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class IoTQueueViewInitializerTest {

    private SubjectViewStore viewStore;
    private SubjectViewOrchestrator orchestrator;
    private IoTQueueViewInitializer initializer;
    private static final String TENANCY = "test-tenant";

    @BeforeEach
    void setUp() throws Exception {
        viewStore = mock(SubjectViewStore.class);
        orchestrator = mock(SubjectViewOrchestrator.class);
        initializer = new IoTQueueViewInitializer();
        var f1 = IoTQueueViewInitializer.class.getDeclaredField("viewStore");
        f1.setAccessible(true);
        f1.set(initializer, viewStore);
        var f2 = IoTQueueViewInitializer.class.getDeclaredField("orchestrator");
        f2.setAccessible(true);
        f2.set(initializer, orchestrator);
        var f3 = IoTQueueViewInitializer.class.getDeclaredField("tenancyId");
        f3.setAccessible(true);
        f3.set(initializer, TENANCY);
    }

    @Test
    void createsAllFourViewsWhenNoneExist() {
        when(viewStore.findByTenancy(TENANCY)).thenReturn(List.of());
        when(orchestrator.saveView(any())).thenAnswer(inv -> inv.getArgument(0));

        initializer.onStartup(null);

        verify(orchestrator, times(4)).saveView(any(SubjectViewSpec.class));
    }

    @Test
    void skipsExistingViews_idempotent() {
        List<SubjectViewSpec> existing = List.of(
            new SubjectViewSpec(java.util.UUID.randomUUID(), "iot-immediate", TENANCY,
                "iot-triage:immediate", null, "enqueuedAt", "ASC", null, java.time.Instant.now()),
            new SubjectViewSpec(java.util.UUID.randomUUID(), "iot-ai-resolution", TENANCY,
                "iot-triage:ai-resolution", null, "enqueuedAt", "ASC", null, java.time.Instant.now()),
            new SubjectViewSpec(java.util.UUID.randomUUID(), "iot-operator-assisted", TENANCY,
                "iot-triage:operator-assisted", null, "enqueuedAt", "ASC", null, java.time.Instant.now()),
            new SubjectViewSpec(java.util.UUID.randomUUID(), "iot-operator-manual", TENANCY,
                "iot-triage:operator-manual", null, "enqueuedAt", "ASC", null, java.time.Instant.now()));
        when(viewStore.findByTenancy(TENANCY)).thenReturn(existing);

        initializer.onStartup(null);

        verify(orchestrator, never()).saveView(any());
    }

    @Test
    void createsOnlyMissingViews() {
        List<SubjectViewSpec> existing = List.of(
            new SubjectViewSpec(java.util.UUID.randomUUID(), "iot-immediate", TENANCY,
                "iot-triage:immediate", null, "enqueuedAt", "ASC", null, java.time.Instant.now()));
        when(viewStore.findByTenancy(TENANCY)).thenReturn(existing);
        when(orchestrator.saveView(any())).thenAnswer(inv -> inv.getArgument(0));

        initializer.onStartup(null);

        verify(orchestrator, times(3)).saveView(any(SubjectViewSpec.class));
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn --batch-mode -pl webapp test -Dtest=IoTQueueViewInitializerTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: FAIL — `IoTQueueViewInitializer` not found

- [ ] **Step 5: Implement IoTQueueViewInitializer**

Create `webapp/src/main/java/io/casehub/iot/webapp/app/triage/IoTQueueViewInitializer.java` using `ide_create_file`:

```java
package io.casehub.iot.webapp.app.triage;

import io.casehub.platform.api.view.SubjectViewSpec;
import io.casehub.platform.api.view.SubjectViewStore;
import io.casehub.platform.view.SubjectViewOrchestrator;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class IoTQueueViewInitializer {

    @Inject SubjectViewStore viewStore;
    @Inject SubjectViewOrchestrator orchestrator;

    @Inject @ConfigProperty(name = "casehub.iot.tenancy-id")
    String tenancyId;

    void onStartup(@Observes StartupEvent event) {
        List<SubjectViewSpec> existing = viewStore.findByTenancy(tenancyId);
        ensureView(existing, "iot-immediate", "iot-triage:immediate");
        ensureView(existing, "iot-ai-resolution", "iot-triage:ai-resolution");
        ensureView(existing, "iot-operator-assisted", "iot-triage:operator-assisted");
        ensureView(existing, "iot-operator-manual", "iot-triage:operator-manual");
    }

    private void ensureView(List<SubjectViewSpec> existing, String name, String labelPattern) {
        boolean found = existing.stream().anyMatch(v -> name.equals(v.name()));
        if (!found) {
            orchestrator.saveView(new SubjectViewSpec(
                UUID.randomUUID(), name, tenancyId, labelPattern,
                null, "enqueuedAt", "ASC", null, Instant.now()));
        }
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode -pl webapp test -Dtest=IoTQueueViewInitializerTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: ALL 3 tests PASS

- [ ] **Step 7: Wire CaseHubs — add label rules and config injection**

**SafetyAlertCaseHub** — add safety-immediate label rule. Use `ide_edit_member` on `augment`:

Add at end of `augment()` body, after the existing `setCbrConfig()` call:

```java
definition.setLabelRules(List.of(
    new LabelRule("safety-immediate",
        new LambdaExpression<>(ctx -> true),
        List.of(new LabelAction.Add("iot-triage:immediate")))));
```

Add imports: `io.casehub.platform.api.label.LabelRule`, `io.casehub.platform.api.label.LabelAction`, `io.casehub.platform.api.expression.LambdaExpression`, `java.util.List`.

**SecurityAlertCaseHub** — same as SafetyAlertCaseHub. Add identical label rule.

**HvacAnomalyCaseHub** — inject `IoTTriageConfig`, add CBR triage rules. Add field:

```java
@Inject IoTTriageConfig triageConfig;
```

Add at end of `augment()`:

```java
definition.setLabelRules(
    IoTTriageLabelRules.cbrTriageRules(
        triageConfig.aiMinSimilarity(), triageConfig.aiMinConsistency()));
```

Add imports: `io.casehub.iot.webapp.cbr.IoTTriageLabelRules`, `io.casehub.iot.webapp.app.triage.IoTTriageConfig`.

**GenericResponseCaseHub** — same as HvacAnomaly. Inject `IoTTriageConfig`, add CBR triage rules.

- [ ] **Step 8: Add config properties to application.properties**

Add to `webapp/src/main/resources/application.properties`:

```properties
# CBR triage thresholds
casehub.iot.triage.ai-resolution.min-similarity=0.85
casehub.iot.triage.ai-resolution.min-consistency=0.80
```

- [ ] **Step 9: Build and verify**

Run: `mvn --batch-mode install -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: BUILD SUCCESS

Use `ide_diagnostics` on each modified CaseHub file to verify no compilation errors.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/pom.xml webapp/src/main/java/io/casehub/iot/webapp/app/triage/ webapp/src/test/java/io/casehub/iot/webapp/app/triage/ webapp/src/main/java/io/casehub/iot/webapp/app/engine/HvacAnomalyCaseHub.java webapp/src/main/java/io/casehub/iot/webapp/app/engine/GenericResponseCaseHub.java webapp/src/main/java/io/casehub/iot/webapp/app/engine/SafetyAlertCaseHub.java webapp/src/main/java/io/casehub/iot/webapp/app/engine/SecurityAlertCaseHub.java webapp/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: IoT queue view initializer + CaseHub triage wiring (#62)"
```
