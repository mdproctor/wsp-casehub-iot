# Topology SSE Tenancy Filtering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #132 — Topology SSE tenancy filtering on broadcast stream
**Issue group:** #132

**Goal:** Filter topology SSE broadcast events per-client by tenancyId, closing the known tenancy leak (ARC42STORIES.MD line 1480).

**Architecture:** Add `tenancyId` to `TopologyStreamEvent` (Jackson `@JsonIgnore` — server-side only), then filter the broadcast `Multi` per subscriber in `streamTopology()`. Both event producers (`onStateChange`, `ScenarioTopologyBinder`) already have access to tenancyId from their source events. Follows the proven pattern from `DefaultIoTDeviceApi.streamDevices()`.

**Tech Stack:** Java 21+, Quarkus, SmallRye Mutiny (`BroadcastProcessor`), Jackson (`@JsonIgnore`), JUnit 5, AssertJ

## Global Constraints

- `TopologyStreamEvent.tenancyId` must NOT appear in JSON serialization (use `@JsonIgnore`)
- Follow existing `DefaultIoTDeviceApi.streamDevices()` filter pattern
- `casehub-iot-api` is a public API surface — `ScenarioBindingEvent` is not modified

---

## Batch 1: Tenancy filtering on topology SSE broadcast

### Task 1: Add tenancyId to TopologyStreamEvent and filter in DefaultIoTTopologyApi

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/service/TopologyStreamEvent.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTTopologyApi.java`
- Modify: `webapp/src/test/java/io/casehub/iot/webapp/app/service/DefaultIoTTopologyApiTest.java`

**Interfaces:**
- Consumes: `TopologyAssembler.assemble(tenancyId)`, `TopologyAssembler.reassembleNode(deviceId, tenancyId)`, `BroadcastProcessor<TopologyStreamEvent>`, `StateChangeEvent.after().tenancyId()`
- Produces: `TopologyStreamEvent(tenancyId, operation, nodes)` constructor, `TopologyStreamEvent.tenancyId()` accessor (used by Task 2)

- [ ] **Step 1: Write failing test — broadcast events are filtered by tenancyId**

Add to `DefaultIoTTopologyApiTest.java`:

```java
@Test
void streamTopology_filters_broadcast_by_tenancyId() {
    registry.addDevice(lightBuilder("l1", "t1").location("HQ/Room1").build());

    var events = new ArrayList<TopologyStreamEvent>();
    api.streamTopology("t1")
            .subscribe().with(events::add);

    // Broadcast an event for tenant t1 — should arrive
    var node = assembler.reassembleNode("l1", "t1");
    api.broadcaster.onNext(new TopologyStreamEvent("t1", "update", List.of(node)));

    // Broadcast an event for tenant t2 — should be filtered out
    registry.addDevice(lightBuilder("l2", "t2").location("Other").build());
    var otherNode = assembler.reassembleNode("l2", "t2");
    api.broadcaster.onNext(new TopologyStreamEvent("t2", "update", List.of(otherNode)));

    // snapshot + 1 matching update = 2 events
    assertEquals(2, events.size());
    assertEquals("snapshot", events.get(0).operation());
    assertEquals("update", events.get(1).operation());
}
```

Add necessary imports at the top of the test file:

```java
import java.util.ArrayList;
import java.util.List;
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl webapp -Dtest=DefaultIoTTopologyApiTest#streamTopology_filters_broadcast_by_tenancyId`
Expected: compilation failure — `TopologyStreamEvent` constructor doesn't accept tenancyId yet

- [ ] **Step 3: Add tenancyId field to TopologyStreamEvent**

Replace the full record in `TopologyStreamEvent.java`:

```java
package io.casehub.iot.webapp.app.service;

import com.fasterxml.jackson.annotation.JsonIgnore;
import io.casehub.iot.webapp.rest.TopologyNode;

import java.util.List;
import java.util.Map;

public record TopologyStreamEvent(
        @JsonIgnore String tenancyId,
        String operation,
        List<TopologyNode> nodes,
        Map<String, Object> binding
) {
    public TopologyStreamEvent(String tenancyId, String operation, List<TopologyNode> nodes) {
        this(tenancyId, operation, nodes, null);
    }

    public static TopologyStreamEvent binding(String tenancyId, String operation, Map<String, Object> data) {
        return new TopologyStreamEvent(tenancyId, operation, List.of(), data);
    }
}
```

- [ ] **Step 4: Update DefaultIoTTopologyApi — filter broadcast, tag events with tenancyId**

In `DefaultIoTTopologyApi.java`, update `streamTopology()`:

```java
@PlatformStream("Stream topology node updates on device state changes")
@RestPath("/stream")
@RolesAllowed("iot-viewer")
public Multi<TopologyStreamEvent> streamTopology(
        @ContextParam("tenancyId") String tenancyId) {
    Multi<TopologyStreamEvent> snapshot = Multi.createFrom().item(() -> {
        var response = assembler.assemble(tenancyId);
        return new TopologyStreamEvent(tenancyId, "snapshot", response.nodes());
    });

    Multi<TopologyStreamEvent> updates = broadcaster
            .filter(e -> e.tenancyId().equals(tenancyId));

    return Multi.createBy().merging().streams(snapshot, updates);
}
```

Update `onStateChange()`:

```java
void onStateChange(@ObservesAsync StateChangeEvent event) {
    var device = event.after();
    try {
        var node = assembler.reassembleNode(device.deviceId(), device.tenancyId());
        broadcaster.onNext(new TopologyStreamEvent(
                device.tenancyId(), "update", List.of(node)));
    } catch (Exception ignored) {
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl webapp -Dtest=DefaultIoTTopologyApiTest#streamTopology_filters_broadcast_by_tenancyId`
Expected: PASS

- [ ] **Step 6: Run all DefaultIoTTopologyApiTest tests**

Run: `mvn --batch-mode test -pl webapp -Dtest=DefaultIoTTopologyApiTest`
Expected: all tests PASS (existing tests don't construct `TopologyStreamEvent` directly)

- [ ] **Step 7: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/service/TopologyStreamEvent.java
git add webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTTopologyApi.java
git add webapp/src/test/java/io/casehub/iot/webapp/app/service/DefaultIoTTopologyApiTest.java
git commit -m "feat(#132): add tenancyId to TopologyStreamEvent and filter broadcast per subscriber"
```

### Task 2: Update ScenarioTopologyBinder to pass tenancyId

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinder.java`
- Modify: `webapp/src/test/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinderTest.java`

**Interfaces:**
- Consumes: `TopologyStreamEvent.binding(tenancyId, operation, data)` from Task 1, `ScenarioBindingEvent.tenancyId()`
- Produces: no new interfaces — same broadcast, now with tenancyId populated

- [ ] **Step 1: Update existing tests to verify tenancyId is set on emitted events**

In `ScenarioTopologyBinderTest.java`, add an assertion to the first test (`stepStartProducesBindingStartEvent`) to verify tenancyId:

```java
assertThat(sse.tenancyId()).isEqualTo("tenant-1");
```

Add the same assertion to all 6 existing test methods (each already constructs `ScenarioBindingEvent` with `"tenant-1"`).

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl webapp -Dtest=ScenarioTopologyBinderTest`
Expected: compilation failure — `TopologyStreamEvent.binding()` signature changed (now requires tenancyId as first arg)

- [ ] **Step 3: Update ScenarioTopologyBinder to pass tenancyId**

In `ScenarioTopologyBinder.java`, update `onBindingEvent()` — add `event.tenancyId()` as the first argument to every `TopologyStreamEvent.binding()` call:

```java
void onBindingEvent(@Observes ScenarioBindingEvent event) {
    var streamEvent = switch (event) {
        case ScenarioBindingEvent.StepStart ss ->
            TopologyStreamEvent.binding(event.tenancyId(), "binding-start", Map.of(
                "executionId", ss.executionId(),
                "stepName", ss.stepName(),
                "deviceIds", ss.deviceIds()));
        case ScenarioBindingEvent.DeviceProvisioned dp ->
            TopologyStreamEvent.binding(event.tenancyId(), "binding-update", Map.of(
                "executionId", dp.executionId(),
                "deviceId", dp.deviceId(),
                "status", "PROVISIONED"));
        case ScenarioBindingEvent.DeviceFailed df ->
            TopologyStreamEvent.binding(event.tenancyId(), "binding-update", Map.of(
                "executionId", df.executionId(),
                "deviceId", df.deviceId(),
                "status", "FAILED"));
        case ScenarioBindingEvent.StepComplete sc ->
            TopologyStreamEvent.binding(event.tenancyId(), "binding-complete", Map.of(
                "executionId", sc.executionId(),
                "stepName", sc.stepName(),
                "outcome", "OK",
                "provisioned", sc.provisioned(),
                "failed", sc.failed()));
        case ScenarioBindingEvent.StepFailed sf ->
            TopologyStreamEvent.binding(event.tenancyId(), "binding-complete", Map.of(
                "executionId", sf.executionId(),
                "stepName", sf.stepName(),
                "outcome", "FAILED",
                "provisioned", sf.provisioned(),
                "failed", sf.failed()));
        case ScenarioBindingEvent.Clear c ->
            TopologyStreamEvent.binding(event.tenancyId(), "binding-clear", Map.of(
                "executionId", c.executionId()));
    };
    broadcaster.onNext(streamEvent);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl webapp -Dtest=ScenarioTopologyBinderTest`
Expected: all 6 tests PASS

- [ ] **Step 5: Run full webapp test suite**

Run: `mvn --batch-mode test -pl webapp`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinder.java
git add webapp/src/test/java/io/casehub/iot/webapp/app/service/ScenarioTopologyBinderTest.java
git commit -m "feat(#132): pass tenancyId through ScenarioTopologyBinder to broadcast events"
```

### Task 3: Full build verification and ARC42STORIES cleanup

**Files:**
- Modify: `ARC42STORIES.MD` (line 1480 — mark SSE tenancy leak as resolved)

**Interfaces:**
- Consumes: none — verification only
- Produces: none

- [ ] **Step 1: Run full project build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 2: Update ARC42STORIES.MD**

Find line 1480 (the SSE tenancy leak row) and update the status to indicate it's resolved. Change the concern status from open to resolved with the fix reference.

- [ ] **Step 3: Commit**

```bash
git add ARC42STORIES.MD
git commit -m "docs(#132): mark SSE tenancy leak as resolved in ARC42STORIES"
```

## References

- [2026-10-06-topology-sse-tenancy-filtering-design.md] — design spec this plan implements
- [DefaultIoTDeviceApi.java:129-143] — existing per-subscriber filter pattern
- [TopologyStreamEvent.java] — record to modify
- [DefaultIoTTopologyApi.java] — SSE stream endpoint with unfiltered broadcast
- [ScenarioTopologyBinder.java] — binding event producer
- [ScenarioBindingEvent.java:7-8] — tenancyId on sealed interface
- [ARC42STORIES.MD:1480] — known SSE tenancy leak concern
- [GitHub #132] — focal issue
- [GitHub #129] — parent issue that added ScenarioBindingEvent with tenancyId
