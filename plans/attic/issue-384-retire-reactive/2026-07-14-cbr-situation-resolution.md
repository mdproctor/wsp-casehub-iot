# CBR Situation Resolution Suggestion — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #50 — feat: situation resolution suggestion via CBR
**Issue group:** #50

**Goal:** Build a CBR retrieval service that queries the case base for
similar past resolutions and surfaces suggestions to operators via REST
and a case detail UI panel.

**Architecture:** `IoTCbrRetrievalService` (webapp-api, Tier 1) takes
`CbrConfig`, raw feature map, and tenantId — builds a `CbrQuery`,
calls `CbrCaseMemoryStore.retrieveSimilar()`, maps results to
`ResolutionSuggestion` records using `PlanTrace` directly from
neocortex-memory-api. REST endpoints on `CaseResource` (webapp)
load case context via `CaseInstanceCache`, extract features using the
`CbrConfig`'s own `LambdaFeatureExtractor`, and delegate to the
retrieval service. TypeScript UI adds a suggestions panel to the
case detail page.

**Tech Stack:** Java 22, Quarkus 3.x, JUnit 5, AssertJ, Mockito,
TypeScript (pages-ui)

## Global Constraints

- `webapp-api` is **Tier 1**: no JPA entities, no CDI runtime beans.
  CDI annotations at `provided` scope for compilation only.
- `ResolutionSuggestion` uses `PlanTrace` from `neocortex-memory-api`
  directly — no wrapper records.
- Query construction uses `CbrConfig` directly from `CaseDefinition` —
  no separate weight registry. Single source of truth.
- `FeatureValue.toFeatureMap(rawFeatures)` converts `Map<String, Object>`
  to `Map<String, FeatureValue>` for the query.
- Feature extraction in the REST layer uses the `CbrConfig`'s own
  `LambdaFeatureExtractor.extract(CaseContext)` — not a switch on case type.
- `RetrievalMode` is derived from `vectorWeight`: `> 0.0` → `HYBRID`,
  else `FEATURE_ONLY`.
- H2 test limitation: JSON columns must use `TEXT` not `JSONB`
  (GE-20260713-b879b2). Existing `cbr_case` table already uses `TEXT`.
- Accept endpoint uses `pastCaseId` (stable), not positional index.
- Idempotency tracked via `acceptedSuggestions` set in working layer context.

---

### Task 1: ResolutionSuggestion and ResolutionConfidence DTOs

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/ResolutionSuggestion.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/ResolutionConfidence.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/ResolutionSuggestionTest.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/ResolutionConfidenceTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.cbr.PlanTrace` (from neocortex-memory-api)
- Produces:
  - `ResolutionSuggestion(String caseId, double similarityScore, String problem, String solution, String outcome, Double confidence, Map<String, Object> matchedFeatures, Map<String, Double> featureSimilarities, List<PlanTrace> planSteps)`
  - `ResolutionConfidence(double bestSimilarity, double outcomeConsistency, int matchCount, ConfidenceLevel level)` with `enum ConfidenceLevel { HIGH, MEDIUM, LOW, NONE }`
  - `ResolutionConfidence.compute(List<ResolutionSuggestion> suggestions, double minSimilarityForHigh, double minConsistencyForHigh)` — static factory

- [ ] **Step 1: Write ResolutionSuggestion tests**

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.neocortex.memory.cbr.PlanTrace;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ResolutionSuggestionTest {

    @Test
    void constructsWithAllFields() {
        var planStep = new PlanTrace("bind-1", "device-control", "set-temperature",
                "SUCCESS", 1, Map.of("target", 22));
        var suggestion = new ResolutionSuggestion(
                "case-123", 0.87, "Temperature rise", "Replaced filter",
                "RESOLVED", 0.95,
                Map.of("deviceClass", "thermostat"),
                Map.of("deviceClass", 1.0),
                List.of(planStep));

        assertThat(suggestion.caseId()).isEqualTo("case-123");
        assertThat(suggestion.similarityScore()).isEqualTo(0.87);
        assertThat(suggestion.problem()).isEqualTo("Temperature rise");
        assertThat(suggestion.solution()).isEqualTo("Replaced filter");
        assertThat(suggestion.outcome()).isEqualTo("RESOLVED");
        assertThat(suggestion.confidence()).isEqualTo(0.95);
        assertThat(suggestion.planSteps()).hasSize(1);
    }

    @Test
    void nullProblemThrows() {
        assertThatThrownBy(() -> new ResolutionSuggestion(
                "case-1", 0.5, null, "solution", "RESOLVED", null,
                Map.of(), Map.of(), List.of()))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullSolutionThrows() {
        assertThatThrownBy(() -> new ResolutionSuggestion(
                "case-1", 0.5, "problem", null, "RESOLVED", null,
                Map.of(), Map.of(), List.of()))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void similarityScoreOutOfRangeThrows() {
        assertThatThrownBy(() -> new ResolutionSuggestion(
                "case-1", 1.5, "problem", "solution", "RESOLVED", null,
                Map.of(), Map.of(), List.of()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void nullOutcomeAllowed() {
        var suggestion = new ResolutionSuggestion(
                "case-1", 0.5, "problem", "solution", null, null,
                Map.of(), Map.of(), List.of());
        assertThat(suggestion.outcome()).isNull();
    }

    @Test
    void nullCaseIdAllowed() {
        var suggestion = new ResolutionSuggestion(
                null, 0.5, "problem", "solution", "RESOLVED", null,
                Map.of(), Map.of(), List.of());
        assertThat(suggestion.caseId()).isNull();
    }

    @Test
    void emptyPlanStepsAllowed() {
        var suggestion = new ResolutionSuggestion(
                "case-1", 0.5, "problem", "solution", "RESOLVED", null,
                Map.of(), Map.of(), List.of());
        assertThat(suggestion.planSteps()).isEmpty();
    }

    @Test
    void defensiveCopies() {
        var features = new java.util.HashMap<String, Object>();
        features.put("key", "val");
        var suggestion = new ResolutionSuggestion(
                "case-1", 0.5, "problem", "solution", "RESOLVED", null,
                features, Map.of(), List.of());
        assertThatThrownBy(() -> suggestion.matchedFeatures().put("x", "y"))
                .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 2: Write ResolutionConfidence tests**

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.neocortex.memory.cbr.PlanTrace;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ResolutionConfidenceTest {

    @Test
    void emptyList_returnsNone() {
        var confidence = ResolutionConfidence.compute(List.of(), 0.85, 0.80);
        assertThat(confidence.level()).isEqualTo(ResolutionConfidence.ConfidenceLevel.NONE);
        assertThat(confidence.matchCount()).isZero();
        assertThat(confidence.bestSimilarity()).isEqualTo(0.0);
        assertThat(confidence.outcomeConsistency()).isEqualTo(0.0);
    }

    @Test
    void highConfidence_allConsistent() {
        var suggestions = List.of(
                suggestion(0.92, "RESOLVED"),
                suggestion(0.88, "RESOLVED"),
                suggestion(0.86, "RESOLVED"));
        var confidence = ResolutionConfidence.compute(suggestions, 0.85, 0.80);
        assertThat(confidence.level()).isEqualTo(ResolutionConfidence.ConfidenceLevel.HIGH);
        assertThat(confidence.bestSimilarity()).isEqualTo(0.92);
        assertThat(confidence.outcomeConsistency()).isEqualTo(1.0);
        assertThat(confidence.matchCount()).isEqualTo(3);
    }

    @Test
    void mediumConfidence_highSimilarityButMixedOutcomes() {
        var suggestions = List.of(
                suggestion(0.90, "RESOLVED"),
                suggestion(0.87, "FAILED"),
                suggestion(0.86, "RESOLVED"));
        var confidence = ResolutionConfidence.compute(suggestions, 0.85, 0.80);
        assertThat(confidence.level()).isEqualTo(ResolutionConfidence.ConfidenceLevel.MEDIUM);
        assertThat(confidence.outcomeConsistency()).isCloseTo(0.667, org.assertj.core.data.Offset.offset(0.01));
    }

    @Test
    void lowConfidence_belowSimilarityThreshold() {
        var suggestions = List.of(
                suggestion(0.60, "RESOLVED"),
                suggestion(0.55, "RESOLVED"));
        var confidence = ResolutionConfidence.compute(suggestions, 0.85, 0.80);
        assertThat(confidence.level()).isEqualTo(ResolutionConfidence.ConfidenceLevel.LOW);
    }

    @Test
    void singleMatch_highSimilarity_isHigh() {
        var suggestions = List.of(suggestion(0.95, "RESOLVED"));
        var confidence = ResolutionConfidence.compute(suggestions, 0.85, 0.80);
        assertThat(confidence.level()).isEqualTo(ResolutionConfidence.ConfidenceLevel.HIGH);
        assertThat(confidence.outcomeConsistency()).isEqualTo(1.0);
    }

    @Test
    void nullOutcomes_treatedAsDistinct() {
        var suggestions = List.of(
                suggestion(0.90, null),
                suggestion(0.88, "RESOLVED"),
                suggestion(0.86, "RESOLVED"));
        var confidence = ResolutionConfidence.compute(suggestions, 0.85, 0.80);
        assertThat(confidence.outcomeConsistency()).isCloseTo(0.667, org.assertj.core.data.Offset.offset(0.01));
    }

    @Test
    void fiveMatches_threeResolved_onePartial_oneFailed() {
        var suggestions = List.of(
                suggestion(0.90, "RESOLVED"),
                suggestion(0.88, "RESOLVED"),
                suggestion(0.87, "RESOLVED_PARTIAL"),
                suggestion(0.86, "RESOLVED"),
                suggestion(0.85, "FAILED"));
        var confidence = ResolutionConfidence.compute(suggestions, 0.85, 0.80);
        assertThat(confidence.outcomeConsistency()).isEqualTo(0.6);
        assertThat(confidence.level()).isEqualTo(ResolutionConfidence.ConfidenceLevel.MEDIUM);
    }

    private static ResolutionSuggestion suggestion(double score, String outcome) {
        return new ResolutionSuggestion(
                "case-id", score, "problem", "solution", outcome, null,
                Map.of(), Map.of(), List.of());
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -pl webapp-api test -Dtest="ResolutionSuggestionTest,ResolutionConfidenceTest" --batch-mode`
Expected: compilation failure — classes don't exist yet.

- [ ] **Step 4: Implement ResolutionSuggestion**

Create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/ResolutionSuggestion.java`:

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.neocortex.memory.cbr.PlanTrace;

import java.util.List;
import java.util.Map;
import java.util.Objects;

public record ResolutionSuggestion(
        String caseId,
        double similarityScore,
        String problem,
        String solution,
        String outcome,
        Double confidence,
        Map<String, Object> matchedFeatures,
        Map<String, Double> featureSimilarities,
        List<PlanTrace> planSteps
) {
    public ResolutionSuggestion {
        Objects.requireNonNull(problem, "problem must not be null");
        Objects.requireNonNull(solution, "solution must not be null");
        if (similarityScore < 0.0 || similarityScore > 1.0) {
            throw new IllegalArgumentException(
                    "similarityScore must be in [0, 1], got: " + similarityScore);
        }
        if (confidence != null && (confidence < 0.0 || confidence > 1.0)) {
            throw new IllegalArgumentException(
                    "confidence must be in [0, 1], got: " + confidence);
        }
        matchedFeatures = matchedFeatures != null ? Map.copyOf(matchedFeatures) : Map.of();
        featureSimilarities = featureSimilarities != null ? Map.copyOf(featureSimilarities) : Map.of();
        planSteps = planSteps != null ? List.copyOf(planSteps) : List.of();
    }
}
```

- [ ] **Step 5: Implement ResolutionConfidence**

Create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/ResolutionConfidence.java`:

```java
package io.casehub.iot.webapp.cbr;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public record ResolutionConfidence(
        double bestSimilarity,
        double outcomeConsistency,
        int matchCount,
        ConfidenceLevel level
) {
    public enum ConfidenceLevel { HIGH, MEDIUM, LOW, NONE }

    public static ResolutionConfidence compute(
            List<ResolutionSuggestion> suggestions,
            double minSimilarityForHigh,
            double minConsistencyForHigh) {

        if (suggestions.isEmpty()) {
            return new ResolutionConfidence(0.0, 0.0, 0, ConfidenceLevel.NONE);
        }

        double best = suggestions.stream()
                .mapToDouble(ResolutionSuggestion::similarityScore)
                .max().orElse(0.0);

        double consistency = computeOutcomeConsistency(suggestions);
        int count = suggestions.size();

        ConfidenceLevel level;
        if (best >= minSimilarityForHigh && consistency >= minConsistencyForHigh) {
            level = ConfidenceLevel.HIGH;
        } else if (best >= 0.5) {
            level = ConfidenceLevel.MEDIUM;
        } else {
            level = ConfidenceLevel.LOW;
        }

        return new ResolutionConfidence(best, consistency, count, level);
    }

    private static double computeOutcomeConsistency(List<ResolutionSuggestion> suggestions) {
        Map<String, Long> counts = suggestions.stream()
                .map(s -> s.outcome() != null ? s.outcome() : "__null__")
                .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
        long mostFrequent = counts.values().stream().mapToLong(Long::longValue).max().orElse(0);
        return (double) mostFrequent / suggestions.size();
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn -pl webapp-api test -Dtest="ResolutionSuggestionTest,ResolutionConfidenceTest" --batch-mode`
Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/src/main/java/io/casehub/iot/webapp/cbr/ResolutionSuggestion.java webapp-api/src/main/java/io/casehub/iot/webapp/cbr/ResolutionConfidence.java webapp-api/src/test/java/io/casehub/iot/webapp/cbr/ResolutionSuggestionTest.java webapp-api/src/test/java/io/casehub/iot/webapp/cbr/ResolutionConfidenceTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(webapp-api): add ResolutionSuggestion and ResolutionConfidence DTOs (#50)"
```

---

### Task 2: IoTCbrRetrievalService

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrRetrievalService.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTCbrRetrievalServiceTest.java`

**Interfaces:**
- Consumes:
  - `CbrCaseMemoryStore.retrieveSimilar(CbrQuery, Class<C>)` → `List<ScoredCbrCase<PlanCbrCase>>`
  - `CbrConfig` (from engine-api) — weights, topK, minSimilarity, domain, caseType, vectorWeight
  - `FeatureValue.toFeatureMap(Map<String, Object>)` → `Map<String, FeatureValue>`
  - `CbrQuery.of(tenantId, domain, caseType, features, topK)` with builder chain
  - `PlanCbrCase` — problem, solution, outcome, confidence, features, planTrace
  - `ScoredCbrCase<C>` — cbrCase, caseId, score, featureSimilarities
- Produces:
  - `IoTCbrRetrievalService(CbrCaseMemoryStore store)` — constructor
  - `List<ResolutionSuggestion> retrieve(CbrConfig config, Map<String, Object> features, String tenantId)` — main method

- [ ] **Step 1: Write IoTCbrRetrievalService tests**

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrQuery;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.PlanTrace;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class IoTCbrRetrievalServiceTest {

    private CbrCaseMemoryStore store;
    private IoTCbrRetrievalService service;

    @BeforeEach
    void setUp() {
        store = mock(CbrCaseMemoryStore.class);
        service = new IoTCbrRetrievalService(store);
    }

    private CbrConfig hvacConfig() {
        return CbrConfig.builder()
                .domain("iot")
                .caseType("hvac-anomaly")
                .featureExtractor(ctx -> Map.of())
                .weight("deviceClass", 2.0)
                .weight("roomType", 1.5)
                .topK(5)
                .minSimilarity(0.3)
                .vectorWeight(0.0)
                .build();
    }

    @Test
    void retrieve_buildsCbrQueryFromConfig() {
        when(store.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of());

        var features = Map.<String, Object>of("deviceClass", "thermostat", "roomType", "bedroom");
        service.retrieve(hvacConfig(), features, "tenant-1");

        var captor = ArgumentCaptor.forClass(CbrQuery.class);
        verify(store).retrieveSimilar(captor.capture(), eq(PlanCbrCase.class));

        var query = captor.getValue();
        assertThat(query.tenantId()).isEqualTo("tenant-1");
        assertThat(query.domain().name()).isEqualTo("iot");
        assertThat(query.caseType()).isEqualTo("hvac-anomaly");
        assertThat(query.topK()).isEqualTo(5);
        assertThat(query.minSimilarity()).isEqualTo(0.3);
        assertThat(query.weights()).containsEntry("deviceClass", 2.0);
        assertThat(query.weights()).containsEntry("roomType", 1.5);
        assertThat(query.vectorWeight()).isEqualTo(0.0);
    }

    @Test
    void retrieve_convertsRawFeaturesToFeatureValues() {
        when(store.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of());

        var features = Map.<String, Object>of("deviceClass", "thermostat", "hourOfDay", 14.0);
        service.retrieve(hvacConfig(), features, "tenant-1");

        var captor = ArgumentCaptor.forClass(CbrQuery.class);
        verify(store).retrieveSimilar(captor.capture(), eq(PlanCbrCase.class));

        var queryFeatures = captor.getValue().features();
        assertThat(queryFeatures.get("deviceClass")).isEqualTo(FeatureValue.string("thermostat"));
        assertThat(queryFeatures.get("hourOfDay")).isEqualTo(FeatureValue.number(14.0));
    }

    @Test
    void retrieve_mapsScoredCasesToSuggestions() {
        var planTrace = new PlanTrace("bind-1", "device-control", "set-temp",
                "SUCCESS", 1, Map.of());
        var cbrCase = new PlanCbrCase(
                "Temperature spike", "Replaced filter", "RESOLVED", 0.95,
                Map.of("deviceClass", FeatureValue.string("thermostat")),
                List.of(planTrace));
        var scored = new ScoredCbrCase<>(cbrCase, "past-case-1", 0.87, false,
                Map.of("deviceClass", 1.0, "roomType", 0.8));

        when(store.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of(scored));

        var results = service.retrieve(hvacConfig(), Map.of("deviceClass", "thermostat"), "t1");

        assertThat(results).hasSize(1);
        var suggestion = results.getFirst();
        assertThat(suggestion.caseId()).isEqualTo("past-case-1");
        assertThat(suggestion.similarityScore()).isEqualTo(0.87);
        assertThat(suggestion.problem()).isEqualTo("Temperature spike");
        assertThat(suggestion.solution()).isEqualTo("Replaced filter");
        assertThat(suggestion.outcome()).isEqualTo("RESOLVED");
        assertThat(suggestion.confidence()).isEqualTo(0.95);
        assertThat(suggestion.planSteps()).hasSize(1);
        assertThat(suggestion.planSteps().getFirst().capabilityName()).isEqualTo("device-control");
        assertThat(suggestion.featureSimilarities()).containsEntry("deviceClass", 1.0);
    }

    @Test
    void retrieve_matchedFeaturesConvertedToRawValues() {
        var cbrCase = new PlanCbrCase(
                "problem", "solution", "RESOLVED", null,
                Map.of("deviceClass", FeatureValue.string("thermostat"),
                        "hourOfDay", FeatureValue.number(14.0)),
                List.of());
        var scored = new ScoredCbrCase<>(cbrCase, "c1", 0.5, false, Map.of());

        when(store.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of(scored));

        var results = service.retrieve(hvacConfig(), Map.of("deviceClass", "thermostat"), "t1");
        assertThat(results.getFirst().matchedFeatures())
                .containsEntry("deviceClass", "thermostat")
                .containsEntry("hourOfDay", 14.0);
    }

    @Test
    void retrieve_emptyFeatures_returnsEmptyList() {
        var results = service.retrieve(hvacConfig(), Map.of(), "t1");
        assertThat(results).isEmpty();
    }

    @Test
    void retrieve_featureOnlyMode_whenVectorWeightZero() {
        when(store.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of());

        service.retrieve(hvacConfig(), Map.of("deviceClass", "thermostat"), "t1");

        var captor = ArgumentCaptor.forClass(CbrQuery.class);
        verify(store).retrieveSimilar(captor.capture(), eq(PlanCbrCase.class));
        assertThat(captor.getValue().retrievalMode().name()).isEqualTo("FEATURE_ONLY");
    }

    @Test
    void retrieve_multipleResults_orderedByScore() {
        var case1 = new PlanCbrCase("p1", "s1", "R", null,
                Map.of("d", FeatureValue.string("a")), List.of());
        var case2 = new PlanCbrCase("p2", "s2", "R", null,
                Map.of("d", FeatureValue.string("b")), List.of());

        when(store.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of(
                        new ScoredCbrCase<>(case1, "c1", 0.9, false, Map.of()),
                        new ScoredCbrCase<>(case2, "c2", 0.7, false, Map.of())));

        var results = service.retrieve(hvacConfig(), Map.of("d", "x"), "t1");
        assertThat(results).hasSize(2);
        assertThat(results.get(0).similarityScore()).isGreaterThan(results.get(1).similarityScore());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl webapp-api test -Dtest="IoTCbrRetrievalServiceTest" --batch-mode`
Expected: compilation failure — class doesn't exist yet.

- [ ] **Step 3: Implement IoTCbrRetrievalService**

Create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrRetrievalService.java`:

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrQuery;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.RetrievalMode;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;

import java.util.List;
import java.util.Map;
import java.util.Objects;

public class IoTCbrRetrievalService {

    private final CbrCaseMemoryStore store;

    public IoTCbrRetrievalService(CbrCaseMemoryStore store) {
        this.store = Objects.requireNonNull(store, "store must not be null");
    }

    public List<ResolutionSuggestion> retrieve(CbrConfig config, Map<String, Object> rawFeatures, String tenantId) {
        Objects.requireNonNull(config, "config must not be null");
        Objects.requireNonNull(tenantId, "tenantId must not be null");

        if (rawFeatures == null || rawFeatures.isEmpty()) {
            return List.of();
        }

        Map<String, FeatureValue> featureMap = FeatureValue.toFeatureMap(rawFeatures);

        CbrQuery query = CbrQuery.of(
                        tenantId,
                        new MemoryDomain(config.domain()),
                        config.caseType(),
                        featureMap,
                        config.topK())
                .withMinSimilarity(config.minSimilarity())
                .withWeights(config.weights())
                .withVectorWeight(config.vectorWeight())
                .withRetrievalMode(config.vectorWeight() > 0.0
                        ? RetrievalMode.HYBRID : RetrievalMode.FEATURE_ONLY);

        List<ScoredCbrCase<PlanCbrCase>> scored = store.retrieveSimilar(query, PlanCbrCase.class);
        return scored.stream().map(this::toSuggestion).toList();
    }

    private ResolutionSuggestion toSuggestion(ScoredCbrCase<PlanCbrCase> scored) {
        PlanCbrCase c = scored.cbrCase();
        return new ResolutionSuggestion(
                scored.caseId(),
                scored.score(),
                c.problem(),
                c.solution(),
                c.outcome(),
                c.confidence(),
                FeatureValue.toRawMap(c.features()),
                scored.featureSimilarities(),
                c.planTrace());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -pl webapp-api test -Dtest="IoTCbrRetrievalServiceTest" --batch-mode`
Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrRetrievalService.java webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTCbrRetrievalServiceTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(webapp-api): add IoTCbrRetrievalService — CBR case base retrieval (#50)"
```

---

### Task 3: Suggestion REST Endpoints on CaseResource

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/CaseResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SuggestionResponse.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/rest/CaseResourceSuggestionsTest.java`

**Interfaces:**
- Consumes:
  - `CaseInstanceCache.get(UUID caseId)` → `CaseInstance` (null if not found)
  - `CaseInstance.getCaseMetaModel().getName()` → case type string
  - `CaseInstance.getCaseContext().layer("working")` → `ReadableLayer`
  - `CaseDefinitionRegistry.findByName(String name)` → `Optional<CaseDefinition>`
  - `CaseDefinition.getCbrConfig()` → `CbrConfig` (nullable)
  - `LambdaFeatureExtractor.extract(CaseContext)` → `Map<String, Object>`
  - `IoTCbrRetrievalService.retrieve(CbrConfig, Map<String, Object>, String)` → `List<ResolutionSuggestion>`
  - `CurrentPrincipal.tenancyId()` → `String`
- Produces:
  - `GET /api/cases/{caseId}/suggestions` → `SuggestionResponse`
  - `POST /api/cases/{caseId}/suggestions/{pastCaseId}/accept` → `void` (200 OK)
  - `SuggestionResponse(UUID caseId, String caseType, int suggestionCount, List<ResolutionSuggestion> suggestions)`

- [ ] **Step 1: Write suggestion endpoint tests**

```java
package io.casehub.iot.webapp.app.rest;

import io.casehub.api.context.CaseContext;
import io.casehub.api.context.ReadableLayer;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.cbr.LambdaFeatureExtractor;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.iot.webapp.cbr.IoTCbrRetrievalService;
import io.casehub.iot.webapp.cbr.ResolutionSuggestion;
import io.casehub.neocortex.memory.cbr.PlanTrace;
import io.casehub.platform.api.identity.CurrentPrincipal;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class CaseResourceSuggestionsTest {

    private CaseInstanceCache cache;
    private CaseDefinitionRegistry registry;
    private IoTCbrRetrievalService retrievalService;
    private CurrentPrincipal principal;
    private CaseResource resource;

    @BeforeEach
    void setUp() {
        cache = mock(CaseInstanceCache.class);
        registry = mock(CaseDefinitionRegistry.class);
        retrievalService = mock(IoTCbrRetrievalService.class);
        principal = mock(CurrentPrincipal.class);
        when(principal.tenancyId()).thenReturn("test-tenant");

        resource = new CaseResource();
        resource.principal = principal;
        resource.caseInstanceCache = cache;
        resource.caseDefinitionRegistry = registry;
        resource.retrievalService = retrievalService;
    }

    @Test
    void getSuggestions_returnsSuggestionsForCase() {
        var caseId = UUID.randomUUID();
        var instance = mockCaseInstance(caseId, "hvac-anomaly",
                Map.of("deviceClass", "thermostat", "roomType", "bedroom"));

        when(cache.get(caseId)).thenReturn(instance);

        var definition = mockDefinitionWithCbrConfig("hvac-anomaly");
        when(registry.findByName("hvac-anomaly")).thenReturn(Optional.of(definition));

        var suggestion = new ResolutionSuggestion(
                "past-1", 0.87, "Temp spike", "Filter replaced", "RESOLVED",
                0.95, Map.of(), Map.of(), List.of());
        when(retrievalService.retrieve(any(), any(), eq("test-tenant")))
                .thenReturn(List.of(suggestion));

        var response = resource.getSuggestions(caseId);

        assertThat(response.caseId()).isEqualTo(caseId);
        assertThat(response.caseType()).isEqualTo("hvac-anomaly");
        assertThat(response.suggestionCount()).isEqualTo(1);
        assertThat(response.suggestions()).hasSize(1);
        assertThat(response.suggestions().getFirst().similarityScore()).isEqualTo(0.87);
    }

    @Test
    void getSuggestions_caseNotFound_throws404() {
        when(cache.get(any())).thenReturn(null);
        assertThatThrownBy(() -> resource.getSuggestions(UUID.randomUUID()))
                .isInstanceOf(jakarta.ws.rs.NotFoundException.class);
    }

    @Test
    void getSuggestions_noCbrConfig_returnsEmptySuggestions() {
        var caseId = UUID.randomUUID();
        var instance = mockCaseInstance(caseId, "unknown-type", Map.of());
        when(cache.get(caseId)).thenReturn(instance);

        var definition = mock(CaseDefinition.class);
        when(definition.getCbrConfig()).thenReturn(null);
        when(registry.findByName("unknown-type")).thenReturn(Optional.of(definition));

        var response = resource.getSuggestions(caseId);
        assertThat(response.suggestions()).isEmpty();
        assertThat(response.suggestionCount()).isZero();
    }

    @Test
    void getSuggestions_definitionNotFound_returnsEmptySuggestions() {
        var caseId = UUID.randomUUID();
        var instance = mockCaseInstance(caseId, "missing-type", Map.of());
        when(cache.get(caseId)).thenReturn(instance);
        when(registry.findByName("missing-type")).thenReturn(Optional.empty());

        var response = resource.getSuggestions(caseId);
        assertThat(response.suggestions()).isEmpty();
    }

    private CaseInstance mockCaseInstance(UUID id, String caseType, Map<String, Object> workingData) {
        var meta = new CaseMetaModel();
        meta.setName(caseType);

        var layer = mock(ReadableLayer.class);
        for (var entry : workingData.entrySet()) {
            when(layer.get(entry.getKey())).thenReturn(entry.getValue());
        }

        var ctx = mock(CaseContext.class);
        when(ctx.layer("working")).thenReturn(layer);

        var instance = mock(CaseInstance.class);
        when(instance.getUuid()).thenReturn(id);
        when(instance.getCaseMetaModel()).thenReturn(meta);
        when(instance.getCaseContext()).thenReturn(ctx);
        when(instance.tenancyId).thenReturn("test-tenant");

        return instance;
    }

    private CaseDefinition mockDefinitionWithCbrConfig(String caseType) {
        var config = CbrConfig.builder()
                .domain("iot")
                .caseType(caseType)
                .featureExtractor(ctx -> {
                    var working = ctx.layer("working");
                    var features = new java.util.LinkedHashMap<String, Object>();
                    var dc = working.get("deviceClass");
                    if (dc != null) features.put("deviceClass", dc);
                    var rt = working.get("roomType");
                    if (rt != null) features.put("roomType", rt);
                    return Map.copyOf(features);
                })
                .weight("deviceClass", 2.0)
                .weight("roomType", 1.5)
                .topK(5)
                .minSimilarity(0.3)
                .vectorWeight(0.0)
                .build();

        var definition = mock(CaseDefinition.class);
        when(definition.getCbrConfig()).thenReturn(config);
        when(definition.getName()).thenReturn(caseType);

        return definition;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl webapp test -Dtest="CaseResourceSuggestionsTest" --batch-mode`
Expected: compilation failure — new fields and methods don't exist yet.

- [ ] **Step 3: Add injected dependencies to CaseResource**

Using `ide_insert_member`, add fields to `CaseResource`:

```java
@Inject
CaseInstanceCache caseInstanceCache;

@Inject
CaseDefinitionRegistry caseDefinitionRegistry;

@Inject
IoTCbrRetrievalService retrievalService;
```

Note: `IoTCbrRetrievalService` is Tier 1 (no CDI annotations on the class itself).
A CDI producer in the webapp module creates the bean:

Create `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/IoTCbrRetrievalServiceProducer.java`:

```java
package io.casehub.iot.webapp.app.cbr;

import io.casehub.iot.webapp.cbr.IoTCbrRetrievalService;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

@ApplicationScoped
public class IoTCbrRetrievalServiceProducer {

    @Inject
    CbrCaseMemoryStore cbrStore;

    @Produces
    @ApplicationScoped
    public IoTCbrRetrievalService retrievalService() {
        return new IoTCbrRetrievalService(cbrStore);
    }
}
```

- [ ] **Step 4: Add SuggestionResponse record**

Create `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SuggestionResponse.java`:

```java
package io.casehub.iot.webapp.app.rest;

import io.casehub.iot.webapp.cbr.ResolutionSuggestion;

import java.util.List;
import java.util.UUID;

public record SuggestionResponse(
        UUID caseId,
        String caseType,
        int suggestionCount,
        List<ResolutionSuggestion> suggestions
) {
    public SuggestionResponse {
        suggestions = suggestions != null ? List.copyOf(suggestions) : List.of();
    }
}
```

- [ ] **Step 5: Implement getSuggestions endpoint**

Using `ide_insert_member`, add to `CaseResource` after the `get()` method:

```java
@GET
@Path("/{caseId}/suggestions")
@RolesAllowed("iot-viewer")
public SuggestionResponse getSuggestions(@PathParam("caseId") UUID caseId) {
    CaseInstance instance = caseInstanceCache.get(caseId);
    if (instance == null) {
        throw new NotFoundException("Case not found: " + caseId);
    }

    String caseType = instance.getCaseMetaModel().getName();

    Optional<CaseDefinition> defOpt = caseDefinitionRegistry.findByName(caseType);
    if (defOpt.isEmpty()) {
        return new SuggestionResponse(caseId, caseType, 0, List.of());
    }

    CbrConfig cbrConfig = defOpt.get().getCbrConfig();
    if (cbrConfig == null) {
        return new SuggestionResponse(caseId, caseType, 0, List.of());
    }

    Map<String, Object> features = extractFeatures(cbrConfig, instance.getCaseContext());
    List<ResolutionSuggestion> suggestions = retrievalService.retrieve(
            cbrConfig, features, principal.tenancyId());

    return new SuggestionResponse(caseId, caseType, suggestions.size(), suggestions);
}

private Map<String, Object> extractFeatures(CbrConfig config, CaseContext context) {
    FeatureExtractor extractor = config.featureExtractor();
    if (extractor instanceof LambdaFeatureExtractor lambda) {
        return lambda.extract(context);
    }
    return Map.of();
}
```

Add required imports to `CaseResource`:
- `io.casehub.api.context.CaseContext`
- `io.casehub.api.model.CaseDefinition`
- `io.casehub.api.model.cbr.CbrConfig`
- `io.casehub.api.model.cbr.FeatureExtractor`
- `io.casehub.api.model.cbr.LambdaFeatureExtractor`
- `io.casehub.engine.common.internal.model.CaseInstance`
- `io.casehub.engine.common.spi.CaseDefinitionRegistry`
- `io.casehub.engine.common.spi.cache.CaseInstanceCache`
- `io.casehub.iot.webapp.cbr.IoTCbrRetrievalService`
- `io.casehub.iot.webapp.cbr.ResolutionSuggestion`
- `java.util.Map`
- `java.util.Optional`

- [ ] **Step 6: Implement acceptSuggestion endpoint**

Using `ide_insert_member`, add after `getSuggestions()`:

```java
@POST
@Path("/{caseId}/suggestions/{pastCaseId}/accept")
@RolesAllowed("iot-operator")
public void acceptSuggestion(
        @PathParam("caseId") UUID caseId,
        @PathParam("pastCaseId") String pastCaseId) {

    CaseInstance instance = caseInstanceCache.get(caseId);
    if (instance == null) {
        throw new NotFoundException("Case not found: " + caseId);
    }

    var context = instance.getCaseContext();
    @SuppressWarnings("unchecked")
    var accepted = (java.util.Set<String>) context.getOrDefault(
            "acceptedSuggestions", new java.util.HashSet<String>());

    if (accepted.contains(pastCaseId)) {
        return;
    }

    // Re-retrieve to get the specific past case's plan
    String caseType = instance.getCaseMetaModel().getName();
    var defOpt = caseDefinitionRegistry.findByName(caseType);
    if (defOpt.isEmpty()) {
        throw new NotFoundException("Case definition not found: " + caseType);
    }

    CbrConfig cbrConfig = defOpt.get().getCbrConfig();
    if (cbrConfig == null) {
        throw new NotFoundException("No CBR config for case type: " + caseType);
    }

    var features = extractFeatures(cbrConfig, context);
    var suggestions = retrievalService.retrieve(cbrConfig, features, principal.tenancyId());
    var match = suggestions.stream()
            .filter(s -> pastCaseId.equals(s.caseId()))
            .findFirst()
            .orElseThrow(() -> new NotFoundException("Suggestion not found: " + pastCaseId));

    // Copy plan steps — PlanTrace → planned action data in working layer
    var planSteps = match.planSteps().stream()
            .map(pt -> Map.<String, Object>of(
                    "description", pt.capabilityName() + " via " + pt.workerName(),
                    "actionType", pt.capabilityName(),
                    "parameters", pt.parameters(),
                    "priority", pt.priority(),
                    "source", "cbr:" + pastCaseId))
            .toList();

    context.set("suggestedPlan", planSteps);

    var newAccepted = new java.util.HashSet<>(accepted);
    newAccepted.add(pastCaseId);
    context.set("acceptedSuggestions", newAccepted);
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn -pl webapp test -Dtest="CaseResourceSuggestionsTest" --batch-mode`
Expected: all tests PASS.

- [ ] **Step 8: Verify build**

Run: `mvn --batch-mode install -DskipTests` to verify the full project compiles.

Check IntelliJ diagnostics:
```
ide_diagnostics(file: "webapp/src/main/java/io/casehub/iot/webapp/app/rest/CaseResource.java")
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/java/io/casehub/iot/webapp/app/rest/CaseResource.java webapp/src/main/java/io/casehub/iot/webapp/app/rest/SuggestionResponse.java webapp/src/main/java/io/casehub/iot/webapp/app/cbr/IoTCbrRetrievalServiceProducer.java webapp/src/test/java/io/casehub/iot/webapp/app/rest/CaseResourceSuggestionsTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(webapp): add CBR suggestion REST endpoints on CaseResource (#50)"
```

---

### Task 4: TypeScript UI — Suggestions Panel

**Files:**
- Modify: `webapp/src/main/webapp/src/app.ts` (dataset registration)
- Modify: `webapp/src/main/webapp/src/pages/cases.ts` (suggestions panel)

**Interfaces:**
- Consumes: `GET /api/cases/{caseId}/suggestions` → `SuggestionResponse` JSON
- Produces: "Resolution Suggestions" panel in the case detail sub-page

- [ ] **Step 1: Add dataset registration**

In `webapp/src/main/webapp/src/app.ts`, add after the `case-workers` dataset:

```typescript
dataset("case-suggestions", "/api/cases/{caseId}/suggestions");
```

- [ ] **Step 2: Add suggestions panel to case detail**

In `webapp/src/main/webapp/src/pages/cases.ts`, add a suggestions panel between
"Worker Results" and "Actions" in the case detail sub-page:

```typescript
panel("Resolution Suggestions", table({
  title: "Similar Past Resolutions",
  sortable: true,
  pageSize: 5,
  lookup: lookup("case-suggestions"),
  columns: [
    { field: "similarityScore", header: "Match", format: "percent" },
    { field: "problem", header: "Past Situation" },
    { field: "solution", header: "Resolution" },
    { field: "outcome", header: "Outcome" },
    { field: "confidence", header: "Confidence", format: "percent" },
  ],
})),
```

Note: The exact column/format API depends on the `@casehubio/pages-ui` library.
Adjust the column definition syntax to match the library's API. The table
uses the `case-suggestions` dataset scoped to the current case via `{caseId}`.

- [ ] **Step 3: Verify TypeScript compiles**

Run: `npm --prefix webapp/src/main/webapp run build` (or equivalent)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/webapp/src/app.ts webapp/src/main/webapp/src/pages/cases.ts
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(webapp): add CBR suggestions panel to case detail UI (#50)"
```

---

### Task 5: Cross-Repo Issues

**Files:** None (GitHub API only)

- [ ] **Step 1: Create cross-repo issues**

File the following GitHub issues. Each references this spec
(`docs/superpowers/specs/2026-07-14-cbr-situation-resolution-design.md`).

**casehub-platform:**
- Title: `feat: generic queue toolkit — AbstractQueueEntity, AbstractQueueService, QueueSubject SPI`
- Body: References spec §4. `platform-api` owns types (QueueSubject, QueueEntry, QueueEntryStatus). New `platform-queue` module owns JPA runtime (AbstractQueueEntity @MappedSuperclass, AbstractQueueService). Each domain builds concrete queues from these templates. Blocked by: nothing. Blocks: casehub-engine case queue.

**casehub-engine:**
- Title: `feat: case queue implementation — CaseQueueEntry, CaseQueueService, CaseQueueRoutingStrategy SPI`
- Body: References spec §5. `CaseQueueEntry extends AbstractQueueEntity<CaseInstance>` with FK to case_instance. `CaseQueueRouter` observes `CaseLifecycleEvent` via `@ObservesAsync`. `CaseQueueRoutingStrategy` SPI with `@DefaultBean` no-op. Blocked by: platform queue toolkit.

**casehubio/iot:**
- Title: `feat: CBR-aware case triage — IoTCbrCaseQueueRoutingStrategy`
- Body: References spec §6. Implements `CaseQueueRoutingStrategy` with CBR similarity-based routing. Four queues: iot-immediate (safety), iot-ai-resolution, iot-operator-assisted, iot-operator-manual. `ResolutionConfidence` computation with configurable thresholds. Error handling falls back to iot-operator-manual. Blocked by: engine case queue, #50.

**casehubio/iot:**
- Title: `feat: LLM resolution agent — IoTAiResolutionAgent`
- Body: References spec §7. Observes `CaseQueueEntryCreated` for iot-ai-resolution queue. Three autonomy levels: plan selection, adaptation, generation. `@Scheduled` timeout sweep. `AiEscalationContext` for human handoff. Blocked by: engine case queue, #50, triage.

**casehubio/iot:**
- Title: `feat: CBR temporal recency weighting`
- Body: References spec R1-11. Add temporal decay to CBR retrieval — older cases contribute less. Deferred from #50 because cold-start case base is sparse. Revisit when case base reaches critical mass.

**casehubio/iot:**
- Title: `feat: situation-level suggestion surfacing`
- Body: References spec §2 scope note. Surface CBR suggestions on `SituationResource` without navigating to the case. Depends on `SituationResource.listActive()` integration.

- [ ] **Step 2: Commit — no files changed, just record issue numbers**

Update spec §8 with the newly filed issue numbers.

```bash
git -C /Users/mdproctor/claude/casehub/iot add docs/superpowers/specs/2026-07-14-cbr-situation-resolution-design.md
git -C /Users/mdproctor/claude/casehub/iot commit -m "docs: add cross-repo issue numbers to spec (#50)"
```
