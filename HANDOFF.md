# Handover — casehub-iot

## Last Session

### #130 — Reactive trigger → drift override binding (landed as 5e4bbd8)

Binds reactive trigger events (motion, manual, schedule) to drift exemptions with policy-governed revert. When a trigger fires, affected devices receive a temporary exemption from desired-state reconciliation that expires according to a declared revert condition.

**What was built:**
- `TriggerSource` enum (MOTION/MANUAL/SCHEDULE/AUTOMATION/UNKNOWN) in api module
- Provider-inferred trigger classification: HA context-based (user_id → MANUAL, parent_id → AUTOMATION, motion/occupancy device_class → MOTION), OH device-class-based (PRESENCE_SENSOR → MOTION)
- `OverrideRule` + `RevertConfig` records with preset YAML `overrides:` section parsing (import composition, last-wins merge)
- `TriggerDriftOverrideObserver` CDI observer grants exemptions via `ExemptionStore` on matching triggers
- `ActivePresetRegistry` tracks applied preset per tenancy, wired into `DesiredStateDeliveryHandler` and `DefaultIoTPresetApi`
- `DriftExemptionGrantedEvent`/`DriftExemptionRevokedEvent`/`DeviceDriftInfo` CDI events and REST records in webapp-api
- `IoTDriftPolicy.isHardConstrained()` — LOCK/CAMERA never overridable
- Code review caught NodeId keying bug (bare deviceId vs deviceId + "-config" suffix). Garden entry: GE-20261008-7cd3c1

**Deferred:** Frontend drift badge overlay → casehubio/casehub-pages#532 (S/Low). topology-viewer is a casehub-pages package.

### #133 — Wire PagesPlaybookController as IoT Scenarios page (landed as 8537204)

Registered a Scenarios page in the IoT Console sidebar. Wired `PagesPlaybookController` (step outline, transport controls, library browser) with `PagesPlaybookYamlViewer` as companion panel. Added `@casehubio/pages-aria` portal resolution.

### Epic #121 — Closed

All 15 items landed. Epic covered: desiredstate module, topology visualization, presets, ordering constraints, drift policy, scenario delivery, scenario-topology binding, playbook UI evaluation, trigger→drift override binding, playbook controller wiring.

## What's Next

1. **#134** — Define IoT step catalog entries for yaml-core (S/Low). Catalog metadata for the scenario editor.
2. **#135** — Expose ScriptRegistry via REST for scenario CRUD (S/Med). REST + persistence.
3. **#136** — IoT preset composer — device picker UI (M/Med). Frontend-heavy.
4. **#137** — Preset diff visualization panel (S/Low). Frontend, builds on preset API.

## Known Issues

- **webapp-api build failure** — pre-existing compilation error in webapp-api module. Does not affect scenario, desiredstate, or other modules.
- **webapp-api test compilation** — `WorkItemOutcomeRecorderTest` and `WorkItemPredictionServiceTest` have pre-existing compile errors. Excluded via `<testExcludes>`.
- **QuarkusTest in provider modules** — pre-existing `UnsatisfiedResolutionException: SimulationRuntime`. Unit tests run fine.
- **topology-viewer tests** — pre-existing tsconfig.base.json issue prevents test runner from loading .casehub-packages/packages/topology-viewer. Not in vitest include glob.

## References

- Design spec for #130: `specs/issue-121-iot-desired-state-epic/2026-10-07-trigger-drift-override-design.md` (in workspace)
- Decisions D12-D15: `specs/issue-121-iot-desired-state-epic/decisions.md` (in workspace)
- Garden entry: GE-20261008-7cd3c1 (ExemptionStore NodeId keying convention)
- Deferred frontend: casehubio/casehub-pages#532
