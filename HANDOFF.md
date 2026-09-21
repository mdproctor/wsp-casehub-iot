# Handover — issue-111-simulation-runtime

## Branch

`issue-111-simulation-runtime` on both project and workspace.

## What's Done

**#111 — REST endpoint for simulation control (complete):**
- `DefaultIoTSimulationApi` `@McpDomain("iot/simulation")` — tri-channel REST/GraphQL/MCP surface
  - `start(profile, speed)` — resolves temporal profile, creates driver, fires `StateChangeEvent` via CDI
  - `stop()`, `setSpeed(speed)`, `status()`, `profiles()`
- `SimulationPayloadConverter` — converts `Map<String, Object>` temporal profile payloads to `StateChangeEvent` for PRESENCE_SENSOR, LIGHT, THERMOSTAT, SENSOR device classes
- Dependencies: `simulation-core`, `simulation-config-core`, `simulation-config` (runtime) added to webapp
- Jandex `index-dependency` for `simulation-config` CDI bean discovery
- `casehub.simulation.config=classpath:simulation/temporal-profiles.yaml` in application.properties
- 15 unit tests (7 converter, 8 API) — all green. Full webapp suite: 90 tests, 0 failures

**Pre-existing fixes (landed on this branch):**
- Adapted to upstream neocortex CBR renames: `CbrCaseMemoryStore` → `CbrRecordStore`, `FeatureVectorCbrCase` → `CbrFeatureRecord`, `ScoredCbrCase` → `CbrMatch`, `CbrCase` → `CbrRecord`, `CbrFeatureSchema` → `CbrRecordSchema`
- Added `@HandWrittenEndpoint` to `DeviceSseResource`, `KpiResource`, `WorkItemResource`

## Queue

3 issues, position 1/3:

1. ~~**#111**~~ (S/Med) — REST endpoint for simulation control. **Done.**

2. **#112** (M/Med) — Pages scenario integration. Complete `morning-routine.yaml` scenario stub with `delivery: 'graphql'` steps calling `TemporalDriverService` to start profiles, `delivery: 'aria'` steps for UI verification. Create additional scenario files (emergency, full-demo). Wire scenario loading in the webapp. **Next up.**

3. **#113** (S/Low) — Seed `SimulationCorpus` for `DeviceProvider` APT decorator.

## Key Context

- `DefaultIoTSimulationApi` manages a single active `TemporalSimulationDriver<Map<String, Object>>` — the event sink converts map payloads to `StateChangeEvent` and fires via CDI `fireAsync()`. This triggers `StateChangeHistoryObserver` (JPA persistence), `DeviceSseResource` (SSE push), and `IoTStateChangeResourceObserver` (MCP resource updates).
- The `SimulationPayloadConverter` builds synthetic `DeviceEntity` subclasses from the simplified YAML payloads. `before` is null; `changedCapabilities` is all capability keys from the device.
- Platform `SimulationConfigBeans` auto-discovers `simulation/temporal-profiles.yaml` via `casehub.simulation.config` property and produces `TemporalProfileRegistry` + `SimulationRuntime` CDI beans.
- Profile composition via `sequence: [{ ref: morning-routine }, { ref: emergency }]` — the `full-demo` profile chains all others.

## Platform Dependencies

Install latest platform SNAPSHOT before starting:
```bash
mvn --batch-mode -f /path/to/platform/pom.xml install -DskipTests -q -o
```

## What's Next

| # | Title | Scale | Complexity | Blocked by |
|---|-------|-------|------------|------------|
| 112 | Pages scenario integration | M | Med | — (platform#372 delivered) |
| 113 | Corpus seeding for DeviceProvider | S | Low | — |
