# Handover — issue-111-simulation-runtime

## Branch

`issue-111-simulation-runtime` on both project and workspace.

## What's Done

The IoT simulation framework is delivered (#110):
- `@SimulationEligible(name = "device-provider")` on `DeviceProvider` — APT generates decorator
- `IoTSimulationBeans` — `TemporalDriverFactory<StateChangeEvent>` CDI producer wired to `Event<StateChangeEvent>.fireAsync()` + `SimulationRuntime`
- 4 temporal profiles in YAML: morning-routine, emergency, quiet-night, full-demo (composite via sequence refs)
- Scenario YAML stub: `webapp/src/main/resources/scenarios/morning-routine.yaml`

Platform dependencies delivered:
- platform#371 — `TemporalSimulationDriver`, `TimedSequence`, `TemporalProfile`, `TemporalProfileRegistry`, `TemporalDriverFactory`
- platform#372 — `TemporalDriverService @McpDomain` control API (start/stop/speed via scenario steps)

## Queue

3 issues in order:

1. **#111** (S/Med) — Wire temporal profiles to webapp REST endpoint. Add `POST /api/simulation/start?profile=morning-routine&speed=10` that injects `TemporalProfileRegistry` + `TemporalDriverFactory<StateChangeEvent>`, resolves the named profile, starts the driver. KPI cards, MCP resources, and RAS situation detection light up with simulated activity. **Console becomes demo-ready.**

2. **#112** (M/Med) — Pages scenario integration. Complete the `morning-routine.yaml` scenario stub with `delivery: 'graphql'` steps calling `TemporalDriverService` to start profiles, `delivery: 'aria'` steps for UI verification (spotlight KPI cards, assert situations). Create additional scenario files (emergency, full-demo). Wire scenario loading in the webapp. **End-to-end guided demos.** Depends on platform#372 (delivered).

3. **#113** (S/Low) — Seed `SimulationCorpus` for `DeviceProvider` APT decorator. Create `IoTCorpusSeed` loading `standard-home.yaml` fixtures into corpus so `discover()` and `dispatch()` resolve from simulation in integration tests. Replace hand-rolled `MockDeviceProvider` usage. **Platform simulation pattern for test infrastructure.**

## Key Context

- Temporal profiles fire `StateChangeEvent` directly (domain-typed path) OR `CloudEvent` (platform pipeline path). Both are available. Decision D3 from the spec chose dual factories.
- `TemporalProfileRegistry.resolve("morning-routine", StateChangeEvent.class, mapper)` does typed conversion from `Map<String, Object>` to domain types in one call.
- Profile composition via `sequence: [{ ref: morning-routine }, { ref: emergency }]` — no code needed for composite scenarios.
- `simulation-starter` is the transitive dependency that pulls in everything needed.
- Platform `TemporalDriverService @McpDomain` (platform#372) provides start/stop/speed as `delivery: 'graphql'` scenario steps — the control plane for Pages scenarios.

## Platform Dependencies

Install latest platform SNAPSHOT before starting:
```bash
mvn --batch-mode -f /path/to/platform/pom.xml install -DskipTests -q -o
```

## What's Next

| # | Title | Scale | Complexity | Blocked by |
|---|-------|-------|------------|------------|
| 111 | REST endpoint for simulation control | S | Med | — |
| 112 | Pages scenario integration | M | Med | — (platform#372 delivered) |
| 113 | Corpus seeding for DeviceProvider | S | Low | — |
