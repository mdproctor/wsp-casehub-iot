# Queue Listing REST Endpoints Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #81 — Queue listing REST endpoints for AI resolution views
**Issue group:** #81

**Goal:** REST endpoints to display AI resolution queue entries enriched with
device context, CBR suggestions, and AI resolution state.

**Architecture:** List + detail pattern on `/api/resolution/queue`. List returns
`QueueEntrySummary` (queue state + device identity). Detail returns
`QueueEntryDetail` (full CBR + AI state). Response records in `webapp-api`
(Tier 1). Resource in `webapp`.

**Tech Stack:** Quarkus JAX-RS, CDI, JUnit 5 + Mockito

## Global Constraints

- Response records are pure data (no CDI, no JPA) — `webapp-api` module
- REST resource uses `@RolesAllowed("iot-viewer")` and `CurrentPrincipal.tenancyId()`
- Default status filter excludes REVOKED entries
- viewName resolved from cached viewId → name mapping (escalation sets viewName to null)
- Detail endpoint uses `CaseQueueEntryStore.findById()` with explicit tenancy verification

---

### Task 1: Response Records (webapp-api)

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/QueueEntrySummary.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/QueueEntryDetail.java`

**Interfaces:**
- Consumes: existing `ResolutionSuggestion`, `AiEscalationContext`, `ExecutedActionResult` from same package
- Produces: `QueueEntrySummary` and `QueueEntryDetail` records used by Task 2's `ResolutionQueueResource`

- [ ] **Step 1: Create QueueEntrySummary record**

```java
// webapp-api/src/main/java/io/casehub/iot/webapp/resolution/QueueEntrySummary.java
package io.casehub.iot.webapp.resolution;

import java.time.Instant;
import java.util.UUID;

public record QueueEntrySummary(
        UUID entryId,
        UUID caseId,
        String caseType,
        String viewName,
        String status,
        String assignedTo,
        Instant createdAt,
        Instant claimedAt,
        Instant escalatedAt,
        String previousViewName,
        String deviceId,
        String deviceClass,
        String roomType,
        String situationId
) {}
```

Use `String` for status (not `QueueEntryStatus` enum) — webapp-api has no
dependency on `casehub-engine-queue`. The resource converts at the boundary.

- [ ] **Step 2: Create QueueEntryDetail record**

```java
// webapp-api/src/main/java/io/casehub/iot/webapp/resolution/QueueEntryDetail.java
package io.casehub.iot.webapp.resolution;

import io.casehub.iot.webapp.cbr.ResolutionSuggestion;

import java.util.List;
import java.util.Map;

public record QueueEntryDetail(
        QueueEntrySummary entry,
        Map<String, Object> workingContext,
        List<ResolutionSuggestion> suggestions,
        AiEscalationContext escalationContext,
        List<ExecutedActionResult> executionResults
) {
    public QueueEntryDetail {
        suggestions = suggestions != null ? List.copyOf(suggestions) : List.of();
        executionResults = executionResults != null ? List.copyOf(executionResults) : List.of();
        workingContext = workingContext != null ? Map.copyOf(workingContext) : Map.of();
    }
}
```

- [ ] **Step 3: Build to verify compilation**

Run: `mvn -pl webapp-api compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add webapp-api/src/main/java/io/casehub/iot/webapp/resolution/QueueEntrySummary.java \
       webapp-api/src/main/java/io/casehub/iot/webapp/resolution/QueueEntryDetail.java
git commit -m "feat(#81): QueueEntrySummary and QueueEntryDetail response records"
```

---

### Task 2: ResolutionQueueResource + Tests

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/ResolutionQueueResource.java`
- Create: `webapp/src/test/java/io/casehub/iot/webapp/app/rest/ResolutionQueueResourceTest.java`

**Interfaces:**
- Consumes: `QueueEntrySummary`, `QueueEntryDetail` from Task 1;
  `CaseQueueService.findPending(UUID, String)`, `CaseQueueService.findByView(UUID, String)`,
  `CaseQueueEntryStore.findById(UUID)`, `CaseInstanceCache.get(UUID)`,
  `CaseDefinitionRegistry.findByName(String)`, `SubjectViewStore.findByTenancy(String)`,
  `IoTCbrRetrievalService.retrieve(CbrConfig, Map, String)`, `CurrentPrincipal.tenancyId()`
- Produces: REST endpoints `GET /api/resolution/queue` and `GET /api/resolution/queue/{entryId}`

- [ ] **Step 1: Write the test class skeleton with setUp**

```java
// webapp/src/test/java/io/casehub/iot/webapp/app/rest/ResolutionQueueResourceTest.java
package io.casehub.iot.webapp.app.rest;

import io.casehub.api.context.CaseContext;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.cbr.CbrConfig;
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
import io.casehub.iot.webapp.resolution.AiEscalationContext;
import io.casehub.iot.webapp.resolution.AiResolutionPlan;
import io.casehub.iot.webapp.resolution.Decision;
import io.casehub.iot.webapp.resolution.ExecutedActionResult;
import io.casehub.iot.webapp.resolution.PlannedActionSpec;
import io.casehub.iot.webapp.resolution.QueueEntryDetail;
import io.casehub.iot.webapp.resolution.QueueEntrySummary;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.view.SubjectViewSpec;
import io.casehub.platform.api.view.SubjectViewStore;
import jakarta.ws.rs.NotFoundException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Field;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class ResolutionQueueResourceTest {

    private static final String TENANCY = "test-tenant";
    private static final UUID AI_VIEW_ID = UUID.randomUUID();
    private static final UUID OPERATOR_VIEW_ID = UUID.randomUUID();

    private CaseQueueService queueService;
    private CaseQueueEntryStore entryStore;
    private CaseInstanceCache caseCache;
    private CaseDefinitionRegistry definitionRegistry;
    private SubjectViewStore viewStore;
    private IoTCbrRetrievalService retrievalService;
    private CurrentPrincipal principal;
    private ResolutionQueueResource resource;

    @BeforeEach
    void setUp() throws Exception {
        queueService = mock(CaseQueueService.class);
        entryStore = mock(CaseQueueEntryStore.class);
        caseCache = mock(CaseInstanceCache.class);
        definitionRegistry = mock(CaseDefinitionRegistry.class);
        viewStore = mock(SubjectViewStore.class);
        retrievalService = mock(IoTCbrRetrievalService.class);
        principal = mock(CurrentPrincipal.class);

        when(principal.tenancyId()).thenReturn(TENANCY);
        when(viewStore.findByTenancy(TENANCY)).thenReturn(List.of(
            new SubjectViewSpec(AI_VIEW_ID, "iot-ai-resolution", TENANCY,
                "iot-triage:ai-resolution", null, "enqueuedAt", "ASC", null, Instant.now()),
            new SubjectViewSpec(OPERATOR_VIEW_ID, "iot-operator-assisted", TENANCY,
                "iot-triage:operator-assisted", null, "enqueuedAt", "ASC", null, Instant.now())
        ));

        resource = new ResolutionQueueResource();
        inject(resource, "queueService", queueService);
        inject(resource, "entryStore", entryStore);
        inject(resource, "caseCache", caseCache);
        inject(resource, "definitionRegistry", definitionRegistry);
        inject(resource, "viewStore", viewStore);
        inject(resource, "retrievalService", retrievalService);
        inject(resource, "principal", principal);

        resource.init();
    }

    // --- helpers ---

    private CaseQueueEntry pendingEntry(UUID caseId, UUID viewId, String viewName) {
        return new CaseQueueEntry(UUID.randomUUID(), caseId, TENANCY,
            viewId, viewName, QueueEntryStatus.PENDING, Instant.now());
    }

    private CaseQueueEntry claimedEntry(UUID caseId, UUID viewId, String viewName) {
        CaseQueueEntry e = new CaseQueueEntry(UUID.randomUUID(), caseId, TENANCY,
            viewId, viewName, QueueEntryStatus.CLAIMED, Instant.now());
        e.setAssignedTo("iot-ai-agent");
        e.setClaimedAt(Instant.now());
        return e;
    }

    private CaseQueueEntry escalatedEntry(UUID caseId) {
        CaseQueueEntry e = new CaseQueueEntry(UUID.randomUUID(), caseId, TENANCY,
            OPERATOR_VIEW_ID, null, QueueEntryStatus.PENDING, Instant.now());
        e.setPreviousViewId(AI_VIEW_ID);
        e.setPreviousViewName("iot-ai-resolution");
        e.setEscalatedAt(Instant.now());
        return e;
    }

    private CaseInstance caseInstance(UUID caseId, String caseType,
                                      Map<String, Object> working) {
        CaseInstance instance = new CaseInstance();
        instance.setUuid(caseId);
        CaseMetaModel meta = new CaseMetaModel();
        meta.setName(caseType);
        instance.setCaseMetaModel(meta);
        CaseContext ctx = mock(CaseContext.class);
        when(ctx.getOrDefault("working", Map.of())).thenReturn(working);
        when(ctx.getOrDefault(eq("working"), any())).thenReturn(working);
        instance.setCaseContext(ctx);
        return instance;
    }

    private Map<String, Object> workingContext() {
        return Map.of(
            "deviceId", "thermo-001",
            "deviceClass", "Thermostat",
            "roomType", "LivingRoom",
            "situationId", "hvac-anomaly-001"
        );
    }

    private static void inject(Object target, String fieldName, Object value)
            throws Exception {
        Field f = target.getClass().getDeclaredField(fieldName);
        f.setAccessible(true);
        f.set(target, value);
    }
}
```

- [ ] **Step 2: Write test — list both views combined**

Add to `ResolutionQueueResourceTest`:

```java
@Test
void list_bothViewsCombined_returnsEntriesFromBothViews() {
    UUID caseId1 = UUID.randomUUID();
    UUID caseId2 = UUID.randomUUID();
    CaseQueueEntry aiEntry = pendingEntry(caseId1, AI_VIEW_ID, "iot-ai-resolution");
    CaseQueueEntry opEntry = escalatedEntry(caseId2);

    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of(aiEntry));
    when(queueService.findByView(OPERATOR_VIEW_ID, TENANCY)).thenReturn(List.of(opEntry));
    when(caseCache.get(caseId1)).thenReturn(caseInstance(caseId1, "hvac-anomaly", workingContext()));
    when(caseCache.get(caseId2)).thenReturn(caseInstance(caseId2, "leak-detected", workingContext()));

    List<QueueEntrySummary> result = resource.list(null, null);

    assertEquals(2, result.size());
    assertTrue(result.stream().anyMatch(s -> "iot-ai-resolution".equals(s.viewName())));
    assertTrue(result.stream().anyMatch(s -> "iot-operator-assisted".equals(s.viewName())));
}
```

- [ ] **Step 3: Write test — view filter**

```java
@Test
void list_viewFilter_returnsOnlyMatchingView() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry aiEntry = pendingEntry(caseId, AI_VIEW_ID, "iot-ai-resolution");

    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of(aiEntry));
    when(caseCache.get(caseId)).thenReturn(caseInstance(caseId, "hvac-anomaly", workingContext()));

    List<QueueEntrySummary> result = resource.list("ai-resolution", null);

    assertEquals(1, result.size());
    assertEquals("iot-ai-resolution", result.get(0).viewName());
    verify(queueService, never()).findByView(OPERATOR_VIEW_ID, TENANCY);
}
```

- [ ] **Step 4: Write test — status filter uses findPending**

```java
@Test
void list_statusPending_usesFindPending() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId, AI_VIEW_ID, "iot-ai-resolution");

    when(queueService.findPending(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.findPending(OPERATOR_VIEW_ID, TENANCY)).thenReturn(List.of());
    when(caseCache.get(caseId)).thenReturn(caseInstance(caseId, "hvac-anomaly", workingContext()));

    List<QueueEntrySummary> result = resource.list(null, "PENDING");

    assertEquals(1, result.size());
    verify(queueService, never()).findByView(any(), any());
}
```

- [ ] **Step 5: Write test — default excludes REVOKED**

```java
@Test
void list_defaultFilter_excludesRevoked() {
    UUID caseId1 = UUID.randomUUID();
    UUID caseId2 = UUID.randomUUID();
    CaseQueueEntry pending = pendingEntry(caseId1, AI_VIEW_ID, "iot-ai-resolution");
    CaseQueueEntry revoked = new CaseQueueEntry(UUID.randomUUID(), caseId2, TENANCY,
        AI_VIEW_ID, "iot-ai-resolution", QueueEntryStatus.REVOKED, Instant.now());

    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of(pending, revoked));
    when(queueService.findByView(OPERATOR_VIEW_ID, TENANCY)).thenReturn(List.of());
    when(caseCache.get(caseId1)).thenReturn(caseInstance(caseId1, "hvac-anomaly", workingContext()));

    List<QueueEntrySummary> result = resource.list(null, null);

    assertEquals(1, result.size());
    assertEquals("PENDING", result.get(0).status());
}
```

- [ ] **Step 6: Write test — enrichment from working context**

```java
@Test
void list_enrichment_extractsDeviceFieldsFromWorkingContext() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId, AI_VIEW_ID, "iot-ai-resolution");
    Map<String, Object> working = Map.of(
        "deviceId", "sensor-042",
        "deviceClass", "TemperatureSensor",
        "roomType", "Bedroom",
        "situationId", "temp-spike-007"
    );

    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.findByView(OPERATOR_VIEW_ID, TENANCY)).thenReturn(List.of());
    when(caseCache.get(caseId)).thenReturn(caseInstance(caseId, "temp-anomaly", working));

    List<QueueEntrySummary> result = resource.list(null, null);

    QueueEntrySummary summary = result.get(0);
    assertEquals("sensor-042", summary.deviceId());
    assertEquals("TemperatureSensor", summary.deviceClass());
    assertEquals("Bedroom", summary.roomType());
    assertEquals("temp-spike-007", summary.situationId());
    assertEquals("temp-anomaly", summary.caseType());
}
```

- [ ] **Step 7: Write test — case not in cache**

```java
@Test
void list_caseNotInCache_returnsEntryWithNullCaseFields() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId, AI_VIEW_ID, "iot-ai-resolution");

    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(queueService.findByView(OPERATOR_VIEW_ID, TENANCY)).thenReturn(List.of());
    when(caseCache.get(caseId)).thenReturn(null);

    List<QueueEntrySummary> result = resource.list(null, null);

    assertEquals(1, result.size());
    QueueEntrySummary summary = result.get(0);
    assertNull(summary.caseType());
    assertNull(summary.deviceId());
    assertNull(summary.deviceClass());
}
```

- [ ] **Step 8: Write test — view not configured**

```java
@Test
void list_viewNotConfigured_returnsEmptyList() throws Exception {
    when(viewStore.findByTenancy(TENANCY)).thenReturn(List.of());
    resource.init();

    List<QueueEntrySummary> result = resource.list(null, null);

    assertTrue(result.isEmpty());
}
```

- [ ] **Step 9: Write test — viewName null on escalated entry resolved from mapping**

```java
@Test
void list_escalatedEntryWithNullViewName_resolvesFromMapping() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = escalatedEntry(caseId);

    when(queueService.findByView(AI_VIEW_ID, TENANCY)).thenReturn(List.of());
    when(queueService.findByView(OPERATOR_VIEW_ID, TENANCY)).thenReturn(List.of(entry));
    when(caseCache.get(caseId)).thenReturn(caseInstance(caseId, "hvac-anomaly", workingContext()));

    List<QueueEntrySummary> result = resource.list(null, null);

    assertEquals(1, result.size());
    assertEquals("iot-operator-assisted", result.get(0).viewName());
    assertEquals("iot-ai-resolution", result.get(0).previousViewName());
}
```

- [ ] **Step 10: Write test — detail happy path**

```java
@Test
void detail_happyPath_returnsFullEnrichment() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = claimedEntry(caseId, AI_VIEW_ID, "iot-ai-resolution");
    Map<String, Object> working = workingContext();
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", working);

    CaseDefinition def = mock(CaseDefinition.class);
    CbrConfig cbrConfig = CbrConfig.builder().domain("iot").caseType("hvac-anomaly")
        .featureExtractor(ctx -> Map.of()).build();
    when(def.getCbrConfig()).thenReturn(cbrConfig);

    List<ExecutedActionResult> execResults = List.of(
        new ExecutedActionResult(
            new PlannedActionSpec("SET_TEMPERATURE", "thermo-001",
                Map.of("target", 22), "Reset temp"),
            true, "SUCCESS"));

    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(entry));
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));
    when(retrievalService.retrieve(eq(cbrConfig), any(), eq(TENANCY)))
        .thenReturn(List.of(new ResolutionSuggestion(
            "past-case-1", 0.92, "HVAC overheat", "Lower temp",
            "RESOLVED", 0.85, Map.of(), Map.of(), List.of())));
    when(instance.getCaseContext().get("aiResolutionResults")).thenReturn(execResults);
    when(instance.getCaseContext().get("aiEscalationContext")).thenReturn(null);

    QueueEntryDetail detail = resource.detail(entry.getId());

    assertNotNull(detail);
    assertEquals("thermo-001", detail.entry().deviceId());
    assertEquals(1, detail.suggestions().size());
    assertEquals(0.92, detail.suggestions().get(0).similarityScore());
    assertEquals(1, detail.executionResults().size());
    assertTrue(detail.executionResults().get(0).succeeded());
}
```

- [ ] **Step 11: Write test — detail entry not found**

```java
@Test
void detail_entryNotFound_throws404() {
    UUID entryId = UUID.randomUUID();
    when(entryStore.findById(entryId)).thenReturn(Optional.empty());

    assertThrows(NotFoundException.class, () -> resource.detail(entryId));
}
```

- [ ] **Step 12: Write test — detail tenancy mismatch**

```java
@Test
void detail_tenancyMismatch_throws404() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = new CaseQueueEntry(UUID.randomUUID(), caseId,
        "other-tenant", AI_VIEW_ID, "iot-ai-resolution",
        QueueEntryStatus.PENDING, Instant.now());

    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(entry));

    assertThrows(NotFoundException.class, () -> resource.detail(entry.getId()));
}
```

- [ ] **Step 13: Write test — detail no AI state**

```java
@Test
void detail_noAiState_returnsNullResolutionFields() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId, AI_VIEW_ID, "iot-ai-resolution");
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", workingContext());

    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(entry));
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.empty());

    QueueEntryDetail detail = resource.detail(entry.getId());

    assertNull(detail.escalationContext());
    assertTrue(detail.suggestions().isEmpty());
    assertTrue(detail.executionResults().isEmpty());
}
```

- [ ] **Step 14: Write test — detail no CBR config**

```java
@Test
void detail_noCbrConfig_returnsEmptySuggestions() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = pendingEntry(caseId, AI_VIEW_ID, "iot-ai-resolution");
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", workingContext());

    CaseDefinition def = mock(CaseDefinition.class);
    when(def.getCbrConfig()).thenReturn(null);

    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(entry));
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.of(def));

    QueueEntryDetail detail = resource.detail(entry.getId());

    assertTrue(detail.suggestions().isEmpty());
    verifyNoInteractions(retrievalService);
}
```

- [ ] **Step 15: Write test — detail escalated entry**

```java
@Test
void detail_escalatedEntry_hasEscalationContext() {
    UUID caseId = UUID.randomUUID();
    CaseQueueEntry entry = escalatedEntry(caseId);
    CaseInstance instance = caseInstance(caseId, "hvac-anomaly", workingContext());

    AiEscalationContext escalation = new AiEscalationContext(
        "Risk gate: LOCK requires approval", List.of(), "Analysis text",
        List.of(new PlannedActionSpec("LOCK", "lock-001", Map.of(), "Secure")),
        null);

    when(entryStore.findById(entry.getId())).thenReturn(Optional.of(entry));
    when(caseCache.get(caseId)).thenReturn(instance);
    when(definitionRegistry.findByName("hvac-anomaly")).thenReturn(Optional.empty());
    when(instance.getCaseContext().get("aiEscalationContext")).thenReturn(escalation);

    QueueEntryDetail detail = resource.detail(entry.getId());

    assertEquals("iot-operator-assisted", detail.entry().viewName());
    assertEquals("iot-ai-resolution", detail.entry().previousViewName());
    assertNotNull(detail.escalationContext());
    assertEquals("Risk gate: LOCK requires approval", detail.escalationContext().reason());
}
```

- [ ] **Step 16: Run tests to verify they fail**

Run: `mvn -pl webapp test -Dtest=ResolutionQueueResourceTest -q`
Expected: COMPILATION FAILURE (ResolutionQueueResource does not exist)

- [ ] **Step 17: Implement ResolutionQueueResource**

```java
// webapp/src/main/java/io/casehub/iot/webapp/app/rest/ResolutionQueueResource.java
package io.casehub.iot.webapp.app.rest;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.engine.queue.model.CaseQueueEntry;
import io.casehub.engine.queue.model.QueueEntryStatus;
import io.casehub.engine.queue.service.CaseQueueService;
import io.casehub.engine.queue.spi.CaseQueueEntryStore;
import io.casehub.iot.webapp.cbr.IoTCbrRetrievalService;
import io.casehub.iot.webapp.cbr.ResolutionSuggestion;
import io.casehub.iot.webapp.resolution.AiEscalationContext;
import io.casehub.iot.webapp.resolution.ExecutedActionResult;
import io.casehub.iot.webapp.resolution.QueueEntryDetail;
import io.casehub.iot.webapp.resolution.QueueEntrySummary;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.view.SubjectViewSpec;
import io.casehub.platform.api.view.SubjectViewStore;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.NotFoundException;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

@Path("/api/resolution/queue")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class ResolutionQueueResource {

    @Inject CaseQueueService queueService;
    @Inject CaseQueueEntryStore entryStore;
    @Inject CaseInstanceCache caseCache;
    @Inject CaseDefinitionRegistry definitionRegistry;
    @Inject SubjectViewStore viewStore;
    @Inject IoTCbrRetrievalService retrievalService;
    @Inject CurrentPrincipal principal;

    private UUID aiResolutionViewId;
    private UUID operatorAssistedViewId;
    private Map<UUID, String> viewNameMapping;

    @PostConstruct
    void init() {
        viewNameMapping = new HashMap<>();
        List<SubjectViewSpec> views = viewStore.findByTenancy(principal.tenancyId());
        for (SubjectViewSpec view : views) {
            viewNameMapping.put(view.id(), view.name());
            if ("iot-ai-resolution".equals(view.name())) {
                aiResolutionViewId = view.id();
            } else if ("iot-operator-assisted".equals(view.name())) {
                operatorAssistedViewId = view.id();
            }
        }
    }

    @GET
    @RolesAllowed("iot-viewer")
    public List<QueueEntrySummary> list(
            @QueryParam("view") String view,
            @QueryParam("status") String status) {

        List<UUID> viewIds = resolveViewIds(view);
        if (viewIds.isEmpty()) {
            return List.of();
        }

        List<CaseQueueEntry> entries = new ArrayList<>();
        for (UUID viewId : viewIds) {
            if ("PENDING".equals(status)) {
                entries.addAll(queueService.findPending(viewId, principal.tenancyId()));
            } else {
                entries.addAll(queueService.findByView(viewId, principal.tenancyId()));
            }
        }

        if (status != null && !"PENDING".equals(status)) {
            QueueEntryStatus filterStatus = QueueEntryStatus.valueOf(status);
            entries = entries.stream()
                .filter(e -> e.getStatus() == filterStatus)
                .toList();
        } else if (status == null) {
            entries = entries.stream()
                .filter(e -> e.getStatus() != QueueEntryStatus.REVOKED)
                .toList();
        }

        return entries.stream().map(this::toSummary).toList();
    }

    @GET
    @Path("/{entryId}")
    @RolesAllowed("iot-viewer")
    public QueueEntryDetail detail(@PathParam("entryId") UUID entryId) {
        CaseQueueEntry entry = entryStore.findById(entryId)
            .filter(e -> principal.tenancyId().equals(e.getTenancyId()))
            .orElseThrow(() -> new NotFoundException("Queue entry not found: " + entryId));

        QueueEntrySummary summary = toSummary(entry);

        CaseInstance instance = caseCache.get(entry.getCaseId());
        Map<String, Object> workingContext = Map.of();
        List<ResolutionSuggestion> suggestions = List.of();
        AiEscalationContext escalation = null;
        List<ExecutedActionResult> results = List.of();

        if (instance != null) {
            @SuppressWarnings("unchecked")
            Map<String, Object> working = (Map<String, Object>)
                instance.getCaseContext().getOrDefault("working", Map.of());
            workingContext = working;

            String caseType = instance.getCaseMetaModel().getName();
            Optional<CaseDefinition> defOpt = definitionRegistry.findByName(caseType);
            if (defOpt.isPresent()) {
                CbrConfig cbrConfig = defOpt.get().getCbrConfig();
                if (cbrConfig != null) {
                    Map<String, Object> features = cbrConfig.featureExtractor()
                        .apply(instance.getCaseContext());
                    suggestions = retrievalService.retrieve(
                        cbrConfig, features, principal.tenancyId());
                }
            }

            escalation = (AiEscalationContext) instance.getCaseContext()
                .get("aiEscalationContext");
            @SuppressWarnings("unchecked")
            List<ExecutedActionResult> execResults = (List<ExecutedActionResult>)
                instance.getCaseContext().get("aiResolutionResults");
            if (execResults != null) {
                results = execResults;
            }
        }

        return new QueueEntryDetail(summary, workingContext, suggestions,
            escalation, results);
    }

    private QueueEntrySummary toSummary(CaseQueueEntry entry) {
        String resolvedViewName = entry.getViewName() != null
            ? entry.getViewName()
            : viewNameMapping.getOrDefault(entry.getViewId(), null);

        CaseInstance instance = caseCache.get(entry.getCaseId());

        String caseType = null;
        String deviceId = null;
        String deviceClass = null;
        String roomType = null;
        String situationId = null;

        if (instance != null) {
            caseType = instance.getCaseMetaModel().getName();
            @SuppressWarnings("unchecked")
            Map<String, Object> working = (Map<String, Object>)
                instance.getCaseContext().getOrDefault("working", Map.of());
            deviceId = (String) working.get("deviceId");
            deviceClass = (String) working.get("deviceClass");
            roomType = (String) working.get("roomType");
            situationId = (String) working.get("situationId");
        }

        return new QueueEntrySummary(
            entry.getId(),
            entry.getCaseId(),
            caseType,
            resolvedViewName,
            entry.getStatus().name(),
            entry.getAssignedTo(),
            entry.getCreatedAt(),
            entry.getClaimedAt(),
            entry.getEscalatedAt(),
            entry.getPreviousViewName(),
            deviceId,
            deviceClass,
            roomType,
            situationId
        );
    }

    private List<UUID> resolveViewIds(String viewFilter) {
        if (viewFilter == null) {
            List<UUID> ids = new ArrayList<>();
            if (aiResolutionViewId != null) ids.add(aiResolutionViewId);
            if (operatorAssistedViewId != null) ids.add(operatorAssistedViewId);
            return ids;
        }
        return switch (viewFilter) {
            case "ai-resolution" -> aiResolutionViewId != null
                ? List.of(aiResolutionViewId) : List.of();
            case "operator-assisted" -> operatorAssistedViewId != null
                ? List.of(operatorAssistedViewId) : List.of();
            default -> List.of();
        };
    }
}
```

- [ ] **Step 18: Run tests to verify they pass**

Run: `mvn -pl webapp test -Dtest=ResolutionQueueResourceTest -q`
Expected: All 14 tests PASS

- [ ] **Step 19: Run full module test suite to check for regressions**

Run: `mvn -pl webapp-api,webapp test -q`
Expected: BUILD SUCCESS

- [ ] **Step 20: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/rest/ResolutionQueueResource.java \
       webapp/src/test/java/io/casehub/iot/webapp/app/rest/ResolutionQueueResourceTest.java
git commit -m "feat(#81): ResolutionQueueResource — list and detail endpoints with triage metadata"
```
