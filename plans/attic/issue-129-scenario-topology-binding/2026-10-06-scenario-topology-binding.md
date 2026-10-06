# Scenario-Topology Binding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #129 — scenario-topology binding
**Issue group:** #129

**Goal:** Connect scenario step execution to the topology dual-view so affected devices highlight during playbook execution with convergence progress.

**Architecture:** CDI event bridge — `DesiredStateDeliveryHandler` fires `ScenarioBindingEvent` variants during step execution. A `ScenarioTopologyBinder` observer in the webapp normalises these into `TopologyStreamEvent` binding operations on the existing SSE stream. Frontend topology components render highlights from SSE-delivered state.

**Tech Stack:** Java 21 (sealed interfaces), CDI (synchronous events), Quarkus SSE (BroadcastProcessor), LitElement (TypeScript), Vitest (frontend tests)

## Global Constraints

- `ScenarioBindingEvent` goes in `casehub-iot-api` — public API surface, semver discipline
- All SPIs are blocking — designed for virtual threads per ADR-0005
- No changes to `casehub-pages-scenario` or `casehub-pages-push`
- `TopologyStreamEvent` must remain backward-compatible (existing `snapshot`/`update` operations unchanged)
- Node IDs use `-config` suffix for config nodes: `NodeId.of(deviceId + "-config")`
- `DeviceCommandEvent` is audit-only — NOT used for scenario binding (deferred to #131)
- `tenancyId` on all events for future per-tenant SSE filtering (#132)

---

## Batch 1: Backend CDI Events

### Task 1: ScenarioBindingEvent sealed interface

**Files:**
- Create: `api/src/main/java/io/casehub/iot/api/ScenarioBindingEvent.java`
- Test: `api/src/test/java/io/casehub/iot/api/ScenarioBindingEventTest.java`

**Interfaces:**
- Consumes: nothing (new root type)
- Produces: `ScenarioBindingEvent` sealed interface with variants: `StepStart(executionId, tenancyId, stepName, deviceIds)`, `DeviceProvisioned(executionId, tenancyId, stepName, deviceId)`, `DeviceFailed(executionId, tenancyId, stepName, deviceId, reason)`, `StepComplete(executionId, tenancyId, stepName, provisioned, failed)`, `StepFailed(executionId, tenancyId, stepName, provisioned, failed, failedDetails)`, `Clear(executionId, tenancyId)`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class ScenarioBindingEventTest {

    @Test
    void stepStartCarriesDeviceIds() {
        var event = new ScenarioBindingEvent.StepStart(
                "exec-1", "tenant-1", "apply-nightmode", Set.of("light-1", "therm-1"));
        assertEquals("exec-1", event.executionId());
        assertEquals("tenant-1", event.tenancyId());
        assertEquals("apply-nightmode", event.stepName());
        assertEquals(Set.of("light-1", "therm-1"), event.deviceIds());
        assertInstanceOf(ScenarioBindingEvent.class, event);
    }

    @Test
    void deviceProvisionedCarriesSingleDevice() {
        var event = new ScenarioBindingEvent.DeviceProvisioned(
                "exec-1", "tenant-1", "apply-nightmode", "light-1");
        assertEquals("light-1", event.deviceId());
    }

    @Test
    void deviceFailedCarriesReason() {
        var event = new ScenarioBindingEvent.DeviceFailed(
                "exec-1", "tenant-1", "apply-nightmode", "therm-1", "timeout");
        assertEquals("therm-1", event.deviceId());
        assertEquals("timeout", event.reason());
    }

    @Test
    void stepCompleteCarriesCounts() {
        var event = new ScenarioBindingEvent.StepComplete(
                "exec-1", "tenant-1", "apply-nightmode", 3, 0);
        assertEquals(3, event.provisioned());
        assertEquals(0, event.failed());
    }

    @Test
    void stepFailedCarriesDetails() {
        var event = new ScenarioBindingEvent.StepFailed(
                "exec-1", "tenant-1", "apply-nightmode", 2, 1,
                List.of("therm-1: timeout"));
        assertEquals(1, event.failed());
        assertEquals(List.of("therm-1: timeout"), event.failedDetails());
    }

    @Test
    void patternMatchOnVariants() {
        ScenarioBindingEvent event = new ScenarioBindingEvent.Clear("exec-1", "tenant-1");
        String result = switch (event) {
            case ScenarioBindingEvent.StepStart ss -> "start:" + ss.stepName();
            case ScenarioBindingEvent.DeviceProvisioned dp -> "prov:" + dp.deviceId();
            case ScenarioBindingEvent.DeviceFailed df -> "fail:" + df.deviceId();
            case ScenarioBindingEvent.StepComplete sc -> "complete:" + sc.provisioned();
            case ScenarioBindingEvent.StepFailed sf -> "failed:" + sf.failed();
            case ScenarioBindingEvent.Clear c -> "clear:" + c.executionId();
        };
        assertEquals("clear:exec-1", result);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=ScenarioBindingEventTest -DfailIfNoTests=false`
Expected: FAIL — `ScenarioBindingEvent` does not exist

- [ ] **Step 3: Implement ScenarioBindingEvent**

```java
package io.casehub.iot.api;

import java.util.List;
import java.util.Set;

public sealed interface ScenarioBindingEvent {
    String executionId();
    String tenancyId();

    record StepStart(
            String executionId, String tenancyId,
            String stepName, Set<String> deviceIds
    ) implements ScenarioBindingEvent {}

    record DeviceProvisioned(
            String executionId, String tenancyId,
            String stepName, String deviceId
    ) implements ScenarioBindingEvent {}

    record DeviceFailed(
            String executionId, String tenancyId,
            String stepName, String deviceId, String reason
    ) implements ScenarioBindingEvent {}

    record StepComplete(
            String executionId, String tenancyId,
            String stepName, int provisioned, int failed
    ) implements ScenarioBindingEvent {}

    record StepFailed(
            String executionId, String tenancyId,
            String stepName, int provisioned, int failed,
            List<String> failedDetails
    ) implements ScenarioBindingEvent {}

    record Clear(
            String executionId, String tenancyId
    ) implements ScenarioBindingEvent {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=ScenarioBindingEventTest`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/iot/api/ScenarioBindingEvent.java api/src/test/java/io/casehub/iot/api/ScenarioBindingEventTest.java
git commit -m "feat(#129): ScenarioBindingEvent sealed interface with typed variants"
```

### Task 2: Emit ScenarioBindingEvent from DesiredStateDeliveryHandler

**Files:**
- Modify: `scenario/src/main/java/io/casehub/iot/scenario/DesiredStateDeliveryHandler.java`
- Modify: `scenario/src/test/java/io/casehub/iot/scenario/DesiredStateDeliveryHandlerTest.java`

**Interfaces:**
- Consumes: `ScenarioBindingEvent` sealed interface (Task 1)
- Produces: CDI events fired synchronously via `Event.fire()` at lifecycle points: `StepStart` → `DeviceProvisioned`/`DeviceFailed` per device → `StepComplete`/`StepFailed`. `Clear` on exception after `StepStart`.

- [ ] **Step 1: Write the test — verify binding events fire during successful reconciliation**

The existing test class uses `MockDeviceRegistry`, `MockDeviceProvider` with `Fixtures.standardHome()`, `IoTGoalCompiler()`, mocked `IoTActualStateAdapter` and `IoTNodeProvisioner`. The handler constructor takes `(registry, presetResolver, compiler, actualStateAdapter, provisioner, tenancyId)`. We add `Event<ScenarioBindingEvent>` as the last constructor parameter.

Add a field and helper to the test class:

```java
private List<ScenarioBindingEvent> capturedEvents;

// In setUp(), after existing setup:
capturedEvents = new ArrayList<>();
jakarta.enterprise.event.Event<ScenarioBindingEvent> bindingEvent = capturedEvents::add;
handler = new DesiredStateDeliveryHandler(
        registry, presetResolver, compiler,
        actualStateAdapter, provisioner, TENANCY_ID, bindingEvent);
```

Update existing `setUp()` constructor call to include the no-op event `e -> {}` (or include the capture list for the binding test).

Add a new test:

```java
@Test
void emitsBindingEventsForSuccessfulReconciliation() {
    when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
            .thenReturn(new ActualState(Map.of()));
    when(provisioner.provision(any(), any()))
            .thenReturn(new ProvisionResult.Success());

    Map<String, Object> data = new LinkedHashMap<>();
    data.put("light-living-1", Map.of("on", false));
    data.put("thermostat-living-1", Map.of("mode", "cool", "target", 22));

    handler.execute("night-mode", data, CTX);

    assertThat(capturedEvents).isNotEmpty();

    // First event: StepStart with both device IDs
    assertThat(capturedEvents.get(0)).isInstanceOf(ScenarioBindingEvent.StepStart.class);
    var start = (ScenarioBindingEvent.StepStart) capturedEvents.get(0);
    assertThat(start.deviceIds()).containsExactlyInAnyOrder("light-living-1", "thermostat-living-1");
    assertThat(start.stepName()).isEqualTo("night-mode");

    // Middle events: per-device provisioned
    long provisionedCount = capturedEvents.stream()
            .filter(e -> e instanceof ScenarioBindingEvent.DeviceProvisioned)
            .count();
    assertThat(provisionedCount).isEqualTo(2);

    // Last event: StepComplete
    var last = capturedEvents.get(capturedEvents.size() - 1);
    assertThat(last).isInstanceOf(ScenarioBindingEvent.StepComplete.class);
    var complete = (ScenarioBindingEvent.StepComplete) last;
    assertThat(complete.provisioned()).isEqualTo(2);
    assertThat(complete.failed()).isZero();
}
```

- [ ] **Step 2: Write the test — verify Clear fires on exception**

```java
@Test
void emitsClearOnReconciliationException() {
    when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
            .thenReturn(new ActualState(Map.of()));
    when(provisioner.provision(any(), any()))
            .thenThrow(new RuntimeException("provider crash"));

    Map<String, Object> data = new LinkedHashMap<>();
    data.put("light-living-1", Map.of("on", false));

    assertThatThrownBy(() -> handler.execute("crash-step", data, CTX))
            .isInstanceOf(RuntimeException.class);

    assertThat(capturedEvents.stream()
            .anyMatch(e -> e instanceof ScenarioBindingEvent.StepStart)).isTrue();
    assertThat(capturedEvents.stream()
            .anyMatch(e -> e instanceof ScenarioBindingEvent.Clear)).isTrue();
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl scenario -Dtest=DesiredStateDeliveryHandlerTest`
Expected: FAIL — new tests fail (handler doesn't fire events yet)

- [ ] **Step 4: Implement — modify DesiredStateDeliveryHandler**

Add `@Inject Event<ScenarioBindingEvent> bindingEvent` field. Modify `execute()` and `reconcile()` methods per the spec:

1. Add `Event<ScenarioBindingEvent> bindingEvent` field with `@Inject`
2. Update constructor to accept `Event<ScenarioBindingEvent>`
3. In `execute()`: generate `executionId = UUID.randomUUID().toString()`, pass to `reconcile()`
4. In `reconcile()`: add `extractDeviceId()` helper (strip `-config` suffix from node ID). Fire `StepStart` after plan computation. Fire `DeviceProvisioned`/`DeviceFailed` per device with deduplication via `deviceOutcomes` map. Fire `StepComplete`/`StepFailed` after loop. Wrap in try/catch that fires `Clear` on exception.

The `extractDeviceId()` helper:
```java
private static String extractDeviceId(DesiredNode node) {
    String id = node.id().value();
    return id.endsWith("-config") ? id.substring(0, id.length() - 7) : id;
}
```

The `reconcile()` method signature changes to accept `executionId`:
```java
private StepOutcome reconcile(String stepName, IoTGoals goals, String executionId)
```

Key deduplication logic: track `Map<String, String> deviceOutcomes` (deviceId → "provisioned"/"failed"). Only fire `DeviceProvisioned` on first success per device. Always fire `DeviceFailed` (overrides prior success).

- [ ] **Step 5: Update test constructor to supply mock Event**

The test needs to construct the handler with a mock `Event<ScenarioBindingEvent>`. Since the existing test uses a package-private constructor, add `Event<ScenarioBindingEvent>` as the last constructor parameter. For tests that don't care about events, pass a no-op event: `e -> {}`.

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl scenario -Dtest=DesiredStateDeliveryHandlerTest`
Expected: PASS — all tests including new binding event tests

- [ ] **Step 7: Run full scenario module tests**

Run: `mvn --batch-mode test -pl scenario`
Expected: PASS — no regressions

- [ ] **Step 8: Commit**

```bash
git add scenario/src/main/java/io/casehub/iot/scenario/DesiredStateDeliveryHandler.java scenario/src/test/java/io/casehub/iot/scenario/DesiredStateDeliveryHandlerTest.java
git commit -m "feat(#129): emit ScenarioBindingEvent from DesiredStateDeliveryHandler"
```

## Batch 2: Backend SSE Bridge

### Task 3: Extract TopologyStreamEvent and create ScenarioTopologyBinder

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/TopologyStreamEvent.java` (standalone record)
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTTopologyApi.java` (remove nested record, use standalone, extract BroadcastProcessor)
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinder.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinderTest.java`

**Interfaces:**
- Consumes: `ScenarioBindingEvent` (Task 1), `BroadcastProcessor<TopologyStreamEvent>` (extracted from DefaultIoTTopologyApi)
- Produces: `TopologyStreamEvent.binding(operation, data)` factory method. Binder `onBindingEvent(@Observes ScenarioBindingEvent)` that writes to BroadcastProcessor.

- [ ] **Step 1: Extract TopologyStreamEvent to standalone record**

Read `DefaultIoTTopologyApi.java:64`. The nested record `TopologyStreamEvent(String operation, List<TopologyNode> nodes)` needs to become a standalone record with the binding factory method.

Create `TopologyStreamEvent.java`:
```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.webapp.rest.TopologyNode;
import java.util.List;
import java.util.Map;

public record TopologyStreamEvent(
        String operation,
        List<TopologyNode> nodes,
        Map<String, Object> binding
) {
    public TopologyStreamEvent(String operation, List<TopologyNode> nodes) {
        this(operation, nodes, null);
    }

    public static TopologyStreamEvent binding(String operation, Map<String, Object> data) {
        return new TopologyStreamEvent(operation, List.of(), data);
    }
}
```

Then use `ide_refactor_safe_delete` on the nested record in `DefaultIoTTopologyApi` after updating all references to use the standalone version. Update imports in `DefaultIoTTopologyApi`.

- [ ] **Step 2: Extract BroadcastProcessor to a shared producer**

In `DefaultIoTTopologyApi`, the `BroadcastProcessor<TopologyStreamEvent> broadcaster` field and `@PostConstruct init()` need to become injectable. Create a `@Produces` method in `DefaultIoTTopologyApi` or a separate producer bean:

```java
// Add to DefaultIoTTopologyApi or a new producer:
@Produces @ApplicationScoped
BroadcastProcessor<TopologyStreamEvent> topologyBroadcaster() {
    return BroadcastProcessor.create();
}
```

Update `DefaultIoTTopologyApi` to `@Inject` the `BroadcastProcessor` instead of creating it internally. Remove `@PostConstruct init()`.

- [ ] **Step 3: Verify existing topology tests still pass**

Run: `mvn --batch-mode test -pl webapp -Dtest=DefaultIoTTopologyApiTest`
Expected: PASS — extraction doesn't change behaviour

- [ ] **Step 4: Write test for ScenarioTopologyBinder**

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.ScenarioBindingEvent;
import io.smallrye.mutiny.operators.multi.processors.BroadcastProcessor;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class ScenarioTopologyBinderTest {

    private BroadcastProcessor<TopologyStreamEvent> broadcaster;
    private ScenarioTopologyBinder binder;
    private List<TopologyStreamEvent> captured;

    @BeforeEach
    void setUp() {
        broadcaster = BroadcastProcessor.create();
        captured = new ArrayList<>();
        broadcaster.subscribe().with(captured::add);
        binder = new ScenarioTopologyBinder(broadcaster);
    }

    @Test
    void stepStartProducesBindingStartEvent() {
        var event = new ScenarioBindingEvent.StepStart(
                "exec-1", "tenant-1", "apply-nightmode", Set.of("light-1", "therm-1"));

        binder.onBindingEvent(event);

        assertEquals(1, captured.size());
        var sse = captured.get(0);
        assertEquals("binding-start", sse.operation());
        assertTrue(sse.nodes().isEmpty());
        assertNotNull(sse.binding());
        assertEquals("exec-1", sse.binding().get("executionId"));
        assertEquals("apply-nightmode", sse.binding().get("stepName"));
        @SuppressWarnings("unchecked")
        var deviceIds = (Set<String>) sse.binding().get("deviceIds");
        assertEquals(Set.of("light-1", "therm-1"), deviceIds);
    }

    @Test
    void deviceProvisionedProducesBindingUpdateEvent() {
        var event = new ScenarioBindingEvent.DeviceProvisioned(
                "exec-1", "tenant-1", "apply-nightmode", "light-1");

        binder.onBindingEvent(event);

        assertEquals(1, captured.size());
        var sse = captured.get(0);
        assertEquals("binding-update", sse.operation());
        assertEquals("light-1", sse.binding().get("deviceId"));
        assertEquals("PROVISIONED", sse.binding().get("status"));
    }

    @Test
    void deviceFailedProducesBindingUpdateWithFailed() {
        var event = new ScenarioBindingEvent.DeviceFailed(
                "exec-1", "tenant-1", "apply-nightmode", "therm-1", "timeout");

        binder.onBindingEvent(event);

        var sse = captured.get(0);
        assertEquals("binding-update", sse.operation());
        assertEquals("FAILED", sse.binding().get("status"));
    }

    @Test
    void stepCompleteProducesBindingCompleteOk() {
        var event = new ScenarioBindingEvent.StepComplete(
                "exec-1", "tenant-1", "apply-nightmode", 3, 0);

        binder.onBindingEvent(event);

        var sse = captured.get(0);
        assertEquals("binding-complete", sse.operation());
        assertEquals("OK", sse.binding().get("outcome"));
        assertEquals(3, sse.binding().get("provisioned"));
    }

    @Test
    void stepFailedProducesBindingCompleteFailed() {
        var event = new ScenarioBindingEvent.StepFailed(
                "exec-1", "tenant-1", "apply-nightmode", 2, 1,
                List.of("therm-1: timeout"));

        binder.onBindingEvent(event);

        var sse = captured.get(0);
        assertEquals("binding-complete", sse.operation());
        assertEquals("FAILED", sse.binding().get("outcome"));
    }

    @Test
    void clearProducesBindingClearEvent() {
        var event = new ScenarioBindingEvent.Clear("exec-1", "tenant-1");

        binder.onBindingEvent(event);

        var sse = captured.get(0);
        assertEquals("binding-clear", sse.operation());
        assertEquals("exec-1", sse.binding().get("executionId"));
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn --batch-mode test -pl webapp -Dtest=ScenarioTopologyBinderTest -DfailIfNoTests=false`
Expected: FAIL — `ScenarioTopologyBinder` does not exist

- [ ] **Step 6: Implement ScenarioTopologyBinder**

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.ScenarioBindingEvent;
import io.smallrye.mutiny.operators.multi.processors.BroadcastProcessor;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import java.util.Map;

@ApplicationScoped
public class ScenarioTopologyBinder {

    private final BroadcastProcessor<TopologyStreamEvent> broadcaster;

    @Inject
    public ScenarioTopologyBinder(BroadcastProcessor<TopologyStreamEvent> broadcaster) {
        this.broadcaster = broadcaster;
    }

    void onBindingEvent(@Observes ScenarioBindingEvent event) {
        var streamEvent = switch (event) {
            case ScenarioBindingEvent.StepStart ss ->
                TopologyStreamEvent.binding("binding-start", Map.of(
                    "executionId", ss.executionId(),
                    "stepName", ss.stepName(),
                    "deviceIds", ss.deviceIds()));
            case ScenarioBindingEvent.DeviceProvisioned dp ->
                TopologyStreamEvent.binding("binding-update", Map.of(
                    "executionId", dp.executionId(),
                    "deviceId", dp.deviceId(),
                    "status", "PROVISIONED"));
            case ScenarioBindingEvent.DeviceFailed df ->
                TopologyStreamEvent.binding("binding-update", Map.of(
                    "executionId", df.executionId(),
                    "deviceId", df.deviceId(),
                    "status", "FAILED"));
            case ScenarioBindingEvent.StepComplete sc ->
                TopologyStreamEvent.binding("binding-complete", Map.of(
                    "executionId", sc.executionId(),
                    "stepName", sc.stepName(),
                    "outcome", "OK",
                    "provisioned", sc.provisioned(),
                    "failed", sc.failed()));
            case ScenarioBindingEvent.StepFailed sf ->
                TopologyStreamEvent.binding("binding-complete", Map.of(
                    "executionId", sf.executionId(),
                    "stepName", sf.stepName(),
                    "outcome", "FAILED",
                    "provisioned", sf.provisioned(),
                    "failed", sf.failed()));
            case ScenarioBindingEvent.Clear c ->
                TopologyStreamEvent.binding("binding-clear", Map.of(
                    "executionId", c.executionId()));
        };
        broadcaster.onNext(streamEvent);
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl webapp -Dtest=ScenarioTopologyBinderTest`
Expected: PASS — all 6 binder tests green

- [ ] **Step 8: Run full webapp backend tests**

Run: `mvn --batch-mode test -pl webapp`
Expected: PASS — no regressions (excluding pre-existing webapp-api compile exclusions)

- [ ] **Step 9: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/service/TopologyStreamEvent.java webapp/src/main/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinder.java webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTTopologyApi.java webapp/src/test/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinderTest.java
git commit -m "feat(#129): extract TopologyStreamEvent, add ScenarioTopologyBinder CDI observer"
```

## Batch 3: Frontend Highlight Model and Tree View

### Task 4: Highlight model and zone aggregation logic

**Files:**
- Modify: `webapp/src/main/webapp/src/components/topology-tree-model.ts`
- Modify: `webapp/src/main/webapp/src/components/topology-tree-model.test.ts`

**Interfaces:**
- Consumes: `TreeBranch`, `TopologyNode` (existing types in `topology-tree-model.ts`)
- Produces: `HighlightState` interface (`{status, executionId, stepName}`), `ZoneHighlight` interface (`{total, counts}`), `BindingEventData` interface, `computeZoneHighlight(branch, highlightMap): ZoneHighlight | null`, `formatZoneHighlightSummary(zone): string`, `applyBindingEvent(current, data): Record<string, HighlightState>`

- [ ] **Step 1: Write tests for highlight model**

```typescript
// Add to topology-tree-model.test.ts
import { computeZoneHighlight, formatZoneHighlightSummary, type HighlightState, type TreeBranch, type TopologyNode } from './topology-tree-model.js';

function makeNode(deviceId: string): TopologyNode {
  return { deviceId, label: deviceId, deviceClass: 'LIGHT', locationPath: ['home', 'bedroom'], available: true, lastUpdated: '', driftStatus: 'CONVERGED', driftDetail: null };
}

function makeBranch(devices: TopologyNode[]): TreeBranch {
  return { name: 'bedroom', path: 'home/bedroom', children: new Map(), devices };
}

describe('computeZoneHighlight', () => {
  test('returns null when fewer than 2 devices highlighted', () => {
    const branch = makeBranch([makeNode('light-1'), makeNode('light-2')]);
    const map: Record<string, HighlightState> = {
      'light-1': { status: 'active', executionId: 'e1', stepName: 's1' },
    };
    expect(computeZoneHighlight(branch, map)).toBeNull();
  });

  test('returns counts when 2+ devices highlighted', () => {
    const branch = makeBranch([makeNode('light-1'), makeNode('light-2'), makeNode('therm-1')]);
    const map: Record<string, HighlightState> = {
      'light-1': { status: 'active', executionId: 'e1', stepName: 's1' },
      'light-2': { status: 'provisioned', executionId: 'e1', stepName: 's1' },
      'therm-1': { status: 'failed', executionId: 'e1', stepName: 's1' },
    };
    const result = computeZoneHighlight(branch, map);
    expect(result).not.toBeNull();
    expect(result!.total).toBe(3);
    expect(result!.counts.active).toBe(1);
    expect(result!.counts.provisioned).toBe(1);
    expect(result!.counts.failed).toBe(1);
  });

  test('returns null for empty highlight map', () => {
    const branch = makeBranch([makeNode('light-1'), makeNode('light-2')]);
    expect(computeZoneHighlight(branch, {})).toBeNull();
  });
});

describe('formatZoneHighlightSummary', () => {
  test('all active', () => {
    expect(formatZoneHighlightSummary({ total: 3, counts: { active: 3, provisioned: 0, failed: 0 } }))
      .toBe('3 active');
  });

  test('mixed statuses', () => {
    expect(formatZoneHighlightSummary({ total: 3, counts: { active: 1, provisioned: 1, failed: 1 } }))
      .toBe('1 active, 1 provisioned, 1 failed');
  });
});

describe('applyBindingEvent (highlight state machine)', () => {
  test('binding-start sets devices to active', () => {
    const map: Record<string, HighlightState> = {};
    const result = applyBindingEvent(map, {
      operation: 'binding-start',
      binding: { executionId: 'e1', stepName: 's1', deviceIds: ['light-1', 'therm-1'] },
    });
    expect(result['light-1']?.status).toBe('active');
    expect(result['therm-1']?.status).toBe('active');
  });

  test('binding-update transitions active to provisioned', () => {
    const map: Record<string, HighlightState> = {
      'light-1': { status: 'active', executionId: 'e1', stepName: 's1' },
    };
    const result = applyBindingEvent(map, {
      operation: 'binding-update',
      binding: { executionId: 'e1', deviceId: 'light-1', status: 'PROVISIONED' },
    });
    expect(result['light-1']?.status).toBe('provisioned');
  });

  test('binding-update transitions active to failed', () => {
    const map: Record<string, HighlightState> = {
      'light-1': { status: 'active', executionId: 'e1', stepName: 's1' },
    };
    const result = applyBindingEvent(map, {
      operation: 'binding-update',
      binding: { executionId: 'e1', deviceId: 'light-1', status: 'FAILED' },
    });
    expect(result['light-1']?.status).toBe('failed');
  });

  test('binding-clear removes entries for matching executionId', () => {
    const map: Record<string, HighlightState> = {
      'light-1': { status: 'active', executionId: 'e1', stepName: 's1' },
      'therm-1': { status: 'active', executionId: 'e2', stepName: 's2' },
    };
    const result = applyBindingEvent(map, {
      operation: 'binding-clear',
      binding: { executionId: 'e1' },
    });
    expect(result['light-1']).toBeUndefined();
    expect(result['therm-1']?.status).toBe('active');
  });

  test('binding-update ignores unknown device', () => {
    const map: Record<string, HighlightState> = {};
    const result = applyBindingEvent(map, {
      operation: 'binding-update',
      binding: { executionId: 'e1', deviceId: 'unknown', status: 'PROVISIONED' },
    });
    expect(Object.keys(result)).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd webapp/src/main/webapp && yarn test -- --run topology-tree-model`
Expected: FAIL — functions not exported

- [ ] **Step 3: Implement highlight model types and functions**

Add to `topology-tree-model.ts`:

```typescript
export interface HighlightState {
  status: 'active' | 'provisioned' | 'failed';
  executionId: string;
  stepName: string;
}

export interface ZoneHighlight {
  total: number;
  counts: { active: number; provisioned: number; failed: number };
}

export function computeZoneHighlight(
  branch: TreeBranch,
  highlightMap: Record<string, HighlightState>,
): ZoneHighlight | null {
  const highlighted = branch.devices.filter(d => highlightMap[d.deviceId]);
  if (highlighted.length < 2) return null;
  const counts = { active: 0, provisioned: 0, failed: 0 };
  for (const d of highlighted) {
    counts[highlightMap[d.deviceId]!.status]++;
  }
  return { total: highlighted.length, counts };
}

export function formatZoneHighlightSummary(zone: ZoneHighlight): string {
  const parts: string[] = [];
  if (zone.counts.active > 0) parts.push(`${zone.counts.active} active`);
  if (zone.counts.provisioned > 0) parts.push(`${zone.counts.provisioned} provisioned`);
  if (zone.counts.failed > 0) parts.push(`${zone.counts.failed} failed`);
  return parts.join(', ');
}

export interface BindingEventData {
  operation: string;
  binding: Record<string, unknown>;
}

export function applyBindingEvent(
  current: Record<string, HighlightState>,
  data: BindingEventData,
): Record<string, HighlightState> {
  const next = { ...current };
  const binding = data.binding;

  switch (data.operation) {
    case 'binding-start': {
      const deviceIds = binding.deviceIds as string[];
      const executionId = binding.executionId as string;
      const stepName = binding.stepName as string;
      for (const id of deviceIds) {
        next[id] = { status: 'active', executionId, stepName };
      }
      break;
    }
    case 'binding-update': {
      const deviceId = binding.deviceId as string;
      const status = (binding.status as string) === 'PROVISIONED' ? 'provisioned' as const : 'failed' as const;
      const existing = current[deviceId];
      if (existing) {
        next[deviceId] = { ...existing, status };
      }
      break;
    }
    case 'binding-clear': {
      const executionId = binding.executionId as string;
      for (const [id, hl] of Object.entries(next)) {
        if (hl.executionId === executionId) delete next[id];
      }
      break;
    }
  }
  return next;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd webapp/src/main/webapp && yarn test -- --run topology-tree-model`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add webapp/src/main/webapp/src/components/topology-tree-model.ts webapp/src/main/webapp/src/components/topology-tree-model.test.ts
git commit -m "feat(#129): highlight model types and zone aggregation logic"
```

### Task 5: Tree view highlighting and zone badges

**Files:**
- Modify: `webapp/src/main/webapp/src/components/topology-tree.ts`

**Interfaces:**
- Consumes: `HighlightState`, `ZoneHighlight`, `computeZoneHighlight()`, `formatZoneHighlightSummary()` (Task 4)
- Produces: `IoTTopologyTree` component with `highlightMap` state, SSE subscription via `SSEManager`, zone badge rendering, CSS highlight classes

- [ ] **Step 1: Add highlightMap state and SSE subscription**

Add to `IoTTopologyTree`:

```typescript
import { computeZoneHighlight, formatZoneHighlightSummary, type HighlightState } from './topology-tree-model.js';

// New state property
@state() private _highlightMap: Record<string, HighlightState> = {};

// New config property for SSE endpoint
@property({ attribute: 'sse-endpoint' }) sseEndpoint?: string;
```

Add SSE subscription lifecycle using `SSEManager` (import from the platform package):

```typescript
private _sseManager = new SSEManager();
private _sseHandler = (event: MessageEvent) => {
  const data = JSON.parse(event.data);
  if (data.operation?.startsWith('binding-')) {
    this._handleBindingEvent(data);
  } else if (data.operation === 'snapshot' || data.operation === 'update') {
    // Update nodes from SSE (existing node update path)
    if (data.nodes) this.nodes = data.nodes;
    if (data.locationAggregates) this.locationAggregates = data.locationAggregates;
  }
};

private _handleBindingEvent(data: { operation: string; binding: Record<string, unknown> }): void {
  const binding = data.binding;
  const next = { ...this._highlightMap };

  switch (data.operation) {
    case 'binding-start': {
      const deviceIds = binding.deviceIds as string[];
      const executionId = binding.executionId as string;
      const stepName = binding.stepName as string;
      for (const id of deviceIds) {
        next[id] = { status: 'active', executionId, stepName };
      }
      break;
    }
    case 'binding-update': {
      const deviceId = binding.deviceId as string;
      const status = (binding.status as string) === 'PROVISIONED' ? 'provisioned' : 'failed';
      const existing = this._highlightMap[deviceId];
      if (existing) {
        next[deviceId] = { ...existing, status };
      }
      break;
    }
    case 'binding-complete': {
      const executionId = binding.executionId as string;
      // Brief flash then clear — use setTimeout for 1.5s delay
      setTimeout(() => {
        const cleared = { ...this._highlightMap };
        for (const [id, hl] of Object.entries(cleared)) {
          if (hl.executionId === executionId) delete cleared[id];
        }
        this._highlightMap = cleared;
      }, 1500);
      break;
    }
    case 'binding-clear': {
      const executionId = binding.executionId as string;
      for (const [id, hl] of Object.entries(next)) {
        if (hl.executionId === executionId) delete next[id];
      }
      break;
    }
  }
  this._highlightMap = next;
}
```

Add SSE lifecycle:

```typescript
override disconnectedCallback(): void {
  if (this.sseEndpoint) {
    this._sseManager.unsubscribe(this.sseEndpoint, this._sseHandler);
  }
  super.disconnectedCallback();
}

override willUpdate(changed: PropertyValues): void {
  if (changed.has('sseEndpoint')) {
    const old = changed.get('sseEndpoint') as string | undefined;
    if (old) this._sseManager.unsubscribe(old, this._sseHandler);
    if (this.sseEndpoint) this._sseManager.subscribe(this.sseEndpoint, this._sseHandler);
  }
}
```

- [ ] **Step 2: Add highlight CSS classes**

Add to `static override styles`:

```css
.device-node.highlight-active {
  animation: pulse-highlight 1.5s ease-in-out infinite;
  border-left: 3px solid var(--pages-accent-color, #1a73e8);
}
.device-node.highlight-provisioned {
  border-left: 3px solid var(--pages-success-color, #22c55e);
}
.device-node.highlight-failed {
  border-left: 3px solid var(--pages-error-color, #ef4444);
}
.branch-node.zone-highlight {
  background: var(--pages-accent-color-subtle, #e8f0fe);
}
.zone-badge {
  font-size: 11px; color: var(--pages-accent-color, #1a73e8); font-weight: 600;
}
@keyframes pulse-highlight {
  0%, 100% { border-left-color: var(--pages-accent-color, #1a73e8); opacity: 1; }
  50% { border-left-color: var(--pages-accent-color, #1a73e8); opacity: 0.5; }
}
```

- [ ] **Step 3: Update _renderDevice to apply highlight classes**

```typescript
private _renderDevice(node: TopologyNode): TemplateResult {
  const color = DRIFT_COLORS[node.driftStatus] ?? DRIFT_COLORS['UNKNOWN']!;
  const isUnmonitored = node.driftStatus === 'UNMONITORED';
  const highlight = this._highlightMap[node.deviceId];
  const highlightClass = highlight ? `highlight-${highlight.status}` : '';
  return html`
    <li role="treeitem" aria-selected="false">
      <div class="device-node ${highlightClass}" tabindex="-1"
        @click=${() => this._selectDevice(node)}
        @keydown=${(e: KeyboardEvent) => { if (e.key === 'Enter') this._selectDevice(node); }}>
        <span class="drift-badge ${isUnmonitored ? 'unmonitored' : ''}"
              style="${isUnmonitored ? '' : `background: ${color}`}"
              title=${node.driftStatus}></span>
        <span class="device-label">${node.label}</span>
        <span class="device-class">${node.deviceClass}</span>
        ${node.driftDetail ? html`<span class="drift-detail">${node.driftDetail}</span>` : nothing}
      </div>
    </li>
  `;
}
```

- [ ] **Step 4: Update _renderBranch to show zone badges**

```typescript
private _renderBranch(branch: TreeBranch): TemplateResult {
  const expanded = this._expanded.has(branch.path);
  const aggregate = this.locationAggregates[branch.path];
  const summary = aggregate ? formatAggregateSummary(aggregate) : '';
  const zoneHighlight = computeZoneHighlight(branch, this._highlightMap);

  return html`
    <li role="treeitem" aria-expanded=${expanded}>
      <div class="branch-node ${zoneHighlight ? 'zone-highlight' : ''}" tabindex="-1"
        @click=${() => this._toggle(branch.path)}
        @keydown=${(e: KeyboardEvent) => {
          if (e.key === 'Enter' || e.key === 'ArrowRight') { if (!expanded) this._toggle(branch.path); }
          if (e.key === 'ArrowLeft') { if (expanded) this._toggle(branch.path); }
        }}>
        <span class="toggle">${expanded ? '▼' : '▶'}</span>
        <span class="branch-name">${branch.name}</span>
        ${zoneHighlight
          ? html`<span class="zone-badge">${formatZoneHighlightSummary(zoneHighlight)}</span>`
          : !expanded && summary
            ? html`<span class="aggregate">${summary}</span>`
            : nothing}
      </div>
      ${expanded ? html`
        <ul role="group">
          ${[...branch.children.values()].map(child => this._renderBranch(child))}
          ${branch.devices.map(device => this._renderDevice(device))}
        </ul>
      ` : nothing}
    </li>
  `;
}
```

- [ ] **Step 5: Run frontend tests**

Run: `cd webapp/src/main/webapp && yarn test -- --run`
Expected: PASS — all existing + new tests

- [ ] **Step 6: Commit**

```bash
git add webapp/src/main/webapp/src/components/topology-tree.ts
git commit -m "feat(#129): tree view highlighting with SSE subscription and zone badges"
```

## Batch 4: Graph View and Build Verification

### Task 6: Graph view highlighting with summary header

**Files:**
- Modify: `webapp/src/main/webapp/src/components/topology-graph.ts`

**Interfaces:**
- Consumes: `HighlightState` (Task 4), SSE binding events (same pattern as Task 5)
- Produces: `IoTTopologyGraph` with `highlightMap` state, node glow highlights, summary header line

- [ ] **Step 1: Add highlightMap state, SSE subscription, and binding handler**

Import `HighlightState` from `./topology-tree-model.js`. Add these to `IoTTopologyGraph`:

```typescript
import { type HighlightState } from './topology-tree-model.js';
import { SSEManager } from '@casehubio/pages-ui';

@property({ attribute: 'sse-endpoint' }) sseEndpoint?: string;
@state() private _highlightMap: Record<string, HighlightState> = {};

private _sseManager = new SSEManager();
private _sseHandler = (event: MessageEvent) => {
  const data = JSON.parse(event.data);
  if (data.operation?.startsWith('binding-')) {
    this._handleBindingEvent(data);
  } else if (data.operation === 'snapshot' || data.operation === 'update') {
    if (data.nodes) this.nodes = data.nodes;
    if (data.edges) this.edges = data.edges;
  }
};

private _handleBindingEvent(data: { operation: string; binding: Record<string, unknown> }): void {
  const binding = data.binding;
  const next = { ...this._highlightMap };

  switch (data.operation) {
    case 'binding-start': {
      const deviceIds = binding.deviceIds as string[];
      const executionId = binding.executionId as string;
      const stepName = binding.stepName as string;
      for (const id of deviceIds) {
        next[id] = { status: 'active', executionId, stepName };
      }
      break;
    }
    case 'binding-update': {
      const deviceId = binding.deviceId as string;
      const status = (binding.status as string) === 'PROVISIONED' ? 'provisioned' : 'failed';
      const existing = this._highlightMap[deviceId];
      if (existing) {
        next[deviceId] = { ...existing, status };
      }
      break;
    }
    case 'binding-complete': {
      const executionId = binding.executionId as string;
      setTimeout(() => {
        const cleared = { ...this._highlightMap };
        for (const [id, hl] of Object.entries(cleared)) {
          if (hl.executionId === executionId) delete cleared[id];
        }
        this._highlightMap = cleared;
      }, 1500);
      break;
    }
    case 'binding-clear': {
      const executionId = binding.executionId as string;
      for (const [id, hl] of Object.entries(next)) {
        if (hl.executionId === executionId) delete next[id];
      }
      break;
    }
  }
  this._highlightMap = next;
}

override disconnectedCallback(): void {
  if (this.sseEndpoint) {
    this._sseManager.unsubscribe(this.sseEndpoint, this._sseHandler);
  }
  super.disconnectedCallback();
}

override willUpdate(changed: PropertyValues): void {
  if (changed.has('sseEndpoint')) {
    const old = changed.get('sseEndpoint') as string | undefined;
    if (old) this._sseManager.unsubscribe(old, this._sseHandler);
    if (this.sseEndpoint) this._sseManager.subscribe(this.sseEndpoint, this._sseHandler);
  }
}
```

- [ ] **Step 2: Add highlight CSS**

Add to `static override styles`:

```css
.graph-node.highlight-active {
  box-shadow: 0 0 8px var(--pages-accent-color, #1a73e8);
  animation: pulse-glow 1.5s ease-in-out infinite;
}
.graph-node.highlight-provisioned {
  border-color: var(--pages-success-color, #22c55e) !important;
  box-shadow: 0 0 4px var(--pages-success-color, #22c55e);
}
.graph-node.highlight-failed {
  border-color: var(--pages-error-color, #ef4444) !important;
  box-shadow: 0 0 4px var(--pages-error-color, #ef4444);
}
.scenario-summary {
  padding: 8px 12px; font-size: 12px; color: var(--pages-accent-color, #1a73e8);
  font-weight: 600; border-bottom: 1px solid var(--pages-border-color, #e5e7eb);
}
@keyframes pulse-glow {
  0%, 100% { box-shadow: 0 0 8px var(--pages-accent-color, #1a73e8); }
  50% { box-shadow: 0 0 16px var(--pages-accent-color, #1a73e8); }
}
```

- [ ] **Step 3: Update node rendering with highlight classes**

In the `render()` method, add highlight class to each node:

```typescript
const highlight = this._highlightMap[node.id];
const highlightClass = highlight ? `highlight-${highlight.status}` : '';
// Apply: class="graph-node ${highlightClass}"
```

- [ ] **Step 4: Add summary header**

Before the graph grid, conditionally render a summary:

```typescript
const highlightedCount = Object.keys(this._highlightMap).length;
const zoneCount = highlightedCount > 0
  ? new Set(this._viewModel.nodes
      .filter(n => this._highlightMap[n.id])
      .map(n => this.nodes.find(orig => orig.deviceId === n.id)?.locationPath.join('/') ?? 'unknown'))
    .size
  : 0;

// In render():
${highlightedCount > 0 ? html`
  <div class="scenario-summary">${highlightedCount} device${highlightedCount !== 1 ? 's' : ''} in ${zoneCount} zone${zoneCount !== 1 ? 's' : ''} affected</div>
` : ''}
```

- [ ] **Step 5: Run frontend tests**

Run: `cd webapp/src/main/webapp && yarn test -- --run`
Expected: PASS

- [ ] **Step 6: Run full Maven build**

Run: `mvn --batch-mode install -DskipTests=false`
Expected: PASS — full build green (excluding pre-existing webapp-api exclusions)

- [ ] **Step 7: Commit**

```bash
git add webapp/src/main/webapp/src/components/topology-graph.ts
git commit -m "feat(#129): graph view highlighting with node glow and summary header"
```

## References

- `2026-10-06-scenario-topology-binding-design.md` — design spec this plan implements
- `DesiredStateDeliveryHandler.java:31` — delivery handler to modify
- `DeviceCommandDispatcher.java:19` — command dispatcher (audit event only)
- `DefaultIoTTopologyApi.java:24-65` — topology SSE with BroadcastProcessor to extract
- `IoTGoalCompiler.java:52-66` — dual-node creation (physical + config) per device
- `IoTNodeTypes.java:33-39` — config/physical node type naming
- `topology-tree.ts:14-112` — tree component to add highlighting
- `topology-graph.ts:14-78` — graph component to add highlighting
- `topology-tree-model.ts:1-81` — model types to extend with highlight
- `topology.ts:1-20` — page definition with hostPanel config
- GitHub #129 — focal issue
- GitHub #131 — deferred: command binding with execution context
- GitHub #132 — deferred: topology SSE tenancy filtering
