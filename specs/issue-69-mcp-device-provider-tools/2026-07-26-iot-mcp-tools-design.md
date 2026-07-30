# IoT MCP Tool Exposure — Design Spec

**Issue:** casehubio/iot#69
**Date:** 2026-07-26
**Status:** Draft

---

## 1. Purpose

Expose `DeviceProvider` operations as MCP tools so LLM agents can interact
with IoT devices. Three tools: `iot_get_devices`, `iot_get_state`,
`iot_send_command`. Follows the `quarkus-mcp-server-http` library pattern
established by `connectors/mcp`.

Prerequisite for casehubio/life#60 (OpenClaw skill integration) — the IoT
row in the SPI bank (§2.3 of that spec).

---

## 2. Module

New module `mcp/` → artifact `casehub-iot-mcp`.

**Dependencies:**

| Scope | Artifact | Why |
|-------|----------|-----|
| compile | `casehub-iot-api` | `DeviceRegistry`, `DeviceProvider`, `DeviceEntity`, `DeviceCommand`, `CommandResult` |
| compile | `quarkus-mcp-server-core` (1.11.1) | `@Tool`, `@ToolArg` annotations |
| test | `casehub-iot-testing` | `MockDeviceProvider`, `MockDeviceRegistry` |
| test | `junit-jupiter` | |
| test | `assertj-core` | |

**Build plugins:** Jandex (`jandex-maven-plugin`) — required for CDI bean
discovery when consumed as a library JAR.

**Not included:** `quarkus-mcp-server-http`. The host app adds the HTTP
transport. This module is a library, not a standalone app.

**Parent POM changes:** Add `casehub-iot-mcp` to `<modules>` and
`<dependencyManagement>`. Add `quarkus-mcp-server.version` property (1.11.1).

---

## 3. Host-Agnostic Design

The MCP tool bean injects `DeviceRegistry` and `Instance<DeviceProvider>`.
Whatever providers are configured in the host app are what the tools see:

- **Bridge app** → tools see local HA/OpenHAB devices
- **Webapp** → tools see remote devices via `BridgeDeviceProvider`
- **Both** → federated view

No host-specific code in this module. Provider activation is controlled by
`@LookupIfProperty` on each provider (existing pattern).

---

## 4. Tool Specifications

### 4.1 `iot_get_devices`

List all devices with optional filters.

**Tool description:** `"List IoT devices with optional filters. Returns device ID, class, label, location, provider, and availability. Use iot_get_state for full device state."`

**Arguments:**

| Name | Required | Description annotation |
|------|----------|-------------|
| `deviceClass` | no | `"Filter by device class. Valid values: SWITCH, LIGHT, THERMOSTAT, SENSOR, PRESENCE_SENSOR, POWER_SENSOR, LOCK, COVER, MEDIA_PLAYER, FAN, CAMERA. Case-insensitive."` |
| `providerId` | no | `"Filter by provider ID (e.g. 'homeassistant', 'openhab')."` |
| `available` | no | `"Filter by availability: true for online devices, false for offline."` |

**`available` type:** `Boolean` — declared as a native boolean parameter.
The MCP schema becomes `"type": "boolean"` and the LLM produces JSON
`true`/`false` directly. Null means no filter (all devices returned).
Consistent with `DeviceResource.list()` which uses `@QueryParam Boolean`.

**Implementation:** `DeviceRegistry.findAll()` → validate `deviceClass`
if provided (case-insensitive `DeviceClass.valueOf()`) → filter → project
to summary format.

**`deviceClass` validation:** If provided and not a valid `DeviceClass`
enum member (case-insensitive), return:
`"Failed: Unknown device class: <value>. Valid values: SWITCH, LIGHT, THERMOSTAT, SENSOR, PRESENCE_SENSOR, POWER_SENSOR, LOCK, COVER, MEDIA_PLAYER, FAN, CAMERA"`

**Return format:** JSON array of summary objects. Each object contains
discovery-level fields only — full typed state is available via
`iot_get_state`:

```json
[
  {
    "deviceId": "light.living_room",
    "deviceClass": "LIGHT",
    "label": "Living Room Light",
    "providerId": "homeassistant",
    "location": "Living Room",
    "available": true,
    "lastUpdated": "2026-07-26T10:30:00Z"
  }
]
```

This separates device discovery from state inspection, keeping list
responses token-efficient for LLM context windows. `lastUpdated` is
included so agents can assess data freshness without calling
`iot_get_state` on every device.

**Error:** Empty array `[]` when no devices match (not an error condition).

### 4.2 `iot_get_state`

Get current state for a single device.

**Tool description:** `"Get current state for a specific IoT device. Returns the full device state including typed fields (temperature, humidity, mode, etc.) and availability."`

**Arguments:**

| Name | Required | Description annotation |
|------|----------|-------------|
| `deviceId` | yes | `"The device ID to query (e.g. 'light.living_room', 'sensor.outdoor_temp'). Use iot_get_devices to discover available device IDs."` |

**Implementation:** `DeviceRegistry.findById(deviceId)` → JSON serialize.

**Return format:** JSON object — full Jackson-serialized `DeviceEntity`
subclass. Typed fields (temperature, humidity, mode, etc.) are flat on each
device subclass per the existing `@JsonAutoDetect` configuration.

**Error:** `"Device not found: <deviceId>"` when device ID doesn't exist
in the registry.

### 4.3 `iot_send_command`

Send a command to a device.

**Tool description:** `"Send a command to an IoT device. Supports actions: turn_on, turn_off, set_temperature, lock, unlock, set_position, set_volume. Returns confirmation with correlation ID on success."`

**Arguments:**

| Name | Required | Description annotation |
|------|----------|-------------|
| `deviceId` | yes | `"Target device ID (e.g. 'light.living_room'). Use iot_get_devices to find available devices."` |
| `action` | yes | `"Command action: turn_on, turn_off, set_temperature, lock, unlock, set_position, set_volume."` |
| `parameters` | no | `"Command parameters (e.g. {\"temperature\": 22.0, \"unit\": \"CELSIUS\"} for set_temperature, {\"position\": 50} for set_position, {\"volume\": 75} for set_volume). Not needed for turn_on, turn_off, lock, unlock."` |

**`parameters` type:** `Map<String, Object>` — declared as a native map
parameter, not a JSON string. The Quarkus MCP framework handles
deserialization via `FeatureManagerBase.handleParam()` which uses
`ObjectMapper.convertValue()` for Map arguments. This avoids
double-encoding (the LLM would otherwise have to produce a JSON string
containing escaped JSON inside the already-JSON tool call).

**Implementation:**
1. `DeviceRegistry.findById(deviceId)` — validate device exists
2. Find `DeviceProvider` matching `device.providerId()` from `Instance<DeviceProvider>`
3. Construct `DeviceCommand` with `dispatchedBy = "mcp-agent"`,
   auto-generated `correlationId`
4. `provider.dispatch(command).await().atMost(Duration.ofSeconds(30))`

**Return format:** Human-readable confirmation string:
`"Command <action> sent to <deviceId> (result=SENT, correlationId=<id>)"`

**Errors:**
- Device not found: `"Failed: Device not found: <deviceId>"`
- Provider not found: `"Failed: Provider not found: <providerId>"`
- Dispatch exception: `"Failed: <exception message>"`
- Dispatch timeout (30s safety net): `"Failed: Command timed out after 30s (correlationId=<id>)"`
- Non-SENT result: `"Command <action> to <deviceId> result: <result> (correlationId=<id>)"` where `<result>` is the `CommandResult` enum value (`FAILED` or `TIMEOUT`). The distinction matters for agent retry decisions: FAILED means the command was rejected (retry likely futile), TIMEOUT means confirmation wasn't received (retry risks double-execution).

**`dispatchedBy`:** Hardcoded to `"mcp-agent"`. Principal propagation is a
cross-repo prerequisite (life#60 §8) — when delivered, this becomes the
propagated identity. No RBAC enforcement in this module; the host app's
security layer handles that.

---

## 5. Implementation

### 5.1 Class: `IoTDeviceMcpTool`

Single `@ApplicationScoped` bean. Three `@Tool`-annotated methods, all
`@Blocking` (registry reads are synchronous, dispatch blocks on `Uni`).

Constructor-injected: `DeviceRegistry`, `Instance<DeviceProvider>`,
Jackson `ObjectMapper`.

**Serialization:**
- `iot_get_state` — `ObjectMapper` serializes the full `DeviceEntity`
  subclass. The existing `@JsonTypeInfo`/`@JsonAutoDetect` configuration
  on `DeviceEntity` controls the output format.
- `iot_get_devices` — projects each `DeviceEntity` to a `DeviceSummary`
  record (defined in the `mcp/` module), then serializes the list via
  `ObjectMapper`. `DeviceSummary` is a simple projection record:
  `DeviceSummary(String deviceId, String deviceClass, String label,
  String providerId, String location, boolean available, Instant lastUpdated)`.
  Constructed from `DeviceEntity` fields — no Jackson polymorphism needed.

**Dispatch timeout:** `iot_send_command` uses `await().atMost(Duration.ofSeconds(30))`
as a safety net for the MCP thread pool. Providers may complete faster with
their own timeouts (e.g., HomeAssistantProvider's 10s REST read timeout).
If no provider-level timeout fires first, the 30s limit ensures the MCP
worker thread is released. `TimeoutException` is caught and returns
`"Failed: Command timed out after 30s (correlationId=<id>)"`.

**Logging:** All failure paths log at WARN level before returning the
fail-soft string, consistent with `ChatPlatformMcpTool` in connectors/mcp.
Format: `LOG.warnf("<tool> failed [%s]: %s", exceptionClass, message)`.
This ensures operators can diagnose provider issues from server logs
without relying on agent conversation transcripts.

### 5.2 Dispatch Routing

The registry-lookup → provider-match → dispatch pattern is the same as
`DeviceResource` and `DeviceCommandWorkerFunction`. Kept inline rather
than extracted to a shared service — the routing is 5 lines of
straightforward lookups, and each consumer has meaningfully different
surrounding concerns (RBAC, worker API, MCP error handling).

### 5.3 Parameter Handling

`parameters` is declared as `@ToolArg Map<String, Object>` — the Quarkus
MCP framework deserializes the JSON object natively via
`FeatureManagerBase.handleParam()` (Jackson `ObjectMapper.convertValue()`).
No manual parsing or dedicated error path needed.

If `parameters` is null (omitted by the LLM): `Map.of()` (empty map),
matching `DeviceCommand`'s null-coalescing behavior.

---

## 6. Testing

Unit tests only — no `@QuarkusTest`. The tool bean takes constructor
injection, so tests instantiate it directly with `MockDeviceRegistry` and
`MockDeviceProvider`.

**Test class:** `IoTDeviceMcpToolTest`

| Test | What it verifies |
|------|-----------------|
| `getDevicesReturnsAllDevices` | JSON array with all registered devices |
| `getDevicesFiltersByDeviceClass` | Only matching class returned |
| `getDevicesFiltersByProviderId` | Only matching provider returned |
| `getDevicesFiltersByAvailability` | Only available/unavailable returned |
| `getDevicesReturnsEmptyArrayWhenNoneMatch` | `[]` not an error |
| `getStateReturnsDeviceJson` | Full serialized device with typed fields |
| `getStateReturnsErrorForUnknownDevice` | `"Device not found: ..."` |
| `sendCommandDispatchesToCorrectProvider` | Command reaches right provider |
| `sendCommandReturnsConfirmation` | Success message with correlationId |
| `sendCommandFailsForUnknownDevice` | `"Failed: Device not found: ..."` |
| `sendCommandFailsForUnknownProvider` | `"Failed: Provider not found: ..."` |
| `sendCommandHandlesDispatchFailure` | Exception → `"Failed: ..."` |
| `sendCommandPassesParametersMap` | Map argument → DeviceCommand parameters |
| `sendCommandHandlesNullParameters` | null → empty Map |
| `sendCommandReportsFailedResult` | FAILED result → `"result: FAILED"` message |
| `sendCommandReportsTimeoutResult` | TIMEOUT result → `"result: TIMEOUT"` message |
| `sendCommandTimesOutAfter30Seconds` | Safety timeout returns `"Failed: Command timed out..."` |
| `getDevicesReturnsErrorForInvalidDeviceClass` | Invalid enum → error with valid values |
| `getDevicesMatchesDeviceClassCaseInsensitively` | "light" matches LIGHT |
| `getDevicesReturnsSummaryFormat` | Response contains discovery fields only, no typed state |

---

## 7. Consumption

Any Quarkus app that wants IoT MCP tools adds two dependencies:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-iot-mcp</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkiverse.mcp</groupId>
    <artifactId>quarkus-mcp-server-http</artifactId>
</dependency>
```

The `@Tool`-annotated beans are discovered via Jandex and auto-registered
at the `/mcp` endpoint. No application code needed.

---

## 8. Not In Scope

- RBAC / `@RolesAllowed`, tenancy filtering, `dispatchedBy` principal propagation — **iot#74** (blocked on life#60 §8)
- Audit / ledger integration for tool commands — **iot#75**
- Device state history queries via MCP — **iot#76** (JPA-dependent)
- WebSocket/SSE streaming of state changes via MCP — **iot#77** (future protocol capability)
