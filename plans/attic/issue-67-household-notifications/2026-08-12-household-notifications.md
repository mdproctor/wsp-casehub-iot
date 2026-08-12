# Household Notifications via Platform Subscription Engine — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #67 — feat: household notifications via platform subscription engine + connector bridge
**Issue group:** #67

**Goal:** Wire IoT situation activations into the platform subscription engine, replacing the stub `HouseholdNotificationWorkerFunction` with real subscription-based notification delivery.

**Architecture:** `IoTSituationEvent` in iot-api implements `SubscribableEvent`. A CDI observer in webapp listens for `SituationChangeEvent` (TRIGGERED/RESOLVED), constructs `IoTSituationEvent`, and pushes it into a platform-global DataSource registered at startup. The `household-notification` worker is removed from all case descriptors.

**Tech Stack:** Quarkus CDI, casehub-platform-api (SubscribableEvent, DataSourceRegistry, DataSource, ObjectType, Path), casehub-ras-api (SituationChangeEvent)

## Global Constraints

- iot-api is a **public API surface** — semver discipline, no breaking changes without major bump
- All SPIs are **blocking** — designed for virtual threads per ADR-0005
- Single tenancy property: `casehub.iot.tenancy-id`
- `casehub-platform-api` is already a direct dependency of iot-api (verified in pom.xml)
- Use `io.casehub.platform.api.path.Path.of("segment1", "segment2")` — varargs segments, NOT slash-delimited strings
- `@JsonIgnore` on `tenancyId()` to match `WorkItemLifecycleEvent` pattern
- Use IntelliJ MCP tools (`ide_insert_member`, `ide_replace_member`, `ide_refactor_safe_delete`) for all code operations — never bash for source files

---

### Task 1: IoTSituationEvent in iot-api

**Files:**
- Create: `api/src/main/java/io/casehub/iot/api/IoTSituationEvent.java`
- Test: `api/src/test/java/io/casehub/iot/api/IoTSituationEventTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.subscription.SubscribableEvent` (type(), tenancyId())
- Produces: `IoTSituationEvent(String situationId, String changeType, String deviceId, String tenancyId, Map<String, Object> metadata, Instant occurredAt)` — used by Task 2's observer

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.iot.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class IoTSituationEventTest {

    private static final Instant NOW = Instant.parse("2026-08-12T10:00:00Z");
    private static final ObjectMapper MAPPER = new ObjectMapper().registerModule(new JavaTimeModule());

    @Test
    void triggeredTypeIncludesSituationId() {
        var event = new IoTSituationEvent(
                "temperature-threshold", "triggered", "sensor.outdoor",
                "tenant-1", Map.of("temperature", 42.0), NOW);
        assertThat(event.type()).isEqualTo("io.casehub.iot.situation.triggered.temperature-threshold");
    }

    @Test
    void resolvedTypeIncludesSituationId() {
        var event = new IoTSituationEvent(
                "temperature-threshold", "resolved", "sensor.outdoor",
                "tenant-1", Map.of(), NOW);
        assertThat(event.type()).isEqualTo("io.casehub.iot.situation.resolved.temperature-threshold");
    }

    @Test
    void tenancyIdReturnsConstructorValue() {
        var event = new IoTSituationEvent(
                "lock-state", "triggered", "lock.front_door",
                "my-tenant", Map.of(), NOW);
        assertThat(event.tenancyId()).isEqualTo("my-tenant");
    }

    @Test
    void tenancyIdIsNotSerialized() throws Exception {
        var event = new IoTSituationEvent(
                "lock-state", "triggered", "lock.front_door",
                "my-tenant", Map.of(), NOW);
        String json = MAPPER.writeValueAsString(event);
        assertThat(json).doesNotContain("tenancyId");
        assertThat(json).doesNotContain("my-tenant");
    }

    @Test
    void metadataIsPreserved() {
        var metadata = Map.<String, Object>of("temperature", 42.0, "threshold", 40.0, "direction", "ABOVE");
        var event = new IoTSituationEvent(
                "temperature-threshold", "triggered", "sensor.outdoor",
                "tenant-1", metadata, NOW);
        assertThat(event.metadata()).containsEntry("temperature", 42.0);
        assertThat(event.metadata()).containsEntry("direction", "ABOVE");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=IoTSituationEventTest -DfailIfNoTests=false --batch-mode`
Expected: FAIL — `IoTSituationEvent` class does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.iot.api;

import com.fasterxml.jackson.annotation.JsonIgnore;
import io.casehub.platform.api.subscription.SubscribableEvent;

import java.time.Instant;
import java.util.Map;
import java.util.Objects;

public final class IoTSituationEvent implements SubscribableEvent {

    private final String situationId;
    private final String changeType;
    private final String deviceId;
    private final String tenancyId;
    private final Map<String, Object> metadata;
    private final Instant occurredAt;

    public IoTSituationEvent(String situationId, String changeType, String deviceId,
                             String tenancyId, Map<String, Object> metadata, Instant occurredAt) {
        this.situationId = Objects.requireNonNull(situationId, "situationId");
        this.changeType = Objects.requireNonNull(changeType, "changeType");
        this.deviceId = Objects.requireNonNull(deviceId, "deviceId");
        this.tenancyId = Objects.requireNonNull(tenancyId, "tenancyId");
        this.metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
        this.occurredAt = Objects.requireNonNull(occurredAt, "occurredAt");
    }

    @Override
    public String type() {
        return "io.casehub.iot.situation." + changeType + "." + situationId;
    }

    @Override
    @JsonIgnore
    public String tenancyId() {
        return tenancyId;
    }

    public String situationId() { return situationId; }
    public String changeType() { return changeType; }
    public String deviceId() { return deviceId; }
    public Map<String, Object> metadata() { return metadata; }
    public Instant occurredAt() { return occurredAt; }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=IoTSituationEventTest --batch-mode`
Expected: PASS — all 5 tests green

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/iot/api/IoTSituationEvent.java api/src/test/java/io/casehub/iot/api/IoTSituationEventTest.java
git commit -m "feat(#67): add IoTSituationEvent implementing SubscribableEvent

Refs #67"
```

---

### Task 2: Case descriptor cleanup — remove household-notification worker

**Files:**
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/SafetyAlertCaseDescriptor.java`
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/SecurityAlertCaseDescriptor.java`
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/HvacAnomalyCaseDescriptor.java`
- Delete: `webapp-api/src/main/java/io/casehub/iot/webapp/worker/HouseholdNotificationWorkerFunction.java` (use `ide_refactor_safe_delete`)
- Delete: `webapp-api/src/test/java/io/casehub/iot/webapp/worker/HouseholdNotificationWorkerFunctionTest.java` (use `ide_refactor_safe_delete`)
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/engine/SafetyAlertCaseDescriptorTest.java`
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/engine/SecurityAlertCaseDescriptorTest.java`
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/engine/HvacAnomalyCaseDescriptorTest.java`

**Interfaces:**
- Consumes: None from other tasks
- Produces: Updated case descriptors with `household-notification` worker removed. No interface change — `workers()` return type unchanged.

- [ ] **Step 1: Write failing tests for all three descriptors**

```java
package io.casehub.iot.webapp.engine;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.work.api.spi.WorkItemCreator;
import jakarta.enterprise.inject.Instance;

class SafetyAlertCaseDescriptorTest {

    @SuppressWarnings("unchecked")
    @Test
    void workersDoNotIncludeHouseholdNotification() {
        var descriptor = new SafetyAlertCaseDescriptor(
                mock(Instance.class), mock(DeviceRegistry.class), mock(WorkItemCreator.class));
        var names = descriptor.workers().stream().map(w -> w.name()).toList();
        assertThat(names).containsExactly("device-command-dispatch", "human-acknowledgement");
        assertThat(names).doesNotContain("household-notification");
    }
}
```

```java
package io.casehub.iot.webapp.engine;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.work.api.spi.WorkItemCreator;
import jakarta.enterprise.inject.Instance;

class SecurityAlertCaseDescriptorTest {

    @SuppressWarnings("unchecked")
    @Test
    void workersDoNotIncludeHouseholdNotification() {
        var descriptor = new SecurityAlertCaseDescriptor(
                mock(Instance.class), mock(DeviceRegistry.class), mock(WorkItemCreator.class));
        var names = descriptor.workers().stream().map(w -> w.name()).toList();
        assertThat(names).containsExactly("device-command-dispatch", "camera-activation", "human-decision");
        assertThat(names).doesNotContain("household-notification");
    }
}
```

```java
package io.casehub.iot.webapp.engine;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.work.api.spi.WorkItemCreator;
import jakarta.enterprise.inject.Instance;

class HvacAnomalyCaseDescriptorTest {

    @SuppressWarnings("unchecked")
    @Test
    void workersDoNotIncludeHouseholdNotification() {
        var descriptor = new HvacAnomalyCaseDescriptor(
                mock(Instance.class), mock(DeviceRegistry.class), mock(WorkItemCreator.class));
        var names = descriptor.workers().stream().map(w -> w.name()).toList();
        assertThat(names).containsExactly("device-command-dispatch", "human-review");
        assertThat(names).doesNotContain("household-notification");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl webapp-api -Dtest="SafetyAlertCaseDescriptorTest,SecurityAlertCaseDescriptorTest,HvacAnomalyCaseDescriptorTest" --batch-mode`
Expected: FAIL — workers lists still contain `household-notification`

- [ ] **Step 3: Remove household-notification from SafetyAlertCaseDescriptor**

In `SafetyAlertCaseDescriptor.java`:
- Delete the `householdNotificationWorker()` method entirely
- Remove `householdNotificationWorker()` from the `workers()` list
- Remove the `import io.casehub.iot.webapp.worker.HouseholdNotificationWorkerFunction;`

Resulting `workers()`:
```java
public List<Worker> workers() {
    return List.of(
            deviceCommandWorker(),
            humanAcknowledgementWorker()
    );
}
```

- [ ] **Step 4: Remove household-notification from SecurityAlertCaseDescriptor**

In `SecurityAlertCaseDescriptor.java`:
- Delete the `householdNotificationWorker()` method entirely
- Remove `householdNotificationWorker()` from the `workers()` list
- Remove the `import io.casehub.iot.webapp.worker.HouseholdNotificationWorkerFunction;`

Resulting `workers()`:
```java
public List<Worker> workers() {
    return List.of(
            deviceCommandWorker(),
            cameraActivationWorker(),
            humanDecisionWorker()
    );
}
```

- [ ] **Step 5: Remove household-notification from HvacAnomalyCaseDescriptor**

In `HvacAnomalyCaseDescriptor.java`:
- Delete the `householdNotificationWorker()` method entirely
- Remove `householdNotificationWorker()` from the `workers()` list
- Remove the `import io.casehub.iot.webapp.worker.HouseholdNotificationWorkerFunction;`

Resulting `workers()`:
```java
public List<Worker> workers() {
    return List.of(
            deviceCommandWorker(),
            humanReviewWorker()
    );
}
```

- [ ] **Step 6: Delete HouseholdNotificationWorkerFunction and its test**

Use `ide_refactor_safe_delete` on:
- `webapp-api/src/main/java/io/casehub/iot/webapp/worker/HouseholdNotificationWorkerFunction.java`
- `webapp-api/src/test/java/io/casehub/iot/webapp/worker/HouseholdNotificationWorkerFunctionTest.java`

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl webapp-api -Dtest="SafetyAlertCaseDescriptorTest,SecurityAlertCaseDescriptorTest,HvacAnomalyCaseDescriptorTest" --batch-mode`
Expected: PASS — all 3 tests green

- [ ] **Step 8: Run full webapp-api tests to verify no regressions**

Run: `mvn test -pl webapp-api --batch-mode`
Expected: PASS — no compilation errors from deleted class, no test failures

- [ ] **Step 9: Commit**

```bash
git add webapp-api/
git commit -m "refactor(#67): remove HouseholdNotificationWorkerFunction from case descriptors

Delete the stub worker and remove household-notification from
SafetyAlert, SecurityAlert, and HvacAnomaly case descriptors.
Notification now handled by platform subscription engine.

Refs #67"
```

---

### Task 3: IoTSituationEventObserver and DataSource registration

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/subscription/IoTNotificationDataSourceRegistrar.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/subscription/IoTSituationEventObserver.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/subscription/IoTSituationEventObserverTest.java`

**Interfaces:**
- Consumes: `IoTSituationEvent` from Task 1; `SituationChangeEvent` from casehub-ras-api; `DataSourceRegistry`, `DataSource`, `DataSourceDescriptor`, `ObjectType`, `Path` from casehub-platform-api
- Produces: Pushes `IoTSituationEvent` instances into the platform subscription engine DataSource

- [ ] **Step 1: Write the failing observer test**

```java
package io.casehub.iot.webapp.app.subscription;

import io.casehub.iot.api.IoTSituationEvent;
import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import io.casehub.ras.api.SituationChangeEvent;
import io.casehub.ras.api.SituationContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class IoTSituationEventObserverTest {

    private DataSourceRegistry registry;
    private DataSource<IoTSituationEvent> dataSource;
    private IoTSituationEventObserver observer;

    @SuppressWarnings("unchecked")
    @BeforeEach
    void setUp() {
        registry = mock(DataSourceRegistry.class);
        dataSource = mock(DataSource.class);
        when(registry.resolveSource(any(Path.class), eq(TenancyConstants.PLATFORM_TENANT_ID)))
                .thenReturn(Optional.of(dataSource));
        observer = new IoTSituationEventObserver(registry);
    }

    @Test
    void triggeredEventPushedToDataSource() {
        var context = mock(SituationContext.class);
        when(context.lastTriggered()).thenReturn(Instant.parse("2026-08-12T10:00:00Z"));
        var event = new SituationChangeEvent(
                "temperature-threshold", "device/sensor.outdoor", "tenant-1",
                SituationChangeEvent.ChangeType.TRIGGERED,
                Map.of("temperature", 42.0), context);

        observer.onSituationChange(event);

        var captor = ArgumentCaptor.forClass(IoTSituationEvent.class);
        verify(dataSource).add(captor.capture());
        var captured = captor.getValue();
        assertThat(captured.type()).isEqualTo("io.casehub.iot.situation.triggered.temperature-threshold");
        assertThat(captured.deviceId()).isEqualTo("sensor.outdoor");
        assertThat(captured.tenancyId()).isEqualTo("tenant-1");
        assertThat(captured.metadata()).containsEntry("temperature", 42.0);
    }

    @Test
    void resolvedEventPushedToDataSource() {
        var context = mock(SituationContext.class);
        when(context.lastTriggered()).thenReturn(Instant.parse("2026-08-12T11:00:00Z"));
        var event = new SituationChangeEvent(
                "temperature-threshold", "device/sensor.outdoor", "tenant-1",
                SituationChangeEvent.ChangeType.RESOLVED,
                Map.of(), context);

        observer.onSituationChange(event);

        var captor = ArgumentCaptor.forClass(IoTSituationEvent.class);
        verify(dataSource).add(captor.capture());
        assertThat(captor.getValue().type()).isEqualTo("io.casehub.iot.situation.resolved.temperature-threshold");
    }

    @Test
    void suppressedEventIsIgnored() {
        var context = mock(SituationContext.class);
        var event = new SituationChangeEvent(
                "temperature-threshold", "device/sensor.outdoor", "tenant-1",
                SituationChangeEvent.ChangeType.SUPPRESSED,
                Map.of(), context);

        observer.onSituationChange(event);

        verify(dataSource, never()).add(any());
    }

    @Test
    void dismissedEventIsIgnored() {
        var context = mock(SituationContext.class);
        var event = new SituationChangeEvent(
                "temperature-threshold", "device/sensor.outdoor", "tenant-1",
                SituationChangeEvent.ChangeType.DISMISSED,
                Map.of(), context);

        observer.onSituationChange(event);

        verify(dataSource, never()).add(any());
    }

    @Test
    void discardedEventIsIgnored() {
        var context = mock(SituationContext.class);
        var event = new SituationChangeEvent(
                "temperature-threshold", "device/sensor.outdoor", "tenant-1",
                SituationChangeEvent.ChangeType.DISCARDED,
                Map.of(), context);

        observer.onSituationChange(event);

        verify(dataSource, never()).add(any());
    }

    @Test
    void dataSourceFailureIsCaughtAndLogged() {
        var context = mock(SituationContext.class);
        when(context.lastTriggered()).thenReturn(Instant.now());
        doThrow(new RuntimeException("DataSource unavailable")).when(dataSource).add(any());

        var event = new SituationChangeEvent(
                "temperature-threshold", "device/sensor.outdoor", "tenant-1",
                SituationChangeEvent.ChangeType.TRIGGERED,
                Map.of(), context);

        // Should not throw
        observer.onSituationChange(event);

        verify(dataSource).add(any());
    }

    @Test
    void correlationKeyWithoutDevicePrefixUsedAsIs() {
        var context = mock(SituationContext.class);
        when(context.lastTriggered()).thenReturn(Instant.now());
        var event = new SituationChangeEvent(
                "power-anomaly", "sensor.power_meter", "tenant-1",
                SituationChangeEvent.ChangeType.TRIGGERED,
                Map.of(), context);

        observer.onSituationChange(event);

        var captor = ArgumentCaptor.forClass(IoTSituationEvent.class);
        verify(dataSource).add(captor.capture());
        assertThat(captor.getValue().deviceId()).isEqualTo("sensor.power_meter");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl webapp -Dtest=IoTSituationEventObserverTest -DfailIfNoTests=false --batch-mode`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement IoTNotificationDataSourceRegistrar**

```java
package io.casehub.iot.webapp.app.subscription;

import io.casehub.iot.api.IoTSituationEvent;
import io.casehub.platform.api.datasource.DataSourceDescriptor;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.platform.api.datasource.ObjectType;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.Set;

@ApplicationScoped
public class IoTNotificationDataSourceRegistrar {

    private static final Logger LOG = Logger.getLogger(IoTNotificationDataSourceRegistrar.class);

    static final Path IOT_SITUATIONS_PATH = Path.of("iot", "situations");

    private static final ObjectType<IoTSituationEvent> OBJECT_TYPE = new ObjectType<>() {
        @Override
        public boolean matches(Object obj) {
            return obj instanceof IoTSituationEvent;
        }

        @Override
        public String getTypeKey() {
            return "io.casehub.iot.situation";
        }
    };

    @Inject
    DataSourceRegistry dataSourceRegistry;

    void onStartup(@Observes StartupEvent event) {
        var descriptor = new DataSourceDescriptor(
                IOT_SITUATIONS_PATH,
                TenancyConstants.PLATFORM_TENANT_ID,
                OBJECT_TYPE,
                IOT_SITUATIONS_PATH,
                Set.of("io.casehub.iot.situation.triggered", "io.casehub.iot.situation.resolved"),
                Map.of(),
                Map.of()
        );
        dataSourceRegistry.register(descriptor);
        LOG.info("Registered IoT situations DataSource at " + IOT_SITUATIONS_PATH);
    }
}
```

- [ ] **Step 4: Implement IoTSituationEventObserver**

```java
package io.casehub.iot.webapp.app.subscription;

import io.casehub.iot.api.IoTSituationEvent;
import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import io.casehub.ras.api.SituationChangeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class IoTSituationEventObserver {

    private static final Logger LOG = Logger.getLogger(IoTSituationEventObserver.class);

    private final DataSourceRegistry dataSourceRegistry;

    @Inject
    public IoTSituationEventObserver(DataSourceRegistry dataSourceRegistry) {
        this.dataSourceRegistry = dataSourceRegistry;
    }

    @SuppressWarnings("unchecked")
    public void onSituationChange(@ObservesAsync SituationChangeEvent event) {
        if (event.changeType() != SituationChangeEvent.ChangeType.TRIGGERED
                && event.changeType() != SituationChangeEvent.ChangeType.RESOLVED) {
            return;
        }

        try {
            String changeType = event.changeType() == SituationChangeEvent.ChangeType.TRIGGERED
                    ? "triggered" : "resolved";
            String deviceId = extractDeviceId(event.correlationKey());

            var situationEvent = new IoTSituationEvent(
                    event.situationId(),
                    changeType,
                    deviceId,
                    event.tenancyId(),
                    event.metadata(),
                    event.context().lastTriggered()
            );

            dataSourceRegistry
                    .resolveSource(IoTNotificationDataSourceRegistrar.IOT_SITUATIONS_PATH,
                            TenancyConstants.PLATFORM_TENANT_ID)
                    .ifPresentOrElse(
                            ds -> ((DataSource<IoTSituationEvent>) ds).add(situationEvent),
                            () -> LOG.warn("IoT situations DataSource not registered — event dropped: "
                                    + situationEvent.type())
                    );
        } catch (Exception e) {
            LOG.warnf(e, "Failed to push IoT situation event [%s/%s]: %s",
                    event.situationId(), event.correlationKey(), e.getMessage());
        }
    }

    private static String extractDeviceId(String correlationKey) {
        if (correlationKey != null && correlationKey.startsWith("device/")) {
            return correlationKey.substring("device/".length());
        }
        return correlationKey != null ? correlationKey : "unknown";
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl webapp -Dtest=IoTSituationEventObserverTest --batch-mode`
Expected: PASS — all 7 tests green

- [ ] **Step 6: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/subscription/ webapp/src/test/java/io/casehub/iot/webapp/app/subscription/
git commit -m "feat(#67): add IoTSituationEventObserver and DataSource registrar

CDI observer listens for SituationChangeEvent (TRIGGERED/RESOLVED),
constructs IoTSituationEvent, pushes into platform-global DataSource.
Best-effort — failures logged, never thrown.

Refs #67"
```

---

### Task 4: Integration test and full build verification

**Files:**
- Create: `webapp/src/test/java/io/casehub/iot/webapp/app/subscription/IoTNotificationIntegrationTest.java`

**Interfaces:**
- Consumes: `IoTSituationEvent` from Task 1, `IoTNotificationDataSourceRegistrar` and `IoTSituationEventObserver` from Task 3
- Produces: Verification that the full pipeline works end-to-end

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.iot.webapp.app.subscription;

import io.casehub.iot.webapp.app.WebappPostgresTestResource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import io.quarkus.test.common.QuarkusTestResource;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@QuarkusTestResource(WebappPostgresTestResource.class)
class IoTNotificationIntegrationTest {

    @Inject
    DataSourceRegistry dataSourceRegistry;

    @Test
    void dataSourceRegisteredAtStartup() {
        var ds = dataSourceRegistry.resolveSource(
                Path.of("iot", "situations"),
                TenancyConstants.PLATFORM_TENANT_ID);
        assertThat(ds).isPresent();
    }

    @Inject
    IoTSituationEventObserver observer;

    @Test
    void observerIsInjectable() {
        assertThat(observer).isNotNull();
    }
}
```

- [ ] **Step 2: Run the integration test**

Run: `mvn test -pl webapp -Dtest=IoTNotificationIntegrationTest --batch-mode`
Expected: PASS — DataSource registered at startup, observer injectable

- [ ] **Step 3: Run full project build**

Run: `mvn --batch-mode install`
Expected: PASS — all modules compile and test green across api, webapp-api, webapp, and all other modules

- [ ] **Step 4: Commit**

```bash
git add webapp/src/test/java/io/casehub/iot/webapp/app/subscription/IoTNotificationIntegrationTest.java
git commit -m "test(#67): integration test for IoT notification DataSource registration

Verifies DataSource is registered at startup and observer is
injectable in the full Quarkus container.

Refs #67"
```
