---
title: "Simulation as a First-Class Citizen"
entry_type: note
subtype: diary
author: mdp
projects: [casehubio/iot]
series: issue-111-simulation-runtime
tags: [simulation, scenario, temporal-profiles, corpus-seeding, pages-aria]
date: 2026-09-22
---

# Simulation as a First-Class Citizen

The IoT operational console can now simulate a full day of device activity — motion sensors triggering at dawn, thermostats adjusting, smoke alarms cascading — all driven by temporal profiles that replay at configurable speed. Three issues landed on one branch, each building on the last.

## The Simulation API

I wanted a single entry point that could be called from REST, GraphQL, and MCP without three separate implementations. The platform's `@McpDomain` annotation already provides tri-channel routing, so `DefaultIoTSimulationApi` became a thin orchestrator: resolve a named temporal profile from the registry, create a `TemporalSimulationDriver` with an event sink that converts map payloads to typed `StateChangeEvent` instances, and fire them through CDI's async event bus.

The interesting design choice was the `SimulationPayloadConverter`. Temporal profiles define events as flat maps — `{ deviceId: "thermostat-living-1", targetTemperature: { value: 22, unit: CELSIUS } }` — because the profile format needs to be simple enough for non-developers to author. The converter rebuilds those maps into the full typed device hierarchy: `PresenceSensor`, `LightDevice`, `ThermostatDevice`, `SensorDevice`. Each conversion produces a `StateChangeEvent` that the rest of the system can't distinguish from a real provider event — the `StateChangeHistoryObserver` persists it, SSE pushes it to the browser, and the RAS situation engine evaluates it. The simulation is invisible to downstream consumers.

A `ReentrantLock` guards the single-driver-at-a-time constraint. Only one simulation can run — start rejects if another is active. This is deliberate: concurrent simulations would interleave events from different temporal narratives, making the output meaningless for demo purposes.

## Wiring Scenarios to the Live Console

The second piece was connecting the Pages scenario infrastructure. The frontend's `PagesScenarioController` expects a server-side orchestrator, a script library REST endpoint, and a WebSocket push channel. None of these existed in the IoT webapp.

The wiring required four new classes — a `SessionSender`, a connection registry, a WebSocket endpoint at `/push`, and a `ScenarioConfig` CDI producer — following the pattern already established in the AML app. The scenario YAML files go under `META-INF/scenarios/` where `BundledScriptSource` auto-discovers them.

Each scenario combines `delivery: 'graphql'` steps (to start and stop the simulation via the `iot/simulation` domain) with `delivery: 'aria'` steps (to spotlight UI panels and wait for device state changes to appear). The `morning-routine` scenario starts the temporal profile at 10× speed, then waits for each device state update to land in the UI before narrating what's happening. `emergency` does the same for a smoke-detection cascade. `full-demo` chains both plus a quiet-night phase using the composite temporal profile.

## Corpus Seeding — From Mocks to Platform Simulation

The third issue replaced the hand-rolled `MockDeviceProvider` pattern with the platform's simulation corpus infrastructure. `IoTCorpusSeed` creates two `CorpusSeed` instances — one for `device-provider.discover` (sequential strategy, returns the standard-home fixture devices) and one for `device-provider.dispatch` (key-lookup by `DeviceCommand.action()`). Known actions map to `CommandResult.SENT`; unknown actions map to `FAILED`.

The key extractor is `DeviceCommand::action` — the same command type always gets the same result regardless of which device it targets. This matches how the `@SimulationEligible` decorator intercepts SPI calls: the decorator resolves a strategy, the strategy looks up by key, and the corpus returns the seeded output.

## What This Opens Up

The simulation stack is now complete from profiles through to UI verification. A new scenario is a YAML file dropped into `META-INF/scenarios/` — no Java code required. The corpus seeder means integration tests can validate the full `DeviceProvider` decorator lifecycle without maintaining separate mock infrastructure. And the scenario controller's `delivery: 'aria'` steps make it possible to script end-to-end guided demos where the console plays through realistic sequences while explaining what's happening at each stage.
