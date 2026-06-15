# Design Journal — issue-13-code-review-and-cleanup

### 2026-06-15 · §9.4·L4 · #11

OpenHAB discovery is now dual-layered: Equipment-based (semantic model) and Thing-based always run in parallel and merge. Equipment-mapped devices retain full semantic-tag resolution; Thing-only devices get basic mapping from channel `itemType` metadata. This makes the provider usable without semantic model configuration while preserving rich mapping when the model exists.

Thing status SSE events (`ThingStatusInfoChangedEvent`) provide real-time availability for all devices — both Equipment-backed (Thing OFFLINE overrides item-derived availability) and Thing-only. The SSE client subscribes to both `openhab/items/*/statechanged` and `openhab/things/*/statuschanged` on a single connection.

The `thingDiscoveryEnabled` config flag controls only Thing *mapping* (whether unmapped Things produce DeviceEntities), not Thing *fetching* — Things are always fetched for the `thingToEquipment` index and availability override.

### 2026-06-15 · §10 · #11

Introduced `ResolvedDeviceFields` + `OpenHabDeviceBuilder` to eliminate mapper duplication. The variation between Equipment-based and Thing-based mapping is *resolution* (how you find the field value from semantic tags vs channel itemType), not *construction* (how you build the DeviceEntity). Both resolution strategies produce a `ResolvedDeviceFields` record; `OpenHabDeviceBuilder.build()` is the single construction path. This also fixes a structural inconsistency where `resolveDeviceClass()` returned SENSOR but `mapSensor()` built entities with POWER_SENSOR or PRESENCE_SENSOR — DeviceClass refinement now happens in the resolution strategy, and `ResolvedDeviceFields.deviceClass` carries the final class.

### 2026-06-14 · §9.4·L4 · #13

C4 code review items: HVAC equipment with a `Control+Switch` member now overrides thermostat mode to OFF when the switch is not ON (powered down). Equipment availability changed from "any member NULL → unavailable" to "all members NULL → unavailable" — models OpenHAB's Thing-level offline correctly. Duplicate `tagSet()` utility consolidated to `OpenHabItemDto.tagSet()`. `buildCommandValue` exposed for testing via `OpenHabCommandDispatchTest`.
