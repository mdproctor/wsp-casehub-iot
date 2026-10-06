# Handover — casehub-iot

## Last Session

Implemented #129 — scenario-topology binding (landed as c735119..c56e306 on main). CDI event bridge connects scenario step execution to topology dual-view. `ScenarioBindingEvent` sealed interface with 6 typed variants fired synchronously from `DesiredStateDeliveryHandler` at lifecycle points with device-level deduplication. `ScenarioTopologyBinder` CDI observer bridges to extended topology SSE stream with binding-start/update/complete/clear operations. Frontend: `applyBindingEvent` pure-function state machine, zone aggregation badges in tree view, node glow animations and summary header in graph view.

Design review (Standard) caught that push WebSocket was wrong transport — SSE stream avoids stale replay from EventStore persistence. Spec review (Standard) surfaced dual-node deduplication requirement and tenancy gap.

Filed #131 (command binding with execution context, M/High), #132 (topology SSE tenancy filtering, S/Med). Updated epic #121.

## Recommended Queue Order

1. **#132** — Topology SSE tenancy filtering (S/Med). Quick win — fixes known broadcast tenancy leak documented in ARC42STORIES. Standalone.
2. **#128** — Evaluate fsitrading playbook UI (S/Low). Investigation task, not implementation. Informs whether the playbook UI direction is viable.
3. **#131** — Command binding with execution context (M/High). Builds on #129's architecture — needs plugin SPI changes to thread executionId through `IoTCommandPlugin`.
4. **#130** — Reactive trigger → drift override (M/High). Largest remaining epic item. Tackle last when smaller items are cleared.

## Known Issues

- **webapp-api build failure** — pre-existing compilation error in webapp-api module. Does not affect scenario, desiredstate, or other modules.
- **webapp-api test compilation** — `WorkItemOutcomeRecorderTest` and `WorkItemPredictionServiceTest` have pre-existing compile errors. Excluded via `<testExcludes>`.
- **QuarkusTest in provider modules** — pre-existing `UnsatisfiedResolutionException: SimulationRuntime`. Unit tests run fine.

## References

- Design spec for #129: `docs/specs/issue-129-scenario-topology-binding/2026-10-06-scenario-topology-binding-design.md`
- Epic issue: casehubio/iot#121 (remaining: #128, #130, #131, #132)
