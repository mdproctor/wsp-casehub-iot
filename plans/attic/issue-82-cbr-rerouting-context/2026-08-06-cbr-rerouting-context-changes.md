# CBR Re-Routing on Context Changes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #82 — Re-routing on context changes (CBR re-evaluation)
**Issue group:** #82

**Goal:** Add an event-driven observer that detects working context changes
on AI-queued cases, re-runs CBR retrieval, and escalates to operator queues
when the similarity band drops.

**Architecture:** New `IoTCbrReEvaluationObserver` in `webapp` observes
`CaseContextUpdatedEvent` (engine CDI event, currently zero observers).
Filters by layer + AI queue membership + debounce, then re-evaluates CBR.
On band drop, calls `queueService.escalate()` directly — same API the agent
and timeout sweep use.

**Tech Stack:** Java 21, CDI (Quarkus), Micrometer, Mockito/JUnit 5

## Global Constraints

- Downward-only re-routing: AI → operator queues. Never operator → AI.
- Thresholds from `IoTTriageConfig` — same config the label rules use.
- Debounce: 30s per case (named constant, not configurable initially).
- No engine changes. Observer is IoT-side only.
- Labels become stale after re-routing — accepted trade-off (§Architecture in spec).

---

### Task 1: IoTCbrReEvaluationObserver — core observer with filtering, band computation, and escalation

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTCbrReEvaluationObserver.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTCbrReEvaluationObserverTest.java`

**Interfaces:**
- Consumes: `CaseContextUpdatedEvent` (engine CDI event), `CaseQueueService.findByView()`, `CaseQueueService.escalate()`, `CaseInstanceCache.get()`, `IoTCbrRetrievalService.retrieve()`, `CaseDefinitionRegistry.findByName()`, `SubjectViewStore.findByTenancy()`, `IoTTriageConfig.aiMinSimilarity()`, `IoTTriageConfig.aiMinConsistency()`
- Produces: Nothing consumed by other tasks. Self-contained component.

- [ ] **Step 1: Create test class with setUp and helper methods**

```java
package io.casehub.iot.webapp.app.resolution;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.CaseDefinition;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.engine.common.spi.event.CaseContextUpdatedEvent;
import io.casehub.engine.queue.model.CaseQueueEntry;
import io.casehub.engine.queue.model.QueueEntryStatus;
import io.casehub.engine.queue.service.CaseQueueService;
import io.casehub.iot.webapp.app.triage.IoTTriageConfig;
import io.casehub.iot.webapp.cbr.IoTCbrRetrievalService;
import io.casehub.iot.webapp.cbr.ResolutionSuggestion;
import io.casehub.platform.api.view.SubjectViewSpec;
import io.casehub.platform.api.view.SubjectViewStore;
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import io.casehub.api.context.CaseContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Field;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class IoTCbrReEvaluationObserverTest {

    private static final String TENANCY = "test-tenant";
    private static final UUID AI_VIEW_ID = UUID.randomUUID();
    private static final UUID OPERATOR_ASSISTED_VIEW_ID = UUID.randomUUID();
    private static final UUID OPERATOR_MANUAL_VIEW_ID = UUID.randomUUID();

    private CaseQueueService queueService;
    private CaseInstanceCache caseCache;
    private IoTCbrRetrievalService retrievalService;
    private CaseDefinitionRegistry definitionRegistry;
    private SubjectViewStore viewStore;
    private IoTTriageConfig triageConfig;
    private SimpleMeterRegistry meterRegistry;
    private IoTCbrReEvaluationObserver observer;

    @BeforeEach
    void setUp() throws Exception {
        queueService = mock(CaseQueueService.class);
        caseCache = mock(CaseInstanceCache.class);
        retrievalService = mock(IoTCbrRetrievalService.class);
        definitionRegistry = mock(CaseDefinitionRegistry.class);
        viewStore = mock(SubjectViewStore.class);
        triageConfig = mock(IoTTriageConfig.class);
        meterRegistry = new SimpleMeterRegistry();

        when(triageConfig.aiMinSimilarity()).thenReturn(0.85);
        when(triageConfig.aiMinConsistency()).thenReturn(0.80);

        when(viewStore.findByTenancy(TENANCY)).thenReturn(List.of(
            new SubjectViewSpec(AI_VIEW_ID, "iot-ai-resolution", TENANCY,
                "iot-triage:ai-resolution", null, "enqueuedAt", "ASC", null, Instant.now()),
            new SubjectViewSpec(OPERATOR_ASSISTED_VIEW_ID, "iot-operator-assisted", TENANCY,
                "iot-triage:operator-assisted", null, "enqueuedAt", "ASC", null, Instant.now()),
            new SubjectViewSpec(OPERATOR_MANUAL_VIEW_ID, "iot-operator-manual", TENANCY,
                "iot-triage:operator-manual", null, "enqueuedAt", "ASC", null, Instant.now())
        ));

        observer = new IoTCbrReEvaluationObserver();
        inject(observer, "queueService", queueService);
        inject(observer, "caseCache", caseCache);
        inject(observer, "retrievalService", retrievalService);
        inject(observer, "definitionRegistry", definitionRegistry);
        inject(observer, "viewStore", viewStore);
        inject(observer, "triageConfig", triageConfig);
        inject(observer, "tenancyId", TENANCY);
        inject(observer, "registry", meterRegistry);
        observer.init();
    }

    private static void inject(Object target, String fieldName, Object value) throws Exception {
        Field f = target.getClass().getDeclaredField(fieldName);
        f.setAccessible(true);
        f.set(target, value);
    }

    private CaseInstance caseInstance(UUID caseId, String caseType, double cbrBestSimilarity) {
        CaseInstance instance = new CaseInstance();
        instance.setUuid(caseId);
        CaseMetaModel meta = new CaseMetaModel();
        meta.setName(caseType);
        instance.setCaseMetaModel(meta);
        CaseContext ctx = mock(CaseContext.class);
        when(ctx.getOrDefault(any(), any())).thenReturn(Map.of(
            "cbrBestSimilarity", cbrBestSimilarity,
            "cbrOutcomeConsistency", 0.9,
            "cbrMatchCount", 3
        ));
        when(ctx.getOrDefault("working", Map.of())).thenReturn(Map.of(
            "deviceClass", "thermostat",
            "roomType", "living_room",
            "cbrBestSimilarity", cbrBestSimilarity,
            "cbrOutcomeConsistency", 0.9,
            "cbrMatchCount", 3
        ));
        instance.setCaseContext(ctx);
        return instance;
    }

    private CaseQueueEntry aiQueueEntry(UUID caseId, QueueEntryStatus status) {
        CaseQueueEntry entry = new CaseQueueEntry(UUID.randomUUID(), caseId, TENANCY,
            AI_VIEW_ID, "iot-ai-resolution", status, Instant.now());
        if (status == QueueEntryStatus.CLAIMED) {
            entry.setAssignedTo("iot-ai-agent");
            entry.setClaimedAt(Instant.now());
        }
        return entry;
    }

    private void setupStandardMocks(UUID caseId, CaseInstance instance, CaseQueueEntry entry) {
        CaseDefinition def = mock(CaseDefinition.class);
        when(def.getCbrConfig()).thenReturn(CbrConfig.builder()
            .domain("iot").caseType("hvac-anomaly")
            .featureExtractor(ctx -> Map.of()).build());
        when(caseCache.get(caseId)).thenReturn(instance);
        when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));
        when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    }

    private ResolutionSuggestion suggestion(double similarity, String outcome) {
        return new ResolutionSuggestion(
            UUID.randomUUID().toString(), similarity,
            "Test problem", "Test solution", outcome, 0.9,
            Map.of(), Map.of(), List.of());
    }

    private double counterValue(String name, String... tags) {
        var counter = meterRegistry.find(name).tags(tags).counter();
        return counter != null ? counter.count() : 0.0;
    }
}
```

- [ ] **Step 2: Write test — band drop HIGH → MEDIUM escalates to operator-assisted**

Add to `IoTCbrReEvaluationObserverTest`:

```java
@Test
void bandDropHighToMedium_escalatesToOperatorAssisted() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", 0.90);
    CaseQueueEntry entry = aiQueueEntry(caseId, QueueEntryStatus.PENDING);
    setupStandardMocks(caseId, instance, entry);

    // CBR retrieval now returns medium-band results
    when(retrievalService.retrieve(any(), any(), eq(TENANCY)))
        .thenReturn(List.of(suggestion(0.65, "RESOLVED")));

    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(queueService).escalate(entry.getId(), TENANCY, OPERATOR_ASSISTED_VIEW_ID);
    assertEquals(1.0, counterValue("casehub.iot.ai.resolution.reevaluation.rerouted",
        "from.band", "high", "to.band", "medium", "target.view", "iot-operator-assisted"));
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#bandDropHighToMedium_escalatesToOperatorAssisted -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `IoTCbrReEvaluationObserver` does not exist yet.

- [ ] **Step 4: Write IoTCbrReEvaluationObserver implementation**

Create `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTCbrReEvaluationObserver.java`:

```java
package io.casehub.iot.webapp.app.resolution;

import io.casehub.api.context.CaseContext;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.engine.common.spi.event.CaseContextUpdatedEvent;
import io.casehub.engine.queue.model.CaseQueueEntry;
import io.casehub.engine.queue.service.CaseQueueService;
import io.casehub.iot.webapp.app.triage.IoTTriageConfig;
import io.casehub.iot.webapp.cbr.IoTCbrRetrievalService;
import io.casehub.iot.webapp.cbr.ResolutionSuggestion;
import io.casehub.platform.api.view.SubjectViewSpec;
import io.casehub.platform.api.view.SubjectViewStore;
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;
import java.util.stream.Collectors;

@ApplicationScoped
public class IoTCbrReEvaluationObserver {

    private static final Logger LOG = Logger.getLogger(IoTCbrReEvaluationObserver.class);
    static final long DEBOUNCE_SECONDS = 30;
    static final double MEDIUM_FLOOR_SIMILARITY = 0.5;

    @Inject CaseQueueService queueService;
    @Inject CaseInstanceCache caseCache;
    @Inject IoTCbrRetrievalService retrievalService;
    @Inject CaseDefinitionRegistry definitionRegistry;
    @Inject SubjectViewStore viewStore;
    @Inject IoTTriageConfig triageConfig;
    @Inject MeterRegistry registry;

    @Inject @ConfigProperty(name = "casehub.iot.tenancy-id")
    String tenancyId;

    private UUID aiResolutionViewId;
    private UUID operatorAssistedViewId;
    private UUID operatorManualViewId;
    private final ConcurrentHashMap<UUID, Instant> lastReEvaluation = new ConcurrentHashMap<>();

    @PostConstruct
    void init() {
        List<SubjectViewSpec> views = viewStore.findByTenancy(tenancyId);
        aiResolutionViewId = resolveView(views, "iot-ai-resolution");
        operatorAssistedViewId = resolveView(views, "iot-operator-assisted");
        operatorManualViewId = resolveView(views, "iot-operator-manual");
    }

    private static UUID resolveView(List<SubjectViewSpec> views, String name) {
        return views.stream()
                    .filter(v -> name.equals(v.name()))
                    .map(SubjectViewSpec::id)
                    .findFirst()
                    .orElse(null);
    }

    public void onContextUpdated(@ObservesAsync CaseContextUpdatedEvent event) {
        if (aiResolutionViewId == null || operatorAssistedViewId == null
            || operatorManualViewId == null) {
            return;
        }
        if (!"working".equals(event.changedLayer())) {
            return;
        }

        UUID caseId = event.caseId();

        Instant lastEval = lastReEvaluation.get(caseId);
        if (lastEval != null && lastEval.plusSeconds(DEBOUNCE_SECONDS).isAfter(Instant.now())) {
            registry.counter("casehub.iot.ai.resolution.reevaluation.debounced").increment();
            return;
        }

        List<CaseQueueEntry> aiEntries = queueService.findByView(aiResolutionViewId, tenancyId);
        CaseQueueEntry matchingEntry = aiEntries.stream()
            .filter(e -> caseId.equals(e.getCaseId()))
            .findFirst()
            .orElse(null);

        if (matchingEntry == null) {
            lastReEvaluation.remove(caseId);
            return;
        }

        CaseInstance instance = caseCache.get(caseId);
        if (instance == null) {
            return;
        }

        String caseType = instance.getCaseMetaModel().getName();
        var defOpt = definitionRegistry.findByName(caseType);
        if (defOpt.isEmpty()) {
            return;
        }

        CbrConfig cbrConfig = defOpt.get().getCbrConfig();
        if (cbrConfig == null) {
            return;
        }

        @SuppressWarnings("unchecked")
        Map<String, Object> features = extractFeatures(instance);
        List<ResolutionSuggestion> suggestions = retrievalService.retrieve(cbrConfig, features, tenancyId);
        lastReEvaluation.put(caseId, Instant.now());
        registry.counter("casehub.iot.ai.resolution.reevaluation.checked").increment();

        String oldBand = currentBand(instance);
        String newBand = computeBand(suggestions);

        if (oldBand.equals(newBand) || "high".equals(newBand)) {
            return;
        }

        UUID targetViewId;
        String targetViewName;
        if ("medium".equals(newBand)) {
            targetViewId = operatorAssistedViewId;
            targetViewName = "iot-operator-assisted";
        } else {
            targetViewId = operatorManualViewId;
            targetViewName = "iot-operator-manual";
        }

        try {
            queueService.escalate(matchingEntry.getId(), tenancyId, targetViewId);
            lastReEvaluation.remove(caseId);
            updateCbrStats(instance, suggestions);
            LOG.infof("CBR re-evaluation: case %s re-routed from ai-resolution to %s "
                      + "(similarity %.2f → %.2f)", caseId, targetViewName,
                      bestSimilarity(instance), bestSimilarity(suggestions));
            registry.counter("casehub.iot.ai.resolution.reevaluation.rerouted",
                "from.band", oldBand, "to.band", newBand,
                "target.view", targetViewName).increment();
        } catch (IllegalStateException e) {
            LOG.debugf("Re-routing case %s failed (already moved): %s", caseId, e.getMessage());
        }
    }

    @SuppressWarnings("unchecked")
    private Map<String, Object> extractFeatures(CaseInstance instance) {
        Object working = instance.getCaseContext().getOrDefault("working", Map.of());
        if (working instanceof Map) {
            return (Map<String, Object>) working;
        }
        return Map.of();
    }

    String computeBand(List<ResolutionSuggestion> suggestions) {
        if (suggestions == null || suggestions.isEmpty()) {
            return "low";
        }
        double bestSim = suggestions.stream()
            .mapToDouble(ResolutionSuggestion::similarityScore)
            .max().orElse(0.0);
        double consistency = computeOutcomeConsistency(suggestions);

        if (bestSim >= triageConfig.aiMinSimilarity()
            && consistency >= triageConfig.aiMinConsistency()) {
            return "high";
        }
        if (bestSim >= MEDIUM_FLOOR_SIMILARITY) {
            return "medium";
        }
        return "low";
    }

    private String currentBand(CaseInstance instance) {
        Map<String, Object> working = extractFeatures(instance);
        Object simObj = working.get("cbrBestSimilarity");
        Object conObj = working.get("cbrOutcomeConsistency");
        double sim = simObj instanceof Number n ? n.doubleValue() : 0.0;
        double con = conObj instanceof Number n ? n.doubleValue() : 0.0;

        if (sim >= triageConfig.aiMinSimilarity()
            && con >= triageConfig.aiMinConsistency()) {
            return "high";
        }
        if (sim >= MEDIUM_FLOOR_SIMILARITY) {
            return "medium";
        }
        return "low";
    }

    private static double computeOutcomeConsistency(List<ResolutionSuggestion> suggestions) {
        Map<String, Long> freq = suggestions.stream()
            .map(ResolutionSuggestion::outcome)
            .filter(Objects::nonNull)
            .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
        if (freq.isEmpty()) {
            return 0.0;
        }
        return (double) Collections.max(freq.values()) / suggestions.size();
    }

    private void updateCbrStats(CaseInstance instance, List<ResolutionSuggestion> suggestions) {
        CaseContext ctx = instance.getCaseContext();
        ctx.set("cbrBestSimilarity", bestSimilarity(suggestions));
        ctx.set("cbrMatchCount", suggestions.size());
        ctx.set("cbrOutcomeConsistency", computeOutcomeConsistency(suggestions));
    }

    private static double bestSimilarity(List<ResolutionSuggestion> suggestions) {
        return suggestions.stream()
            .mapToDouble(ResolutionSuggestion::similarityScore)
            .max().orElse(0.0);
    }

    private double bestSimilarity(CaseInstance instance) {
        Object v = extractFeatures(instance).get("cbrBestSimilarity");
        return v instanceof Number n ? n.doubleValue() : 0.0;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#bandDropHighToMedium_escalatesToOperatorAssisted`
Expected: PASS

- [ ] **Step 6: Write test — band drop HIGH → LOW escalates to operator-manual**

```java
@Test
void bandDropHighToLow_escalatesToOperatorManual() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", 0.90);
    CaseQueueEntry entry = aiQueueEntry(caseId, QueueEntryStatus.PENDING);
    setupStandardMocks(caseId, instance, entry);

    when(retrievalService.retrieve(any(), any(), eq(TENANCY)))
        .thenReturn(List.of(suggestion(0.30, "RESOLVED")));

    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(queueService).escalate(entry.getId(), TENANCY, OPERATOR_MANUAL_VIEW_ID);
    assertEquals(1.0, counterValue("casehub.iot.ai.resolution.reevaluation.rerouted",
        "from.band", "high", "to.band", "low", "target.view", "iot-operator-manual"));
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#bandDropHighToLow_escalatesToOperatorManual`
Expected: PASS

- [ ] **Step 8: Write test — band unchanged is no-op**

```java
@Test
void bandUnchanged_noEscalation() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", 0.90);
    CaseQueueEntry entry = aiQueueEntry(caseId, QueueEntryStatus.PENDING);
    setupStandardMocks(caseId, instance, entry);

    // CBR retrieval still returns high-band results
    when(retrievalService.retrieve(any(), any(), eq(TENANCY)))
        .thenReturn(List.of(
            suggestion(0.92, "RESOLVED"),
            suggestion(0.88, "RESOLVED")));

    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(queueService, never()).escalate(any(), any(), any());
    assertEquals(1.0, counterValue("casehub.iot.ai.resolution.reevaluation.checked"));
    assertEquals(0.0, counterValue("casehub.iot.ai.resolution.reevaluation.rerouted",
        "from.band", "high", "to.band", "medium", "target.view", "iot-operator-assisted"));
}
```

- [ ] **Step 9: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#bandUnchanged_noEscalation`
Expected: PASS

- [ ] **Step 10: Write test — non-working layer change filtered out**

```java
@Test
void nonWorkingLayerChange_filteredOut() {
    UUID caseId = UUID.randomUUID();

    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "episodic", TENANCY));

    verify(queueService, never()).findByView(any(), any());
    verify(caseCache, never()).get(any());
}
```

- [ ] **Step 11: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#nonWorkingLayerChange_filteredOut`
Expected: PASS

- [ ] **Step 12: Write test — case not in AI queue filtered out**

```java
@Test
void caseNotInAiQueue_filteredOut() {
    UUID caseId = UUID.randomUUID();
    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());

    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(caseCache, never()).get(any());
    verify(retrievalService, never()).retrieve(any(), any(), any());
}
```

- [ ] **Step 13: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#caseNotInAiQueue_filteredOut`
Expected: PASS

- [ ] **Step 14: Write test — debounce coalesces rapid events**

```java
@Test
void debounce_coalesces_rapidEvents() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", 0.90);
    CaseQueueEntry entry = aiQueueEntry(caseId, QueueEntryStatus.PENDING);
    setupStandardMocks(caseId, instance, entry);

    when(retrievalService.retrieve(any(), any(), eq(TENANCY)))
        .thenReturn(List.of(suggestion(0.92, "RESOLVED")));

    // First event passes through
    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));
    // Second event within 30s is debounced
    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(retrievalService, times(1)).retrieve(any(), any(), eq(TENANCY));
    assertEquals(1.0, counterValue("casehub.iot.ai.resolution.reevaluation.debounced"));
}
```

- [ ] **Step 15: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#debounce_coalesces_rapidEvents`
Expected: PASS

- [ ] **Step 16: Write test — PENDING entry re-routed**

```java
@Test
void pendingEntry_rerouted() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", 0.90);
    CaseQueueEntry entry = aiQueueEntry(caseId, QueueEntryStatus.PENDING);
    setupStandardMocks(caseId, instance, entry);

    when(retrievalService.retrieve(any(), any(), eq(TENANCY)))
        .thenReturn(List.of(suggestion(0.65, "RESOLVED")));

    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(queueService).escalate(entry.getId(), TENANCY, OPERATOR_ASSISTED_VIEW_ID);
}
```

- [ ] **Step 17: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#pendingEntry_rerouted`
Expected: PASS

- [ ] **Step 18: Write test — CLAIMED entry re-routed**

```java
@Test
void claimedEntry_rerouted() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", 0.90);
    CaseQueueEntry entry = aiQueueEntry(caseId, QueueEntryStatus.CLAIMED);
    setupStandardMocks(caseId, instance, entry);

    when(retrievalService.retrieve(any(), any(), eq(TENANCY)))
        .thenReturn(List.of(suggestion(0.65, "RESOLVED")));

    observer.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(queueService).escalate(entry.getId(), TENANCY, OPERATOR_ASSISTED_VIEW_ID);
}
```

- [ ] **Step 19: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#claimedEntry_rerouted`
Expected: PASS

- [ ] **Step 20: Write test — observer disabled when views not resolved**

```java
@Test
void viewsNotResolved_noErrors() throws Exception {
    // Create observer with empty view store
    IoTCbrReEvaluationObserver disabledObserver = new IoTCbrReEvaluationObserver();
    SubjectViewStore emptyStore = mock(SubjectViewStore.class);
    when(emptyStore.findByTenancy(TENANCY)).thenReturn(List.of());

    inject(disabledObserver, "queueService", queueService);
    inject(disabledObserver, "caseCache", caseCache);
    inject(disabledObserver, "retrievalService", retrievalService);
    inject(disabledObserver, "definitionRegistry", definitionRegistry);
    inject(disabledObserver, "viewStore", emptyStore);
    inject(disabledObserver, "triageConfig", triageConfig);
    inject(disabledObserver, "tenancyId", TENANCY);
    inject(disabledObserver, "registry", meterRegistry);
    disabledObserver.init();

    UUID caseId = UUID.randomUUID();
    disabledObserver.onContextUpdated(new CaseContextUpdatedEvent(caseId, "working", TENANCY));

    verify(queueService, never()).findByView(any(), any());
}
```

- [ ] **Step 21: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest#viewsNotResolved_noErrors`
Expected: PASS

- [ ] **Step 22: Run full test class**

Run: `mvn --batch-mode test -pl webapp -Dtest=IoTCbrReEvaluationObserverTest`
Expected: all 9 tests PASS

- [ ] **Step 23: Run full webapp module tests to check for regressions**

Run: `mvn --batch-mode test -pl webapp`
Expected: all existing tests still PASS

- [ ] **Step 24: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTCbrReEvaluationObserver.java webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTCbrReEvaluationObserverTest.java
git commit -m "feat(#82): IoTCbrReEvaluationObserver — event-driven CBR re-routing on context changes

Observes CaseContextUpdatedEvent for working layer changes on AI-queued
cases. Re-evaluates CBR similarity and escalates to operator queues when
the band drops. Filters by layer, queue membership, and 30s debounce.
Uses direct queueService.escalate() — same API as agent and timeout sweep.

Refs #82"
```
