# IoT Device State Streaming via MCP Resources

**Issue:** casehubio/iot#77
**Date:** 2026-08-20
**Status:** Design
**Depends on:** casehubio/platform#241 (McpResourceRegistry SPI — delivered)

## Summary

Expose IoT device state changes as subscribable MCP resources through the platform's `McpResourceRegistry` SPI. LLM agents subscribe to per-device or global resources and receive push notifications when state changes. No new MCP module — the IoT `mcp` module registers resources through the platform infrastructure.

## Architecture

The platform's `McpResourceRegistry` SPI (platform#241) provides:
- `McpResourceDescriptor.template(name, uriTemplate, mimeType, description).withSubscribable(true)` — register a subscribable resource template
- `McpResourceHandle.notifyUpdate(uri)` — push `notifications/resources/updated` to subscribers
- `McpResourceRegistryBridge` — adapts to Quarkus `ResourceManager` with virtual-thread handlers

IoT contributes:
1. **`IoTResourceRegistrar`** — `@Startup` CDI bean that registers two resources via `McpResourceRegistry`
2. **`IoTStateChangeResourceObserver`** — CDI observer on `StateChangeEvent` that calls `handle.notifyUpdate()`

### Resources

| Resource | Type | URI | Subscribable | Content |
|---|---|---|---|---|
| Per-device state | Template | `iot://devices/{deviceId}/state` | Yes | Device state JSON (full DeviceEntity capabilities) |
| Global change feed | Static | `iot://devices/changes` | Yes | Last N state changes (ring buffer) |

### Data Flow

```
DeviceProvider → StateChangeEvent (CDI) → IoTStateChangeResourceObserver
                                            ├── handle.notifyUpdate("iot://devices/{deviceId}/state")
                                            └── handle.notifyUpdate("iot://devices/changes")
                                                    ↓
                                          McpResourceRegistryBridge
                                                    ↓
                                          ResourceInfo.sendUpdateAndForget()
                                                    ↓
                                          MCP client receives notifications/resources/updated
                                          Client re-reads resource → gets latest state
```

## Components

### `IoTResourceRegistrar` (new — in `casehub-iot-mcp`)

```java
@ApplicationScoped
public class IoTResourceRegistrar {

    @Inject McpResourceRegistry resourceRegistry;
    @Inject DeviceRegistry deviceRegistry;
    @Inject ObjectMapper objectMapper;
    @Inject McpIdentityContext identityContext;

    private McpResourceHandle deviceStateHandle;
    private McpResourceHandle changesHandle;

    @PostConstruct
    void register() {
        // Per-device state (template, subscribable)
        deviceStateHandle = resourceRegistry
            .newResource(TemplateResourceDescriptor.template(
                "iot-device-state",
                "iot://devices/{deviceId}/state",
                "application/json",
                "Current state of an IoT device including all capabilities")
                .withSubscribable(true))
            .handler(this::readDeviceState)
            .completion("deviceId", this::listDeviceIds)
            .register();

        // Global change feed (static, subscribable)
        changesHandle = resourceRegistry
            .newResource(StaticResourceDescriptor.of(
                "iot-device-changes",
                "iot://devices/changes",
                "application/json",
                "Recent device state changes across all providers")
                .withSubscribable(true))
            .handler(this::readChanges)
            .register();
    }

    McpResourceContent readDeviceState(McpResourceReadRequest request) throws Exception {
        String deviceId = request.templateArgs().get("deviceId");
        var device = deviceRegistry.findById(deviceId)
            .orElseThrow(() -> new IllegalArgumentException("Device not found: " + deviceId));
        // Tenancy filtering via McpIdentityContext (same pattern as IoTDeviceMcpTool)
        return McpResourceContent.of(request.uri(), objectMapper.writeValueAsString(device));
    }

    McpResourceContent readChanges(McpResourceReadRequest request) throws Exception {
        // Returns the ring buffer contents
        return McpResourceContent.of(request.uri(), objectMapper.writeValueAsString(recentChanges));
    }

    Supplier<List<String>> listDeviceIds() {
        return () -> deviceRegistry.findAll().stream()
            .map(DeviceEntity::deviceId)
            .toList();
    }
}
```

### `IoTStateChangeResourceObserver` (new — in `casehub-iot-mcp`)

```java
@ApplicationScoped
public class IoTStateChangeResourceObserver {

    @Inject IoTResourceRegistrar registrar;

    private final Deque<StateChangeSummary> recentChanges = new ConcurrentLinkedDeque<>();
    private static final int MAX_RECENT = 50;

    void onStateChange(@ObservesAsync StateChangeEvent event) {
        // Add to ring buffer
        recentChanges.addFirst(StateChangeSummary.from(event));
        while (recentChanges.size() > MAX_RECENT) recentChanges.removeLast();

        // Notify per-device subscribers
        String deviceId = event.after().deviceId();
        registrar.notifyDeviceUpdate(deviceId);

        // Notify global feed subscribers
        registrar.notifyChangesUpdate();
    }
}
```

### Dependency Changes

`casehub-iot-mcp/pom.xml` — add `casehub-platform-api` compile dependency (for `McpResourceRegistry` SPI, `McpResourceDescriptor`, etc.). This is the correct direction — IoT already depends on `casehub-platform-api` via `casehub-iot-api`.

## Test Plan

| Test | Assertion |
|------|-----------|
| `IoTResourceRegistrar` registers two resources | `register()` called twice on `McpResourceRegistry` — one template, one static |
| Per-device state read returns device JSON | Mock `DeviceRegistry` with test device → handler returns serialised `DeviceEntity` |
| Device not found returns error | Unknown deviceId → `IllegalArgumentException` |
| Template completion lists device IDs | Returns all device IDs from registry |
| State change fires per-device notification | `StateChangeEvent` → `handle.notifyUpdate("iot://devices/{id}/state")` called |
| State change fires global notification | Same event → `handle.notifyUpdate("iot://devices/changes")` called |
| Ring buffer bounded at 50 | 60 events → only 50 in recent changes |
| Changes feed returns recent events | Read handler returns ring buffer as JSON array |

## References

- casehubio/platform#241 — McpResourceRegistry SPI + McpResourceRegistryBridge
- `platform-api/src/main/java/io/casehub/platform/api/mcp/` — SPI types
- `mcp/src/main/java/io/casehub/iot/mcp/IoTDeviceMcpTool.java` — existing tool patterns
- `api/src/main/java/io/casehub/iot/api/StateChangeEvent.java` — CDI event to observe
- GE-20260806-0dadb3 — dual MCP transport (streamable HTTP + SSE)
- GE-20260804-52ba5f — SSE event dispatch semantics
