# CloudEvent Adapter for StateChangeEvent — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `IoTCloudEventAdapter` — an `@ApplicationScoped` CDI bean in `casehub-iot-api` that observes `StateChangeEvent` and fires `Event<CloudEvent>.fireAsync()` with the correct envelope mapping.

**Architecture:** Single adapter class in `api` module. Observes `@ObservesAsync StateChangeEvent`, builds a `CloudEvent` using `CloudEventBuilder.v1()`, serializes the event payload with Jackson `ObjectMapper`, and fires it as a CDI async event. No CDI container needed for tests — constructor injection with a capturing `Event<CloudEvent>` implementation.

**Tech Stack:** Java 21, Quarkus CDI (ArC), CloudEvents Java SDK 4.x (`io.cloudevents:cloudevents-core`), Jackson, JUnit 5, AssertJ.

**Spec:** `docs/superpowers/specs/2026-06-20-cloudevent-adapter-design.md`

---

### Task 1: Add `casehub-platform-api` dependency to `api/pom.xml`

**Files:**
- Modify: `api/pom.xml`

- [ ] **Step 1: Add the dependency**

Add after the `jackson-datatype-jsr310` dependency block in `api/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-api</artifactId>
</dependency>
```

Version is managed by `casehub-parent` BOM — no `<version>` tag needed.

- [ ] **Step 2: Verify the build compiles**

Run: `mvn --batch-mode compile -pl api`
Expected: BUILD SUCCESS. `cloudevents-core` now transitively on the classpath.

- [ ] **Step 3: Commit**

```
git add api/pom.xml
git commit -m "build: add casehub-platform-api dependency to iot-api #19"
```

---

### Task 2: Write failing tests for IoTCloudEventAdapter

**Files:**
- Create: `api/src/test/java/io/casehub/iot/api/IoTCloudEventAdapterTest.java`

The adapter needs a non-CDI-managed `Event<CloudEvent>` for testing. Use a simple capturing implementation inline in the test class — no mock framework needed.

- [ ] **Step 1: Write all four test methods**

```java
package io.casehub.iot.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.cloudevents.CloudEvent;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.NotificationOptions;
import jakarta.enterprise.util.TypeLiteral;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.UncheckedIOException;
import java.lang.annotation.Annotation;
import java.math.BigDecimal;
import java.net.URI;
import java.time.Instant;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;

class IoTCloudEventAdapterTest {

    private static final Instant OCCURRED = Instant.parse("2026-06-20T12:00:00Z");
    private static final ObjectMapper MAPPER = new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

    private List<CloudEvent> firedEvents;
    private IoTCloudEventAdapter adapter;

    @BeforeEach
    void setUp() {
        firedEvents = new ArrayList<>();
        Event<CloudEvent> capturingEvent = new CapturingEvent(firedEvents);
        adapter = new IoTCloudEventAdapter(capturingEvent, MAPPER);
    }

    @Test
    void correctMapping() throws Exception {
        var after = new ThermostatDevice.Builder()
            .deviceId("therm-1").deviceClass(DeviceClass.THERMOSTAT).label("Living Room")
            .available(true).lastUpdated(OCCURRED).tenancyId("tenant-a").providerId("homeassistant")
            .currentTemperature(new Temperature(BigDecimal.valueOf(21), Temperature.TemperatureUnit.CELSIUS))
            .targetTemperature(new Temperature(BigDecimal.valueOf(22), Temperature.TemperatureUnit.CELSIUS))
            .mode(ThermostatMode.HEAT)
            .build();
        var before = new ThermostatDevice.Builder()
            .deviceId("therm-1").deviceClass(DeviceClass.THERMOSTAT).label("Living Room")
            .available(true).lastUpdated(OCCURRED).tenancyId("tenant-a").providerId("homeassistant")
            .currentTemperature(new Temperature(BigDecimal.valueOf(20), Temperature.TemperatureUnit.CELSIUS))
            .targetTemperature(new Temperature(BigDecimal.valueOf(22), Temperature.TemperatureUnit.CELSIUS))
            .mode(ThermostatMode.HEAT)
            .build();
        var event = new StateChangeEvent(before, after,
            Set.of(ThermostatDevice.CAP_CURRENT_TEMPERATURE), OCCURRED, "homeassistant");

        adapter.onStateChange(event);

        assertThat(firedEvents).hasSize(1);
        CloudEvent ce = firedEvents.get(0);
        assertThat(ce.getType()).isEqualTo("io.casehub.iot.state_change.thermostat");
        assertThat(ce.getSource()).isEqualTo(URI.create("/casehub-iot"));
        assertThat(ce.getSubject()).isEqualTo("device/therm-1");
        assertThat(ce.getTime()).isEqualTo(OCCURRED.atOffset(ZoneOffset.UTC));
        assertThat(ce.getId()).isNotBlank();
        assertThat(ce.getDataContentType()).isEqualTo("application/json");
        assertThat(ce.getExtension("tenancyid")).isEqualTo("tenant-a");
        assertThat(ce.getExtension("providerid")).isEqualTo("homeassistant");

        byte[] data = ce.getData().toBytes();
        StateChangeEvent deserialized = MAPPER.readValue(data, StateChangeEvent.class);
        assertThat(deserialized.after().deviceId()).isEqualTo("therm-1");
        assertThat(deserialized.before().deviceId()).isEqualTo("therm-1");
        assertThat(deserialized.changedCapabilities()).containsExactly(ThermostatDevice.CAP_CURRENT_TEMPERATURE);
    }

    @Test
    void polymorphismGuard() {
        // Simulates a vendor supplement type (like HomeAssistantThermostat) — a subclass
        // whose getSimpleName() differs from the base class. The adapter must use
        // deviceClass() (→ THERMOSTAT), not getClass().getSimpleName() (→ FakeThermostat).
        var after = new FakeThermostat.Builder()
            .deviceId("ha-therm").deviceClass(DeviceClass.THERMOSTAT).label("HA Thermostat")
            .available(true).lastUpdated(OCCURRED).tenancyId("tenant-a").providerId("homeassistant")
            .currentTemperature(new Temperature(BigDecimal.valueOf(21), Temperature.TemperatureUnit.CELSIUS))
            .targetTemperature(new Temperature(BigDecimal.valueOf(22), Temperature.TemperatureUnit.CELSIUS))
            .mode(ThermostatMode.HEAT)
            .build();
        var event = new StateChangeEvent(null, after, Set.of(), OCCURRED, "homeassistant");

        adapter.onStateChange(event);

        assertThat(firedEvents).hasSize(1);
        assertThat(firedEvents.get(0).getType())
            .isEqualTo("io.casehub.iot.state_change.thermostat");
    }

    @Test
    void nullBefore_initialDiscovery() throws Exception {
        var after = new ThermostatDevice.Builder()
            .deviceId("therm-new").deviceClass(DeviceClass.THERMOSTAT).label("New Thermostat")
            .available(true).lastUpdated(OCCURRED).tenancyId("tenant-b").providerId("openhab")
            .currentTemperature(new Temperature(BigDecimal.valueOf(19), Temperature.TemperatureUnit.CELSIUS))
            .targetTemperature(new Temperature(BigDecimal.valueOf(21), Temperature.TemperatureUnit.CELSIUS))
            .mode(ThermostatMode.AUTO)
            .build();
        var event = new StateChangeEvent(null, after, Set.of(), OCCURRED, "openhab");

        adapter.onStateChange(event);

        assertThat(firedEvents).hasSize(1);
        CloudEvent ce = firedEvents.get(0);
        assertThat(ce.getType()).isEqualTo("io.casehub.iot.state_change.thermostat");
        assertThat(ce.getExtension("providerid")).isEqualTo("openhab");

        StateChangeEvent deserialized = MAPPER.readValue(ce.getData().toBytes(), StateChangeEvent.class);
        assertThat(deserialized.before()).isNull();
        assertThat(deserialized.after().deviceId()).isEqualTo("therm-new");
    }

    @Test
    void compoundDeviceClass_underscoreConvention() {
        var after = PresenceSensor.builder()
            .deviceId("ps-1").deviceClass(DeviceClass.PRESENCE_SENSOR).label("Front Door")
            .available(true).lastUpdated(OCCURRED).tenancyId("tenant-a").providerId("homeassistant")
            .present(true).lastSeen(OCCURRED)
            .build();
        var event = new StateChangeEvent(null, after, Set.of(), OCCURRED, "homeassistant");

        adapter.onStateChange(event);

        assertThat(firedEvents).hasSize(1);
        assertThat(firedEvents.get(0).getType())
            .isEqualTo("io.casehub.iot.state_change.presence_sensor");
    }

    /**
     * Test-only subclass simulating a vendor supplement type.
     * getSimpleName() returns "FakeThermostat", not "ThermostatDevice" —
     * ensures the adapter uses deviceClass(), not getClass().getSimpleName().
     */
    private static class FakeThermostat extends ThermostatDevice {
        private FakeThermostat(Builder builder) { super(builder); }
        static class Builder extends ThermostatDevice.AbstractBuilder<FakeThermostat, Builder> {
            @Override protected Builder self() { return this; }
            @Override public FakeThermostat build() { return new FakeThermostat(this); }
        }
    }

    private static class CapturingEvent implements Event<CloudEvent> {
        private final List<CloudEvent> captured;

        CapturingEvent(List<CloudEvent> captured) {
            this.captured = captured;
        }

        @Override
        public void fire(CloudEvent event) {
            throw new UnsupportedOperationException("adapter must use fireAsync");
        }

        @Override
        public <U extends CloudEvent> CompletionStage<U> fireAsync(U event) {
            captured.add(event);
            return CompletableFuture.completedFuture(event);
        }

        @Override
        public <U extends CloudEvent> CompletionStage<U> fireAsync(U event, NotificationOptions options) {
            return fireAsync(event);
        }

        @Override public Event<CloudEvent> select(Annotation... qualifiers) { throw new UnsupportedOperationException(); }
        @Override public <U extends CloudEvent> Event<U> select(Class<U> subtype, Annotation... qualifiers) { throw new UnsupportedOperationException(); }
        @Override public <U extends CloudEvent> Event<U> select(TypeLiteral<U> subtype, Annotation... qualifiers) { throw new UnsupportedOperationException(); }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest=IoTCloudEventAdapterTest`
Expected: COMPILATION FAILURE — `IoTCloudEventAdapter` does not exist yet.

- [ ] **Step 3: Commit the failing tests**

```
git add api/src/test/java/io/casehub/iot/api/IoTCloudEventAdapterTest.java
git commit -m "test: add failing tests for IoTCloudEventAdapter #19"
```

---

### Task 3: Implement IoTCloudEventAdapter

**Files:**
- Create: `api/src/main/java/io/casehub/iot/api/IoTCloudEventAdapter.java`

- [ ] **Step 1: Write the adapter**

```java
package io.casehub.iot.api;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import java.io.UncheckedIOException;
import java.net.URI;
import java.time.ZoneOffset;
import java.util.UUID;

@ApplicationScoped
public class IoTCloudEventAdapter {

    private static final URI SOURCE = URI.create("/casehub-iot");
    private static final String TYPE_PREFIX = "io.casehub.iot.state_change.";

    private final Event<CloudEvent> cloudEvents;
    private final ObjectMapper mapper;

    @Inject
    public IoTCloudEventAdapter(Event<CloudEvent> cloudEvents, ObjectMapper mapper) {
        this.cloudEvents = cloudEvents;
        this.mapper = mapper;
    }

    void onStateChange(@ObservesAsync StateChangeEvent event) {
        String deviceClass = event.after().deviceClass().name().toLowerCase();
        byte[] data;
        try {
            data = mapper.writeValueAsBytes(event);
        } catch (JsonProcessingException e) {
            throw new UncheckedIOException(e);
        }

        CloudEvent ce = CloudEventBuilder.v1()
            .withId(UUID.randomUUID().toString())
            .withType(TYPE_PREFIX + deviceClass)
            .withSource(SOURCE)
            .withSubject("device/" + event.after().deviceId())
            .withTime(event.occurredAt().atOffset(ZoneOffset.UTC))
            .withData("application/json", data)
            .withExtension("tenancyid", event.after().tenancyId())
            .withExtension("providerid", event.providerId())
            .build();

        cloudEvents.fireAsync(ce);
    }
}
```

- [ ] **Step 2: Run all tests**

Run: `mvn --batch-mode test -pl api -Dtest=IoTCloudEventAdapterTest`
Expected: All 4 tests PASS.

- [ ] **Step 3: Commit**

```
git add api/src/main/java/io/casehub/iot/api/IoTCloudEventAdapter.java
git commit -m "feat: implement IoTCloudEventAdapter — StateChangeEvent to CloudEvent #19"
```

---

### Task 4: Full build validation

**Files:** None — verification only.

- [ ] **Step 1: Run the full multi-module build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile and all tests pass (68+ existing tests plus the 4 new ones).

- [ ] **Step 2: Commit if any Jandex or generated files changed**

Check `git status --short`. If the Jandex index was regenerated in `api/target/`, it's not tracked — nothing to commit. If clean, skip this step.

---

### Task 5: Update ARC42STORIES.MD

**Files:**
- Modify: `ARC42STORIES.MD`

- [ ] **Step 1: Update header**

Change:
```
**Build position:** Foundation — no casehubio dependencies
```
To:
```
**Build position:** Foundation — depends on casehub-platform-api (shared vocabulary)
```

Change:
```
**Depends on:** none
```
To:
```
**Depends on:** casehub-platform-api (types only, no runtime behaviour)
```

- [ ] **Step 2: Update §2 Constraints — Dependencies table**

Change:
```
No casehubio dependencies. Foundation-tier peer to casehub-connectors.
```
To:
```
Depends on casehub-platform-api (shared vocabulary — types only, no runtime behaviour). Foundation-tier peer to casehub-connectors.
```

- [ ] **Step 3: Commit**

```
git add ARC42STORIES.MD
git commit -m "docs: update ARC42STORIES — casehub-platform-api dependency #19"
```

---

### Task 6: Code review

- [ ] **Step 1: Run `superpowers:requesting-code-review`**

Review the diff from the branch against main. Any finding Minor or above that isn't fixed this session must be captured as a GitHub issue.

- [ ] **Step 2: Fix any findings, commit fixes**

- [ ] **Step 3: File issues for deferred findings (if any)**

---

### Task 7: Doc sync

- [ ] **Step 1: Run `implementation-doc-sync`**

Check whether CLAUDE.md, DESIGN.md, or any other docs need updating to reflect the new adapter and the `casehub-platform-api` dependency.
