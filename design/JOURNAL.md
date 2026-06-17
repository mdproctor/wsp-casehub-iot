# Design Journal — issue-5-bridge-runtime

## §Bridge Architecture

**Two-sided tunnel via DeviceProvider SPI** — the core architectural decision. The bridge ships both a local agent (`casehub-iot-bridge`) and a cloud-side `BridgeDeviceProvider` (`casehub-iot-bridge-server`). Cloud consumers add bridge-server as a dependency and treat remote devices identically to local ones. The hexagonal architecture works: `HomeAutomationEventObserver @ObservesAsync StateChangeEvent` fires the same whether the event came from a local HA provider or a remote bridge.

**Hybrid is deployment topology, not a bridge feature.** The original issue spec had `local-automations`/`cloud-automations` config. Replaced with CDI classpath extension — add application-tier logic to the bridge's classpath, it observes the same CDI events. No routing config.

## §Serialization

**Compound type ID** — `"{DeviceClass}:{ClassName}"` (e.g., `"THERMOSTAT:HomeAssistantThermostat"`). Chosen because `deviceClass` alone can't distinguish supplements from common types. `DeserializationProblemHandler.handleUnknownTypeId()` has no JSON field access — fallback must be self-contained in the type ID. Three code review iterations to get this right.

**L1 boundary change** — Jackson `@JsonTypeInfo` added to `DeviceEntity` in iot-api. L1 was "zero framework dependency." Now includes Jackson annotations. Deliberate — serialization is a first-class concern for types crossing process boundaries.

## §Server-Side Namespacing

Device IDs namespaced server-side (not agent-side) via Jackson tree copy. `DeviceIdNamespacer` uses `valueToTree` → modify `ObjectNode` → `treeToValue`. Avoids `toBuilder()` type-slicing trap on supplementable types. Agent sends raw local IDs — wire format is debuggable.

## §Snapshot-Only Reconnection

No event buffer. On reconnect, send `STATE_SNAPSHOT` from `discover()`. Replaying buffered events risks ghost automations (presence event from 30s ago triggers security case for past arrival). Durable store-and-forward deferred to #20.
