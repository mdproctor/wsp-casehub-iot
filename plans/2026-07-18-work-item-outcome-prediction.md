# Work Item Outcome Prediction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #51 — feat: work item outcome prediction via CBR
**Issue group:** #51

**Goal:** Predict likely outcome, resolution time, and best-fit assignees for work items using CBR similarity matching against historical work item completions.

**Architecture:** Uses `FeatureVectorCbrCase` (already supported by all CBR store backends) with case type `"iot-work-item"`. Feature extraction is shared between Retain (store completed outcomes) and Retrieve (predict for new work items) paths. Prediction aggregation is a new stateless computation layer. Advisory REST endpoint on `WorkItemResource`.

**Tech Stack:** Java 21, Quarkus CDI, neocortex CBR (`CbrCaseMemoryStore`, `FeatureVectorCbrCase`, `CbrFeatureSchema`), casehub-work-api (`WorkItemObserver`, `WorkItemCreator`)

## Global Constraints

- `webapp-api` is Tier 1 — pure Java, no CDI annotations, no Quarkus dependencies
- `webapp` is the CDI wiring module — `@ApplicationScoped`, `@ConfigMapping`, producers
- `FeatureVectorCbrCase` requires non-blank `problem` and `solution` fields
- CBR query case type: `"iot-work-item"` — must match schema registration
- All config under prefix `casehub.iot.webapp.cbr.work-item`
- Use `WorkItemService.findById()` for work item access (not direct EntityManager)
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing

---

### Task 1: WorkItemContext and WorkItemFeatureExtractor

Foundation types for feature extraction — shared by Retain (Task 3) and Retrieve (Task 5) paths.

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemContext.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemFeatureExtractor.java`
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureExtractors.java` — add `deriveTemporalFeatures(Map, Instant)` overload
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/WorkItemFeatureExtractorTest.java`

**Interfaces:**
- Consumes: `IoTCbrFeatureExtractors.deriveTemporalFeatures()` (existing, refactored)
- Produces:
  - `WorkItemContext` — record with 14 fields (see spec §2)
  - `WorkItemFeatureExtractor.extractForRetain(WorkItemContext) → Map<String, Object>` — input + output features
  - `WorkItemFeatureExtractor.extractForRetrieve(WorkItemContext) → Map<String, Object>` — input features only

- [ ] **Step 1: Write failing tests for WorkItemFeatureExtractor**

Create `WorkItemFeatureExtractorTest.java`:

```java
package io.casehub.iot.webapp.cbr;

import org.junit.jupiter.api.Test;
import java.time.*;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class WorkItemFeatureExtractorTest {

    @Test
    void extractForRetrieve_allFieldsPopulated_returnsInputFeaturesOnly() {
        var ctx = new WorkItemContext(
                "Inspect sensor", "Temperature sensor offline", List.of("human-review"),
                "HIGH", "hvac-technicians", "human-review", "hvac-anomaly",
                "thermostat", "bedroom", Instant.parse("2026-07-17T14:30:00Z"),
                "COMPLETED", "tech-1", Instant.parse("2026-07-17T12:00:00Z"),
                Instant.parse("2026-07-17T14:30:00Z"));

        Map<String, Object> features = WorkItemFeatureExtractor.extractForRetrieve(ctx);

        assertThat(features).containsEntry("caseType", "hvac-anomaly");
        assertThat(features).containsEntry("workerName", "human-review");
        assertThat(features).containsEntry("deviceClass", "thermostat");
        assertThat(features).containsEntry("roomType", "bedroom");
        assertThat(features).containsEntry("priority", "HIGH");
        assertThat(features).containsEntry("candidateGroups", "hvac-technicians");
        assertThat(features).containsKey("hourOfDay");
        assertThat(features).containsKey("dayType");
        assertThat(features).containsKey("season");
        assertThat(features).doesNotContainKey("resolutionDurationMinutes");
        assertThat(features).doesNotContainKey("resolvedBy");
        assertThat(features).doesNotContainKey("terminalStatus");
    }

    @Test
    void extractForRetain_includesOutputFeatures() {
        var ctx = new WorkItemContext(
                "Inspect sensor", "Sensor offline", List.of("human-review"),
                "HIGH", "hvac-technicians", "human-review", "hvac-anomaly",
                "thermostat", "bedroom", Instant.parse("2026-07-17T14:30:00Z"),
                "COMPLETED", "tech-1",
                Instant.parse("2026-07-17T12:00:00Z"),
                Instant.parse("2026-07-17T14:30:00Z"));

        Map<String, Object> features = WorkItemFeatureExtractor.extractForRetain(ctx);

        assertThat(features).containsEntry("terminalStatus", "COMPLETED");
        assertThat(features).containsEntry("resolvedBy", "tech-1");
        assertThat(features).containsEntry("resolutionDurationMinutes", 150.0);
    }

    @Test
    void extractForRetrieve_nullableFieldsAbsent_omittedFromMap() {
        var ctx = new WorkItemContext(
                "Review alert", "Generic alert", List.of("human-review"),
                "MEDIUM", "ops-team", "human-review", "generic-response",
                null, null, null,
                null, null, null, null);

        Map<String, Object> features = WorkItemFeatureExtractor.extractForRetrieve(ctx);

        assertThat(features).containsEntry("caseType", "generic-response");
        assertThat(features).containsEntry("priority", "MEDIUM");
        assertThat(features).doesNotContainKey("deviceClass");
        assertThat(features).doesNotContainKey("roomType");
        assertThat(features).doesNotContainKey("hourOfDay");
        assertThat(features).doesNotContainKey("dayType");
        assertThat(features).doesNotContainKey("season");
    }

    @Test
    void extractForRetain_nullOutputFields_omittedFromMap() {
        var ctx = new WorkItemContext(
                "Review", "Alert", List.of("human-review"),
                "LOW", "ops", "human-review", "generic-response",
                null, null, null,
                null, null, null, null);

        Map<String, Object> features = WorkItemFeatureExtractor.extractForRetain(ctx);

        assertThat(features).doesNotContainKey("terminalStatus");
        assertThat(features).doesNotContainKey("resolvedBy");
        assertThat(features).doesNotContainKey("resolutionDurationMinutes");
    }

    @Test
    void temporalDerivation_weekday() {
        var ctx = new WorkItemContext(
                "T", "D", List.of(), "HIGH", "g", "w", "c",
                null, null, Instant.parse("2026-07-15T09:00:00Z"),
                null, null, null, null);

        Map<String, Object> features = WorkItemFeatureExtractor.extractForRetrieve(ctx);

        assertThat(features).containsEntry("hourOfDay", 9.0);
        assertThat(features).containsEntry("dayType", "weekday");
        assertThat(features).containsEntry("season", "summer");
    }

    @Test
    void temporalDerivation_weekend() {
        var ctx = new WorkItemContext(
                "T", "D", List.of(), "HIGH", "g", "w", "c",
                null, null, Instant.parse("2026-07-18T22:00:00Z"),
                null, null, null, null);

        Map<String, Object> features = WorkItemFeatureExtractor.extractForRetrieve(ctx);

        assertThat(features).containsEntry("hourOfDay", 22.0);
        assertThat(features).containsEntry("dayType", "weekend");
    }

    @Test
    void temporalDerivation_winterSeason() {
        var ctx = new WorkItemContext(
                "T", "D", List.of(), "HIGH", "g", "w", "c",
                null, null, Instant.parse("2026-01-15T10:00:00Z"),
                null, null, null, null);

        Map<String, Object> features = WorkItemFeatureExtractor.extractForRetrieve(ctx);

        assertThat(features).containsEntry("season", "winter");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl webapp-api -Dtest=WorkItemFeatureExtractorTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: compilation failure — `WorkItemContext` and `WorkItemFeatureExtractor` do not exist.

- [ ] **Step 3: Create WorkItemContext record**

Use `ide_create_file` to create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemContext.java`:

```java
package io.casehub.iot.webapp.cbr;

import java.time.Instant;
import java.util.List;

public record WorkItemContext(
        String workItemTitle,
        String workItemDescription,
        List<String> workItemTypes,
        String priority,
        String candidateGroups,
        String workerName,
        String caseTypeName,
        String deviceClass,
        String roomType,
        Instant eventTimestamp,
        String terminalStatus,
        String resolvedBy,
        Instant createdAt,
        Instant completedAt
) {}
```

- [ ] **Step 4: Refactor IoTCbrFeatureExtractors — add Instant overload for temporal derivation**

Add a new overload `deriveTemporalFeatures(Map<String, Object>, Instant)` using `ide_insert_member`. The existing `deriveTemporalFeatures(Map<String, Object>, ReadableLayer)` delegates to it.

New method:

```java
static void deriveTemporalFeatures(Map<String, Object> features, Instant instant) {
    if (instant == null) return;
    ZonedDateTime zdt = instant.atZone(ZoneOffset.UTC);
    features.put("hourOfDay", (double) zdt.getHour());
    DayOfWeek dow = zdt.getDayOfWeek();
    features.put("dayType", (dow == DayOfWeek.SATURDAY || dow == DayOfWeek.SUNDAY)
            ? "weekend" : "weekday");
    features.put("season", deriveSeason(zdt.getMonthValue()));
}
```

Replace existing method body to delegate:

```java
private static void deriveTemporalFeatures(Map<String, Object> features, ReadableLayer working) {
    Object tsRaw = working.get("eventTimestamp");
    if (tsRaw == null) return;
    Instant instant;
    if (tsRaw instanceof Instant i) {
        instant = i;
    } else if (tsRaw instanceof String s) {
        instant = Instant.parse(s);
    } else {
        return;
    }
    deriveTemporalFeatures(features, instant);
}
```

- [ ] **Step 5: Create WorkItemFeatureExtractor**

Use `ide_create_file` to create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemFeatureExtractor.java`:

```java
package io.casehub.iot.webapp.cbr;

import java.time.Duration;
import java.util.LinkedHashMap;
import java.util.Map;

public final class WorkItemFeatureExtractor {

    private WorkItemFeatureExtractor() {}

    public static Map<String, Object> extractForRetain(WorkItemContext ctx) {
        var features = extractInputFeatures(ctx);
        if (ctx.terminalStatus() != null) {
            features.put("terminalStatus", ctx.terminalStatus());
        }
        if (ctx.resolvedBy() != null) {
            features.put("resolvedBy", ctx.resolvedBy());
        }
        if (ctx.createdAt() != null && ctx.completedAt() != null) {
            long minutes = Duration.between(ctx.createdAt(), ctx.completedAt()).toMinutes();
            features.put("resolutionDurationMinutes", (double) minutes);
        }
        return Map.copyOf(features);
    }

    public static Map<String, Object> extractForRetrieve(WorkItemContext ctx) {
        return Map.copyOf(extractInputFeatures(ctx));
    }

    private static Map<String, Object> extractInputFeatures(WorkItemContext ctx) {
        var features = new LinkedHashMap<String, Object>();
        features.put("caseType", ctx.caseTypeName());
        features.put("workerName", ctx.workerName());
        if (ctx.deviceClass() != null) features.put("deviceClass", ctx.deviceClass());
        if (ctx.roomType() != null) features.put("roomType", ctx.roomType());
        features.put("priority", ctx.priority());
        features.put("candidateGroups", ctx.candidateGroups());
        IoTCbrFeatureExtractors.deriveTemporalFeatures(features, ctx.eventTimestamp());
        return features;
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl webapp-api -Dtest=WorkItemFeatureExtractorTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: all 7 tests PASS.

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on `WorkItemFeatureExtractor.java` and `IoTCbrFeatureExtractors.java` to check for errors.

- [ ] **Step 8: Run full webapp-api test suite**

Run: `mvn test -pl webapp-api -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: all existing tests still pass (IoTCbrFeatureExtractors refactoring is backward-compatible).

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemContext.java webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemFeatureExtractor.java webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureExtractors.java webapp-api/src/test/java/io/casehub/iot/webapp/cbr/WorkItemFeatureExtractorTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: WorkItemContext and WorkItemFeatureExtractor (#51)"
```

---

### Task 2: WorkItemPrediction and WorkItemPredictionService

Prediction model and aggregation logic — consumes scored CBR results, produces predictions.

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemPrediction.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemPredictionService.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/WorkItemPredictionServiceTest.java`

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.retrieveSimilar(CbrQuery, FeatureVectorCbrCase.class)`, `FeatureValue.toFeatureMap()`
- Produces:
  - `WorkItemPrediction` record with `outcomeDistribution`, `resolutionTimeP50/P90`, `suggestedAssignees`, `confidence`, `sampleSize`
  - `WorkItemPrediction.AssigneeSuggestion` record
  - `WorkItemPrediction.empty()` factory
  - `WorkItemPredictionService(CbrCaseMemoryStore, int topK, double minSimilarity)`
  - `WorkItemPredictionService.predict(Map<String, Object>, String) → WorkItemPrediction`

- [ ] **Step 1: Write failing tests for WorkItemPredictionService**

Create `WorkItemPredictionServiceTest.java`:

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.*;
import static io.casehub.neocortex.memory.cbr.FeatureValue.*;
import static org.assertj.core.api.Assertions.*;

class WorkItemPredictionServiceTest {

    @Test
    void emptyResults_returnsEmptyPrediction() {
        var store = new StubCbrStore(List.of());
        var service = new WorkItemPredictionService(store, 20, 0.3);

        var prediction = service.predict(Map.of("caseType", "hvac-anomaly"), "tenant-1");

        assertThat(prediction.sampleSize()).isZero();
        assertThat(prediction.confidence()).isZero();
        assertThat(prediction.outcomeDistribution()).isEmpty();
        assertThat(prediction.resolutionTimeP50()).isNull();
        assertThat(prediction.resolutionTimeP90()).isNull();
        assertThat(prediction.suggestedAssignees()).isEmpty();
    }

    @Test
    void singleCompletedResult_fullPrediction() {
        var scored = scoredCase(0.9, "COMPLETED", 120.0, "tech-1");
        var store = new StubCbrStore(List.of(scored));
        var service = new WorkItemPredictionService(store, 20, 0.3);

        var prediction = service.predict(Map.of("caseType", "hvac-anomaly"), "tenant-1");

        assertThat(prediction.sampleSize()).isEqualTo(1);
        assertThat(prediction.outcomeDistribution()).containsEntry("COMPLETED", 1.0);
        assertThat(prediction.suggestedAssignees()).hasSize(1);
        assertThat(prediction.suggestedAssignees().getFirst().assigneeId()).isEqualTo("tech-1");
    }

    @Test
    void diverseOutcomes_weightedDistribution() {
        var completed1 = scoredCase(0.9, "COMPLETED", 120.0, "tech-1");
        var completed2 = scoredCase(0.8, "COMPLETED", 180.0, "tech-2");
        var rejected = scoredCase(0.7, "REJECTED", 60.0, "tech-1");
        var store = new StubCbrStore(List.of(completed1, completed2, rejected));
        var service = new WorkItemPredictionService(store, 20, 0.3);

        var prediction = service.predict(Map.of("caseType", "hvac-anomaly"), "tenant-1");

        assertThat(prediction.sampleSize()).isEqualTo(3);
        assertThat(prediction.outcomeDistribution().get("COMPLETED"))
                .isGreaterThan(prediction.outcomeDistribution().get("REJECTED"));
        double total = prediction.outcomeDistribution().values().stream()
                .mapToDouble(d -> d).sum();
        assertThat(total).isCloseTo(1.0, within(0.001));
    }

    @Test
    void resolutionTime_onlyCompletedItems() {
        var completed = scoredCase(0.9, "COMPLETED", 120.0, "tech-1");
        var completed2 = scoredCase(0.8, "COMPLETED", 180.0, "tech-2");
        var completed3 = scoredCase(0.7, "COMPLETED", 60.0, "tech-3");
        var cancelled = scoredCase(0.6, "CANCELLED", 300.0, null);
        var store = new StubCbrStore(List.of(completed, completed2, completed3, cancelled));
        var service = new WorkItemPredictionService(store, 20, 0.3);

        var prediction = service.predict(Map.of("caseType", "x"), "t");

        assertThat(prediction.resolutionTimeP50()).isNotNull();
        assertThat(prediction.resolutionTimeP90()).isNotNull();
        assertThat(prediction.resolutionTimeP90()).isGreaterThanOrEqualTo(prediction.resolutionTimeP50());
    }

    @Test
    void resolutionTime_fewerThan3Completed_null() {
        var completed = scoredCase(0.9, "COMPLETED", 120.0, "tech-1");
        var cancelled = scoredCase(0.8, "CANCELLED", 300.0, null);
        var store = new StubCbrStore(List.of(completed, cancelled));
        var service = new WorkItemPredictionService(store, 20, 0.3);

        var prediction = service.predict(Map.of("caseType", "x"), "t");

        assertThat(prediction.resolutionTimeP50()).isNull();
        assertThat(prediction.resolutionTimeP90()).isNull();
    }

    @Test
    void assigneeRanking_excludesNullResolvedBy() {
        var completed = scoredCase(0.9, "COMPLETED", 120.0, "tech-1");
        var cancelled = scoredCase(0.8, "CANCELLED", 300.0, null);
        var store = new StubCbrStore(List.of(completed, cancelled));
        var service = new WorkItemPredictionService(store, 20, 0.3);

        var prediction = service.predict(Map.of("caseType", "x"), "t");

        assertThat(prediction.suggestedAssignees()).hasSize(1);
        assertThat(prediction.suggestedAssignees().getFirst().assigneeId()).isEqualTo("tech-1");
    }

    @Test
    void assigneeRanking_controllableSuccessRate() {
        var c1 = scoredCase(0.9, "COMPLETED", 120.0, "tech-1");
        var c2 = scoredCase(0.9, "COMPLETED", 60.0, "tech-1");
        var c3 = scoredCase(0.9, "COMPLETED", 180.0, "tech-1");
        var r1 = scoredCase(0.9, "REJECTED", 30.0, "tech-1");
        var cancelled = scoredCase(0.9, "CANCELLED", 300.0, "tech-1");
        var store = new StubCbrStore(List.of(c1, c2, c3, r1, cancelled));
        var service = new WorkItemPredictionService(store, 20, 0.3);

        var prediction = service.predict(Map.of("caseType", "x"), "t");

        var assignee = prediction.suggestedAssignees().getFirst();
        assertThat(assignee.successRate()).isCloseTo(0.75, within(0.001));
        assertThat(assignee.taskCount()).isEqualTo(5);
    }

    @Test
    void confidence_scalesWithSampleSize() {
        var single = scoredCase(0.9, "COMPLETED", 120.0, "t");
        var storeSingle = new StubCbrStore(List.of(single));
        var serviceSingle = new WorkItemPredictionService(storeSingle, 20, 0.3);

        var many = new ArrayList<ScoredCbrCase<FeatureVectorCbrCase>>();
        for (int i = 0; i < 16; i++) many.add(scoredCase(0.9, "COMPLETED", 120.0, "t"));
        var storeMany = new StubCbrStore(many);
        var serviceMany = new WorkItemPredictionService(storeMany, 20, 0.3);

        var predSingle = serviceSingle.predict(Map.of("caseType", "x"), "t");
        var predMany = serviceMany.predict(Map.of("caseType", "x"), "t");

        assertThat(predMany.confidence()).isGreaterThan(predSingle.confidence());
    }

    // --- helpers ---

    private static ScoredCbrCase<FeatureVectorCbrCase> scoredCase(
            double score, String status, double durationMinutes, String assignee) {
        var featureMap = new LinkedHashMap<String, FeatureValue>();
        featureMap.put("terminalStatus", string(status));
        featureMap.put("resolutionDurationMinutes", number(durationMinutes));
        if (assignee != null) featureMap.put("resolvedBy", string(assignee));
        var cbrCase = new FeatureVectorCbrCase(
                "work item title", "resolution", status, 1.0, featureMap);
        return new ScoredCbrCase<>(cbrCase, score);
    }

    /**
     * Stub store that returns pre-configured results for any query.
     */
    private record StubCbrStore(
            List<ScoredCbrCase<FeatureVectorCbrCase>> results
    ) implements CbrCaseMemoryStore {

        @Override
        @SuppressWarnings("unchecked")
        public <C extends CbrCase> List<ScoredCbrCase<C>> retrieveSimilar(
                CbrQuery query, Class<C> caseType) {
            return (List<ScoredCbrCase<C>>) (List<?>) results;
        }

        @Override public void registerSchema(CbrFeatureSchema schema) {}
        @Override public String store(CbrCase c, String ct, String eid,
                MemoryDomain d, String tid, String cid,
                io.casehub.platform.api.path.Path scope) { return "id"; }
        @Override public Integer erase(EraseRequest r) { return 0; }
        @Override public Integer eraseEntity(String eid, String tid) { return 0; }
        @Override public Integer eraseByScope(
                io.casehub.platform.api.path.Path scope, String tid) { return 0; }
        @Override public void recordOutcome(String cid, String tid, CbrOutcome o) {}
        @Override public Integer purge(CbrRetentionPolicy p) { return 0; }
        @Override public void supersede(String cid, String tid, String scid, String r) {}
        @Override public void reinstate(String cid, String tid) {}
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl webapp-api -Dtest=WorkItemPredictionServiceTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: compilation failure — `WorkItemPrediction` and `WorkItemPredictionService` do not exist.

- [ ] **Step 3: Create WorkItemPrediction record**

Use `ide_create_file` to create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemPrediction.java`:

```java
package io.casehub.iot.webapp.cbr;

import java.time.Duration;
import java.util.List;
import java.util.Map;

public record WorkItemPrediction(
        Map<String, Double> outcomeDistribution,
        Duration resolutionTimeP50,
        Duration resolutionTimeP90,
        List<AssigneeSuggestion> suggestedAssignees,
        double confidence,
        int sampleSize
) {
    public record AssigneeSuggestion(
            String assigneeId,
            double successRate,
            Duration avgResolutionTime,
            int taskCount
    ) {}

    public static WorkItemPrediction empty() {
        return new WorkItemPrediction(Map.of(), null, null, List.of(), 0.0, 0);
    }
}
```

- [ ] **Step 4: Create WorkItemPredictionService**

Use `ide_create_file` to create `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemPredictionService.java`:

```java
package io.casehub.iot.webapp.cbr;

import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;

import java.time.Duration;
import java.util.*;
import java.util.stream.Collectors;

public class WorkItemPredictionService {

    private final CbrCaseMemoryStore store;
    private final int topK;
    private final double minSimilarity;

    public WorkItemPredictionService(CbrCaseMemoryStore store, int topK, double minSimilarity) {
        this.store = Objects.requireNonNull(store);
        this.topK = topK;
        this.minSimilarity = minSimilarity;
    }

    public WorkItemPrediction predict(Map<String, Object> inputFeatures, String tenantId) {
        CbrQuery query = CbrQuery.of(
                tenantId,
                new MemoryDomain("iot"),
                Path.root(),
                "iot-work-item",
                FeatureValue.toFeatureMap(inputFeatures),
                topK
        ).withMinSimilarity(minSimilarity)
         .withRetrievalMode(RetrievalMode.FEATURE_ONLY);

        List<ScoredCbrCase<FeatureVectorCbrCase>> results =
                store.retrieveSimilar(query, FeatureVectorCbrCase.class);

        if (results.isEmpty()) {
            return WorkItemPrediction.empty();
        }

        return aggregate(results);
    }

    private WorkItemPrediction aggregate(List<ScoredCbrCase<FeatureVectorCbrCase>> results) {
        int sampleSize = results.size();
        var outcomeDistribution = computeOutcomeDistribution(results);
        var resolutionTimes = computeResolutionTimes(results);
        var assignees = computeAssigneeRankings(results);
        double confidence = computeConfidence(results);

        return new WorkItemPrediction(
                outcomeDistribution,
                resolutionTimes.p50(),
                resolutionTimes.p90(),
                assignees,
                confidence,
                sampleSize);
    }

    private Map<String, Double> computeOutcomeDistribution(
            List<ScoredCbrCase<FeatureVectorCbrCase>> results) {
        Map<String, Double> weighted = new LinkedHashMap<>();
        double totalWeight = 0;
        for (var scored : results) {
            String status = featureString(scored, "terminalStatus");
            if (status == null) continue;
            double w = scored.score();
            weighted.merge(status, w, Double::sum);
            totalWeight += w;
        }
        if (totalWeight == 0) return Map.of();
        double total = totalWeight;
        weighted.replaceAll((k, v) -> v / total);
        return Map.copyOf(weighted);
    }

    private record ResolutionTimes(Duration p50, Duration p90) {}

    private ResolutionTimes computeResolutionTimes(
            List<ScoredCbrCase<FeatureVectorCbrCase>> results) {
        List<Double> durations = results.stream()
                .filter(s -> "COMPLETED".equals(featureString(s, "terminalStatus")))
                .map(s -> featureNumber(s, "resolutionDurationMinutes"))
                .filter(Objects::nonNull)
                .toList();

        if (durations.size() < 3) return new ResolutionTimes(null, null);

        List<Double> sorted = durations.stream().sorted().toList();
        return new ResolutionTimes(
                Duration.ofMinutes(Math.round(percentile(sorted, 50))),
                Duration.ofMinutes(Math.round(percentile(sorted, 90))));
    }

    private List<WorkItemPrediction.AssigneeSuggestion> computeAssigneeRankings(
            List<ScoredCbrCase<FeatureVectorCbrCase>> results) {

        record AssigneeStats(int completed, int controllable, int total,
                             List<Double> completedDurations) {}

        Map<String, AssigneeStats> byAssignee = new LinkedHashMap<>();
        Set<String> controllable = Set.of("COMPLETED", "REJECTED", "FAULTED");

        for (var scored : results) {
            String assignee = featureString(scored, "resolvedBy");
            if (assignee == null) continue;
            String status = featureString(scored, "terminalStatus");
            if (status == null) continue;

            byAssignee.compute(assignee, (k, prev) -> {
                int c = (prev != null ? prev.completed() : 0) +
                        ("COMPLETED".equals(status) ? 1 : 0);
                int ctrl = (prev != null ? prev.controllable() : 0) +
                        (controllable.contains(status) ? 1 : 0);
                int t = (prev != null ? prev.total() : 0) + 1;
                var durations = new ArrayList<>(
                        prev != null ? prev.completedDurations() : List.of());
                if ("COMPLETED".equals(status)) {
                    Double dur = featureNumber(scored, "resolutionDurationMinutes");
                    if (dur != null) durations.add(dur);
                }
                return new AssigneeStats(c, ctrl, t, durations);
            });
        }

        return byAssignee.entrySet().stream()
                .map(e -> {
                    var s = e.getValue();
                    double successRate = s.controllable() > 0
                            ? (double) s.completed() / s.controllable() : 0.0;
                    Duration avgDur = s.completedDurations().isEmpty() ? null
                            : Duration.ofMinutes(Math.round(
                                    s.completedDurations().stream()
                                            .mapToDouble(d -> d).average().orElse(0)));
                    return new WorkItemPrediction.AssigneeSuggestion(
                            e.getKey(), successRate, avgDur, s.total());
                })
                .sorted(Comparator
                        .comparingDouble(WorkItemPrediction.AssigneeSuggestion::successRate)
                        .reversed()
                        .thenComparing(a -> a.avgResolutionTime() != null
                                ? a.avgResolutionTime() : Duration.ofDays(365)))
                .limit(5)
                .toList();
    }

    private double computeConfidence(List<ScoredCbrCase<FeatureVectorCbrCase>> results) {
        double meanScore = results.stream()
                .mapToDouble(ScoredCbrCase::score).average().orElse(0);
        double sampleFactor = Math.min(1.0,
                Math.log(results.size() + 1) / Math.log(2) / 4.0);
        return meanScore * sampleFactor;
    }

    private static double percentile(List<Double> sorted, int p) {
        if (sorted.isEmpty()) return 0;
        double index = (p / 100.0) * (sorted.size() - 1);
        int lower = (int) Math.floor(index);
        int upper = Math.min(lower + 1, sorted.size() - 1);
        double fraction = index - lower;
        return sorted.get(lower) + fraction * (sorted.get(upper) - sorted.get(lower));
    }

    private static String featureString(ScoredCbrCase<FeatureVectorCbrCase> scored, String key) {
        FeatureValue fv = scored.cbrCase().features().get(key);
        return fv instanceof FeatureValue.StringVal sv ? sv.value() : null;
    }

    private static Double featureNumber(ScoredCbrCase<FeatureVectorCbrCase> scored, String key) {
        FeatureValue fv = scored.cbrCase().features().get(key);
        return fv instanceof FeatureValue.NumberVal nv ? nv.value() : null;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl webapp-api -Dtest=WorkItemPredictionServiceTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: all 8 tests PASS.

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on `WorkItemPredictionService.java` and `WorkItemPrediction.java`.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemPrediction.java webapp-api/src/main/java/io/casehub/iot/webapp/cbr/WorkItemPredictionService.java webapp-api/src/test/java/io/casehub/iot/webapp/cbr/WorkItemPredictionServiceTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: WorkItemPredictionService with aggregation (#51)"
```

---

### Task 3: Configuration, Schema Registration, and CDI Producers

Wires the Tier 1 services into CDI and registers the CBR feature schema.

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemCbrConfig.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemPredictionServiceProducer.java`
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureSchemas.java` — add `workItemOutcome()`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/IoTCbrSchemaRegistration.java` — register new schema
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureSchemasTest.java` — add test for new schema

**Interfaces:**
- Consumes: `CbrCaseMemoryStore`, `IoTCbrFeatureSchemas.commonFields()`
- Produces:
  - `WorkItemCbrConfig` — `@ConfigMapping` with `enabled()`, `topK()`, `minSimilarity()`
  - `WorkItemPredictionServiceProducer` — CDI producer
  - `IoTCbrFeatureSchemas.workItemOutcome()` → `CbrFeatureSchema`

- [ ] **Step 1: Write failing test for workItemOutcome schema**

Add to `IoTCbrFeatureSchemasTest.java`:

```java
@Test
void workItemOutcome_hasCorrectCaseTypeAndFields() {
    var schema = IoTCbrFeatureSchemas.workItemOutcome();
    assertThat(schema.caseType()).isEqualTo("iot-work-item");
    var fieldNames = schema.fields().stream()
            .map(FeatureField::name).toList();
    assertThat(fieldNames).contains("deviceClass", "roomType", "hourOfDay",
            "dayType", "season", "caseType", "workerName", "priority",
            "candidateGroups");
    assertThat(fieldNames).doesNotContain("resolutionDurationMinutes",
            "resolvedBy", "terminalStatus");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl webapp-api -Dtest=IoTCbrFeatureSchemasTest#workItemOutcome_hasCorrectCaseTypeAndFields -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: compilation failure — `workItemOutcome()` does not exist.

- [ ] **Step 3: Make commonFields() package-visible**

The existing `commonFields()` is `private`. Change visibility to package-private so `WorkItemFeatureExtractor` tests and the schema method can reference the common set. Use `ide_edit_member` on `IoTCbrFeatureSchemas`, member `commonFields`:

Change `private static List<FeatureField> commonFields()` → `static List<FeatureField> commonFields()`.

- [ ] **Step 4: Add workItemOutcome() schema method**

Use `ide_insert_member` on `IoTCbrFeatureSchemas`, after `genericResponse()`:

```java
public static CbrFeatureSchema workItemOutcome() {
    var fields = new ArrayList<>(commonFields());
    fields.add(FeatureField.categorical("caseType"));
    fields.add(FeatureField.categorical("workerName"));
    fields.add(FeatureField.categorical("priority"));
    fields.add(FeatureField.categorical("candidateGroups"));
    return new CbrFeatureSchema("iot-work-item", fields);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl webapp-api -Dtest=IoTCbrFeatureSchemasTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: PASS.

- [ ] **Step 6: Create WorkItemCbrConfig**

Use `ide_create_file` to create `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemCbrConfig.java`:

```java
package io.casehub.iot.webapp.app.cbr;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.iot.webapp.cbr.work-item")
public interface WorkItemCbrConfig {

    @WithDefault("true")
    boolean enabled();

    @WithDefault("20")
    int topK();

    @WithDefault("0.3")
    double minSimilarity();
}
```

- [ ] **Step 7: Create WorkItemPredictionServiceProducer**

Use `ide_create_file` to create `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemPredictionServiceProducer.java`:

```java
package io.casehub.iot.webapp.app.cbr;

import io.casehub.iot.webapp.cbr.WorkItemPredictionService;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

@ApplicationScoped
public class WorkItemPredictionServiceProducer {

    @Inject
    CbrCaseMemoryStore cbrStore;

    @Inject
    WorkItemCbrConfig config;

    @Produces
    @ApplicationScoped
    public WorkItemPredictionService predictionService() {
        return new WorkItemPredictionService(cbrStore, config.topK(), config.minSimilarity());
    }
}
```

- [ ] **Step 8: Register workItemOutcome schema**

Use `ide_replace_member` on `IoTCbrSchemaRegistration`, member `onStartup`, to add the new registration:

```java
void onStartup(@Observes StartupEvent event) {
    cbrStore.registerSchema(IoTCbrFeatureSchemas.hvacAnomaly());
    cbrStore.registerSchema(IoTCbrFeatureSchemas.safetyAlert());
    cbrStore.registerSchema(IoTCbrFeatureSchemas.securityAlert());
    cbrStore.registerSchema(IoTCbrFeatureSchemas.genericResponse());
    cbrStore.registerSchema(IoTCbrFeatureSchemas.workItemOutcome());
}
```

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on all new/modified files.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/src/main/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureSchemas.java webapp-api/src/test/java/io/casehub/iot/webapp/cbr/IoTCbrFeatureSchemasTest.java webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemCbrConfig.java webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemPredictionServiceProducer.java webapp/src/main/java/io/casehub/iot/webapp/app/cbr/IoTCbrSchemaRegistration.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: WorkItemCbrConfig, schema registration, CDI producer (#51)"
```

---

### Task 4: WorkItemOutcomeRecorder

CDI observer that stores completed work item outcomes as CBR cases.

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemOutcomeRecorder.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/cbr/WorkItemOutcomeRecorderTest.java`

**Interfaces:**
- Consumes: `WorkItemObserver` (SPI from casehub-work-api), `WorkItemService.findById()`, `CbrCaseMemoryStore.store()`, `WorkItemFeatureExtractor.extractForRetain()`, `WorkItemCbrConfig.enabled()`, `CaseInstanceCache.getAll()`
- Produces: stored `FeatureVectorCbrCase` entries in the CBR store

- [ ] **Step 1: Write failing tests for WorkItemOutcomeRecorder**

Create `WorkItemOutcomeRecorderTest.java`. This test uses unit-level mocking — construct the recorder with mock/stub dependencies:

```java
package io.casehub.iot.webapp.app.cbr;

import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.iot.webapp.cbr.WorkItemFeatureExtractor;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.work.api.*;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.assertj.core.api.Assertions.*;

class WorkItemOutcomeRecorderTest {

    @Test
    void terminalStatus_storesCbrCase() {
        var captured = new ArrayList<CbrCase>();
        var store = captureStore(captured);
        var workItem = testWorkItem(WorkItemStatus.COMPLETED, "tech-1",
                """
                {"caseId":"case-1","caseType":"hvac-anomaly","workerName":"human-review",
                 "deviceClass":"thermostat","roomType":"bedroom"}
                """);
        var service = stubWorkItemService(workItem);
        var recorder = new WorkItemOutcomeRecorder(store, service, stubCache(), enabledConfig());

        recorder.onStatusChange(terminalEvent(workItem.id, WorkItemStatus.COMPLETED, "tech-1"));

        assertThat(captured).hasSize(1);
        assertThat(captured.getFirst().outcome()).isEqualTo("COMPLETED");
    }

    @Test
    void nonTerminalStatus_noOp() {
        var captured = new ArrayList<CbrCase>();
        var store = captureStore(captured);
        var recorder = new WorkItemOutcomeRecorder(store, stubWorkItemService(null),
                stubCache(), enabledConfig());

        recorder.onStatusChange(new WorkItemStatusEvent(
                WorkEventType.ASSIGNED, UUID.randomUUID(), WorkItemStatus.ASSIGNED,
                "actor", null, null, "tech-1", "group", null, "t1",
                Instant.now()));

        assertThat(captured).isEmpty();
    }

    @Test
    void disabledConfig_noOp() {
        var captured = new ArrayList<CbrCase>();
        var store = captureStore(captured);
        var workItem = testWorkItem(WorkItemStatus.COMPLETED, "tech-1", "{}");
        var recorder = new WorkItemOutcomeRecorder(store, stubWorkItemService(workItem),
                stubCache(), disabledConfig());

        recorder.onStatusChange(terminalEvent(workItem.id, WorkItemStatus.COMPLETED, "tech-1"));

        assertThat(captured).isEmpty();
    }

    // --- helpers (stubs for WorkItemService, CaseInstanceCache, CbrCaseMemoryStore, config) ---
    // Implementation deferred to the actual test file — these follow standard stub patterns:
    // - captureStore: CbrCaseMemoryStore that captures store() calls
    // - testWorkItem: WorkItem entity with fields set
    // - stubWorkItemService: returns Optional.of(workItem) from findById
    // - stubCache: returns empty CaseInstanceCache
    // - enabledConfig/disabledConfig: WorkItemCbrConfig with enabled true/false
    // - terminalEvent: WorkItemStatusEvent for a terminal status
}
```

**Note to implementer:** The test stubs follow the same pattern as `WorkItemPredictionServiceTest.StubCbrStore`. Create inline record stubs or anonymous implementations. The `WorkItem` entity uses public fields (`workItem.id`, `workItem.status`, etc.) so construction is straightforward.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl webapp -Dtest=WorkItemOutcomeRecorderTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: compilation failure.

- [ ] **Step 3: Create WorkItemOutcomeRecorder**

Use `ide_create_file` to create `webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemOutcomeRecorder.java`:

```java
package io.casehub.iot.webapp.app.cbr;

import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.iot.webapp.cbr.WorkItemContext;
import io.casehub.iot.webapp.cbr.WorkItemFeatureExtractor;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import io.casehub.work.api.*;
import io.casehub.work.api.spi.WorkItemObserver;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.*;

@ApplicationScoped
public class WorkItemOutcomeRecorder implements WorkItemObserver {

    private static final Logger LOG = Logger.getLogger(WorkItemOutcomeRecorder.class);
    private static final Set<WorkItemStatus> TERMINAL = EnumSet.of(
            WorkItemStatus.COMPLETED, WorkItemStatus.REJECTED, WorkItemStatus.FAULTED,
            WorkItemStatus.CANCELLED, WorkItemStatus.EXPIRED, WorkItemStatus.ESCALATED,
            WorkItemStatus.OBSOLETE);

    private final CbrCaseMemoryStore store;
    private final WorkItemService workItemService;
    private final CaseInstanceCache caseInstanceCache;
    private final WorkItemCbrConfig config;

    @Inject
    public WorkItemOutcomeRecorder(CbrCaseMemoryStore store,
                                    WorkItemService workItemService,
                                    CaseInstanceCache caseInstanceCache,
                                    WorkItemCbrConfig config) {
        this.store = store;
        this.workItemService = workItemService;
        this.caseInstanceCache = caseInstanceCache;
        this.config = config;
    }

    @Override
    public void onStatusChange(WorkItemStatusEvent event) {
        if (!config.enabled()) return;
        if (!TERMINAL.contains(event.status())) return;

        try {
            var workItemOpt = workItemService.findById(event.workItemId());
            if (workItemOpt.isEmpty()) {
                LOG.warnv("Work item {0} not found for CBR recording", event.workItemId());
                return;
            }
            var workItem = workItemOpt.get();
            var ctx = buildContext(workItem, event);
            var rawFeatures = WorkItemFeatureExtractor.extractForRetain(ctx);
            String solution = coalesce(workItem.resolution, event.outcome(),
                    event.detail(), event.status().name());

            var cbrCase = new FeatureVectorCbrCase(
                    workItem.title != null ? workItem.title : "work-item",
                    solution,
                    event.status().name(),
                    1.0,
                    FeatureValue.toFeatureMap(rawFeatures));

            String caseId = parseCaseId(workItem.payload);
            store.store(cbrCase, "iot-work-item", event.workItemId().toString(),
                    new MemoryDomain("iot"), event.tenancyId(),
                    caseId, Path.root());
        } catch (Exception e) {
            LOG.warnv(e, "CBR recording failed for work item {0}", event.workItemId());
        }
    }

    private WorkItemContext buildContext(
            io.casehub.work.runtime.model.WorkItem workItem,
            WorkItemStatusEvent event) {
        var payload = parsePayload(workItem.payload);
        String deviceClass = (String) payload.get("deviceClass");
        String roomType = (String) payload.get("roomType");
        String caseType = (String) payload.get("caseType");
        String workerName = (String) payload.get("workerName");
        String eventTs = (String) payload.get("eventTimestamp");

        if (caseType == null) {
            var caseOpt = findCaseForWorkItem(event.workItemId(), event.tenancyId());
            if (caseOpt.isPresent()) {
                var ci = caseOpt.get();
                caseType = ci.getCaseMetaModel().getName();
                var working = ci.getCaseContext().layer("working");
                if (deviceClass == null) {
                    Object dc = working.get("deviceClass");
                    if (dc instanceof String s) deviceClass = s;
                }
                if (roomType == null) {
                    Object rt = working.get("roomType");
                    if (rt instanceof String s) roomType = s;
                }
            }
        }

        return new WorkItemContext(
                workItem.title, workItem.description,
                workItem.types != null
                        ? workItem.types.stream().map(t -> t.path).toList()
                        : List.of(),
                workItem.priority != null ? workItem.priority.name() : "MEDIUM",
                workItem.candidateGroups,
                workerName != null ? workerName : "unknown",
                caseType != null ? caseType : "unknown",
                deviceClass, roomType,
                eventTs != null ? Instant.parse(eventTs) : null,
                event.status().name(),
                workItem.assigneeId,
                workItem.createdAt,
                Instant.now());
    }

    private Optional<io.casehub.engine.common.internal.model.CaseInstance> findCaseForWorkItem(
            UUID workItemId, String tenancyId) {
        return caseInstanceCache.getAll().stream()
                .filter(ci -> tenancyId.equals(ci.tenancyId))
                .filter(ci -> workItemId.toString().equals(ci.getWaitingForWorkId()))
                .findFirst();
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> parsePayload(String payload) {
        if (payload == null || payload.isBlank()) return Map.of();
        try {
            return new com.fasterxml.jackson.databind.ObjectMapper()
                    .readValue(payload, Map.class);
        } catch (Exception e) {
            return Map.of();
        }
    }

    private static String parseCaseId(String payload) {
        var map = parsePayload(payload);
        Object caseId = map.get("caseId");
        return caseId instanceof String s ? s : null;
    }

    private static String coalesce(String... values) {
        for (String v : values) {
            if (v != null && !v.isBlank()) return v;
        }
        return "unknown";
    }
}
```

- [ ] **Step 4: Complete the test stubs and run tests**

Flesh out the test helper methods (stubs for WorkItemService, CaseInstanceCache, config, WorkItem construction). Run:

`mvn test -pl webapp -Dtest=WorkItemOutcomeRecorderTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: all tests PASS.

- [ ] **Step 5: Verify with ide_diagnostics**

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/java/io/casehub/iot/webapp/app/cbr/WorkItemOutcomeRecorder.java webapp/src/test/java/io/casehub/iot/webapp/app/cbr/WorkItemOutcomeRecorderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: WorkItemOutcomeRecorder — CBR retain path (#51)"
```

---

### Task 5: HumanDecisionWorkerFunction Replacement

Replace stub with real work item creation via `WorkItemCreator`. Embeds IoT context in payload for the Retain path.

**Files:**
- Modify: `webapp-api/pom.xml` — add `casehub-work-api` dependency
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/worker/HumanDecisionWorkerFunction.java` — replace stub
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/HvacAnomalyCaseDescriptor.java` — pass WorkItemCreator
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/SafetyAlertCaseDescriptor.java` — pass WorkItemCreator
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/SecurityAlertCaseDescriptor.java` — pass WorkItemCreator
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/GenericResponseCaseDescriptor.java` — pass WorkItemCreator
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/HvacAnomalyCaseHub.java` — inject WorkItemCreator
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/SafetyAlertCaseHub.java` — inject WorkItemCreator
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/SecurityAlertCaseHub.java` — inject WorkItemCreator
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/GenericResponseCaseHub.java` — inject WorkItemCreator
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/worker/HumanDecisionWorkerFunctionTest.java` — rewrite

**Interfaces:**
- Consumes: `WorkItemCreator.create(WorkItemCreateRequest)` → `WorkItemRef`, `WorkItemPriority`
- Produces: work items with IoT context in payload

- [ ] **Step 1: Add casehub-work-api dependency to webapp-api pom.xml**

Add to `webapp-api/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-api</artifactId>
</dependency>
```

(Version managed by parent POM.)

- [ ] **Step 2: Write failing tests for the new HumanDecisionWorkerFunction**

Rewrite `HumanDecisionWorkerFunctionTest.java`:

```java
package io.casehub.iot.webapp.worker;

import io.casehub.work.api.*;
import io.casehub.work.api.spi.WorkItemCreator;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class HumanDecisionWorkerFunctionTest {

    @Test
    void createsWorkItemWithCorrectFields() {
        var captured = new ArrayList<WorkItemCreateRequest>();
        var creator = captureCreator(captured);
        var fn = new HumanDecisionWorkerFunction(creator);

        var input = Map.<String, Object>of(
                "caseId", "case-uuid-1",
                "caseType", "hvac-anomaly",
                "workerName", "human-review",
                "deviceClass", "thermostat",
                "roomType", "bedroom",
                "eventTimestamp", "2026-07-17T14:30:00Z",
                "situationId", "sustained-temp-rise",
                "urgency", "high",
                "candidateGroups", "hvac-technicians",
                "planItemId", "pi-uuid-1",
                "situationContext", "Temperature anomaly detected");

        fn.apply(input);

        assertThat(captured).hasSize(1);
        var req = captured.getFirst();
        assertThat(req.types).containsExactly("human-review");
        assertThat(req.priority).isEqualTo(WorkItemPriority.HIGH);
        assertThat(req.candidateGroups).isEqualTo("hvac-technicians");
        assertThat(req.payload).contains("\"deviceClass\":\"thermostat\"");
        assertThat(req.payload).contains("\"caseId\":\"case-uuid-1\"");
    }

    @Test
    void returnsWorkItemIdInResult() {
        var creator = captureCreator(new ArrayList<>());
        var fn = new HumanDecisionWorkerFunction(creator);

        var result = fn.apply(minimalInput());

        assertThat(result.getData()).containsKey("workItemId");
    }

    // helpers
    private static WorkItemCreator captureCreator(List<WorkItemCreateRequest> captured) {
        // Stub WorkItemCreator that captures the request and returns a WorkItemRef
        // with a random UUID
        return new WorkItemCreator() {
            @Override
            public WorkItemRef create(WorkItemCreateRequest req) {
                captured.add(req);
                return new WorkItemRef(UUID.randomUUID(), WorkItemStatus.PENDING,
                        req.callerRef, null, null, req.candidateGroups,
                        null, req.tenancyId, req.payload);
            }
            @Override
            public Optional<WorkItemRef> findByCallerRef(String ref) {
                return Optional.empty();
            }
            @Override
            public Optional<WorkItemRef> findActiveByCallerRef(String ref) {
                return Optional.empty();
            }
            @Override
            public void obsoleteByCallerRef(String ref) {}
        };
    }

    private static Map<String, Object> minimalInput() {
        return Map.of(
                "caseId", "c1",
                "caseType", "generic-response",
                "workerName", "human-review",
                "situationContext", "test",
                "planItemId", "pi1");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl webapp-api -Dtest=HumanDecisionWorkerFunctionTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: failure — constructor signature changed.

- [ ] **Step 4: Replace HumanDecisionWorkerFunction implementation**

Use `ide_edit_member` on `HumanDecisionWorkerFunction`, member `HumanDecisionWorkerFunction` (the class declaration):

```java
public class HumanDecisionWorkerFunction implements Function<Map<String, Object>, WorkerResult> {

    private final WorkItemCreator workItemCreator;

    public HumanDecisionWorkerFunction(WorkItemCreator workItemCreator) {
        this.workItemCreator = Objects.requireNonNull(workItemCreator);
    }

    @Override
    public WorkerResult apply(Map<String, Object> input) {
        String caseId = str(input, "caseId");
        String caseType = str(input, "caseType");
        String workerName = str(input, "workerName");
        String situationContext = str(input, "situationContext");
        String planItemId = str(input, "planItemId");

        String title = (caseType != null ? caseType.replace('-', ' ') : "Review")
                + " — " + (situationContext != null ? situationContext : "manual review");

        var payload = buildPayload(input);

        var request = WorkItemCreateRequest.builder()
                .title(title)
                .types(List.of("human-review"))
                .priority(mapPriority(str(input, "urgency")))
                .candidateGroups(str(input, "candidateGroups"))
                .callerRef("case:" + caseId + "/pi:" + planItemId)
                .payload(payload)
                .build();

        var ref = workItemCreator.create(request);
        return WorkerResult.of(Map.of("workItemId", ref.id().toString()));
    }

    private static WorkItemPriority mapPriority(String urgency) {
        if (urgency == null) return WorkItemPriority.MEDIUM;
        return switch (urgency.toLowerCase()) {
            case "critical" -> WorkItemPriority.URGENT;
            case "high" -> WorkItemPriority.HIGH;
            case "low" -> WorkItemPriority.LOW;
            default -> WorkItemPriority.MEDIUM;
        };
    }

    private static String buildPayload(Map<String, Object> input) {
        var payload = new LinkedHashMap<String, Object>();
        copyIfPresent(payload, input, "caseId");
        copyIfPresent(payload, input, "caseType");
        copyIfPresent(payload, input, "workerName");
        copyIfPresent(payload, input, "deviceClass");
        copyIfPresent(payload, input, "roomType");
        copyIfPresent(payload, input, "eventTimestamp");
        copyIfPresent(payload, input, "situationId");
        try {
            return new com.fasterxml.jackson.databind.ObjectMapper()
                    .writeValueAsString(payload);
        } catch (Exception e) {
            return "{}";
        }
    }

    private static void copyIfPresent(Map<String, Object> target,
                                       Map<String, Object> source, String key) {
        Object value = source.get(key);
        if (value != null) target.put(key, value);
    }

    private static String str(Map<String, Object> map, String key) {
        Object v = map.get(key);
        return v instanceof String s ? s : null;
    }
}
```

- [ ] **Step 5: Update all four case descriptors — convert humanReviewWorker from static to instance**

For each descriptor (`HvacAnomalyCaseDescriptor`, `SafetyAlertCaseDescriptor`, `SecurityAlertCaseDescriptor`, `GenericResponseCaseDescriptor`):

1. Add `WorkItemCreator` as a constructor parameter and field
2. Change `humanReviewWorker()` from `private static` to `private`
3. Pass `workItemCreator` to `new HumanDecisionWorkerFunction(workItemCreator)`

Use `ide_edit_member` for each.

- [ ] **Step 6: Update all four CaseHub beans — inject WorkItemCreator**

For each CaseHub bean (`HvacAnomalyCaseHub`, `SafetyAlertCaseHub`, `SecurityAlertCaseHub`, `GenericResponseCaseHub`):

1. Add `@Inject WorkItemCreator workItemCreator;` field
2. Pass `workItemCreator` to the descriptor constructor in `augment()`

Use `ide_insert_member` for the field, `ide_replace_member` for `augment()`.

- [ ] **Step 7: Run tests**

Run: `mvn test -pl webapp-api -Dtest=HumanDecisionWorkerFunctionTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: PASS.

Run: `mvn test -pl webapp-api -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: all webapp-api tests pass (existing descriptor tests may need updating for new constructor signature).

- [ ] **Step 8: Verify with ide_diagnostics on all modified files**

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/pom.xml webapp-api/src/main/java/io/casehub/iot/webapp/worker/HumanDecisionWorkerFunction.java webapp-api/src/main/java/io/casehub/iot/webapp/engine/ webapp-api/src/test/java/io/casehub/iot/webapp/worker/HumanDecisionWorkerFunctionTest.java webapp/src/main/java/io/casehub/iot/webapp/app/engine/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: replace HumanDecisionWorkerFunction stub with WorkItemCreator (#51)"
```

---

### Task 6: REST Prediction Endpoint

`GET /api/workitems/{workItemId}/prediction` on WorkItemResource.

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/WorkItemResource.java` — add prediction endpoint + response record
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/rest/WorkItemResourcePredictionTest.java`

**Interfaces:**
- Consumes: `WorkItemService.findById()`, `WorkItemPredictionService.predict()`, `WorkItemFeatureExtractor.extractForRetrieve()`, `CaseInstanceCache.getAll()`, `CurrentPrincipal.tenancyId()`
- Produces: `GET /api/workitems/{workItemId}/prediction` → `WorkItemPredictionResponse` (JSON)

- [ ] **Step 1: Write failing test for the prediction endpoint**

Create `WorkItemResourcePredictionTest.java`. This is a unit test (not @QuarkusTest) that constructs the resource with mocked dependencies:

```java
package io.casehub.iot.webapp.app.rest;

import io.casehub.iot.webapp.cbr.*;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.ws.rs.NotFoundException;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.*;

class WorkItemResourcePredictionTest {

    @Test
    void validWorkItem_returnsPrediction() {
        // Set up a work item with payload, mock services returning prediction
        // Assert response has expected shape
    }

    @Test
    void workItemNotFound_throws404() {
        // WorkItemService returns empty Optional
        // Assert NotFoundException
    }

    @Test
    void tenancyMismatch_throws404() {
        // Work item exists but tenancyId doesn't match principal
        // Assert NotFoundException
    }

    @Test
    void emptyCaseBase_returnsEmptyPrediction() {
        // PredictionService returns WorkItemPrediction.empty()
        // Assert response with sampleSize 0
    }

    @Test
    void cbrFailure_returnsEmptyPrediction() {
        // PredictionService throws RuntimeException
        // Assert empty prediction (not exception propagation)
    }
}
```

**Note to implementer:** fill in the test bodies following the patterns from `WorkItemOutcomeRecorderTest` — construct the resource, inject stub dependencies, call the method directly.

- [ ] **Step 2: Add prediction endpoint and response record to WorkItemResource**

Use `ide_insert_member` on `WorkItemResource` to add:

1. Inject `WorkItemService`, `WorkItemPredictionService`, `CaseInstanceCache` fields
2. Add `prediction()` method
3. Add `WorkItemPredictionResponse` inner record

```java
@GET
@Path("/{workItemId}/prediction")
@RolesAllowed("iot-viewer")
public WorkItemPredictionResponse prediction(@PathParam("workItemId") UUID workItemId) {
    var workItemOpt = workItemService.findById(workItemId);
    if (workItemOpt.isEmpty()) {
        throw new NotFoundException("WorkItem not found: " + workItemId);
    }
    var workItem = workItemOpt.get();
    if (!principal.tenancyId().equals(workItem.tenancyId)) {
        throw new NotFoundException("WorkItem not found: " + workItemId);
    }

    try {
        var ctx = buildPredictionContext(workItem);
        var features = WorkItemFeatureExtractor.extractForRetrieve(ctx);
        var prediction = predictionService.predict(features, principal.tenancyId());
        return WorkItemPredictionResponse.from(workItemId, prediction);
    } catch (Exception e) {
        org.jboss.logging.Logger.getLogger(WorkItemResource.class)
                .warnv(e, "CBR prediction failed for work item {0}", workItemId);
        return WorkItemPredictionResponse.empty(workItemId);
    }
}
```

- [ ] **Step 3: Run tests**

Run: `mvn test -pl webapp -Dtest=WorkItemResourcePredictionTest -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: PASS.

- [ ] **Step 4: Verify with ide_diagnostics**

- [ ] **Step 5: Run full build**

Run: `mvn --batch-mode install -f /Users/mdproctor/claude/casehub/iot/pom.xml`
Expected: BUILD SUCCESS — all modules compile and all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/java/io/casehub/iot/webapp/app/rest/WorkItemResource.java webapp/src/test/java/io/casehub/iot/webapp/app/rest/WorkItemResourcePredictionTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat: GET /api/workitems/{id}/prediction endpoint (#51)"
```
