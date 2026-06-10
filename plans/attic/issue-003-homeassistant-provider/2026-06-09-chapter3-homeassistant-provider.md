# C3: Home Assistant Provider — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the Home Assistant provider and fix C1 API design errors surfaced by real HA data.

**Architecture:** Section 0 corrects `api/` SPI to reactive (`Uni<>`), makes `PowerSensor` and `CoverDevice` fields optional, adds `DRY`/`CO` enum values. Section 1+ implements `homeassistant/` module: REST client for discovery/dispatch, WebSocket Next client for state subscription with application-level heartbeat, entity mapper for all HA domain→DeviceEntity conversions, three supplement types (HomeAssistantLight, HomeAssistantThermostat, HomeAssistantLock).

**Tech Stack:** Java 21, Quarkus 3.x, Mutiny (`Uni<>`), `quarkus-rest-client` + `quarkus-rest-client-jackson`, `quarkus-websockets-next`, JUnit 5, AssertJ, OkHttp MockWebServer.

**Spec:** `docs/superpowers/specs/2026-06-09-chapter3-homeassistant-provider-design.md` (rev 5)

**Issue:** casehubio/iot#3

---

## Task 1: DeviceProvider + DeviceRegistry — Reactive SPI Correction

Corrects the blocking SPI to reactive. This task touches `api/` only — the public contract.

**Files:**
- Modify: `api/pom.xml` — add Mutiny dep
- Modify: `api/src/main/java/io/casehub/iot/api/spi/DeviceProvider.java` — `Uni<>` returns
- Modify: `api/src/main/java/io/casehub/iot/api/spi/DeviceRegistry.java` — `Uni<Void> refresh()`
- Modify: `api/src/main/java/io/casehub/iot/spi/CdiDeviceRegistry.java` — reactive refresh + synchronized swap
- Modify: `testing/src/main/java/io/casehub/iot/testing/MockDeviceProvider.java` — `Uni<>` returns
- Modify: `api/src/test/java/io/casehub/iot/spi/CdiDeviceRegistryTest.java` — `.await().indefinitely()`
- Modify: `testing/src/test/java/io/casehub/iot/testing/MockDeviceProviderTest.java` — `Uni<>` assertions

- [ ] **Step 1:** Add `io.smallrye.reactive:mutiny` to `api/pom.xml` `<dependencies>`. No version — managed by parent BOM.

- [ ] **Step 2:** Update `DeviceProvider.java`: change `List<DeviceEntity> discover()` → `Uni<List<DeviceEntity>> discover()`; change `CommandResult dispatch(DeviceCommand command)` → `Uni<CommandResult> dispatch(DeviceCommand command)`. Add `import io.smallrye.mutiny.Uni`. `providerId()` and `status()` stay sync.

- [ ] **Step 3:** Update `DeviceRegistry.java`: change `void refresh()` → `Uni<Void> refresh()`. Add `import io.smallrye.mutiny.Uni`. Read methods stay sync.

- [ ] **Step 4:** Rewrite `CdiDeviceRegistry.refresh()` per spec §0.3: `Uni.join().all(...)` with per-provider `.onFailure().recoverWithItem(List.of())`, synchronized map swap inside `.map()`. Update `onStartup()` to `refresh().await().indefinitely()`. Add `import io.smallrye.mutiny.Uni` and other needed Mutiny imports. Add `private static final org.jboss.logging.Logger LOG = org.jboss.logging.Logger.getLogger(CdiDeviceRegistry.class)`.

- [ ] **Step 5:** Update `MockDeviceProvider.java` per spec §0.8: `discover()` → `Uni.createFrom().item(() -> List.copyOf(devices.values()))`, `dispatch()` → `Uni.createFrom().item(dispatchResult)`.

- [ ] **Step 6:** Fix all tests. `CdiDeviceRegistryTest`: every `registry.refresh()` call → `registry.refresh().await().indefinitely()`. `MockDeviceProviderTest`: assertions on `discover()` use `.await().indefinitely()` to get the list, then assert.

- [ ] **Step 7:** Run `mvn --batch-mode install -pl api,testing` — all tests must pass.

- [ ] **Step 8:** Commit: `refactor(api): correct DeviceProvider/DeviceRegistry SPI to reactive Uni<> #3`

---

## Task 2: PowerSensor, CoverDevice, ThermostatMode, SensorType — API Type Fixes

Fixes C1 type design errors surfaced by real HA data.

**Files:**
- Modify: `api/src/main/java/io/casehub/iot/api/PowerSensor.java` — Optional fields
- Modify: `api/src/main/java/io/casehub/iot/api/CoverDevice.java` — Optional position
- Modify: `api/src/main/java/io/casehub/iot/api/ThermostatMode.java` — add DRY
- Modify: `api/src/main/java/io/casehub/iot/api/SensorType.java` — add CO
- Modify: `testing/src/main/java/io/casehub/iot/testing/Fixtures.java` — solarPanel, bedroomBlinds
- Modify: `api/src/test/java/io/casehub/iot/api/EnumTest.java` — hasSize(6), hasSize(8)
- Modify: `api/src/test/java/io/casehub/iot/api/ExtensibleDeviceTest.java` — CoverDevice position Optional
- Test: relevant existing tests

- [ ] **Step 1: Write/update failing tests first.**
  - `EnumTest`: change `thermostatModeHasFiveValues` → `thermostatModeHasSixValues` with `hasSize(6)` and add `ThermostatMode.DRY` to `containsExactly`. Change `sensorTypeHasSevenValues` → `sensorTypeHasEightValues` with `hasSize(8)` and verify both `CO` and `CO2`.
  - Add a test in the existing PowerSensor test: `powerSensorBuildsWithPowerOnly` — builder sets `power(new BigDecimal("3200"))` but no `energy()` → asserts `power().isPresent()`, `energy().isEmpty()`.
  - Update any CoverDevice test: `device.position()` → `assertThat(device.position()).hasValue(75)`.

- [ ] **Step 2:** Run `mvn --batch-mode test -pl api` — confirm tests FAIL (enum counts wrong, PowerSensor NPE, CoverDevice type mismatch).

- [ ] **Step 3: Implement fixes.**
  - `ThermostatMode.java`: add `DRY` after `FAN_ONLY`.
  - `SensorType.java`: add `CO` after `CO2`.
  - `PowerSensor.java`: remove both `Objects.requireNonNull` from constructor. Change `public BigDecimal power()` → `public Optional<BigDecimal> power() { return Optional.ofNullable(power); }`. Same for `energy()`. Update `toBuilder()` accordingly.
  - `CoverDevice.java`: change `private final int position` → `private final Integer position`. Change `public int position()` → `public Optional<Integer> position() { return Optional.ofNullable(position); }`. In `AbstractBuilder`: change `int position` → `Integer position`; change `public B position(int position)` → `public B position(Integer position)`. Update `toBuilder()`.
  - `Fixtures.solarPanel()`: change `.power(BigDecimal.ZERO).energy(BigDecimal.ZERO)` → `.power(new BigDecimal("3200"))` (no `.energy()` call — defaults to null).
  - `Fixtures.bedroomBlinds()`: remove `.position(0)` or change to `.position(null)`.

- [ ] **Step 4:** Run `mvn --batch-mode test -pl api` — all tests pass.

- [ ] **Step 5:** Run `mvn --batch-mode install -pl api,testing` — full build passes.

- [ ] **Step 6:** Commit: `refactor(api): PowerSensor Optional fields, CoverDevice Optional position, ThermostatMode.DRY, SensorType.CO #3`

---

## Task 3: homeassistant/ Module Setup + Config + Internal DTOs

Creates the module scaffold, config mapping, and internal DTO records.

**Files:**
- Modify: `pom.xml` (root) — add `runtime` module position (N/A — no runtime/ in C3; just ensure homeassistant module order is correct)
- Modify: `homeassistant/pom.xml` — add deps
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantConfig.java`
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/internal/HaStateDto.java`
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/internal/HaServiceCallDto.java`
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/internal/ServiceCallSpec.java`
- Create: `homeassistant/src/main/resources/application.properties` — REST client bridge
- Test: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantConfigTest.java`

- [ ] **Step 1:** Update `homeassistant/pom.xml`: add `quarkus-rest-client`, `quarkus-rest-client-jackson`, `quarkus-websockets-next` as compile deps. Add `quarkus-junit` and `assertj-core` as test deps. Add `io.smallrye:jandex-maven-plugin` (3.3.1) per `library-jars-require-jandex` protocol.

- [ ] **Step 2:** Create `HomeAssistantConfig.java` — `@ConfigMapping(prefix = "casehub.iot.homeassistant")` interface with 7 methods per spec §2.1.

- [ ] **Step 3:** Create `internal/HaStateDto.java` — package-private record. Fields: `String entity_id`, `String state`, `String last_updated`, `String last_changed`, `Map<String, Object> attributes`. Jackson `@JsonIgnoreProperties(ignoreUnknown = true)`.

- [ ] **Step 4:** Create `internal/HaServiceCallDto.java` — package-private record. Fields: `String entity_id`, `Map<String, Object> service_data`. Jackson annotations as needed.

- [ ] **Step 5:** Create `internal/ServiceCallSpec.java` — package-private record. Fields: `String domain`, `String service`, `HaServiceCallDto body`.

- [ ] **Step 6:** Create `homeassistant/src/main/resources/application.properties`:
  ```properties
  quarkus.rest-client."homeassistant".url=${casehub.iot.homeassistant.url}
  quarkus.rest-client."homeassistant".connect-timeout=5000
  quarkus.rest-client."homeassistant".read-timeout=10000
  ```

- [ ] **Step 7:** Write `HomeAssistantConfigTest` — `@QuarkusTest` that injects `HomeAssistantConfig` and asserts default values for reconnect/ping/pong fields. Requires `src/test/resources/application.properties` with test url/token/tenancy-id values.

- [ ] **Step 8:** Run `mvn --batch-mode test -pl homeassistant` — config test passes.

- [ ] **Step 9:** Commit: `feat(homeassistant): module scaffold — config, internal DTOs, REST client bridge #3`

---

## Task 4: Supplement Types — HomeAssistantLight, HomeAssistantThermostat, HomeAssistantLock

TDD: write tests first, then implement the three supplement types.

**Files:**
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantLight.java`
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantThermostat.java`
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantLock.java`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantSupplementTest.java`

- [ ] **Step 1: Write failing tests.** `HomeAssistantSupplementTest` — plain JUnit 5 + AssertJ, no Quarkus:
  - `lightBuildsWithAllSupplementFields`: build `HomeAssistantLight` via builder, assert `rgbColor().isPresent()`, `effect().isPresent()`, `supportedColorModes()` not empty, `isInstanceOf(LightDevice.class)`.
  - `lightBuildsWithAbsentSupplementFields`: build without supplement fields, assert all `isEmpty()`.
  - `lightCapabilitiesIncludeSupplementFields`: build with supplement fields, call `capabilities()`, assert map contains keys `CAP_RGB_COLOR`, `CAP_EFFECT`, `CAP_SUPPORTED_COLOR_MODES` plus all inherited keys from `LightDevice`.
  - Same pattern for `HomeAssistantThermostat` (presetMode, swingMode, hvacAction) and `HomeAssistantLock` (changedBy, codeSlot).
  - `supplementDeriveChangedCapabilitiesDetectsSupplementFieldChange`: build two `HomeAssistantLight` instances differing only in `effect`, call `deriveChangedCapabilities`, assert `changedCapabilities` contains `"effect"`.

- [ ] **Step 2:** Run `mvn --batch-mode test -pl homeassistant` — FAIL (classes don't exist).

- [ ] **Step 3: Implement supplement types.** Each follows the `AbstractBuilder` pattern from `LightDevice`/`ThermostatDevice`/`LockDevice`:
  - `HomeAssistantLight extends LightDevice`: fields `int[] rgbColor`, `String effect`, `Set<String> supportedColorModes` (all nullable). CAP_ constants. `capabilities()` override calls `super.capabilities()` and adds three entries. `AbstractBuilder<HomeAssistantLight, Builder> extends LightDevice.AbstractBuilder<T, B>`. `toBuilder()`.
  - `HomeAssistantThermostat extends ThermostatDevice`: fields `String presetMode`, `String swingMode`, `String hvacAction`. CAP_ constants. `capabilities()` override. `AbstractBuilder`.
  - `HomeAssistantLock extends LockDevice`: fields `String changedBy`, `Integer codeSlot`. CAP_ constants. `capabilities()` override. Builder (LockDevice doesn't have AbstractBuilder — use direct extension of `LockDevice.Builder` if possible, or create one).

- [ ] **Step 4:** Run `mvn --batch-mode test -pl homeassistant` — all supplement tests pass.

- [ ] **Step 5:** Commit: `feat(homeassistant): supplement types — HomeAssistantLight, Thermostat, Lock with CAP_* and capabilities() #3`

---

## Task 5: HomeAssistantEntityMapper — Type Mapping

TDD: write mapper tests first against the full domain→type table, then implement.

**Files:**
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantEntityMapper.java`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantEntityMapperTest.java`

- [ ] **Step 1: Write failing tests.** `HomeAssistantEntityMapperTest` — plain JUnit 5 + AssertJ. Instantiate mapper with anonymous `HomeAssistantConfig` impl (all 7 methods, per spec §5). Build `HaStateDto` instances for each domain mapping and assert correct DeviceEntity type and field values:
  - `mapsSwitch`: entity_id `switch.hallway`, state `on` → `SwitchDevice`, `isOn()=true`
  - `mapsLight`: entity_id `light.kitchen`, state `on`, attributes: brightness=200, color_temp=370, rgb_color=[255,0,0], effect="rainbow", supported_color_modes=["hs","rgb"] → `HomeAssistantLight`, all fields populated
  - `mapsClimate`: entity_id `climate.living`, attributes: current_temperature=21.5, temperature=22.0, hvac_mode="heat", temperature_unit="°C", preset_mode="home" → `HomeAssistantThermostat`, temperature unit=CELSIUS
  - `mapsClimateFahrenheit`: temperature_unit="°F" → FAHRENHEIT
  - `mapsLock`: entity_id `lock.front`, state `locked`, attributes: changed_by="John", code_slot=3 → `HomeAssistantLock`
  - `mapsCover`: entity_id `cover.blinds`, state `open`, attributes: current_position=75 → `CoverDevice`, position=Optional.of(75)
  - `mapsCoverWithoutPosition`: no current_position attribute → position=Optional.empty()
  - `mapsMediaPlayer`: entity_id `media_player.speaker`, state `playing`, attributes: volume_level=0.65 → `MediaPlayerDevice`, playing=true, volume=Optional.of(65)
  - `mapsMediaPlayerPaused`: state `paused` → playing=false
  - `mapsFan`: entity_id `fan.bedroom`, state `on`, attributes: percentage=50 → `FanDevice`
  - `mapsPowerSensorPowerOnly`: entity_id `sensor.solar_power`, device_class=power, state="3200" → `PowerSensor`, power=3200, energy=empty
  - `mapsPowerSensorEnergyOnly`: device_class=energy, state="15.2" → power=empty, energy=15.2
  - `mapsPresenceSensor`: entity_id `binary_sensor.motion`, device_class=motion → `PresenceSensor`, present="on".equals(state), lastSeen from last_changed
  - `mapsSensorTemperature`: device_class=temperature → `SensorDevice`, sensorType=TEMPERATURE
  - `mapsSensorCO`: device_class=carbon_monoxide → sensorType=CO
  - `mapsSensorCO2`: device_class=carbon_dioxide → sensorType=CO2
  - `mapsBinarySensorNonPresence`: device_class=door → `SensorDevice`, binaryValue=true, numericValue=null
  - `skipsUnknownDomain`: entity_id `unknown.x` → `mapOne()` returns null
  - `unavailableEntityHasAvailableFalse`: state="unavailable" → available=false, numeric fields null
  - `commonFieldsPopulated`: assert deviceId=entity_id, label=friendly_name, lastUpdated from last_updated, tenancyId from config
  - `hvacModeDryMapsToDry`: hvac_mode="dry" → ThermostatMode.DRY
  - `hvacModeNullDefaultsToOff`: no hvac_mode attribute → ThermostatMode.OFF
  - `parseOrNullHandlesNonNumeric`: state="unavailable" for sensor → numericValue null (not NPE)

- [ ] **Step 2:** Run `mvn --batch-mode test -pl homeassistant` — FAIL (mapper doesn't exist).

- [ ] **Step 3: Implement `HomeAssistantEntityMapper`.** `@ApplicationScoped`, constructor injection per spec §3. `mapAll(List<HaStateDto>)` filters nulls from `mapOne()`. `mapOne(HaStateDto)` — big switch on domain prefix of entity_id. Uses `parseOrNull()` static helper. All attribute lookups null-safe via `Map.getOrDefault`.

- [ ] **Step 4:** Run `mvn --batch-mode test -pl homeassistant` — all mapper tests pass.

- [ ] **Step 5:** Commit: `feat(homeassistant): HomeAssistantEntityMapper — full domain-to-type mapping with 22+ test cases #3`

---

## Task 6: HomeAssistantRestClient

TDD: test the REST client with MockWebServer, then implement.

**Files:**
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantRestClient.java`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantRestClientTest.java`

- [ ] **Step 1:** Add `com.squareup.okhttp3:mockwebserver` as test dep in `homeassistant/pom.xml`.

- [ ] **Step 2: Write failing tests.** `HomeAssistantRestClientTest` — `@QuarkusTest` with `MockWebServer`:
  - `getStatesReturnsDeviceList`: enqueue JSON response matching HA `/api/states` shape, call `restClient.getStates().await().indefinitely()`, assert list size and first entity fields.
  - `callServiceReturnsResponse`: enqueue 200 response, call `restClient.callService("light", "turn_on", dto).await().indefinitely()`, assert response status 200.
  - `authorizationHeaderSent`: after a call, assert `mockServer.takeRequest().getHeader("Authorization")` starts with "Bearer ".

- [ ] **Step 3: Implement `HomeAssistantRestClient`.** `@RegisterRestClient(configKey = "homeassistant")` interface per spec §2.2. `@ClientHeaderParam` with `lookupToken()` default method using `ConfigProvider.getConfig()`.

- [ ] **Step 4:** Run `mvn --batch-mode test -pl homeassistant` — REST client tests pass.

- [ ] **Step 5:** Commit: `feat(homeassistant): HomeAssistantRestClient — REST client for /api/states and /api/services #3`

---

## Task 7: HomeAssistantWebSocketClient — Connection + Auth + State Machine

The core WebSocket client with auth handshake, state_changed processing, heartbeat, and reconnection.

**Files:**
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantWebSocketClient.java`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantWebSocketClientTest.java`

- [ ] **Step 1: Write failing tests.** `HomeAssistantWebSocketClientTest` — `@QuarkusTest` with OkHttp `MockWebServer` (WebSocket support). Test the state machine:
  - `authHandshakeCompletes`: server sends auth_required → client sends auth → server sends auth_ok → client sends subscribe → verify status transitions DISCONNECTED→CONNECTING→CONNECTED.
  - `authInvalidSchedulesReconnect`: server sends auth_invalid → verify status stays DISCONNECTED.
  - `stateChangedFiresCdiEvent`: server sends state_changed event with old_state and new_state → verify `Event<StateChangeEvent>` fired (inject a CDI observer in the test).
  - `nullOldStateUsesAllKeysAsChanged`: state_changed with null old_state → changedCapabilities = all keys of after.capabilities().
  - `pongResetsPongTimeout`: send ping, receive pong → verify no reconnect triggered.
  - `closedConnectionReconnects`: server closes → verify scheduleReconnect called (status becomes DISCONNECTED).
  - `unparsableMessageIgnored`: server sends garbage → no crash, no event.

- [ ] **Step 2:** Run `mvn --batch-mode test -pl homeassistant` — FAIL.

- [ ] **Step 3: Implement `HomeAssistantWebSocketClient`.** Full implementation per spec §2.3 — `@WebSocketClient`, `@ApplicationScoped`, all fields, `connect()` via `Instance<WebSocketConnector<>>`, `@OnOpen`, `@OnTextMessage` switch, `@OnClose`, `@OnError`, `@PreDestroy` shutdown sequence, heartbeat via `ScheduledExecutorService`, `sendHeartbeat()`, `handlePongTimeout()`, `cancelHeartbeat()`, `cancelPongTimeout()`, `scheduleReconnect()` with `AtomicInteger`.

- [ ] **Step 4:** Run `mvn --batch-mode test -pl homeassistant` — WebSocket tests pass.

- [ ] **Step 5:** Commit: `feat(homeassistant): WebSocket client — auth handshake, state_changed events, heartbeat, reconnection #3`

---

## Task 8: HomeAssistantProvider — Wiring Discovery + Dispatch

Wires the REST client, WebSocket client, and mapper into the `DeviceProvider` implementation.

**Files:**
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantProvider.java`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantProviderTest.java`

- [ ] **Step 1: Write failing tests.** `HomeAssistantProviderTest` — `@QuarkusTest` with MockWebServer:
  - `providerIdIsHomeassistant`: assert `providerId()` returns `"homeassistant"`.
  - `discoverMapsHaStatesToDeviceEntities`: mock `/api/states` response with multiple entity types, call `discover().await().indefinitely()`, assert correct types and count.
  - `dispatchTurnOnSendsCorrectServiceCall`: mock 200 response, dispatch a `DeviceCommand.turnOn(...)`, verify mock received `POST /api/services/light/turn_on` with correct body.
  - `dispatchReturnsFailedOnHttp500`: mock 500 response → `CommandResult.FAILED`.
  - `dispatchReturnsFailedOnUnknownAction`: unknown action → `CommandResult.FAILED`.
  - `dispatchSetVolumeConvertsToFloat`: dispatch `set_volume` with volume=65, verify body has `volume_level=0.65`.
  - `statusReturnsWebSocketStatus`: verify `status()` delegates to `wsClient.currentStatus()`.

- [ ] **Step 2:** Run tests — FAIL.

- [ ] **Step 3: Implement `HomeAssistantProvider`.** Per spec §2.4. `@ApplicationScoped`, `@Inject @RestClient` for REST client, `@Inject` WebSocket client, `@Inject` mapper. `@PostConstruct` connects WebSocket. `buildServiceCall()` with full switch table. `dispatch()` with `Uni<Response>` status check + `ProcessingException`/`TimeoutException` recovery.

- [ ] **Step 4:** Run `mvn --batch-mode test -pl homeassistant` — provider tests pass.

- [ ] **Step 5:** Commit: `feat(homeassistant): HomeAssistantProvider — discovery, dispatch, status wiring #3`

---

## Task 9: Full Build + ARC42STORIES + CLAUDE.md Updates

Integration verification and documentation sync.

**Files:**
- Modify: `ARC42STORIES.MD` — §2 Constraints, §12 Risks
- Modify: `CLAUDE.md` — Module Structure table (add homeassistant/ description update if needed)

- [ ] **Step 1:** Run `mvn --batch-mode install` — full multi-module build passes.

- [ ] **Step 2:** Update `ARC42STORIES.MD`:
  - §2 Constraints "Async model" row: change "Blocking SPIs; reactive variant deferred" → "Reactive SPIs (Uni<>) for discover/dispatch; corrected in C3"
  - §12: deviceId risk mitigation → "entity_id verbatim; renames require refresh(); accepted risk"
  - §9.3 C3 status → update from 🔲 to delivery metadata

- [ ] **Step 3:** Run code review (`superpowers:requesting-code-review`) before committing. Any finding Minor+ not fixed this session → file as GitHub issue.

- [ ] **Step 4:** Commit: `docs: sync ARC42STORIES.MD — C3 reactive SPI correction, deviceId accepted risk #3`

- [ ] **Step 5:** Run `implementation-doc-sync` after committing.

---

## Execution Order

Tasks MUST be executed in order. Each task depends on the prior:

```
Task 1 (reactive SPI) → Task 2 (type fixes) → Task 3 (module scaffold)
  → Task 4 (supplement types) → Task 5 (mapper)
  → Task 6 (REST client) → Task 7 (WebSocket client)
  → Task 8 (provider wiring) → Task 9 (full build + docs)
```

Tasks 4 and 5 can potentially run in parallel (supplement types are consumed by mapper but can be built independently). All others are strictly sequential.
