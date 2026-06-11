# C4: OpenHAB Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the OpenHAB provider module — REST discovery, SSE state subscription with 50ms coalescing, command dispatch, three supplement types.

**Architecture:** `OpenHabProvider` implements `DeviceProvider` SPI. Discovery via REST (`GET /rest/items?tags=Equipment&recursive=true`), state via SSE (`GET /rest/events`), commands via REST (`POST /rest/items/{itemName}`). Two REST client interfaces (separate timeout configs). Pure stateless mapper. SSE client owns state cache and coalescing.

**Tech Stack:** Quarkus 3.x, MicroProfile REST Client (RESTEasy Reactive), Jackson, Mutiny, Awaitility, MockWebServer.

**Spec:** `docs/superpowers/specs/2026-06-10-chapter4-openhab-provider-design.md` (rev 4)

**Reference implementation:** `homeassistant/` module (C3) — same patterns, follow its structure.

---

### Task 1: Project scaffold — pom.xml, test config, CoverDevice Javadoc

**Files:**
- Modify: `openhab/pom.xml`
- Modify: `api/src/main/java/io/casehub/iot/api/CoverDevice.java`
- Create: `openhab/src/main/resources/application.properties`
- Create: `openhab/src/test/resources/application.properties`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabMockServerResource.java`

- [ ] **Step 1: Update openhab/pom.xml with all dependencies**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-iot-parent</artifactId>
        <version>0.1-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-iot-openhab</artifactId>
    <name>CaseHub IoT — OpenHAB Provider</name>
    <description>OpenHAB provider (REST + SSE, semantic model) and OpenHAB supplement types</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-client</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-client-jackson</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>mockwebserver</artifactId>
            <version>5.3.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.awaitility</groupId>
            <artifactId>awaitility</artifactId>
            <version>4.2.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

- [ ] **Step 2: Create main application.properties**

File: `openhab/src/main/resources/application.properties`
```properties
# OpenHAB REST client — short-lived requests (discovery, commands)
quarkus.rest-client."openhab".url=${casehub.iot.openhab.url}
quarkus.rest-client."openhab".connect-timeout=5000
quarkus.rest-client."openhab".read-timeout=10000

# OpenHAB SSE client — long-lived event stream, NO read-timeout
quarkus.rest-client."openhab-sse".url=${casehub.iot.openhab.url}
quarkus.rest-client."openhab-sse".connect-timeout=5000
```

- [ ] **Step 3: Create test application.properties**

File: `openhab/src/test/resources/application.properties`
```properties
casehub.iot.openhab.url=http://localhost:8082
casehub.iot.openhab.token=test-token
casehub.iot.openhab.tenancy-id=test-tenant
casehub.iot.openhab.coalesce-window-ms=50
quarkus.http.test-port=8082
```

- [ ] **Step 4: Create OpenHabMockServerResource**

File: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabMockServerResource.java`

Follow `homeassistant/src/test/java/io/casehub/iot/homeassistant/MockWebServerResource.java` pattern. Inject URL into both REST client config keys.

```java
package io.casehub.iot.openhab;

import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;
import okhttp3.mockwebserver.MockWebServer;
import java.io.IOException;
import java.util.Map;

public class OpenHabMockServerResource implements QuarkusTestResourceLifecycleManager {

    static volatile MockWebServer INSTANCE;

    @Override
    public Map<String, String> start() {
        var server = new MockWebServer();
        try {
            server.start();
        } catch (IOException e) {
            throw new RuntimeException("Failed to start MockWebServer", e);
        }
        INSTANCE = server;
        String url = "http://localhost:" + server.getPort();
        return Map.of(
            "quarkus.rest-client.\"openhab\".url", url,
            "quarkus.rest-client.\"openhab-sse\".url", url,
            "casehub.iot.openhab.url", url
        );
    }

    @Override
    public void stop() {
        if (INSTANCE != null) {
            try { INSTANCE.shutdown(); } catch (IOException e) { /* best-effort */ }
            INSTANCE = null;
        }
    }
}
```

- [ ] **Step 5: Add CoverDevice position convention Javadoc**

In `api/src/main/java/io/casehub/iot/api/CoverDevice.java`, add Javadoc to `position()`:

```java
/**
 * Position as a percentage: 0 = fully closed, 100 = fully open.
 * Providers that use the opposite convention (e.g., OpenHAB Rollershutter:
 * 0=open, 100=closed) must invert before populating this field.
 */
public Optional<Integer> position() {
    return Optional.ofNullable(position);
}
```

- [ ] **Step 6: Verify build compiles**

Run: `mvn --batch-mode compile -pl openhab`
Expected: BUILD SUCCESS (empty module with dependencies resolved)

- [ ] **Step 7: Commit**

```
git add openhab/pom.xml openhab/src/ api/src/main/java/io/casehub/iot/api/CoverDevice.java
git commit -m "chore(openhab): project scaffold — pom deps, test infra, CoverDevice Javadoc #4"
```

---

### Task 2: Internal DTOs

**Files:**
- Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabItemDto.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabStateDescriptionDto.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabSseEventDto.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabStatePayloadDto.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabDtoTest.java`

- [ ] **Step 1: Write DTO deserialization tests**

File: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabDtoTest.java`

```java
package io.casehub.iot.openhab;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.openhab.internal.OpenHabItemDto;
import io.casehub.iot.openhab.internal.OpenHabSseEventDto;
import io.casehub.iot.openhab.internal.OpenHabStatePayloadDto;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class OpenHabDtoTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void itemDtoDeserializesEquipmentGroupWithMembers() throws Exception {
        String json = """
            {
              "type": "Group",
              "name": "LivingRoom_Thermostat",
              "label": "Living Room Thermostat",
              "state": "NULL",
              "tags": ["Equipment", "HVAC"],
              "members": [
                {
                  "type": "Number:Temperature",
                  "name": "LivingRoom_Temperature",
                  "label": "Current Temperature",
                  "state": "21.5",
                  "tags": ["Measurement", "Temperature"],
                  "stateDescription": {"pattern": "%.1f °C"}
                },
                {
                  "type": "Number:Temperature",
                  "name": "LivingRoom_Setpoint",
                  "label": "Target Temperature",
                  "state": "22.0",
                  "tags": ["Setpoint", "Temperature"],
                  "stateDescription": {"pattern": "%.1f °C"}
                }
              ]
            }
            """;

        OpenHabItemDto item = mapper.readValue(json, OpenHabItemDto.class);

        assertThat(item.type()).isEqualTo("Group");
        assertThat(item.name()).isEqualTo("LivingRoom_Thermostat");
        assertThat(item.tags()).containsExactly("Equipment", "HVAC");
        assertThat(item.members()).hasSize(2);
        assertThat(item.members().get(0).name()).isEqualTo("LivingRoom_Temperature");
        assertThat(item.members().get(0).stateDescription()).isNotNull();
        assertThat(item.members().get(0).stateDescription().pattern()).isEqualTo("%.1f °C");
    }

    @Test
    void itemDtoIgnoresUnknownFields() throws Exception {
        String json = """
            {"type":"Switch","name":"Light_1","state":"ON","tags":[],"unknownField":"value"}
            """;
        OpenHabItemDto item = mapper.readValue(json, OpenHabItemDto.class);
        assertThat(item.name()).isEqualTo("Light_1");
    }

    @Test
    void sseEventDtoDeserializes() throws Exception {
        String json = """
            {
              "topic": "openhab/items/LivingRoom_Temperature/statechanged",
              "payload": "{\\"type\\":\\"Decimal\\",\\"value\\":\\"22.0\\",\\"oldType\\":\\"Decimal\\",\\"oldValue\\":\\"21.5\\"}",
              "type": "ItemStateChangedEvent"
            }
            """;

        OpenHabSseEventDto event = mapper.readValue(json, OpenHabSseEventDto.class);

        assertThat(event.topic()).isEqualTo("openhab/items/LivingRoom_Temperature/statechanged");
        assertThat(event.type()).isEqualTo("ItemStateChangedEvent");

        OpenHabStatePayloadDto payload = mapper.readValue(event.payload(), OpenHabStatePayloadDto.class);
        assertThat(payload.type()).isEqualTo("Decimal");
        assertThat(payload.value()).isEqualTo("22.0");
        assertThat(payload.oldValue()).isEqualTo("21.5");
    }
}
```

- [ ] **Step 2: Run test — verify it fails (classes don't exist)**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabDtoTest`
Expected: COMPILATION ERROR

- [ ] **Step 3: Implement all four DTOs**

Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabStateDescriptionDto.java`
```java
package io.casehub.iot.openhab.internal;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record OpenHabStateDescriptionDto(String pattern) {}
```

Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabItemDto.java`
```java
package io.casehub.iot.openhab.internal;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
public record OpenHabItemDto(
    String type,
    String name,
    String label,
    String state,
    List<String> tags,
    List<OpenHabItemDto> members,
    @JsonProperty("stateDescription") OpenHabStateDescriptionDto stateDescription
) {}
```

Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabSseEventDto.java`
```java
package io.casehub.iot.openhab.internal;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record OpenHabSseEventDto(String topic, String payload, String type) {}
```

Create: `openhab/src/main/java/io/casehub/iot/openhab/internal/OpenHabStatePayloadDto.java`
```java
package io.casehub.iot.openhab.internal;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record OpenHabStatePayloadDto(String type, String value, String oldType, String oldValue) {}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabDtoTest`
Expected: 3 tests PASS

- [ ] **Step 5: Commit**

```
git add openhab/src/
git commit -m "feat(openhab): internal DTOs — OpenHabItemDto, SSE event/payload records #4"
```

---

### Task 3: Supplement types

**Files:**
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabUpDownType.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabHsbType.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabThermostat.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabRollershutter.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabLight.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabSupplementTest.java`

- [ ] **Step 1: Write supplement tests**

File: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabSupplementTest.java`

Follow `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantSupplementTest.java` pattern exactly. Cover:
- Each supplement builds with all fields populated
- Each supplement builds with absent optional fields → `Optional.empty()`
- Each supplement's `capabilities()` includes supplement fields AND inherited fields
- `deriveChangedCapabilities()` detects supplement-only field changes
- `toBuilder()` round-trip preserves all fields
- `OpenHabRollershutter.upDown()` returns `Optional.empty()` when not set

Key test — supplement diff detection (prevents the silent-drop bug from C3 §3.6):
```java
@Test
void supplementDeriveChangedCapabilitiesDetectsHeatingDemandChange() {
    OpenHabThermostat before = OpenHabThermostat.builder()
        .deviceId("t1").deviceClass(DeviceClass.THERMOSTAT).label("HVAC")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .currentTemperature(temp21).targetTemperature(temp22).mode(ThermostatMode.HEAT)
        .heatingDemand(new BigDecimal("50"))
        .build();
    OpenHabThermostat after = OpenHabThermostat.builder()
        .deviceId("t1").deviceClass(DeviceClass.THERMOSTAT).label("HVAC")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .currentTemperature(temp21).targetTemperature(temp22).mode(ThermostatMode.HEAT)
        .heatingDemand(new BigDecimal("75"))
        .build();

    Set<String> changed = StateChangeEvent.deriveChangedCapabilities(before, after);
    assertThat(changed).containsExactly("heatingDemand");
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabSupplementTest`
Expected: COMPILATION ERROR

- [ ] **Step 3: Implement supporting types**

`OpenHabUpDownType.java`:
```java
package io.casehub.iot.openhab;

public enum OpenHabUpDownType { UP, DOWN, STOP }
```

`OpenHabHsbType.java`:
```java
package io.casehub.iot.openhab;

import java.math.BigDecimal;
import java.util.Objects;

public record OpenHabHsbType(BigDecimal hue, BigDecimal saturation, BigDecimal brightness) {
    public OpenHabHsbType {
        Objects.requireNonNull(hue, "hue");
        Objects.requireNonNull(saturation, "saturation");
        Objects.requireNonNull(brightness, "brightness");
    }
}
```

- [ ] **Step 4: Implement OpenHabThermostat**

Follow `HomeAssistantThermostat.java` pattern — AbstractBuilder extends `ThermostatDevice.AbstractBuilder`, `capabilities()` override adds `CAP_HEATING_DEMAND` and `CAP_COOLING_DEMAND`, `toBuilder()` copies all fields.

- [ ] **Step 5: Implement OpenHabRollershutter**

Follow `HomeAssistantLock.java` pattern (extends a type with AbstractBuilder). AbstractBuilder extends `CoverDevice.AbstractBuilder`, `capabilities()` override adds `CAP_UP_DOWN`, `upDown` is `Optional<OpenHabUpDownType>`.

- [ ] **Step 6: Implement OpenHabLight**

Follow `HomeAssistantLight.java` pattern. AbstractBuilder extends `LightDevice.AbstractBuilder`, `capabilities()` override adds `CAP_HSB`, `hsb` is `Optional<OpenHabHsbType>`.

- [ ] **Step 7: Run tests — verify all pass**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabSupplementTest`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```
git add openhab/src/
git commit -m "feat(openhab): supplement types — OpenHabThermostat, OpenHabRollershutter, OpenHabLight #4"
```

---

### Task 4: Config + REST client interfaces

**Files:**
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabConfig.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabRestClient.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabSseRestClient.java`

No dedicated tests — these are interfaces/config validated by the @QuarkusTest integration tests in Tasks 6-7.

- [ ] **Step 1: Implement OpenHabConfig**

```java
package io.casehub.iot.openhab;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.iot.openhab")
public interface OpenHabConfig {
    String url();
    String token();
    String tenancyId();
    @WithDefault("5")   int reconnectBaseSeconds();
    @WithDefault("300") int reconnectMaxSeconds();
    @WithDefault("50")  int coalesceWindowMs();
}
```

- [ ] **Step 2: Implement OpenHabRestClient**

```java
package io.casehub.iot.openhab;

import io.casehub.iot.openhab.internal.OpenHabItemDto;
import io.smallrye.mutiny.Uni;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.config.ConfigProvider;
import org.eclipse.microprofile.rest.client.annotation.ClientHeaderParam;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import java.util.List;

@RegisterRestClient(configKey = "openhab")
@ClientHeaderParam(name = "Authorization", value = "{lookupToken}")
public interface OpenHabRestClient {

    @GET @Path("/rest/items")
    Uni<List<OpenHabItemDto>> getItems(
        @QueryParam("tags") String tags,
        @QueryParam("recursive") boolean recursive);

    @POST @Path("/rest/items/{itemName}")
    @Consumes(MediaType.TEXT_PLAIN)
    Uni<Response> sendCommand(
        @PathParam("itemName") String itemName,
        String command);

    default String lookupToken() {
        return "Bearer " + ConfigProvider.getConfig()
                .getValue("casehub.iot.openhab.token", String.class);
    }
}
```

- [ ] **Step 3: Implement OpenHabSseRestClient**

```java
package io.casehub.iot.openhab;

import io.smallrye.mutiny.Multi;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.config.ConfigProvider;
import org.eclipse.microprofile.rest.client.annotation.ClientHeaderParam;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import org.jboss.resteasy.reactive.client.SseEvent;

@RegisterRestClient(configKey = "openhab-sse")
@ClientHeaderParam(name = "Authorization", value = "{lookupToken}")
public interface OpenHabSseRestClient {

    @GET @Path("/rest/events")
    @Produces(MediaType.SERVER_SENT_EVENTS)
    Multi<SseEvent<String>> subscribeEvents(
        @QueryParam("topics") String topics);

    default String lookupToken() {
        return "Bearer " + ConfigProvider.getConfig()
                .getValue("casehub.iot.openhab.token", String.class);
    }
}
```

- [ ] **Step 4: Verify build compiles**

Run: `mvn --batch-mode compile -pl openhab`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
git add openhab/src/
git commit -m "feat(openhab): config + REST client interfaces (openhab + openhab-sse) #4"
```

---

### Task 5: Entity mapper

The core mapping logic — pure stateless function: `OpenHabItemDto → DeviceEntity`. This is the largest task.

**Files:**
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabEntityMapper.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabEntityMapperTest.java`

- [ ] **Step 1: Write mapper tests**

File: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabEntityMapperTest.java`

Unit test (not @QuarkusTest). Construct mapper directly with a mock config.

Cover all device class mappings from the spec:

```java
package io.casehub.iot.openhab;

import io.casehub.iot.api.*;
import io.casehub.iot.openhab.internal.OpenHabItemDto;
import io.casehub.iot.openhab.internal.OpenHabStateDescriptionDto;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class OpenHabEntityMapperTest {

    private static final Instant NOW = Instant.parse("2026-06-10T10:00:00Z");
    private OpenHabEntityMapper mapper;

    @BeforeEach
    void setUp() {
        mapper = new OpenHabEntityMapper("test-tenant");
    }
```

Test cases (one per method):
- `hvacEquipmentMapsToThermostat` — Equipment with HVAC tag, Temperature Measurement + Setpoint members → `ThermostatDevice`
- `hvacWithHeatingDemandMapsToOpenHabThermostat` — adds Measurement+Level "heating demand" → `OpenHabThermostat`
- `lightbulbMapsToLightDevice` — Equipment with Lightbulb tag, Control+Switch member
- `colorItemMapsToOpenHabLight` — adds Color item with HSB state → `OpenHabLight` with hsb parsed
- `rollershutterMapsToOpenHabRollershutter` — Equipment with Rollershutter tag → position inverted
- `rollershutterPositionIsInverted` — OH 30% → CaseHub 70% (100 - 30)
- `contactOpenStateMapsCorrectly` — Contact OPEN→100, CLOSED→0
- `switchEquipmentMapsToSwitch` — PowerOutlet/WallSwitch tag
- `lockEquipmentMapsToLock` — Lock tag, Control+Switch → isLocked
- `fanEquipmentMapsToFan` — Fan tag
- `mediaPlayerMapsToMediaPlayer` — Television/Speaker/Receiver tag
- `sensorEquipmentMapsToSensor` — Sensor tag with Measurement+Humidity member
- `motionDetectorMapsToSensorMotion` — MotionDetector tag → SensorType.MOTION
- `batteryMapsToSensorGeneric` — Battery tag → SensorType.GENERIC, unit="%"
- `smokeDetectorMapsToSensorGeneric` — SmokeDetector tag → SensorType.GENERIC (not CO)
- `airConditionerMapsToThermostat` — AirConditioner tag → THERMOSTAT
- `unknownTagIsSkipped` — unrecognised Equipment tag → returns null
- `nullUndefStateMarksUnavailable` — state="NULL" → available=false
- `temperatureUnitDefaultsCelsius` — no °F in pattern → Celsius
- `temperatureUnitDetectsFahrenheit` — pattern contains °F → Fahrenheit
- `hvacModeResolvedFromStringItemContainingMode` — String item with "mode" in name → ThermostatMode mapping
- `hvacModeDefaultsToOffWhenNoModeItem` — no String item with "mode" → ThermostatMode.OFF
- `deviceIdIsEquipmentName` — `device.deviceId()` equals `equipment.name()`

Each test constructs an `OpenHabItemDto` with appropriate structure and asserts the mapped `DeviceEntity`.

Example test:
```java
@Test
void rollershutterPositionIsInverted() {
    OpenHabItemDto equipment = new OpenHabItemDto(
        "Group", "Blinds_LivingRoom", "Living Room Blinds", "NULL",
        List.of("Equipment", "Rollershutter"),
        List.of(new OpenHabItemDto(
            "Rollershutter", "Blinds_LivingRoom_Position", "Position", "30",
            List.of("Status", "OpenState"), null, null)),
        null);

    DeviceEntity device = mapper.mapEquipment(equipment, NOW);

    assertThat(device).isInstanceOf(CoverDevice.class);
    CoverDevice cover = (CoverDevice) device;
    assertThat(cover.position()).hasValue(70); // 100 - 30 = 70
}
```

- [ ] **Step 2: Run tests — verify compilation failure**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabEntityMapperTest`
Expected: COMPILATION ERROR

- [ ] **Step 3: Implement OpenHabEntityMapper**

`OpenHabEntityMapper.java` — follow structure of `HomeAssistantEntityMapper`. Key differences:
- Constructor takes `tenancyId` directly (injected from config in the CDI bean; for tests, passed directly)
- `mapEquipment(OpenHabItemDto equipment, Instant now)` is the entry point
- `resolveDeviceClass(List<String> tags)` maps Equipment tags per spec table
- Per-device-class mapping methods iterate `equipment.members()` to populate builders
- Position inversion: `100 - parseInt(state)` for Rollershutter items
- HVAC mode: search for String item with "mode" in name/label
- Temperature unit: default Celsius, check `stateDescription.pattern` for °F
- Contact vs Rollershutter: check item `type` field

The mapper is `@ApplicationScoped` but also constructable directly for unit tests via a package-private constructor taking `String tenancyId`.

```java
@ApplicationScoped
public class OpenHabEntityMapper {
    private final String tenancyId;

    @Inject
    public OpenHabEntityMapper(OpenHabConfig config) {
        this.tenancyId = config.tenancyId();
    }

    OpenHabEntityMapper(String tenancyId) {
        this.tenancyId = tenancyId;
    }

    public DeviceEntity mapEquipment(OpenHabItemDto equipment, Instant now) { ... }
    // ... per-device-class methods
}
```

- [ ] **Step 4: Run tests — verify all pass**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabEntityMapperTest`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
git add openhab/src/
git commit -m "feat(openhab): entity mapper — Equipment semantic model to DeviceEntity hierarchy #4"
```

---

### Task 6: Provider + command dispatch

**Files:**
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabProvider.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabProviderTest.java`

- [ ] **Step 1: Write provider tests**

File: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabProviderTest.java`

Follow `HomeAssistantProviderTest.java` pattern — `@QuarkusTest` with `@QuarkusTestResource(OpenHabMockServerResource.class)`.

Tests:
- `providerIdIsOpenhab` — returns `"openhab"`
- `discoverMapsEquipmentToDeviceEntities` — enqueue Equipment JSON, verify mapped devices
- `dispatchTurnOnSendsCorrectCommand` — verify POST to correct item with "ON" body
- `dispatchTurnOffSendsCorrectCommand` — POST "OFF"
- `dispatchSetTemperatureSendsCorrectCommand` — POST numeric value to Setpoint item
- `dispatchLockSendsCorrectCommand` — POST "ON" to Lock's Control+Switch item
- `dispatchUnlockSendsCorrectCommand` — POST "OFF"
- `dispatchSetPositionInvertsValue` — position 75 → POST "25" (100-75)
- `dispatchSetVolumeSendsCorrectCommand` — POST volume value
- `dispatchReturnsFailedOnHttp500` — 500 response → CommandResult.FAILED
- `dispatchReturnsFailedOnUnknownAction` — unknown action → FAILED
- `statusDelegatesToSseClient` — provider.status() returns SSE client's current status

- [ ] **Step 2: Run tests — verify compilation failure**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabProviderTest`
Expected: COMPILATION ERROR

- [ ] **Step 3: Implement OpenHabProvider**

```java
@ApplicationScoped
public class OpenHabProvider implements DeviceProvider {

    private static final Logger LOG = Logger.getLogger(OpenHabProvider.class);

    @Inject @RestClient OpenHabRestClient restClient;
    @Inject OpenHabSseClient sseClient;
    @Inject OpenHabEntityMapper mapper;

    @PostConstruct
    void start() {
        sseClient.connect().subscribe().with(
            v -> {},
            e -> LOG.warnf(e, "OpenHAB initial connect failed")
        );
    }

    @Override public String providerId() { return "openhab"; }

    @Override public ProviderStatus status() { return sseClient.currentStatus(); }

    @Override
    public Uni<List<DeviceEntity>> discover() {
        return restClient.getItems("Equipment", true)
            .map(items -> items.stream()
                .map(item -> mapper.mapEquipment(item, Instant.now()))
                .filter(Objects::nonNull)
                .toList());
    }

    @Override
    public Uni<CommandResult> dispatch(DeviceCommand command) {
        String targetItem = sseClient.resolveTargetItem(command);
        if (targetItem == null) {
            return Uni.createFrom().item(CommandResult.FAILED);
        }
        String commandValue = buildCommandValue(command);
        if (commandValue == null) {
            return Uni.createFrom().item(CommandResult.FAILED);
        }
        return restClient.sendCommand(targetItem, commandValue)
            .map(resp -> resp.getStatus() < 300 ? CommandResult.SENT : CommandResult.FAILED)
            .onFailure(WebApplicationException.class).recoverWithItem(CommandResult.FAILED)
            .onFailure(ProcessingException.class).recoverWithItem(CommandResult.FAILED)
            .onFailure(TimeoutException.class).recoverWithItem(CommandResult.TIMEOUT);
    }

    private String buildCommandValue(DeviceCommand command) {
        return switch (command.action()) {
            case DeviceCommand.ACTION_TURN_ON -> "ON";
            case DeviceCommand.ACTION_TURN_OFF -> "OFF";
            case DeviceCommand.ACTION_LOCK -> "ON";
            case DeviceCommand.ACTION_UNLOCK -> "OFF";
            case DeviceCommand.ACTION_SET_TEMPERATURE ->
                command.parameters().get("temperature").toString();
            case DeviceCommand.ACTION_SET_POSITION -> {
                int pos = ((Number) command.parameters().get("position")).intValue();
                yield String.valueOf(100 - pos); // invert for OpenHAB
            }
            case DeviceCommand.ACTION_SET_VOLUME ->
                command.parameters().get("volume").toString();
            default -> null;
        };
    }
}
```

The `resolveTargetItem(DeviceCommand)` method lives on `OpenHabSseClient` because it needs the Equipment member index. It finds the member item whose tags match the action's required Point tags (per the spec's action → item resolution table). Implement a stub that returns null initially — Task 7 completes it.

- [ ] **Step 4: Create OpenHabSseClient stub (enough for provider tests to compile)**

A minimal stub of `OpenHabSseClient` with `currentStatus()`, `connect()`, and `resolveTargetItem()` — just enough for `OpenHabProvider` to compile and basic tests to pass. The full SSE implementation is Task 7.

```java
@ApplicationScoped
public class OpenHabSseClient {
    private volatile ProviderStatus currentStatus = ProviderStatus.DISCONNECTED;

    public ProviderStatus currentStatus() { return currentStatus; }

    public Uni<Void> connect() {
        return Uni.createFrom().voidItem();
    }

    public String resolveTargetItem(DeviceCommand command) {
        return null; // full implementation in Task 7
    }
}
```

- [ ] **Step 5: Run tests — verify passing**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabProviderTest`
Expected: All tests PASS (dispatch tests using mock responses; resolveTargetItem tests deferred to Task 7)

- [ ] **Step 6: Commit**

```
git add openhab/src/
git commit -m "feat(openhab): provider — discovery, command dispatch with error recovery #4"
```

---

### Task 7: SSE client — state cache, coalescing, lifecycle

The most complex task. Implements the full SSE subscription lifecycle with state cache, 50ms coalescing, reconnection, and item-to-Equipment resolution.

**Files:**
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabSseClient.java` (replace stub)
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabSseClientTest.java`

- [ ] **Step 1: Write SSE client tests**

File: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabSseClientTest.java`

Unit tests for:
- `populatesCachesOnDiscover` — after discover(), itemToEquipment and deviceCache are populated
- `resolveTargetItemFindsControlSwitchForTurnOn` — finds member with `Control` + `Switch` tags
- `resolveTargetItemFindsSetpointForSetTemperature` — finds member with `Setpoint` + `Temperature`
- `resolveTargetItemReturnsNullForUnknownDevice` — deviceId not in cache → null
- `resolveTargetItemDisambiguatesMultipleControlItems` — HVAC with multiple Controls → correct one
- `handleSseEventUpdatesDeviceCache` — simulates SSE event processing, verifies cache updated
- `coalescingProducesSingleEventForRapidChanges` — two item changes within 50ms → one StateChangeEvent
- `coalescingCapturesBeforeOnFirstEvent` — before snapshot is from before any change in window
- `itemNotInEquipmentIsIgnored` — SSE event for unknown item → no cache update
- `setPositionInvertsCommandValue` — position 75 → "25" via resolveTargetItem + provider

Integration test with `@QuarkusTest` for coalescing (needs CDI Event for StateChangeEvent):
- Uses Awaitility to verify coalescing behavior with real timers

- [ ] **Step 2: Run tests — verify compilation failure**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabSseClientTest`
Expected: COMPILATION ERROR

- [ ] **Step 3: Implement full OpenHabSseClient**

Replace the stub from Task 6. Full implementation per spec §SSE Subscription:

```java
@ApplicationScoped
public class OpenHabSseClient {

    private static final Logger LOG = Logger.getLogger(OpenHabSseClient.class);

    @Inject @RestClient OpenHabRestClient restClient;
    @Inject @RestClient OpenHabSseRestClient sseRestClient;
    @Inject OpenHabEntityMapper mapper;
    @Inject OpenHabConfig config;
    @Inject Event<StateChangeEvent> stateEvents;
    @Inject Event<ProviderStatusEvent> statusEvents;
    @Inject ObjectMapper objectMapper;

    private volatile ProviderStatus currentStatus = ProviderStatus.DISCONNECTED;
    private volatile boolean shuttingDown = false;
    private volatile Cancellable sseSubscription;
    private final AtomicInteger reconnectAttempt = new AtomicInteger(0);

    // State caches — see spec §State cache structure
    private final ConcurrentHashMap<String, OpenHabItemDto> equipmentCache = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ConcurrentHashMap<String, OpenHabItemDto>> itemStateCache = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, DeviceEntity> deviceCache = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> itemToEquipment = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, DeviceEntity> coalescingBefore = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ScheduledFuture<?>> coalescingTimers = new ConcurrentHashMap<>();

    private final ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread t = new Thread(r, "oh-coalesce");
        t.setDaemon(true);
        return t;
    });

    // ... connect(), scheduleReconnect(), handleSseEvent(), fireCoalesced(),
    //     resolveTargetItem(), populateCaches(), currentStatus(), stop()
}
```

Key methods:
- `connect()` — discover via REST, populate caches, subscribe to SSE Multi
- `handleSseEvent(SseEvent<String>)` — parse, resolve Equipment, update cache, coalesce
- `fireCoalesced(String equipment)` — timer expiry handler, diffs before/after, fires StateChangeEvent
- `resolveTargetItem(DeviceCommand)` — finds member item by tag combination for command dispatch
- `populateCaches(List<OpenHabItemDto>)` — builds equipmentCache, itemStateCache, itemToEquipment, deviceCache from discovery results
- `scheduleReconnect()` — exponential backoff, same pattern as HA `scheduleReconnect()`

- [ ] **Step 4: Run tests — verify all pass**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabSseClientTest`
Expected: All tests PASS

- [ ] **Step 5: Run full module tests**

Run: `mvn --batch-mode test -pl openhab`
Expected: All tests PASS across all test classes

- [ ] **Step 6: Commit**

```
git add openhab/src/
git commit -m "feat(openhab): SSE client — state cache, 50ms coalescing, reconnection lifecycle #4"
```

---

### Task 8: Build verification + final integration

- [ ] **Step 1: Run full build from root**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS for all modules including openhab

- [ ] **Step 2: Verify Jandex index is generated**

Check: `openhab/target/classes/META-INF/jandex.idx` exists

- [ ] **Step 3: Review test coverage**

Verify all test classes exist and pass:
- `OpenHabDtoTest` — DTO serialization
- `OpenHabSupplementTest` — supplement types + capabilities
- `OpenHabEntityMapperTest` — all device class mappings
- `OpenHabProviderTest` — discovery + dispatch
- `OpenHabSseClientTest` — state cache + coalescing

- [ ] **Step 4: Commit if any final fixes needed**

```
git add -A
git commit -m "feat(openhab): C4 OpenHAB provider complete #4"
```

---

## File Map Summary

| File | Task | Purpose |
|------|------|---------|
| `openhab/pom.xml` | 1 | Dependencies: iot-api, rest-client, rest-client-jackson, junit, assertj, mockwebserver, awaitility |
| `api/.../CoverDevice.java` | 1 | Add position() Javadoc: 0=closed, 100=open |
| `openhab/src/main/resources/application.properties` | 1 | REST client URL bridge config for both configKeys |
| `openhab/src/test/resources/application.properties` | 1 | Test config: localhost URL, test-token, test-tenant |
| `openhab/.../OpenHabMockServerResource.java` | 1 | MockWebServer lifecycle, injects URLs for both REST clients |
| `openhab/.../internal/OpenHabItemDto.java` | 2 | Item JSON record — Groups + members |
| `openhab/.../internal/OpenHabStateDescriptionDto.java` | 2 | State description — unit pattern |
| `openhab/.../internal/OpenHabSseEventDto.java` | 2 | SSE event wrapper — topic + payload |
| `openhab/.../internal/OpenHabStatePayloadDto.java` | 2 | State change payload — type + value |
| `openhab/.../OpenHabUpDownType.java` | 3 | Enum: UP, DOWN, STOP |
| `openhab/.../OpenHabHsbType.java` | 3 | Record: hue, saturation, brightness |
| `openhab/.../OpenHabThermostat.java` | 3 | Supplement: heatingDemand, coolingDemand |
| `openhab/.../OpenHabRollershutter.java` | 3 | Supplement: upDown (Optional) |
| `openhab/.../OpenHabLight.java` | 3 | Supplement: hsb (Optional) |
| `openhab/.../OpenHabConfig.java` | 4 | @ConfigMapping: url, token, tenancyId, reconnect, coalesce |
| `openhab/.../OpenHabRestClient.java` | 4 | REST interface: getItems(), sendCommand() |
| `openhab/.../OpenHabSseRestClient.java` | 4 | SSE interface: subscribeEvents() → Multi<SseEvent> |
| `openhab/.../OpenHabEntityMapper.java` | 5 | Pure mapper: Equipment DTO → DeviceEntity |
| `openhab/.../OpenHabProvider.java` | 6 | DeviceProvider SPI: discover, dispatch, status |
| `openhab/.../OpenHabSseClient.java` | 7 | SSE lifecycle, state cache, coalescing, indexes |
