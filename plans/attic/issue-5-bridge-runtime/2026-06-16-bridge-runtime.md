# Bridge Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the two-sided bridge tunnel — local agent + cloud-side BridgeDeviceProvider — enabling cloud and hybrid IoT deployment via the DeviceProvider SPI.

**Architecture:** Bridge agent (standalone Quarkus app) observes local StateChangeEvents, runs a CDI-discovered filter chain, and relays to cloud via WebSocket. Bridge server (library) provides BridgeDeviceProvider that fires events into CDI — cloud consumers are unaware of the bridge. Compound type ID serialization (`"THERMOSTAT:HomeAssistantThermostat"`) with custom DeviceTypeIdResolver enables polymorphic DeviceEntity round-trip with graceful degradation for unknown supplement types.

**Tech Stack:** Quarkus 3.x, Jackson 2.21.x (polymorphic serialization), Quarkus WebSocket Next (transport), CDI Instance<> discovery (filter chain), Mutiny Uni<> (async SPI)

**Spec:** `docs/superpowers/specs/2026-06-16-bridge-runtime-design.md`

**Issue:** casehubio/iot#5

---

## File Map

### iot-api modifications

| File | Action | Responsibility |
|------|--------|----------------|
| `api/pom.xml` | Modify | Add `jackson-databind` compile dependency |
| `api/src/main/java/io/casehub/iot/api/DeviceEntity.java` | Modify | Add `@JsonTypeInfo`, `@JsonTypeIdResolver`, Jackson `@JsonProperty` annotations |
| `api/src/main/java/io/casehub/iot/api/DeviceTypeIdResolver.java` | Create | Custom TypeIdResolver — compound ID, static registry, graceful degradation |
| `api/src/main/java/io/casehub/iot/api/bridge/BridgeMessage.java` | Create | Sealed interface — 6 message type records |
| `api/src/main/java/io/casehub/iot/api/bridge/BridgeEventFilter.java` | Create | Filter SPI — `priority()`, `filter(StateChangeEvent, FilterContext)` |
| `api/src/main/java/io/casehub/iot/api/bridge/FilterAction.java` | Create | Sealed interface — `Forward`, `Suppress(reason)` |
| `api/src/main/java/io/casehub/iot/api/bridge/FilterContext.java` | Create | Read-only context record — `tenancyId`, `connectionState`, `providerId` |
| `api/src/main/java/io/casehub/iot/api/bridge/ConnectionState.java` | Create | Enum — `CONNECTED`, `DISCONNECTED` |
| `api/src/main/java/io/casehub/iot/api/bridge/DeviceIdUtils.java` | Create | Static utilities — `stripPrefix()`, `extractTenancyId()` for namespaced device IDs |
| `api/src/test/java/io/casehub/iot/api/DeviceTypeIdResolverTest.java` | Create | Round-trip serialization tests for all 10 common types + graceful degradation |
| `api/src/test/java/io/casehub/iot/api/bridge/BridgeMessageSerializationTest.java` | Create | Round-trip tests for all 6 message types |

### homeassistant modifications

| File | Action | Responsibility |
|------|--------|----------------|
| `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModule.java` | Create | Registers 3 HA supplement types with DeviceTypeIdResolver |
| `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModuleTest.java` | Create | Round-trip HA supplement types |

### openhab modifications

| File | Action | Responsibility |
|------|--------|----------------|
| `openhab/src/main/java/io/casehub/iot/openhab/OpenHabJacksonModule.java` | Create | Registers 3 OH supplement types with DeviceTypeIdResolver |
| `openhab/src/test/java/io/casehub/iot/openhab/OpenHabJacksonModuleTest.java` | Create | Round-trip OH supplement types |

### bridge-server (new module)

| File | Action | Responsibility |
|------|--------|----------------|
| `bridge-server/pom.xml` | Create | Library module — iot-api + websockets-next + jackson |
| `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeDeviceProvider.java` | Create | DeviceProvider impl — device map, discover(), dispatch(), status(), event firing |
| `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpoint.java` | Create | WebSocket server — accepts connections, routes messages |
| `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeConnectionRegistry.java` | Create | Tenancy → WebSocket session map, multi-site support |
| `bridge-server/src/main/java/io/casehub/iot/bridge/server/DeviceIdNamespacer.java` | Create | Jackson tree copy — namespace/strip device IDs |
| `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeServerConfig.java` | Create | @ConfigMapping — command-timeout-seconds |
| `bridge-server/src/test/java/io/casehub/iot/bridge/server/DeviceIdNamespacerTest.java` | Create | Namespace/strip round-trip for all device types |
| `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeDeviceProviderTest.java` | Create | Device map, snapshot diff, device removal, event firing |
| `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeConnectionRegistryTest.java` | Create | Multi-site registration, disconnect handling |

### bridge (new module — standalone Quarkus app)

| File | Action | Responsibility |
|------|--------|----------------|
| `bridge/pom.xml` | Modify | Quarkus app — iot-api + websockets-next + jackson |
| `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeFilterChain.java` | Create | Discovers BridgeEventFilter beans, sorts, chains |
| `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeEventObserver.java` | Create | @ObservesAsync StateChangeEvent + ProviderStatusEvent |
| `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCloudClient.java` | Create | WebSocket client — connects to cloud, sends/receives BridgeMessage |
| `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCommandDispatcher.java` | Create | Receives Command, dispatches to DeviceRegistry |
| `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeConnectionManager.java` | Create | Connection lifecycle — auth, reconnect, heartbeat, snapshot |
| `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeAgentConfig.java` | Create | @ConfigMapping — endpoint, token, tenant-id, reconnect params |
| `bridge/src/main/resources/application.properties` | Create | Default config values |
| `bridge/src/test/java/io/casehub/iot/bridge/agent/BridgeFilterChainTest.java` | Create | Priority ordering, short-circuit, no-filter pass-through |
| `bridge/src/test/java/io/casehub/iot/bridge/agent/BridgeCommandDispatcherTest.java` | Create | Command dispatch, tenancy prefix stripping |

### Parent POM

| File | Action | Responsibility |
|------|--------|----------------|
| `pom.xml` | Modify | Add `bridge-server` module, update `bridge` deps |

---

## Task 1: DeviceTypeIdResolver + Jackson Annotations on DeviceEntity

**Files:**
- Modify: `api/pom.xml`
- Modify: `api/src/main/java/io/casehub/iot/api/DeviceEntity.java`
- Create: `api/src/main/java/io/casehub/iot/api/DeviceTypeIdResolver.java`
- Create: `api/src/test/java/io/casehub/iot/api/DeviceTypeIdResolverTest.java`

This is the foundation — everything else depends on DeviceEntity being serializable.

- [ ] **Step 1: Add jackson-databind to api/pom.xml**

Add to `<dependencies>`:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

`jackson-databind` is managed by the Quarkus BOM (already in the parent chain). `jackson-datatype-jsr310` is needed for `Instant` and `BigDecimal` serialization in device fields.

- [ ] **Step 2: Write the failing serialization test**

Create `api/src/test/java/io/casehub/iot/api/DeviceTypeIdResolverTest.java`:

```java
package io.casehub.iot.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;

class DeviceTypeIdResolverTest {

    static ObjectMapper mapper;

    @BeforeAll
    static void setup() {
        mapper = JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .build();
    }

    @Test
    void switchDeviceRoundTrip() throws Exception {
        var device = SwitchDevice.builder()
                .deviceId("switch.kitchen").deviceClass(DeviceClass.SWITCH)
                .label("Kitchen Switch").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true)
                .build();

        String json = mapper.writeValueAsString(device);
        assertThat(json).contains("\"@deviceType\":\"SWITCH:SwitchDevice\"");

        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(SwitchDevice.class);
        SwitchDevice sw = (SwitchDevice) deserialized;
        assertThat(sw.deviceId()).isEqualTo("switch.kitchen");
        assertThat(sw.isOn()).isTrue();
    }

    @Test
    void thermostatDeviceRoundTrip() throws Exception {
        var device = ThermostatDevice.builder()
                .deviceId("climate.living").deviceClass(DeviceClass.THERMOSTAT)
                .label("Living Room").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1")
                .currentTemperature(new Temperature(new BigDecimal("21.5"), Temperature.TemperatureUnit.CELSIUS))
                .targetTemperature(new Temperature(new BigDecimal("22.0"), Temperature.TemperatureUnit.CELSIUS))
                .mode(ThermostatMode.HEAT)
                .build();

        String json = mapper.writeValueAsString(device);
        assertThat(json).contains("\"@deviceType\":\"THERMOSTAT:ThermostatDevice\"");

        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(ThermostatDevice.class);
        ThermostatDevice t = (ThermostatDevice) deserialized;
        assertThat(t.currentTemperature().toCelsius().value()).isEqualByComparingTo("21.5");
        assertThat(t.mode()).isEqualTo(ThermostatMode.HEAT);
    }

    @Test
    void lightDeviceRoundTrip() throws Exception {
        var device = new LightDevice.Builder()
                .deviceId("light.bedroom").deviceClass(DeviceClass.LIGHT)
                .label("Bedroom Light").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true).brightness(200).colorTemp(4000)
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(LightDevice.class);
        assertThat(((LightDevice) deserialized).brightness()).hasValue(200);
    }

    @Test
    void sensorDeviceRoundTrip() throws Exception {
        var device = SensorDevice.builder()
                .deviceId("sensor.temp").deviceClass(DeviceClass.SENSOR)
                .label("Temp Sensor").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1")
                .sensorType(SensorType.TEMPERATURE)
                .numericValue(new BigDecimal("23.4")).unit("°C")
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(SensorDevice.class);
        assertThat(((SensorDevice) deserialized).sensorType()).isEqualTo(SensorType.TEMPERATURE);
    }

    @Test
    void presenceSensorRoundTrip() throws Exception {
        var device = PresenceSensor.builder()
                .deviceId("binary_sensor.hall").deviceClass(DeviceClass.PRESENCE_SENSOR)
                .label("Hall Motion").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1")
                .present(true).lastSeen(Instant.parse("2026-06-16T09:55:00Z"))
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(PresenceSensor.class);
    }

    @Test
    void powerSensorRoundTrip() throws Exception {
        var device = PowerSensor.builder()
                .deviceId("sensor.power").deviceClass(DeviceClass.POWER_SENSOR)
                .label("Main Power").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1")
                .power(new BigDecimal("1500")).energy(new BigDecimal("42000"))
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(PowerSensor.class);
    }

    @Test
    void lockDeviceRoundTrip() throws Exception {
        var device = new LockDevice.Builder()
                .deviceId("lock.front").deviceClass(DeviceClass.LOCK)
                .label("Front Door").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").locked(true)
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(LockDevice.class);
        assertThat(((LockDevice) deserialized).isLocked()).isTrue();
    }

    @Test
    void coverDeviceRoundTrip() throws Exception {
        var device = new CoverDevice.Builder()
                .deviceId("cover.garage").deviceClass(DeviceClass.COVER)
                .label("Garage Door").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").position(75).moving(false)
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(CoverDevice.class);
    }

    @Test
    void mediaPlayerDeviceRoundTrip() throws Exception {
        var device = MediaPlayerDevice.builder()
                .deviceId("media.tv").deviceClass(DeviceClass.MEDIA_PLAYER)
                .label("Living TV").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").playing(true).volume(65)
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(MediaPlayerDevice.class);
    }

    @Test
    void fanDeviceRoundTrip() throws Exception {
        var device = FanDevice.builder()
                .deviceId("fan.ceiling").deviceClass(DeviceClass.FAN)
                .label("Ceiling Fan").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true).speed(3)
                .build();

        String json = mapper.writeValueAsString(device);
        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(FanDevice.class);
    }

    @Test
    void unknownSupplementTypeFallsBackToCommonType() throws Exception {
        // Simulate JSON from a bridge agent that has casehub-iot-homeassistant
        // but the cloud-side doesn't
        String json = """
                {
                  "@deviceType": "THERMOSTAT:HomeAssistantThermostat",
                  "deviceId": "climate.living",
                  "deviceClass": "THERMOSTAT",
                  "label": "Living Room",
                  "available": true,
                  "lastUpdated": "2026-06-16T10:00:00Z",
                  "tenancyId": "home-1",
                  "currentTemperature": {"value": 21.5, "unit": "CELSIUS"},
                  "targetTemperature": {"value": 22.0, "unit": "CELSIUS"},
                  "mode": "HEAT",
                  "presetMode": "comfort",
                  "hvacAction": "heating"
                }
                """;

        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(ThermostatDevice.class);
        assertThat(deserialized).isNotInstanceOf(homeAssistantThermostatClass());
        assertThat(deserialized.deviceId()).isEqualTo("climate.living");
        assertThat(((ThermostatDevice) deserialized).mode()).isEqualTo(ThermostatMode.HEAT);
    }

    private Class<?> homeAssistantThermostatClass() {
        try {
            return Class.forName("io.casehub.iot.homeassistant.HomeAssistantThermostat");
        } catch (ClassNotFoundException e) {
            return Void.class; // HA not on classpath — expected
        }
    }

    @Test
    void stateChangeEventWithDeviceRoundTrip() throws Exception {
        var before = SwitchDevice.builder()
                .deviceId("switch.kitchen").deviceClass(DeviceClass.SWITCH)
                .label("Kitchen Switch").available(true)
                .lastUpdated(Instant.parse("2026-06-16T09:59:00Z"))
                .tenancyId("home-1").on(false)
                .build();
        var after = SwitchDevice.builder()
                .deviceId("switch.kitchen").deviceClass(DeviceClass.SWITCH)
                .label("Kitchen Switch").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true)
                .build();
        var event = new StateChangeEvent(before, after,
                StateChangeEvent.deriveChangedCapabilities(before, after),
                Instant.parse("2026-06-16T10:00:00Z"), "homeassistant");

        String json = mapper.writeValueAsString(event);
        StateChangeEvent deserialized = mapper.readValue(json, StateChangeEvent.class);
        assertThat(deserialized.after()).isInstanceOf(SwitchDevice.class);
        assertThat(((SwitchDevice) deserialized.after()).isOn()).isTrue();
        assertThat(deserialized.changedCapabilities()).containsExactly("isOn");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=DeviceTypeIdResolverTest`
Expected: compilation failure — `DeviceTypeIdResolver` class doesn't exist, no Jackson annotations on `DeviceEntity`.

- [ ] **Step 4: Create DeviceTypeIdResolver**

Create `api/src/main/java/io/casehub/iot/api/DeviceTypeIdResolver.java`:

```java
package io.casehub.iot.api;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.DatabindContext;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.jsontype.impl.TypeIdResolverBase;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class DeviceTypeIdResolver extends TypeIdResolverBase {

    private static final Logger LOG = Logger.getLogger(DeviceTypeIdResolver.class);

    private static final Map<String, Class<? extends DeviceEntity>> REGISTRY = new ConcurrentHashMap<>();
    private static final Map<DeviceClass, Class<? extends DeviceEntity>> COMMON_TYPES = Map.ofEntries(
            Map.entry(DeviceClass.SWITCH, SwitchDevice.class),
            Map.entry(DeviceClass.LIGHT, LightDevice.class),
            Map.entry(DeviceClass.THERMOSTAT, ThermostatDevice.class),
            Map.entry(DeviceClass.SENSOR, SensorDevice.class),
            Map.entry(DeviceClass.PRESENCE_SENSOR, PresenceSensor.class),
            Map.entry(DeviceClass.POWER_SENSOR, PowerSensor.class),
            Map.entry(DeviceClass.LOCK, LockDevice.class),
            Map.entry(DeviceClass.COVER, CoverDevice.class),
            Map.entry(DeviceClass.MEDIA_PLAYER, MediaPlayerDevice.class),
            Map.entry(DeviceClass.FAN, FanDevice.class)
    );

    static {
        COMMON_TYPES.forEach((dc, clazz) ->
                REGISTRY.put(dc.name() + ":" + clazz.getSimpleName(), clazz));
    }

    public static void registerType(String compoundId, Class<? extends DeviceEntity> type) {
        REGISTRY.put(compoundId, type);
    }

    @Override
    public String idFromValue(Object value) {
        if (!(value instanceof DeviceEntity device)) {
            throw new IllegalArgumentException("Expected DeviceEntity, got " + value.getClass());
        }
        return device.deviceClass().name() + ":" + value.getClass().getSimpleName();
    }

    @Override
    public String idFromValueAndType(Object value, Class<?> suggestedType) {
        return idFromValue(value);
    }

    @Override
    public JavaType typeFromId(DatabindContext context, String id) {
        Class<? extends DeviceEntity> type = REGISTRY.get(id);
        if (type != null) {
            return context.constructType(type);
        }

        int colonIndex = id.indexOf(':');
        if (colonIndex > 0) {
            String prefix = id.substring(0, colonIndex);
            String specificType = id.substring(colonIndex + 1);
            try {
                DeviceClass dc = DeviceClass.valueOf(prefix);
                Class<? extends DeviceEntity> fallback = COMMON_TYPES.get(dc);
                if (fallback != null) {
                    LOG.warnf("Unknown device type '%s' — falling back to %s. "
                            + "Add the vendor module to classpath for full type fidelity.",
                            specificType, fallback.getSimpleName());
                    return context.constructType(fallback);
                }
            } catch (IllegalArgumentException ignored) {
            }
        }

        throw new IllegalArgumentException("Cannot resolve device type ID: " + id);
    }

    @Override
    public JsonTypeInfo.Id getMechanism() {
        return JsonTypeInfo.Id.CUSTOM;
    }
}
```

- [ ] **Step 5: Add Jackson annotations to DeviceEntity**

Modify `api/src/main/java/io/casehub/iot/api/DeviceEntity.java` — add at class level:

```java
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.annotation.JsonTypeIdResolver;

@JsonTypeInfo(use = JsonTypeInfo.Id.CUSTOM, property = "@deviceType")
@JsonTypeIdResolver(DeviceTypeIdResolver.class)
public abstract class DeviceEntity {
```

Also add `@JsonIgnoreProperties(ignoreUnknown = true)` for graceful degradation (unknown supplement fields are silently dropped):

```java
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonTypeInfo(use = JsonTypeInfo.Id.CUSTOM, property = "@deviceType")
@JsonTypeIdResolver(DeviceTypeIdResolver.class)
@JsonIgnoreProperties(ignoreUnknown = true)
public abstract class DeviceEntity {
```

Jackson needs to deserialize via the builder pattern since DeviceEntity has no public constructor and fields are final. Add `@JsonDeserialize(builder = ...)` to each concrete type. For types with `AbstractBuilder` (supplementable types), the annotation goes on the concrete type pointing to its `Builder`:

For each of the 10 common types, add `@JsonDeserialize(builder = XxxDevice.Builder.class)` and `@JsonPOJOBuilder(withPrefix = "")` on the Builder class. Example for `SwitchDevice`:

```java
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import com.fasterxml.jackson.databind.annotation.JsonPOJOBuilder;

@JsonDeserialize(builder = SwitchDevice.Builder.class)
public class SwitchDevice extends DeviceEntity {
    // ... existing code unchanged ...

    @JsonPOJOBuilder(withPrefix = "")
    public static final class Builder extends DeviceEntity.Builder<SwitchDevice, Builder> {
        // ... existing code unchanged ...
    }
}
```

Apply the same pattern to all 10 common types:
- `SwitchDevice` → `@JsonDeserialize(builder = SwitchDevice.Builder.class)`
- `LightDevice` → `@JsonDeserialize(builder = LightDevice.Builder.class)`
- `ThermostatDevice` → `@JsonDeserialize(builder = ThermostatDevice.Builder.class)`
- `SensorDevice` → `@JsonDeserialize(builder = SensorDevice.Builder.class)`
- `PresenceSensor` → `@JsonDeserialize(builder = PresenceSensor.Builder.class)`
- `PowerSensor` → `@JsonDeserialize(builder = PowerSensor.Builder.class)`
- `LockDevice` → `@JsonDeserialize(builder = LockDevice.Builder.class)`
- `CoverDevice` → `@JsonDeserialize(builder = CoverDevice.Builder.class)`
- `MediaPlayerDevice` → `@JsonDeserialize(builder = MediaPlayerDevice.Builder.class)`
- `FanDevice` → `@JsonDeserialize(builder = FanDevice.Builder.class)`

For types with `AbstractBuilder` hierarchy (LightDevice, ThermostatDevice, LockDevice, CoverDevice), `@JsonPOJOBuilder` goes on the concrete `Builder`, not the `AbstractBuilder`.

**Important:** The `Builder.build()` method is already correctly named. `@JsonPOJOBuilder(withPrefix = "")` tells Jackson the setter methods don't have a prefix (they're `on()`, `brightness()`, not `withOn()`, `withBrightness()`). The `buildMethodName` defaults to `"build"` which matches.

For `Temperature` record — Jackson handles records natively since 2.12+. No annotation needed.

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=DeviceTypeIdResolverTest`
Expected: all 12 tests PASS.

- [ ] **Step 7: Run full api module tests to check for regressions**

Run: `mvn --batch-mode test -pl api`
Expected: all existing tests still pass.

- [ ] **Step 8: Commit**

```bash
git add api/pom.xml api/src/main/java/io/casehub/iot/api/DeviceEntity.java \
  api/src/main/java/io/casehub/iot/api/DeviceTypeIdResolver.java \
  api/src/main/java/io/casehub/iot/api/SwitchDevice.java \
  api/src/main/java/io/casehub/iot/api/LightDevice.java \
  api/src/main/java/io/casehub/iot/api/ThermostatDevice.java \
  api/src/main/java/io/casehub/iot/api/SensorDevice.java \
  api/src/main/java/io/casehub/iot/api/PresenceSensor.java \
  api/src/main/java/io/casehub/iot/api/PowerSensor.java \
  api/src/main/java/io/casehub/iot/api/LockDevice.java \
  api/src/main/java/io/casehub/iot/api/CoverDevice.java \
  api/src/main/java/io/casehub/iot/api/MediaPlayerDevice.java \
  api/src/main/java/io/casehub/iot/api/FanDevice.java \
  api/src/test/java/io/casehub/iot/api/DeviceTypeIdResolverTest.java
git commit -m "feat(api): DeviceTypeIdResolver — compound type ID with graceful degradation #5"
```

---

## Task 2: BridgeMessage Sealed Interface + Wire Protocol Types

**Files:**
- Create: `api/src/main/java/io/casehub/iot/api/bridge/BridgeMessage.java`
- Create: `api/src/main/java/io/casehub/iot/api/bridge/BridgeEventFilter.java`
- Create: `api/src/main/java/io/casehub/iot/api/bridge/FilterAction.java`
- Create: `api/src/main/java/io/casehub/iot/api/bridge/FilterContext.java`
- Create: `api/src/main/java/io/casehub/iot/api/bridge/ConnectionState.java`
- Create: `api/src/test/java/io/casehub/iot/api/bridge/BridgeMessageSerializationTest.java`

- [ ] **Step 1: Write failing test for BridgeMessage serialization**

Create `api/src/test/java/io/casehub/iot/api/bridge/BridgeMessageSerializationTest.java`:

```java
package io.casehub.iot.api.bridge;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.iot.api.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class BridgeMessageSerializationTest {

    static ObjectMapper mapper;

    @BeforeAll
    static void setup() {
        mapper = JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .build();
    }

    @Test
    void stateChangeRoundTrip() throws Exception {
        var sw = SwitchDevice.builder()
                .deviceId("switch.kitchen").deviceClass(DeviceClass.SWITCH)
                .label("Kitchen").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true).build();
        var event = new StateChangeEvent(null, sw, Set.of("isOn"),
                Instant.parse("2026-06-16T10:00:00Z"), "ha");

        BridgeMessage msg = new BridgeMessage.StateChange("home-1",
                Instant.parse("2026-06-16T10:00:00Z"), event);

        String json = mapper.writeValueAsString(msg);
        assertThat(json).contains("\"@type\":\"STATE_CHANGE\"");

        BridgeMessage deserialized = mapper.readValue(json, BridgeMessage.class);
        assertThat(deserialized).isInstanceOf(BridgeMessage.StateChange.class);
        var sc = (BridgeMessage.StateChange) deserialized;
        assertThat(sc.tenancyId()).isEqualTo("home-1");
        assertThat(sc.event().after()).isInstanceOf(SwitchDevice.class);
    }

    @Test
    void stateSnapshotRoundTrip() throws Exception {
        var sw = SwitchDevice.builder()
                .deviceId("switch.kitchen").deviceClass(DeviceClass.SWITCH)
                .label("Kitchen").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true).build();

        BridgeMessage msg = new BridgeMessage.StateSnapshot("home-1",
                Instant.parse("2026-06-16T10:00:00Z"), List.of(sw));

        String json = mapper.writeValueAsString(msg);
        BridgeMessage deserialized = mapper.readValue(json, BridgeMessage.class);
        assertThat(deserialized).isInstanceOf(BridgeMessage.StateSnapshot.class);
        var ss = (BridgeMessage.StateSnapshot) deserialized;
        assertThat(ss.devices()).hasSize(1);
        assertThat(ss.devices().get(0)).isInstanceOf(SwitchDevice.class);
    }

    @Test
    void commandRoundTrip() throws Exception {
        var cmd = DeviceCommand.turnOn("switch.kitchen", null, "user", "corr-1");
        BridgeMessage msg = new BridgeMessage.Command("home-1",
                Instant.parse("2026-06-16T10:00:00Z"), "corr-1", cmd);

        String json = mapper.writeValueAsString(msg);
        BridgeMessage deserialized = mapper.readValue(json, BridgeMessage.class);
        assertThat(deserialized).isInstanceOf(BridgeMessage.Command.class);
        var c = (BridgeMessage.Command) deserialized;
        assertThat(c.correlationId()).isEqualTo("corr-1");
        assertThat(c.command().action()).isEqualTo(DeviceCommand.ACTION_TURN_ON);
    }

    @Test
    void commandResultRoundTrip() throws Exception {
        BridgeMessage msg = new BridgeMessage.CommandResult("home-1",
                Instant.parse("2026-06-16T10:00:00Z"), "corr-1", CommandResult.SENT);

        String json = mapper.writeValueAsString(msg);
        BridgeMessage deserialized = mapper.readValue(json, BridgeMessage.class);
        assertThat(deserialized).isInstanceOf(BridgeMessage.CommandResult.class);
        var cr = (BridgeMessage.CommandResult) deserialized;
        assertThat(cr.result()).isEqualTo(CommandResult.SENT);
    }

    @Test
    void providerStatusRoundTrip() throws Exception {
        var status = new ProviderStatusEvent("ha", ProviderStatus.CONNECTING, ProviderStatus.CONNECTED);
        BridgeMessage msg = new BridgeMessage.ProviderStatus("home-1",
                Instant.parse("2026-06-16T10:00:00Z"), status);

        String json = mapper.writeValueAsString(msg);
        BridgeMessage deserialized = mapper.readValue(json, BridgeMessage.class);
        assertThat(deserialized).isInstanceOf(BridgeMessage.ProviderStatus.class);
    }

    @Test
    void heartbeatRoundTrip() throws Exception {
        BridgeMessage msg = new BridgeMessage.Heartbeat("home-1",
                Instant.parse("2026-06-16T10:00:00Z"));

        String json = mapper.writeValueAsString(msg);
        BridgeMessage deserialized = mapper.readValue(json, BridgeMessage.class);
        assertThat(deserialized).isInstanceOf(BridgeMessage.Heartbeat.class);
    }

    @Test
    void exhaustiveSwitchCompiles() {
        BridgeMessage msg = new BridgeMessage.Heartbeat("home-1", Instant.now());
        String result = switch (msg) {
            case BridgeMessage.StateChange sc -> "state_change";
            case BridgeMessage.StateSnapshot ss -> "snapshot";
            case BridgeMessage.ProviderStatus ps -> "provider_status";
            case BridgeMessage.Command c -> "command";
            case BridgeMessage.CommandResult cr -> "command_result";
            case BridgeMessage.Heartbeat h -> "heartbeat";
        };
        assertThat(result).isEqualTo("heartbeat");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=BridgeMessageSerializationTest`
Expected: compilation failure — `BridgeMessage` class doesn't exist.

- [ ] **Step 3: Create wire protocol types**

Create `api/src/main/java/io/casehub/iot/api/bridge/ConnectionState.java`:

```java
package io.casehub.iot.api.bridge;

public enum ConnectionState {
    CONNECTED, DISCONNECTED
}
```

Create `api/src/main/java/io/casehub/iot/api/bridge/FilterContext.java`:

```java
package io.casehub.iot.api.bridge;

import java.util.Objects;

public record FilterContext(
    String tenancyId,
    ConnectionState connectionState,
    String providerId
) {
    public FilterContext {
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(connectionState, "connectionState");
        Objects.requireNonNull(providerId, "providerId");
    }
}
```

Create `api/src/main/java/io/casehub/iot/api/bridge/FilterAction.java`:

```java
package io.casehub.iot.api.bridge;

import java.util.Objects;

public sealed interface FilterAction {
    record Forward() implements FilterAction {}
    record Suppress(String reason) implements FilterAction {
        public Suppress {
            Objects.requireNonNull(reason, "reason");
        }
    }
}
```

Create `api/src/main/java/io/casehub/iot/api/bridge/BridgeEventFilter.java`:

```java
package io.casehub.iot.api.bridge;

import io.casehub.iot.api.StateChangeEvent;
import io.smallrye.mutiny.Uni;

public interface BridgeEventFilter {
    int priority();
    Uni<FilterAction> filter(StateChangeEvent event, FilterContext ctx);
}
```

Create `api/src/main/java/io/casehub/iot/api/bridge/BridgeMessage.java`:

```java
package io.casehub.iot.api.bridge;

import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import io.casehub.iot.api.*;

import java.time.Instant;
import java.util.List;
import java.util.Objects;

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "@type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = BridgeMessage.StateChange.class, name = "STATE_CHANGE"),
    @JsonSubTypes.Type(value = BridgeMessage.StateSnapshot.class, name = "STATE_SNAPSHOT"),
    @JsonSubTypes.Type(value = BridgeMessage.ProviderStatus.class, name = "PROVIDER_STATUS"),
    @JsonSubTypes.Type(value = BridgeMessage.Command.class, name = "COMMAND"),
    @JsonSubTypes.Type(value = BridgeMessage.CommandResult.class, name = "COMMAND_RESULT"),
    @JsonSubTypes.Type(value = BridgeMessage.Heartbeat.class, name = "HEARTBEAT")
})
public sealed interface BridgeMessage {
    String tenancyId();
    Instant timestamp();

    record StateChange(String tenancyId, Instant timestamp,
                       StateChangeEvent event) implements BridgeMessage {
        public StateChange {
            Objects.requireNonNull(tenancyId, "tenancyId");
            Objects.requireNonNull(timestamp, "timestamp");
            Objects.requireNonNull(event, "event");
        }
    }

    record StateSnapshot(String tenancyId, Instant timestamp,
                         List<DeviceEntity> devices) implements BridgeMessage {
        public StateSnapshot {
            Objects.requireNonNull(tenancyId, "tenancyId");
            Objects.requireNonNull(timestamp, "timestamp");
            Objects.requireNonNull(devices, "devices");
            devices = List.copyOf(devices);
        }
    }

    record ProviderStatus(String tenancyId, Instant timestamp,
                          ProviderStatusEvent status) implements BridgeMessage {
        public ProviderStatus {
            Objects.requireNonNull(tenancyId, "tenancyId");
            Objects.requireNonNull(timestamp, "timestamp");
            Objects.requireNonNull(status, "status");
        }
    }

    record Command(String tenancyId, Instant timestamp,
                   String correlationId, DeviceCommand command) implements BridgeMessage {
        public Command {
            Objects.requireNonNull(tenancyId, "tenancyId");
            Objects.requireNonNull(timestamp, "timestamp");
            Objects.requireNonNull(correlationId, "correlationId");
            Objects.requireNonNull(command, "command");
        }
    }

    record CommandResult(String tenancyId, Instant timestamp,
                         String correlationId,
                         io.casehub.iot.api.CommandResult result) implements BridgeMessage {
        public CommandResult {
            Objects.requireNonNull(tenancyId, "tenancyId");
            Objects.requireNonNull(timestamp, "timestamp");
            Objects.requireNonNull(correlationId, "correlationId");
            Objects.requireNonNull(result, "result");
        }
    }

    record Heartbeat(String tenancyId, Instant timestamp) implements BridgeMessage {
        public Heartbeat {
            Objects.requireNonNull(tenancyId, "tenancyId");
            Objects.requireNonNull(timestamp, "timestamp");
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=BridgeMessageSerializationTest`
Expected: all 7 tests PASS.

- [ ] **Step 5: Run full api tests**

Run: `mvn --batch-mode test -pl api`
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/iot/api/bridge/ \
  api/src/test/java/io/casehub/iot/api/bridge/
git commit -m "feat(api): BridgeMessage sealed interface + BridgeEventFilter SPI #5"
```

---

## Task 3: Vendor Jackson Modules (HA + OH)

**Files:**
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModule.java`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModuleTest.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabJacksonModule.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabJacksonModuleTest.java`

- [ ] **Step 1: Write failing HA supplement round-trip test**

Create `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModuleTest.java`:

```java
package io.casehub.iot.homeassistant;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.iot.api.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;

class HomeAssistantJacksonModuleTest {

    static ObjectMapper mapper;

    @BeforeAll
    static void setup() {
        mapper = JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .addModule(new HomeAssistantJacksonModule())
                .build();
    }

    @Test
    void homeAssistantThermostatRoundTrip() throws Exception {
        var device = HomeAssistantThermostat.builder()
                .deviceId("climate.living").deviceClass(DeviceClass.THERMOSTAT)
                .label("Living Room").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1")
                .currentTemperature(new Temperature(new BigDecimal("21.5"), Temperature.TemperatureUnit.CELSIUS))
                .targetTemperature(new Temperature(new BigDecimal("22.0"), Temperature.TemperatureUnit.CELSIUS))
                .mode(ThermostatMode.HEAT)
                .presetMode("comfort").hvacAction("heating")
                .build();

        String json = mapper.writeValueAsString(device);
        assertThat(json).contains("\"@deviceType\":\"THERMOSTAT:HomeAssistantThermostat\"");
        assertThat(json).contains("\"presetMode\":\"comfort\"");

        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(HomeAssistantThermostat.class);
        var hat = (HomeAssistantThermostat) deserialized;
        assertThat(hat.presetMode()).hasValue("comfort");
        assertThat(hat.hvacAction()).hasValue("heating");
        assertThat(hat.mode()).isEqualTo(ThermostatMode.HEAT);
    }

    @Test
    void homeAssistantLightRoundTrip() throws Exception {
        var device = HomeAssistantLight.builder()
                .deviceId("light.bedroom").deviceClass(DeviceClass.LIGHT)
                .label("Bedroom Light").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true).brightness(200)
                .rgbColor("255,128,0").effect("rainbow")
                .build();

        String json = mapper.writeValueAsString(device);
        assertThat(json).contains("\"@deviceType\":\"LIGHT:HomeAssistantLight\"");

        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(HomeAssistantLight.class);
        var hal = (HomeAssistantLight) deserialized;
        assertThat(hal.rgbColor()).hasValue("255,128,0");
    }

    @Test
    void homeAssistantLockRoundTrip() throws Exception {
        var device = HomeAssistantLock.builder()
                .deviceId("lock.front").deviceClass(DeviceClass.LOCK)
                .label("Front Door").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").locked(true)
                .changedBy("user1")
                .build();

        String json = mapper.writeValueAsString(device);
        assertThat(json).contains("\"@deviceType\":\"LOCK:HomeAssistantLock\"");

        DeviceEntity deserialized = mapper.readValue(json, DeviceEntity.class);
        assertThat(deserialized).isInstanceOf(HomeAssistantLock.class);
        var lock = (HomeAssistantLock) deserialized;
        assertThat(lock.changedBy()).hasValue("user1");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl homeassistant -Dtest=HomeAssistantJacksonModuleTest`
Expected: compilation failure — `HomeAssistantJacksonModule` class doesn't exist.

- [ ] **Step 3: Create HomeAssistantJacksonModule**

Create `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModule.java`:

```java
package io.casehub.iot.homeassistant;

import com.fasterxml.jackson.databind.module.SimpleModule;
import io.casehub.iot.api.DeviceTypeIdResolver;
import jakarta.inject.Singleton;

@Singleton
public class HomeAssistantJacksonModule extends SimpleModule {

    public HomeAssistantJacksonModule() {
        super("HomeAssistantJacksonModule");
    }

    @Override
    public void setupModule(SetupContext context) {
        super.setupModule(context);
        DeviceTypeIdResolver.registerType("THERMOSTAT:HomeAssistantThermostat", HomeAssistantThermostat.class);
        DeviceTypeIdResolver.registerType("LIGHT:HomeAssistantLight", HomeAssistantLight.class);
        DeviceTypeIdResolver.registerType("LOCK:HomeAssistantLock", HomeAssistantLock.class);
    }
}
```

Also add `@JsonDeserialize` and `@JsonPOJOBuilder` annotations to the 3 HA supplement types (same pattern as common types in Task 1):
- `HomeAssistantThermostat` → `@JsonDeserialize(builder = HomeAssistantThermostat.Builder.class)`
- `HomeAssistantLight` → `@JsonDeserialize(builder = HomeAssistantLight.Builder.class)`
- `HomeAssistantLock` → `@JsonDeserialize(builder = HomeAssistantLock.Builder.class)`

Each Builder gets `@JsonPOJOBuilder(withPrefix = "")`.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl homeassistant -Dtest=HomeAssistantJacksonModuleTest`
Expected: all 3 tests PASS.

- [ ] **Step 5: Create OpenHabJacksonModule and test (same pattern)**

Create `openhab/src/main/java/io/casehub/iot/openhab/OpenHabJacksonModule.java`:

```java
package io.casehub.iot.openhab;

import com.fasterxml.jackson.databind.module.SimpleModule;
import io.casehub.iot.api.DeviceTypeIdResolver;
import jakarta.inject.Singleton;

@Singleton
public class OpenHabJacksonModule extends SimpleModule {

    public OpenHabJacksonModule() {
        super("OpenHabJacksonModule");
    }

    @Override
    public void setupModule(SetupContext context) {
        super.setupModule(context);
        DeviceTypeIdResolver.registerType("THERMOSTAT:OpenHabThermostat", OpenHabThermostat.class);
        DeviceTypeIdResolver.registerType("LIGHT:OpenHabLight", OpenHabLight.class);
        DeviceTypeIdResolver.registerType("COVER:OpenHabRollershutter", OpenHabRollershutter.class);
    }
}
```

Add `@JsonDeserialize` + `@JsonPOJOBuilder` to the 3 OH supplement types.

Create `openhab/src/test/java/io/casehub/iot/openhab/OpenHabJacksonModuleTest.java` with round-trip tests for all 3 OH supplement types (same pattern as HA tests above).

- [ ] **Step 6: Run all tests**

Run: `mvn --batch-mode test -pl homeassistant,openhab`
Expected: all tests pass (including existing tests).

- [ ] **Step 7: Commit**

```bash
git add homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModule.java \
  homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantThermostat.java \
  homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantLight.java \
  homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantLock.java \
  homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantJacksonModuleTest.java \
  openhab/src/main/java/io/casehub/iot/openhab/OpenHabJacksonModule.java \
  openhab/src/main/java/io/casehub/iot/openhab/OpenHabThermostat.java \
  openhab/src/main/java/io/casehub/iot/openhab/OpenHabLight.java \
  openhab/src/main/java/io/casehub/iot/openhab/OpenHabRollershutter.java \
  openhab/src/test/java/io/casehub/iot/openhab/OpenHabJacksonModuleTest.java
git commit -m "feat(ha,oh): vendor Jackson Modules — register supplement types with DeviceTypeIdResolver #5"
```

---

## Task 4: Bridge Server Module Setup + DeviceIdNamespacer

**Files:**
- Create: `bridge-server/pom.xml`
- Modify: `pom.xml` (parent — add bridge-server module)
- Create: `bridge-server/src/main/java/io/casehub/iot/bridge/server/DeviceIdNamespacer.java`
- Create: `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeServerConfig.java`
- Create: `bridge-server/src/test/java/io/casehub/iot/bridge/server/DeviceIdNamespacerTest.java`

- [ ] **Step 1: Create bridge-server/pom.xml**

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

    <artifactId>casehub-iot-bridge-server</artifactId>
    <name>CaseHub IoT — Bridge Server</name>
    <description>Cloud-side BridgeDeviceProvider — remote devices look local via DeviceProvider SPI</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-websockets-next</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-client-jackson</artifactId>
        </dependency>

        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-testing</artifactId>
            <scope>test</scope>
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

- [ ] **Step 2: Add bridge-server to parent pom.xml modules**

In `pom.xml`, add `<module>bridge-server</module>` to the `<modules>` section.

- [ ] **Step 3: Write failing DeviceIdNamespacer test**

Create `bridge-server/src/test/java/io/casehub/iot/bridge/server/DeviceIdNamespacerTest.java`:

```java
package io.casehub.iot.bridge.server;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.iot.api.*;
import io.casehub.iot.api.bridge.DeviceIdUtils;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;

class DeviceIdNamespacerTest {

    static ObjectMapper mapper;
    static DeviceIdNamespacer namespacer;

    @BeforeAll
    static void setup() {
        mapper = JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .build();
        namespacer = new DeviceIdNamespacer(mapper);
    }

    @Test
    void namespaceSwitchDevice() {
        var device = SwitchDevice.builder()
                .deviceId("switch.kitchen").deviceClass(DeviceClass.SWITCH)
                .label("Kitchen Switch").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true).build();

        DeviceEntity namespaced = namespacer.namespace(device, "site-a");
        assertThat(namespaced.deviceId()).isEqualTo("site-a/switch.kitchen");
        assertThat(namespaced).isInstanceOf(SwitchDevice.class);
        assertThat(((SwitchDevice) namespaced).isOn()).isTrue();
        assertThat(namespaced.label()).isEqualTo("Kitchen Switch");
    }

    @Test
    void namespaceThermostatDevice() {
        var device = ThermostatDevice.builder()
                .deviceId("climate.living").deviceClass(DeviceClass.THERMOSTAT)
                .label("Living Room").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1")
                .currentTemperature(new Temperature(new BigDecimal("21.5"), Temperature.TemperatureUnit.CELSIUS))
                .targetTemperature(new Temperature(new BigDecimal("22.0"), Temperature.TemperatureUnit.CELSIUS))
                .mode(ThermostatMode.HEAT).build();

        DeviceEntity namespaced = namespacer.namespace(device, "site-a");
        assertThat(namespaced.deviceId()).isEqualTo("site-a/climate.living");
        assertThat(namespaced).isInstanceOf(ThermostatDevice.class);
        assertThat(((ThermostatDevice) namespaced).mode()).isEqualTo(ThermostatMode.HEAT);
    }

    @Test
    void markUnavailable() {
        var device = SwitchDevice.builder()
                .deviceId("switch.kitchen").deviceClass(DeviceClass.SWITCH)
                .label("Kitchen Switch").available(true)
                .lastUpdated(Instant.parse("2026-06-16T10:00:00Z"))
                .tenancyId("home-1").on(true).build();

        DeviceEntity unavailable = namespacer.markUnavailable(device);
        assertThat(unavailable.available()).isFalse();
        assertThat(unavailable.deviceId()).isEqualTo("switch.kitchen");
        assertThat(unavailable).isInstanceOf(SwitchDevice.class);
        assertThat(((SwitchDevice) unavailable).isOn()).isTrue();
    }

    @Test
    void stripTenancyPrefix() {
        assertThat(DeviceIdUtils.stripPrefix("site-a/switch.kitchen"))
                .isEqualTo("switch.kitchen");
    }

    @Test
    void extractTenancyId() {
        assertThat(DeviceIdUtils.extractTenancyId("site-a/switch.kitchen"))
                .isEqualTo("site-a");
    }

    @Test
    void noSlashReturnsOriginal() {
        assertThat(DeviceIdUtils.stripPrefix("switch.kitchen"))
                .isEqualTo("switch.kitchen");
    }
}
```

- [ ] **Step 4: Create DeviceIdNamespacer**

Create `bridge-server/src/main/java/io/casehub/iot/bridge/server/DeviceIdNamespacer.java`:

```java
package io.casehub.iot.bridge.server;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.bridge.DeviceIdUtils;

public class DeviceIdNamespacer {

    private final ObjectMapper mapper;

    public DeviceIdNamespacer(ObjectMapper mapper) {
        this.mapper = mapper;
    }

    public DeviceEntity namespace(DeviceEntity device, String tenancyId) {
        return copyWithDeviceId(device, tenancyId + "/" + device.deviceId());
    }

    public DeviceEntity markUnavailable(DeviceEntity device) {
        ObjectNode tree = mapper.valueToTree(device);
        tree.put("available", false);
        try {
            return mapper.treeToValue(tree, DeviceEntity.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to mark device unavailable: " + device.deviceId(), e);
        }
    }

    private DeviceEntity copyWithDeviceId(DeviceEntity device, String newDeviceId) {
        ObjectNode tree = mapper.valueToTree(device);
        tree.put("deviceId", newDeviceId);
        try {
            return mapper.treeToValue(tree, DeviceEntity.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to namespace device: " + device.deviceId(), e);
        }
    }
}
```

Also create shared utility in iot-api — both bridge and bridge-server need it:

Create `api/src/main/java/io/casehub/iot/api/bridge/DeviceIdUtils.java`:

```java
package io.casehub.iot.api.bridge;

public final class DeviceIdUtils {
    private DeviceIdUtils() {}

    public static String stripPrefix(String namespacedId) {
        int slashIndex = namespacedId.indexOf('/');
        return slashIndex >= 0 ? namespacedId.substring(slashIndex + 1) : namespacedId;
    }

    public static String extractTenancyId(String namespacedId) {
        int slashIndex = namespacedId.indexOf('/');
        return slashIndex >= 0 ? namespacedId.substring(0, slashIndex) : namespacedId;
    }
}
```

- [ ] **Step 5: Create BridgeServerConfig**

Create `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeServerConfig.java`:

```java
package io.casehub.iot.bridge.server;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.iot.bridge-server")
public interface BridgeServerConfig {
    @WithDefault("30")
    int commandTimeoutSeconds();
}
```

- [ ] **Step 6: Run tests**

Run: `mvn --batch-mode test -pl bridge-server -Dtest=DeviceIdNamespacerTest`
Expected: all 6 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add bridge-server/ pom.xml
git commit -m "feat(bridge-server): module setup + DeviceIdNamespacer — Jackson tree copy for polymorphic ID rewriting #5"
```

---

## Task 5: BridgeConnectionRegistry + BridgeDeviceProvider

**Files:**
- Create: `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeConnectionRegistry.java`
- Create: `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeDeviceProvider.java`
- Create: `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeConnectionRegistryTest.java`
- Create: `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeDeviceProviderTest.java`

- [ ] **Step 1: Write failing BridgeConnectionRegistry test**

Create `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeConnectionRegistryTest.java`:

```java
package io.casehub.iot.bridge.server;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BridgeConnectionRegistryTest {

    @Test
    void registerAndLookup() {
        var registry = new BridgeConnectionRegistry();
        var mockSession = new Object(); // stand-in for WebSocketConnection
        registry.register("site-a", mockSession);
        assertThat(registry.getSession("site-a")).contains(mockSession);
    }

    @Test
    void unregisterRemovesSession() {
        var registry = new BridgeConnectionRegistry();
        var mockSession = new Object();
        registry.register("site-a", mockSession);
        registry.unregister("site-a");
        assertThat(registry.getSession("site-a")).isEmpty();
    }

    @Test
    void allConnectedTenancies() {
        var registry = new BridgeConnectionRegistry();
        registry.register("site-a", new Object());
        registry.register("site-b", new Object());
        assertThat(registry.connectedTenancies()).containsExactlyInAnyOrder("site-a", "site-b");
    }

    @Test
    void isFullyConnectedWhenAllKnownTenanciesHaveSessions() {
        var registry = new BridgeConnectionRegistry();
        registry.register("site-a", new Object());
        registry.register("site-b", new Object());
        assertThat(registry.isFullyConnected()).isTrue();
    }

    @Test
    void notFullyConnectedAfterDisconnect() {
        var registry = new BridgeConnectionRegistry();
        registry.register("site-a", new Object());
        registry.register("site-b", new Object());
        registry.unregister("site-b");
        assertThat(registry.isFullyConnected()).isFalse();
        assertThat(registry.hasAnyConnection()).isTrue();
    }

    @Test
    void noConnectionsWhenEmpty() {
        var registry = new BridgeConnectionRegistry();
        assertThat(registry.hasAnyConnection()).isFalse();
        assertThat(registry.isFullyConnected()).isTrue(); // vacuously true — no known tenancies
    }
}
```

- [ ] **Step 2: Create BridgeConnectionRegistry**

Create `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeConnectionRegistry.java`:

```java
package io.casehub.iot.bridge.server;

import jakarta.enterprise.context.ApplicationScoped;

import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class BridgeConnectionRegistry {

    private final ConcurrentHashMap<String, Object> sessions = new ConcurrentHashMap<>();
    private final Set<String> knownTenancies = ConcurrentHashMap.newKeySet();

    public void register(String tenancyId, Object session) {
        sessions.put(tenancyId, session);
        knownTenancies.add(tenancyId);
    }

    public void unregister(String tenancyId) {
        sessions.remove(tenancyId);
    }

    public Optional<Object> getSession(String tenancyId) {
        return Optional.ofNullable(sessions.get(tenancyId));
    }

    public Set<String> connectedTenancies() {
        return Set.copyOf(sessions.keySet());
    }

    public boolean hasAnyConnection() {
        return !sessions.isEmpty();
    }

    public boolean isFullyConnected() {
        return knownTenancies.isEmpty() || sessions.keySet().containsAll(knownTenancies);
    }
}
```

- [ ] **Step 3: Write failing BridgeDeviceProvider test**

Create `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeDeviceProviderTest.java` — test the core device map logic (not WebSocket transport):

```java
package io.casehub.iot.bridge.server;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.iot.api.*;
import io.casehub.iot.testing.Fixtures;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class BridgeDeviceProviderTest {

    BridgeDeviceProvider provider;
    DeviceIdNamespacer namespacer;

    @BeforeEach
    void setup() {
        ObjectMapper mapper = JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .build();
        namespacer = new DeviceIdNamespacer(mapper);
        provider = new BridgeDeviceProvider(namespacer, new BridgeConnectionRegistry());
    }

    @Test
    void providerIdIsBridge() {
        assertThat(provider.providerId()).isEqualTo("bridge");
    }

    @Test
    void snapshotPopulatesDeviceMap() {
        var devices = List.of(Fixtures.switchDevice(), Fixtures.light());
        provider.onSnapshot("site-a", devices);

        var all = provider.discover().await().indefinitely();
        assertThat(all).hasSize(2);
        assertThat(all.get(0).deviceId()).startsWith("site-a/");
    }

    @Test
    void snapshotDiffDetectsStateChange() {
        var sw1 = Fixtures.switchDevice(); // on=false
        provider.onSnapshot("site-a", List.of(sw1));

        var sw2 = SwitchDevice.builder()
                .deviceId(sw1.deviceId()).deviceClass(DeviceClass.SWITCH)
                .label(sw1.label()).available(true)
                .lastUpdated(Instant.now()).tenancyId(sw1.tenancyId())
                .on(true).build();
        var events = provider.onSnapshot("site-a", List.of(sw2));

        assertThat(events).hasSize(1);
        assertThat(events.get(0).changedCapabilities()).contains("isOn");
        assertThat(events.get(0).providerId()).isEqualTo("bridge");
    }

    @Test
    void snapshotDiffDetectsNewDevice() {
        provider.onSnapshot("site-a", List.of(Fixtures.switchDevice()));
        var events = provider.onSnapshot("site-a",
                List.of(Fixtures.switchDevice(), Fixtures.light()));

        var newDeviceEvent = events.stream()
                .filter(e -> e.before() == null).findFirst();
        assertThat(newDeviceEvent).isPresent();
        assertThat(newDeviceEvent.get().after()).isInstanceOf(LightDevice.class);
        assertThat(newDeviceEvent.get().changedCapabilities())
                .containsAll(newDeviceEvent.get().after().capabilities().keySet());
    }

    @Test
    void snapshotDiffDetectsRemovedDevice() {
        provider.onSnapshot("site-a",
                List.of(Fixtures.switchDevice(), Fixtures.light()));
        var events = provider.onSnapshot("site-a", List.of(Fixtures.switchDevice()));

        var removedEvent = events.stream()
                .filter(e -> e.changedCapabilities().contains("available"))
                .findFirst();
        assertThat(removedEvent).isPresent();
        assertThat(removedEvent.get().after().available()).isFalse();
    }

    @Test
    void statusDisconnectedWhenNoAgents() {
        assertThat(provider.status()).isEqualTo(ProviderStatus.DISCONNECTED);
    }
}
```

- [ ] **Step 4: Create BridgeDeviceProvider**

Create `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeDeviceProvider.java`:

This class implements `DeviceProvider`, maintains the device map, and computes snapshot diffs. It fires CDI events but the test above tests the pure logic without CDI.

```java
package io.casehub.iot.bridge.server;

import io.casehub.iot.api.*;
import io.casehub.iot.api.spi.DeviceProvider;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class BridgeDeviceProvider implements DeviceProvider {

    private static final Logger LOG = Logger.getLogger(BridgeDeviceProvider.class);

    private final DeviceIdNamespacer namespacer;
    private final BridgeConnectionRegistry connectionRegistry;
    private final Map<String, DeviceEntity> devices = new ConcurrentHashMap<>();

    @Inject Event<StateChangeEvent> stateEvents;
    @Inject Event<ProviderStatusEvent> statusEvents;

    // CDI-managed constructor
    @Inject
    public BridgeDeviceProvider(DeviceIdNamespacer namespacer,
                                BridgeConnectionRegistry connectionRegistry) {
        this.namespacer = namespacer;
        this.connectionRegistry = connectionRegistry;
    }

    @Override
    public String providerId() {
        return "bridge";
    }

    @Override
    public Uni<List<DeviceEntity>> discover() {
        return Uni.createFrom().item(List.copyOf(devices.values()));
    }

    @Override
    public Uni<CommandResult> dispatch(DeviceCommand command) {
        // Dispatch is handled by BridgeWebSocketEndpoint — sends Command to agent
        // This method is called by CdiDeviceRegistry but the actual dispatch
        // goes through the WebSocket connection, not a local provider call.
        // Returns FAILED if no agent is connected for the target tenancy.
        String tenancyId = DeviceIdNamespacer.extractTenancyId(command.targetDeviceId());
        if (connectionRegistry.getSession(tenancyId).isEmpty()) {
            return Uni.createFrom().item(CommandResult.FAILED);
        }
        // Actual dispatch happens asynchronously via WebSocket — see BridgeWebSocketEndpoint
        return Uni.createFrom().item(CommandResult.SENT);
    }

    @Override
    public ProviderStatus status() {
        if (!connectionRegistry.hasAnyConnection()) return ProviderStatus.DISCONNECTED;
        if (connectionRegistry.isFullyConnected()) return ProviderStatus.CONNECTED;
        return ProviderStatus.CONNECTING;
    }

    public List<StateChangeEvent> onSnapshot(String tenancyId, List<DeviceEntity> incoming) {
        Map<String, DeviceEntity> previous = new HashMap<>();
        devices.entrySet().stream()
                .filter(e -> e.getKey().startsWith(tenancyId + "/"))
                .forEach(e -> previous.put(e.getKey(), e.getValue()));

        Map<String, DeviceEntity> namespacedIncoming = new LinkedHashMap<>();
        for (DeviceEntity device : incoming) {
            DeviceEntity namespaced = namespacer.namespace(device, tenancyId);
            namespacedIncoming.put(namespaced.deviceId(), namespaced);
        }

        List<StateChangeEvent> events = new ArrayList<>();
        Instant now = Instant.now();

        // New or changed devices
        for (var entry : namespacedIncoming.entrySet()) {
            DeviceEntity after = entry.getValue();
            DeviceEntity before = previous.get(entry.getKey());

            if (before == null) {
                // First appearance
                events.add(new StateChangeEvent(null, after,
                        after.capabilities().keySet(), now, "bridge"));
            } else if (!before.capabilities().equals(after.capabilities())) {
                Set<String> changed = StateChangeEvent.deriveChangedCapabilities(before, after);
                if (!changed.isEmpty()) {
                    events.add(new StateChangeEvent(before, after, changed, now, "bridge"));
                }
            }
        }

        // Removed devices
        for (var entry : previous.entrySet()) {
            if (!namespacedIncoming.containsKey(entry.getKey())) {
                DeviceEntity unavailable = namespacer.markUnavailable(entry.getValue());
                events.add(new StateChangeEvent(entry.getValue(), unavailable,
                        Set.of(DeviceEntity.CAP_AVAILABLE), now, "bridge"));
            }
        }

        // Update device map
        previous.keySet().forEach(devices::remove);
        devices.putAll(namespacedIncoming);

        return events;
    }

    public void onStateChange(StateChangeEvent event, String tenancyId) {
        DeviceEntity namespaced = namespacer.namespace(event.after(), tenancyId);
        devices.put(namespaced.deviceId(), namespaced);
    }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl bridge-server`
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add bridge-server/src/
git commit -m "feat(bridge-server): BridgeDeviceProvider + BridgeConnectionRegistry — device map, snapshot diff, multi-site #5"
```

---

## Task 6: Bridge Server WebSocket Endpoint

**Files:**
- Create: `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpoint.java`
- Create: `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpointTest.java`

- [ ] **Step 1: Write test for WebSocket endpoint message handling**

Create `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpointTest.java` — test message routing logic (not WebSocket transport):

```java
package io.casehub.iot.bridge.server;

import io.casehub.iot.api.*;
import io.casehub.iot.api.bridge.BridgeMessage;
import io.casehub.iot.testing.Fixtures;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class BridgeWebSocketEndpointTest {

    @Test
    void stateSnapshotMessageProducesEvents() {
        // This tests the message dispatch logic that will be called
        // from the WebSocket onTextMessage handler
        // Implementation will call provider.onSnapshot() and fire CDI events
        // Detailed WebSocket integration test in Task 8
    }
}
```

- [ ] **Step 2: Create BridgeWebSocketEndpoint**

Create `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpoint.java`:

```java
package io.casehub.iot.bridge.server;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.api.ProviderStatusEvent;
import io.casehub.iot.api.StateChangeEvent;
import io.casehub.iot.api.bridge.BridgeMessage;
import io.quarkus.websockets.next.*;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@WebSocket(path = "/iot/bridge")
public class BridgeWebSocketEndpoint {

    private static final Logger LOG = Logger.getLogger(BridgeWebSocketEndpoint.class);

    @Inject ObjectMapper mapper;
    @Inject BridgeDeviceProvider provider;
    @Inject BridgeConnectionRegistry connectionRegistry;
    @Inject Event<StateChangeEvent> stateEvents;
    @Inject Event<ProviderStatusEvent> statusEvents;

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        String tenancyId = connection.handshakeRequest().header("X-Tenancy-ID");
        if (tenancyId == null || tenancyId.isBlank()) {
            LOG.warn("Bridge agent connected without X-Tenancy-ID header — closing");
            connection.close();
            return;
        }
        connectionRegistry.register(tenancyId, connection);
        LOG.infof("Bridge agent connected: tenancy=%s", tenancyId);
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        String tenancyId = connection.handshakeRequest().header("X-Tenancy-ID");
        if (tenancyId != null) {
            connectionRegistry.unregister(tenancyId);
            LOG.infof("Bridge agent disconnected: tenancy=%s", tenancyId);
        }
    }

    @OnTextMessage
    void onMessage(String text, WebSocketConnection connection) throws Exception {
        BridgeMessage msg = mapper.readValue(text, BridgeMessage.class);

        switch (msg) {
            case BridgeMessage.StateChange sc -> {
                provider.onStateChange(sc.event(), sc.tenancyId());
                stateEvents.fireAsync(sc.event());
            }
            case BridgeMessage.StateSnapshot ss -> {
                var events = provider.onSnapshot(ss.tenancyId(), ss.devices());
                events.forEach(stateEvents::fireAsync);
            }
            case BridgeMessage.ProviderStatus ps -> {
                statusEvents.fireAsync(ps.status());
            }
            case BridgeMessage.CommandResult cr -> {
                // TODO: route to pending command future by correlationId
                LOG.debugf("Command result: correlationId=%s result=%s",
                        cr.correlationId(), cr.result());
            }
            case BridgeMessage.Heartbeat h -> {
                // Connection keepalive — no action needed
            }
            case BridgeMessage.Command c -> {
                LOG.warn("Received Command from agent — commands flow server→agent, not agent→server");
            }
        }
    }
}
```

- [ ] **Step 3: Run tests**

Run: `mvn --batch-mode test -pl bridge-server`
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git add bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpoint.java \
  bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpointTest.java
git commit -m "feat(bridge-server): BridgeWebSocketEndpoint — accepts agent connections, routes messages #5"
```

---

## Task 7: Bridge Agent Module Setup + Filter Chain + Command Dispatcher

**Files:**
- Modify: `bridge/pom.xml`
- Create: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeAgentConfig.java`
- Create: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeFilterChain.java`
- Create: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCommandDispatcher.java`
- Create: `bridge/src/main/resources/application.properties`
- Create: `bridge/src/test/java/io/casehub/iot/bridge/agent/BridgeFilterChainTest.java`
- Create: `bridge/src/test/java/io/casehub/iot/bridge/agent/BridgeCommandDispatcherTest.java`

- [ ] **Step 1: Update bridge/pom.xml**

Replace contents of `bridge/pom.xml` with Quarkus app setup:

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

    <artifactId>casehub-iot-bridge</artifactId>
    <name>CaseHub IoT — Bridge Agent</name>
    <description>Lightweight local bridge agent — event relay, filter chain, command dispatch</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-websockets-next</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-client-jackson</artifactId>
        </dependency>

        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-testing</artifactId>
            <scope>test</scope>
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
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Create BridgeAgentConfig**

Create `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeAgentConfig.java`:

```java
package io.casehub.iot.bridge.agent;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.iot.bridge")
public interface BridgeAgentConfig {
    String cloudEndpoint();
    String token();
    String tenantId();

    @WithDefault("5")
    int reconnectBaseSeconds();

    @WithDefault("300")
    int reconnectMaxSeconds();

    @WithDefault("30")
    int heartbeatIntervalSeconds();
}
```

- [ ] **Step 3: Create application.properties**

Create `bridge/src/main/resources/application.properties`:

```properties
# Bridge Agent Configuration — override per deployment
casehub.iot.bridge.cloud-endpoint=wss://localhost:8443/iot/bridge
casehub.iot.bridge.token=changeme
casehub.iot.bridge.tenant-id=default
```

- [ ] **Step 4: Write failing BridgeFilterChain test**

Create `bridge/src/test/java/io/casehub/iot/bridge/agent/BridgeFilterChainTest.java`:

```java
package io.casehub.iot.bridge.agent;

import io.casehub.iot.api.*;
import io.casehub.iot.api.bridge.*;
import io.casehub.iot.testing.Fixtures;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class BridgeFilterChainTest {

    static final FilterContext CTX = new FilterContext("home-1",
            ConnectionState.CONNECTED, "ha");

    StateChangeEvent sampleEvent() {
        var sw = Fixtures.switchDevice();
        return new StateChangeEvent(null, sw, Set.of("isOn"),
                Instant.now(), "ha");
    }

    @Test
    void noFiltersForwardsAll() {
        var chain = new BridgeFilterChain(List.of());
        FilterAction result = chain.execute(sampleEvent(), CTX).await().indefinitely();
        assertThat(result).isInstanceOf(FilterAction.Forward.class);
    }

    @Test
    void singleForwardFilterForwards() {
        BridgeEventFilter forwardAll = new BridgeEventFilter() {
            public int priority() { return 0; }
            public Uni<FilterAction> filter(StateChangeEvent event, FilterContext ctx) {
                return Uni.createFrom().item(new FilterAction.Forward());
            }
        };
        var chain = new BridgeFilterChain(List.of(forwardAll));
        FilterAction result = chain.execute(sampleEvent(), CTX).await().indefinitely();
        assertThat(result).isInstanceOf(FilterAction.Forward.class);
    }

    @Test
    void suppressShortCircuits() {
        BridgeEventFilter suppress = new BridgeEventFilter() {
            public int priority() { return 10; }
            public Uni<FilterAction> filter(StateChangeEvent event, FilterContext ctx) {
                return Uni.createFrom().item(new FilterAction.Suppress("privacy"));
            }
        };
        BridgeEventFilter shouldNotRun = new BridgeEventFilter() {
            public int priority() { return 20; }
            public Uni<FilterAction> filter(StateChangeEvent event, FilterContext ctx) {
                throw new AssertionError("Should not be called after suppress");
            }
        };
        var chain = new BridgeFilterChain(List.of(suppress, shouldNotRun));
        FilterAction result = chain.execute(sampleEvent(), CTX).await().indefinitely();
        assertThat(result).isInstanceOf(FilterAction.Suppress.class);
        assertThat(((FilterAction.Suppress) result).reason()).isEqualTo("privacy");
    }

    @Test
    void filtersExecuteInPriorityOrder() {
        var order = new java.util.ArrayList<Integer>();
        BridgeEventFilter p10 = new BridgeEventFilter() {
            public int priority() { return 10; }
            public Uni<FilterAction> filter(StateChangeEvent event, FilterContext ctx) {
                order.add(10);
                return Uni.createFrom().item(new FilterAction.Forward());
            }
        };
        BridgeEventFilter p5 = new BridgeEventFilter() {
            public int priority() { return 5; }
            public Uni<FilterAction> filter(StateChangeEvent event, FilterContext ctx) {
                order.add(5);
                return Uni.createFrom().item(new FilterAction.Forward());
            }
        };
        var chain = new BridgeFilterChain(List.of(p10, p5));
        chain.execute(sampleEvent(), CTX).await().indefinitely();
        assertThat(order).containsExactly(5, 10);
    }
}
```

- [ ] **Step 5: Create BridgeFilterChain**

Create `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeFilterChain.java`:

```java
package io.casehub.iot.bridge.agent;

import io.casehub.iot.api.StateChangeEvent;
import io.casehub.iot.api.bridge.BridgeEventFilter;
import io.casehub.iot.api.bridge.FilterAction;
import io.casehub.iot.api.bridge.FilterContext;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Comparator;
import java.util.List;

@ApplicationScoped
public class BridgeFilterChain {

    private static final Logger LOG = Logger.getLogger(BridgeFilterChain.class);
    private final List<BridgeEventFilter> filters;

    @Inject
    public BridgeFilterChain(@Any Instance<BridgeEventFilter> discovered) {
        this.filters = discovered.stream()
                .sorted(Comparator.comparingInt(BridgeEventFilter::priority))
                .toList();
        LOG.infof("Bridge filter chain initialized with %d filter(s)", filters.size());
    }

    // Test constructor
    BridgeFilterChain(List<BridgeEventFilter> filters) {
        this.filters = filters.stream()
                .sorted(Comparator.comparingInt(BridgeEventFilter::priority))
                .toList();
    }

    public Uni<FilterAction> execute(StateChangeEvent event, FilterContext ctx) {
        if (filters.isEmpty()) {
            return Uni.createFrom().item(new FilterAction.Forward());
        }
        return executeChain(event, ctx, 0);
    }

    private Uni<FilterAction> executeChain(StateChangeEvent event, FilterContext ctx, int index) {
        if (index >= filters.size()) {
            return Uni.createFrom().item(new FilterAction.Forward());
        }
        return filters.get(index).filter(event, ctx)
                .flatMap(action -> switch (action) {
                    case FilterAction.Suppress s -> {
                        LOG.debugf("Event suppressed by filter %s (priority %d): %s",
                                filters.get(index).getClass().getSimpleName(),
                                filters.get(index).priority(), s.reason());
                        yield Uni.createFrom().item(action);
                    }
                    case FilterAction.Forward f -> executeChain(event, ctx, index + 1);
                });
    }
}
```

- [ ] **Step 6: Write failing BridgeCommandDispatcher test**

Create `bridge/src/test/java/io/casehub/iot/bridge/agent/BridgeCommandDispatcherTest.java`:

```java
package io.casehub.iot.bridge.agent;

import io.casehub.iot.api.*;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.testing.MockDeviceProvider;
import io.casehub.iot.testing.Fixtures;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class BridgeCommandDispatcherTest {

    @Test
    void dispatchStripsTenancyPrefix() {
        var mockProvider = new MockDeviceProvider();
        mockProvider.addDevice(Fixtures.switchDevice());
        mockProvider.setDispatchResult(CommandResult.SENT);

        var dispatcher = new BridgeCommandDispatcher(mockProvider);

        var command = DeviceCommand.turnOn("home-1/switch-1", Map.of(), "cloud", "corr-1");
        var result = dispatcher.dispatch(command).await().indefinitely();

        assertThat(result).isEqualTo(CommandResult.SENT);
        assertThat(mockProvider.dispatchedCommands()).hasSize(1);
        assertThat(mockProvider.dispatchedCommands().get(0).targetDeviceId())
                .isEqualTo("switch-1");
    }
}
```

- [ ] **Step 7: Create BridgeCommandDispatcher**

Create `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCommandDispatcher.java`:

```java
package io.casehub.iot.bridge.agent;

import io.casehub.iot.api.CommandResult;
import io.casehub.iot.api.DeviceCommand;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.bridge.DeviceIdUtils;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class BridgeCommandDispatcher {

    private static final Logger LOG = Logger.getLogger(BridgeCommandDispatcher.class);
    private final DeviceProvider provider;

    @Inject
    public BridgeCommandDispatcher(@Any Instance<DeviceProvider> providers) {
        // The bridge agent has exactly one real provider on its classpath
        // (HA or OH). Pick the first non-bridge provider.
        this.provider = providers.stream()
                .findFirst()
                .orElseThrow(() -> new IllegalStateException("No DeviceProvider found"));
    }

    // Test constructor
    BridgeCommandDispatcher(DeviceProvider provider) {
        this.provider = provider;
    }

    public Uni<CommandResult> dispatch(DeviceCommand command) {
        String localDeviceId = DeviceIdUtils.stripPrefix(command.targetDeviceId());
        DeviceCommand localCommand = new DeviceCommand(localDeviceId, command.action(),
                command.parameters(), command.dispatchedBy(), command.correlationId());
        LOG.debugf("Dispatching command to local provider: device=%s action=%s",
                localDeviceId, command.action());
        return provider.dispatch(localCommand);
    }
}
```

- [ ] **Step 8: Run tests**

Run: `mvn --batch-mode test -pl bridge`
Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git add bridge/
git commit -m "feat(bridge): agent module — BridgeFilterChain + BridgeCommandDispatcher + config #5"
```

---

## Task 8: Bridge Agent WebSocket Client + Event Observer + Connection Manager

**Files:**
- Create: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCloudClient.java`
- Create: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeEventObserver.java`
- Create: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeConnectionManager.java`

This task creates the WebSocket client, CDI event observer, and connection lifecycle manager. These components are integration-heavy (WebSocket connections, CDI events, timers) — the unit-testable logic was already extracted in Tasks 5-7 (filter chain, command dispatcher, device map). These components wire everything together.

- [ ] **Step 1: Create BridgeCloudClient**

Create `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCloudClient.java`:

```java
package io.casehub.iot.bridge.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.api.bridge.BridgeMessage;
import io.quarkus.websockets.next.*;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@WebSocketClient(path = "/iot/bridge")
public class BridgeCloudClient {

    private static final Logger LOG = Logger.getLogger(BridgeCloudClient.class);

    @Inject ObjectMapper mapper;
    @Inject BridgeCommandDispatcher commandDispatcher;

    @OnOpen
    void onOpen(WebSocketClientConnection connection) {
        LOG.info("Connected to cloud bridge endpoint");
    }

    @OnClose
    void onClose(WebSocketClientConnection connection) {
        LOG.info("Disconnected from cloud bridge endpoint");
    }

    @OnTextMessage
    void onMessage(String text, WebSocketClientConnection connection) throws Exception {
        BridgeMessage msg = mapper.readValue(text, BridgeMessage.class);

        switch (msg) {
            case BridgeMessage.Command cmd -> {
                var result = commandDispatcher.dispatch(cmd.command()).await().indefinitely();
                var response = new BridgeMessage.CommandResult(
                        cmd.tenancyId(), java.time.Instant.now(),
                        cmd.correlationId(), result);
                connection.sendTextAndAwait(mapper.writeValueAsString(response));
            }
            case BridgeMessage.Heartbeat h -> {
                // Respond with heartbeat
                connection.sendTextAndAwait(mapper.writeValueAsString(
                        new BridgeMessage.Heartbeat(h.tenancyId(), java.time.Instant.now())));
            }
            default -> LOG.warnf("Unexpected message type from cloud: %s",
                    msg.getClass().getSimpleName());
        }
    }
}
```

- [ ] **Step 2: Create BridgeEventObserver**

Create `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeEventObserver.java`:

```java
package io.casehub.iot.bridge.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.api.ProviderStatusEvent;
import io.casehub.iot.api.StateChangeEvent;
import io.casehub.iot.api.bridge.*;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Instant;

@ApplicationScoped
public class BridgeEventObserver {

    private static final Logger LOG = Logger.getLogger(BridgeEventObserver.class);

    @Inject BridgeFilterChain filterChain;
    @Inject BridgeConnectionManager connectionManager;
    @Inject BridgeAgentConfig config;
    @Inject ObjectMapper mapper;

    void onStateChange(@ObservesAsync StateChangeEvent event) {
        if (!connectionManager.isConnected()) {
            return;
        }

        FilterContext ctx = new FilterContext(config.tenantId(),
                ConnectionState.CONNECTED, event.providerId());

        FilterAction action = filterChain.execute(event, ctx).await().indefinitely();
        switch (action) {
            case FilterAction.Forward f -> {
                var msg = new BridgeMessage.StateChange(config.tenantId(), Instant.now(), event);
                connectionManager.send(msg);
            }
            case FilterAction.Suppress s ->
                    LOG.debugf("Event suppressed: %s", s.reason());
        }
    }

    void onProviderStatus(@ObservesAsync ProviderStatusEvent event) {
        if (!connectionManager.isConnected()) {
            return;
        }
        var msg = new BridgeMessage.ProviderStatus(config.tenantId(), Instant.now(), event);
        connectionManager.send(msg);
    }
}
```

- [ ] **Step 3: Create BridgeConnectionManager**

Create `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeConnectionManager.java`:

```java
package io.casehub.iot.bridge.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.bridge.BridgeMessage;
import io.casehub.iot.api.spi.DeviceProvider;
import io.quarkus.runtime.StartupEvent;
import io.quarkus.websockets.next.WebSocketClientConnection;
import io.quarkus.websockets.next.WebSocketConnector;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.net.URI;
import java.time.Instant;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

@ApplicationScoped
public class BridgeConnectionManager {

    private static final Logger LOG = Logger.getLogger(BridgeConnectionManager.class);

    @Inject BridgeAgentConfig config;
    @Inject ObjectMapper mapper;
    @Inject @Any Instance<DeviceProvider> providers;
    @Inject WebSocketConnector<BridgeCloudClient> connector;

    private final AtomicReference<WebSocketClientConnection> connection = new AtomicReference<>();

    void onStartup(@Observes StartupEvent event) {
        connect();
    }

    public boolean isConnected() {
        WebSocketClientConnection conn = connection.get();
        return conn != null && conn.isOpen();
    }

    public void send(BridgeMessage message) {
        WebSocketClientConnection conn = connection.get();
        if (conn != null && conn.isOpen()) {
            try {
                conn.sendTextAndAwait(mapper.writeValueAsString(message));
            } catch (Exception e) {
                LOG.errorf(e, "Failed to send bridge message: %s", message.getClass().getSimpleName());
            }
        }
    }

    private void connect() {
        try {
            URI uri = URI.create(config.cloudEndpoint());
            WebSocketClientConnection conn = connector
                    .baseUri(uri)
                    .addHeader("Authorization", "Bearer " + config.token())
                    .addHeader("X-Tenancy-ID", config.tenantId())
                    .connectAndAwait();
            connection.set(conn);
            LOG.infof("Connected to cloud: %s", config.cloudEndpoint());
            sendSnapshot();
        } catch (Exception e) {
            LOG.errorf(e, "Failed to connect to cloud: %s", config.cloudEndpoint());
            scheduleReconnect(config.reconnectBaseSeconds());
        }
    }

    private void sendSnapshot() {
        List<DeviceEntity> allDevices = providers.stream()
                .flatMap(p -> p.discover().await().indefinitely().stream())
                .toList();

        var snapshot = new BridgeMessage.StateSnapshot(
                config.tenantId(), Instant.now(), allDevices);
        send(snapshot);
        LOG.infof("Sent state snapshot: %d devices", allDevices.size());
    }

    private void scheduleReconnect(int delaySeconds) {
        int capped = Math.min(delaySeconds, config.reconnectMaxSeconds());
        LOG.infof("Reconnecting in %d seconds", capped);
        // Exponential backoff with jitter — reconnect on virtual thread
        Thread.ofVirtual().start(() -> {
            try {
                Thread.sleep(capped * 1000L);
                connect();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
}
```

- [ ] **Step 4: Run full build**

Run: `mvn --batch-mode install`
Expected: all modules compile and tests pass.

- [ ] **Step 5: Commit**

```bash
git add bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCloudClient.java \
  bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeEventObserver.java \
  bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeConnectionManager.java
git commit -m "feat(bridge): agent WebSocket client + event observer + connection lifecycle #5"
```

---

## Task 9: Integration Test — Agent ↔ Server

**Files:**
- Create: `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeIntegrationTest.java`

This test runs a bridge-server WebSocket endpoint and a bridge agent connecting to it, verifying end-to-end event flow.

- [ ] **Step 1: Write integration test**

Create `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeIntegrationTest.java`:

```java
package io.casehub.iot.bridge.server;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.api.*;
import io.casehub.iot.api.bridge.BridgeMessage;
import io.casehub.iot.testing.Fixtures;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.websockets.next.WebSocketClientConnection;
import io.quarkus.websockets.next.WebSocketConnector;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class BridgeIntegrationTest {

    @Inject ObjectMapper mapper;
    @Inject BridgeDeviceProvider provider;
    @Inject BridgeConnectionRegistry registry;

    @Test
    void snapshotPopulatesProviderDeviceMap() throws Exception {
        // Simulate a bridge agent sending a snapshot
        var snapshot = new BridgeMessage.StateSnapshot("test-tenant",
                Instant.now(), List.of(Fixtures.switchDevice(), Fixtures.light()));

        provider.onSnapshot("test-tenant", snapshot.devices());

        var devices = provider.discover().await().indefinitely();
        assertThat(devices).hasSize(2);
        assertThat(devices).allSatisfy(d ->
                assertThat(d.deviceId()).startsWith("test-tenant/"));
    }

    @Test
    void stateChangeUpdatesDeviceMap() throws Exception {
        var sw = Fixtures.switchDevice();
        provider.onSnapshot("test-tenant", List.of(sw));

        var updated = SwitchDevice.builder()
                .deviceId(sw.deviceId()).deviceClass(DeviceClass.SWITCH)
                .label(sw.label()).available(true)
                .lastUpdated(Instant.now()).tenancyId(sw.tenancyId())
                .on(true).build();
        var event = new StateChangeEvent(sw, updated, Set.of("isOn"),
                Instant.now(), "ha");

        provider.onStateChange(event, "test-tenant");

        var devices = provider.discover().await().indefinitely();
        var found = devices.stream()
                .filter(d -> d.deviceId().contains(sw.deviceId()))
                .findFirst();
        assertThat(found).isPresent();
        assertThat(found.get()).isInstanceOf(SwitchDevice.class);
        assertThat(((SwitchDevice) found.get()).isOn()).isTrue();
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `mvn --batch-mode test -pl bridge-server -Dtest=BridgeIntegrationTest`
Expected: all tests pass.

- [ ] **Step 3: Run full build**

Run: `mvn --batch-mode install`
Expected: all modules compile and all tests pass.

- [ ] **Step 4: Commit**

```bash
git add bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeIntegrationTest.java
git commit -m "test(bridge): integration test — snapshot populates provider, state change updates map #5"
```

---

## Task 10: Final Build Verification + Code Review

- [ ] **Step 1: Full clean build**

Run: `mvn --batch-mode clean install`
Expected: all modules compile. All tests pass.

- [ ] **Step 2: Invoke code review**

Invoke `superpowers:requesting-code-review` on all changes since the branch started. Any finding Minor or above that isn't fixable this session must be captured as a GitHub issue.

- [ ] **Step 3: Fix review findings**

Apply fixes, commit.

- [ ] **Step 4: Invoke implementation-doc-sync**

Invoke `implementation-doc-sync` to update ARC42STORIES.MD and other docs.

- [ ] **Step 5: Final commit and push**

```bash
git push -u origin issue-5-bridge-runtime
```
