# IoT MCP Tools Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #69 — feat: MCP tool exposure for DeviceProvider operations
**Issue group:** #69

**Goal:** Add a new `mcp/` module exposing `iot_get_devices`, `iot_get_state`,
and `iot_send_command` as Quarkus MCP server tools, following the
`connectors/mcp` library pattern.

**Architecture:** Single `@ApplicationScoped` bean (`IoTDeviceMcpTool`) with
three `@Tool`-annotated methods. Injects `DeviceRegistry` for reads and
`Instance<DeviceProvider>` for command dispatch. Host-agnostic library — any
Quarkus app adding this module + `quarkus-mcp-server-http` gets IoT tools at
`/mcp`.

**Tech Stack:** Quarkus MCP Server Core 1.11.1, Jackson, CDI

## Global Constraints

- `quarkus-mcp-server-core` version: 1.11.1 (matches connectors)
- Module artifact: `casehub-iot-mcp`
- Package: `io.casehub.iot.mcp`
- No `quarkus-mcp-server-http` dependency — host app provides HTTP transport
- Jandex plugin required for library CDI discovery
- `@Blocking` on all tool methods (registry reads are synchronous)
- All failure paths log WARN before returning fail-soft string
- `dispatchedBy` = `"mcp-agent"` (no principal propagation yet)

---

### Task 1: Module scaffold, DeviceSummary, and read tools (iot_get_devices, iot_get_state)

**Files:**
- Create: `mcp/pom.xml`
- Modify: `pom.xml` (parent — add module, dependency management, version property)
- Create: `mcp/src/main/java/io/casehub/iot/mcp/DeviceSummary.java`
- Create: `mcp/src/main/java/io/casehub/iot/mcp/IoTDeviceMcpTool.java`
- Test: `mcp/src/test/java/io/casehub/iot/mcp/IoTDeviceMcpToolTest.java`

**Interfaces:**
- Consumes: `DeviceRegistry` (api), `DeviceEntity` subclasses (api), `DeviceClass` (api), `Instance<DeviceProvider>` (CDI), `ObjectMapper` (Jackson)
- Produces: `IoTDeviceMcpTool.getDevices()`, `IoTDeviceMcpTool.getState()`, `DeviceSummary` record — Task 2 adds `sendCommand()` to same class

- [ ] **Step 1: Create module directory and pom.xml**

```bash
mkdir -p /Users/mdproctor/claude/casehub/iot/mcp/src/main/java/io/casehub/iot/mcp
mkdir -p /Users/mdproctor/claude/casehub/iot/mcp/src/test/java/io/casehub/iot/mcp
```

Create `mcp/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-iot-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-iot-mcp</artifactId>
    <name>CaseHub IoT — MCP Tools</name>
    <description>MCP tool surface for casehub-iot. Exposes iot_get_devices, iot_get_state,
and iot_send_command as Quarkus MCP server tools. Add this module and
quarkus-mcp-server-http to any Quarkus app to let LLM agents interact with IoT devices.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.mcp</groupId>
            <artifactId>quarkus-mcp-server-core</artifactId>
            <version>${quarkus-mcp-server.version}</version>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
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

- [ ] **Step 2: Update parent pom.xml**

Add the version property, dependency management entry, and module declaration.
Use `ide_replace_text_in_file` for each change in `pom.xml` (root).

Add property after `<version.drools>10.1.0</version.drools>`:
```xml
        <quarkus-mcp-server.version>1.11.1</quarkus-mcp-server.version>
```

Add dependency management entry after `casehub-iot-bridge-server`:
```xml
            <dependency>
                <groupId>io.casehub</groupId>
                <artifactId>casehub-iot-mcp</artifactId>
                <version>${project.version}</version>
            </dependency>
```

Add module after `<module>bridge-server</module>`:
```xml
        <module>mcp</module>
```

- [ ] **Step 3: Verify module compiles**

Run: `mvn --batch-mode -pl mcp -am compile`
Expected: BUILD SUCCESS

- [ ] **Step 4: Create DeviceSummary record**

Use `ide_create_file` to create `mcp/src/main/java/io/casehub/iot/mcp/DeviceSummary.java`:

```java
package io.casehub.iot.mcp;

import io.casehub.iot.api.DeviceEntity;
import java.time.Instant;

record DeviceSummary(
    String deviceId,
    String deviceClass,
    String label,
    String providerId,
    String location,
    boolean available,
    Instant lastUpdated
) {
    static DeviceSummary from(DeviceEntity device) {
        return new DeviceSummary(
            device.deviceId(),
            device.deviceClass().name(),
            device.label(),
            device.providerId(),
            device.location(),
            device.available(),
            device.lastUpdated()
        );
    }
}
```

- [ ] **Step 5: Write failing tests for iot_get_devices and iot_get_state**

Use `ide_create_file` to create `mcp/src/test/java/io/casehub/iot/mcp/IoTDeviceMcpToolTest.java`:

```java
package io.casehub.iot.mcp;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.iot.api.CommandResult;
import io.casehub.iot.api.DeviceClass;
import io.casehub.iot.api.LightDevice;
import io.casehub.iot.api.ThermostatDevice;
import io.casehub.iot.api.Temperature;
import io.casehub.iot.api.ThermostatMode;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.testing.MockDeviceProvider;
import io.casehub.iot.testing.MockDeviceRegistry;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class IoTDeviceMcpToolTest {

    private static final Instant NOW = Instant.parse("2026-07-26T10:00:00Z");
    private static final ObjectMapper MAPPER = new ObjectMapper()
            .registerModule(new JavaTimeModule());

    private MockDeviceRegistry registry;
    private MockDeviceProvider provider;
    private Instance<DeviceProvider> providers;
    private IoTDeviceMcpTool tool;

    @SuppressWarnings("unchecked")
    @BeforeEach
    void setUp() {
        registry = new MockDeviceRegistry();
        provider = new MockDeviceProvider("test-provider");
        providers = mock(Instance.class);
        when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
        tool = new IoTDeviceMcpTool(registry, providers, MAPPER);
    }

    private LightDevice light() {
        return new LightDevice.Builder()
                .deviceId("light.living_room").deviceClass(DeviceClass.LIGHT)
                .label("Living Room Light").available(true).lastUpdated(NOW)
                .tenancyId("default-tenant").providerId("test-provider")
                .on(true).brightness(80).build();
    }

    private ThermostatDevice thermostat() {
        return new ThermostatDevice.Builder()
                .deviceId("thermostat.hallway").deviceClass(DeviceClass.THERMOSTAT)
                .label("Hallway Thermostat").available(true).lastUpdated(NOW)
                .tenancyId("default-tenant").providerId("test-provider")
                .currentTemperature(new Temperature(new BigDecimal("21.5"), Temperature.TemperatureUnit.CELSIUS))
                .targetTemperature(new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS))
                .mode(ThermostatMode.HEAT).build();
    }

    // --- iot_get_devices ---

    @Test
    void getDevicesReturnsAllDevices() throws Exception {
        registry.addDevices(light(), thermostat());
        String result = tool.getDevices(null, null, null);
        JsonNode array = MAPPER.readTree(result);
        assertThat(array.isArray()).isTrue();
        assertThat(array).hasSize(2);
    }

    @Test
    void getDevicesFiltersByDeviceClass() throws Exception {
        registry.addDevices(light(), thermostat());
        String result = tool.getDevices("LIGHT", null, null);
        JsonNode array = MAPPER.readTree(result);
        assertThat(array).hasSize(1);
        assertThat(array.get(0).get("deviceClass").asText()).isEqualTo("LIGHT");
    }

    @Test
    void getDevicesMatchesDeviceClassCaseInsensitively() throws Exception {
        registry.addDevices(light());
        String result = tool.getDevices("light", null, null);
        JsonNode array = MAPPER.readTree(result);
        assertThat(array).hasSize(1);
    }

    @Test
    void getDevicesReturnsErrorForInvalidDeviceClass() {
        String result = tool.getDevices("INVALID_CLASS", null, null);
        assertThat(result).startsWith("Failed: Unknown device class: INVALID_CLASS");
        assertThat(result).contains("LIGHT");
    }

    @Test
    void getDevicesFiltersByProviderId() throws Exception {
        registry.addDevices(light(), thermostat());
        String result = tool.getDevices(null, "test-provider", null);
        JsonNode array = MAPPER.readTree(result);
        assertThat(array).hasSize(2);

        String result2 = tool.getDevices(null, "unknown-provider", null);
        JsonNode array2 = MAPPER.readTree(result2);
        assertThat(array2).hasSize(0);
    }

    @Test
    void getDevicesFiltersByAvailability() throws Exception {
        var unavailableLight = new LightDevice.Builder()
                .deviceId("light.off").deviceClass(DeviceClass.LIGHT)
                .label("Offline Light").available(false).lastUpdated(NOW)
                .tenancyId("default-tenant").providerId("test-provider")
                .on(false).build();
        registry.addDevices(light(), unavailableLight);

        String onlineResult = tool.getDevices(null, null, true);
        assertThat(MAPPER.readTree(onlineResult)).hasSize(1);

        String offlineResult = tool.getDevices(null, null, false);
        assertThat(MAPPER.readTree(offlineResult)).hasSize(1);
    }

    @Test
    void getDevicesReturnsEmptyArrayWhenNoneMatch() throws Exception {
        String result = tool.getDevices(null, null, null);
        JsonNode array = MAPPER.readTree(result);
        assertThat(array.isArray()).isTrue();
        assertThat(array).isEmpty();
    }

    @Test
    void getDevicesReturnsSummaryFormat() throws Exception {
        registry.addDevice(light());
        String result = tool.getDevices(null, null, null);
        JsonNode device = MAPPER.readTree(result).get(0);
        assertThat(device.has("deviceId")).isTrue();
        assertThat(device.has("deviceClass")).isTrue();
        assertThat(device.has("label")).isTrue();
        assertThat(device.has("providerId")).isTrue();
        assertThat(device.has("available")).isTrue();
        assertThat(device.has("lastUpdated")).isTrue();
        // Summary format must NOT contain typed device state
        assertThat(device.has("on")).isFalse();
        assertThat(device.has("brightness")).isFalse();
        assertThat(device.has("@deviceType")).isFalse();
    }

    // --- iot_get_state ---

    @Test
    void getStateReturnsDeviceJson() throws Exception {
        registry.addDevice(thermostat());
        String result = tool.getState("thermostat.hallway");
        JsonNode node = MAPPER.readTree(result);
        assertThat(node.get("@deviceType").asText()).isEqualTo("thermostat");
        assertThat(node.get("deviceId").asText()).isEqualTo("thermostat.hallway");
        assertThat(node.has("currentTemperature")).isTrue();
        assertThat(node.has("mode")).isTrue();
    }

    @Test
    void getStateReturnsErrorForUnknownDevice() {
        String result = tool.getState("nonexistent");
        assertThat(result).isEqualTo("Device not found: nonexistent");
    }
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `mvn --batch-mode -pl mcp test`
Expected: COMPILATION FAILURE — `IoTDeviceMcpTool` class doesn't exist yet

- [ ] **Step 7: Implement IoTDeviceMcpTool (read tools only)**

Use `ide_create_file` to create `mcp/src/main/java/io/casehub/iot/mcp/IoTDeviceMcpTool.java`:

```java
package io.casehub.iot.mcp;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.api.DeviceClass;
import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import io.smallrye.common.annotation.Blocking;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

@ApplicationScoped
public class IoTDeviceMcpTool {

    private static final Logger LOG = Logger.getLogger(IoTDeviceMcpTool.class);

    private final DeviceRegistry deviceRegistry;
    private final Instance<DeviceProvider> providers;
    private final ObjectMapper objectMapper;

    @Inject
    public IoTDeviceMcpTool(final DeviceRegistry deviceRegistry,
                            final Instance<DeviceProvider> providers,
                            final ObjectMapper objectMapper) {
        this.deviceRegistry = deviceRegistry;
        this.providers = providers;
        this.objectMapper = objectMapper;
    }

    @Tool(name = "iot_get_devices",
          description = "List IoT devices with optional filters. Returns device ID, "
                      + "class, label, location, provider, and availability. "
                      + "Use iot_get_state for full device state.")
    @Blocking
    public String getDevices(
            @ToolArg(description = "Filter by device class. Valid values: SWITCH, LIGHT, "
                                 + "THERMOSTAT, SENSOR, PRESENCE_SENSOR, POWER_SENSOR, "
                                 + "LOCK, COVER, MEDIA_PLAYER, FAN, CAMERA. Case-insensitive.",
                     required = false)
            final String deviceClass,
            @ToolArg(description = "Filter by provider ID (e.g. 'homeassistant', 'openhab').",
                     required = false)
            final String providerId,
            @ToolArg(description = "Filter by availability: true for online devices, "
                                 + "false for offline.",
                     required = false)
            final Boolean available) {

        final DeviceClass parsedClass;
        if (deviceClass != null && !deviceClass.isBlank()) {
            try {
                parsedClass = DeviceClass.valueOf(deviceClass.toUpperCase());
            } catch (final IllegalArgumentException e) {
                return "Failed: Unknown device class: " + deviceClass
                     + ". Valid values: " + Arrays.stream(DeviceClass.values())
                         .map(Enum::name)
                         .collect(Collectors.joining(", "));
            }
        } else {
            parsedClass = null;
        }

        List<DeviceSummary> summaries = deviceRegistry.findAll().stream()
                .filter(d -> parsedClass == null || d.deviceClass() == parsedClass)
                .filter(d -> providerId == null || d.providerId().equals(providerId))
                .filter(d -> available == null || d.available() == available)
                .map(DeviceSummary::from)
                .toList();

        try {
            return objectMapper.writeValueAsString(summaries);
        } catch (final JsonProcessingException e) {
            LOG.warnf("iot_get_devices failed [%s]: %s",
                    e.getClass().getSimpleName(), e.getMessage());
            return "Failed: " + e.getMessage();
        }
    }

    @Tool(name = "iot_get_state",
          description = "Get current state for a specific IoT device. Returns the full "
                      + "device state including typed fields (temperature, humidity, "
                      + "mode, etc.) and availability.")
    @Blocking
    public String getState(
            @ToolArg(description = "The device ID to query (e.g. 'light.living_room', "
                                 + "'sensor.outdoor_temp'). Use iot_get_devices to "
                                 + "discover available device IDs.")
            final String deviceId) {
        return deviceRegistry.findById(deviceId)
                .map(device -> {
                    try {
                        return objectMapper.writeValueAsString(device);
                    } catch (final JsonProcessingException e) {
                        LOG.warnf("iot_get_state failed [%s]: %s",
                                e.getClass().getSimpleName(), e.getMessage());
                        return "Failed: " + e.getMessage();
                    }
                })
                .orElse("Device not found: " + deviceId);
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn --batch-mode -pl mcp test`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on both new source files and the test file to check
for compilation errors, unused imports, or warnings.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add mcp/ pom.xml
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#69): add casehub-iot-mcp module with read tools

Add iot_get_devices and iot_get_state MCP tools following the
connectors/mcp library pattern. Includes DeviceSummary projection
record and full test coverage."
```

---

### Task 2: iot_send_command tool with tests

**Files:**
- Modify: `mcp/src/main/java/io/casehub/iot/mcp/IoTDeviceMcpTool.java`
- Modify: `mcp/src/test/java/io/casehub/iot/mcp/IoTDeviceMcpToolTest.java`

**Interfaces:**
- Consumes: `DeviceCommand` (api), `CommandResult` (api), `DeviceProvider.dispatch()` (api), `Instance<DeviceProvider>` (CDI)
- Produces: `IoTDeviceMcpTool.sendCommand()` — completes the MCP tool surface

- [ ] **Step 1: Write failing tests for iot_send_command**

Add these tests to `IoTDeviceMcpToolTest.java` using `ide_insert_member`:

```java
// --- iot_send_command ---

@Test
void sendCommandDispatchesToCorrectProvider() {
    registry.addDevice(light());
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
    String result = tool.sendCommand("light.living_room", "turn_on", null);
    assertThat(result).contains("result=SENT");
    assertThat(provider.dispatchedCommands()).hasSize(1);
    assertThat(provider.dispatchedCommands().get(0).action()).isEqualTo("turn_on");
    assertThat(provider.dispatchedCommands().get(0).targetDeviceId()).isEqualTo("light.living_room");
}

@Test
void sendCommandReturnsConfirmation() {
    registry.addDevice(light());
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
    String result = tool.sendCommand("light.living_room", "turn_on", null);
    assertThat(result).startsWith("Command turn_on sent to light.living_room");
    assertThat(result).contains("result=SENT");
    assertThat(result).contains("correlationId=");
}

@Test
void sendCommandFailsForUnknownDevice() {
    String result = tool.sendCommand("nonexistent", "turn_on", null);
    assertThat(result).isEqualTo("Failed: Device not found: nonexistent");
}

@Test
void sendCommandFailsForUnknownProvider() {
    var deviceWithUnknownProvider = new LightDevice.Builder()
            .deviceId("light.orphan").deviceClass(DeviceClass.LIGHT)
            .label("Orphan Light").available(true).lastUpdated(NOW)
            .tenancyId("default-tenant").providerId("unknown-provider")
            .on(false).build();
    registry.addDevice(deviceWithUnknownProvider);
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
    String result = tool.sendCommand("light.orphan", "turn_on", null);
    assertThat(result).isEqualTo("Failed: Provider not found: unknown-provider");
}

@Test
void sendCommandHandlesDispatchFailure() {
    registry.addDevice(light());
    var failingProvider = mock(DeviceProvider.class);
    when(failingProvider.providerId()).thenReturn("test-provider");
    when(failingProvider.dispatch(any())).thenReturn(
            io.smallrye.mutiny.Uni.createFrom().failure(new RuntimeException("connection lost")));
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(failingProvider));
    String result = tool.sendCommand("light.living_room", "turn_on", null);
    assertThat(result).startsWith("Failed: ");
    assertThat(result).contains("connection lost");
}

@Test
void sendCommandPassesParametersMap() {
    registry.addDevice(light());
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
    var params = java.util.Map.<String, Object>of("brightness", 50);
    tool.sendCommand("light.living_room", "turn_on", params);
    assertThat(provider.dispatchedCommands().get(0).parameters())
            .containsEntry("brightness", 50);
}

@Test
void sendCommandHandlesNullParameters() {
    registry.addDevice(light());
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
    tool.sendCommand("light.living_room", "turn_on", null);
    assertThat(provider.dispatchedCommands().get(0).parameters()).isEmpty();
}

@Test
void sendCommandReportsFailedResult() {
    registry.addDevice(light());
    provider.setDispatchResult(CommandResult.FAILED);
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
    String result = tool.sendCommand("light.living_room", "turn_on", null);
    assertThat(result).contains("result: FAILED");
    assertThat(result).doesNotContain("result=SENT");
}

@Test
void sendCommandReportsTimeoutResult() {
    registry.addDevice(light());
    provider.setDispatchResult(CommandResult.TIMEOUT);
    when(providers.stream()).thenReturn(java.util.stream.Stream.of(provider));
    String result = tool.sendCommand("light.living_room", "turn_on", null);
    assertThat(result).contains("result: TIMEOUT");
}
```

Add these imports at the top of the test file (they'll be needed):
```java
import io.casehub.iot.api.CommandResult;
import static org.mockito.ArgumentMatchers.any;
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl mcp test`
Expected: COMPILATION FAILURE — `sendCommand` method doesn't exist yet

- [ ] **Step 3: Implement sendCommand method**

Add to `IoTDeviceMcpTool.java` using `ide_insert_member`, after `getState`:

```java
@Tool(name = "iot_send_command",
      description = "Send a command to an IoT device. Supports actions: "
                  + "turn_on, turn_off, set_temperature, lock, unlock, "
                  + "set_position, set_volume. Returns confirmation with "
                  + "correlation ID on success.")
@Blocking
public String sendCommand(
        @ToolArg(description = "Target device ID (e.g. 'light.living_room'). "
                             + "Use iot_get_devices to find available devices.")
        final String deviceId,
        @ToolArg(description = "Command action: turn_on, turn_off, set_temperature, "
                             + "lock, unlock, set_position, set_volume.")
        final String action,
        @ToolArg(description = "Command parameters (e.g. {\"temperature\": 22.0, "
                             + "\"unit\": \"CELSIUS\"} for set_temperature, "
                             + "{\"position\": 50} for set_position, "
                             + "{\"volume\": 75} for set_volume). Not needed for "
                             + "turn_on, turn_off, lock, unlock.",
                 required = false)
        final Map<String, Object> parameters) {
    var deviceOpt = deviceRegistry.findById(deviceId);
    if (deviceOpt.isEmpty()) {
        LOG.warnf("iot_send_command failed: Device not found: %s", deviceId);
        return "Failed: Device not found: " + deviceId;
    }
    var device = deviceOpt.get();

    var providerOpt = providers.stream()
            .filter(p -> p.providerId().equals(device.providerId()))
            .findFirst();
    if (providerOpt.isEmpty()) {
        LOG.warnf("iot_send_command failed: Provider not found: %s", device.providerId());
        return "Failed: Provider not found: " + device.providerId();
    }

    String correlationId = java.util.UUID.randomUUID().toString();
    var command = new DeviceCommand(
            deviceId,
            action,
            parameters != null ? parameters : Map.of(),
            "mcp-agent",
            correlationId
    );

    try {
        var result = providerOpt.get().dispatch(command)
                .await().atMost(java.time.Duration.ofSeconds(30));

        if (result == CommandResult.SENT) {
            return "Command " + action + " sent to " + deviceId
                 + " (result=SENT, correlationId=" + correlationId + ")";
        }
        return "Command " + action + " to " + deviceId
             + " result: " + result.name()
             + " (correlationId=" + correlationId + ")";
    } catch (final Exception e) {
        LOG.warnf("iot_send_command failed [%s]: %s",
                e.getClass().getSimpleName(), e.getMessage());
        if (e instanceof io.smallrye.mutiny.TimeoutException) {
            return "Failed: Command timed out after 30s"
                 + " (correlationId=" + correlationId + ")";
        }
        return "Failed: " + e.getMessage();
    }
}
```

Add these imports to `IoTDeviceMcpTool.java`:
```java
import io.casehub.iot.api.CommandResult;
import io.casehub.iot.api.DeviceCommand;
import java.util.Map;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -pl mcp test`
Expected: BUILD SUCCESS, all tests pass (Task 1 tests + Task 2 tests)

- [ ] **Step 5: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on `IoTDeviceMcpTool.java` and `IoTDeviceMcpToolTest.java`
to confirm no errors or warnings.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add mcp/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#69): add iot_send_command MCP tool

Complete the IoT MCP tool surface with command dispatch. Routes
commands via DeviceRegistry lookup + provider match. 30s safety
timeout, fail-soft error handling, WARN logging on all failures."
```
