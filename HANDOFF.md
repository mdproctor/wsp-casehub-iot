# Handover — casehub-iot

## Last Session

### #134 — Define IoT step catalog entries for yaml-core (landed as aad4ac7)

Created `iot-steps.yaml` in `webapp/src/main/resources/steps/` with three step catalog entries for the `PagesActionCatalog` UI:
- `iot.command` — device command dispatch, MCP invoke binding to `iot_send_command`
- `iot.state` — device state query, MCP invoke binding to `iot_get_state`
- `iot.desired-state` — desired-state delivery handler, metadata-only (no invoke)

Test via `StepCatalogService` (not `DeclarationParser` — the latter throws on null invoke bindings). 9 tests covering parsing, invoke bindings, parameter types, required/optional constraints.

### Epic #140 — IoT Playbook Console (created, not started)

New epic to upgrade the "Scenarios" page to a full operational "Playbooks" console. Compose platform UI components into a `dockWorkbench()` layout for browsing, executing, and monitoring IoT playbooks against simulated devices.

**Decisions made:**
- D1: Operational console scope — no visual authoring (PagesBuilderShell) or CRUD (ScriptRegistry REST) in this epic
- D2: Presets (#136, #137) are a separate follow-on epic
- D3: Keep playbook controller variants, rename page from "Scenarios" to "Playbooks"

**Sub-issues (8):**
- #141 — Rename Scenarios → Playbooks + dockWorkbench layout (S/Low) — start here
- #142 — Wire PagesActionCatalog panel (S/Low)
- #143 — Wire PagesLibraryView panel (S/Low)
- #144 — Wire PagesPlaybookNarrative panel (XS/Low)
- #145 — Wire OrchestrationWorkbench panel (S/Med)
- #146 — Dataset bindings and status bar (XS/Low)
- #148 — Realistic device topology fixtures (M/Med) — apartments, houses, offices with HA/OH/hybrid providers
- #147 — End-to-end simulation verification (S/Med) — acceptance gate, blocked by all above

**Design spec:** `specs/iot-playbook-console-epic/2026-10-13-iot-playbook-console-design.md` (in workspace)

## What's Next

1. **#141** — Rename + dockWorkbench layout (S/Low). Unblocks all other panel wiring issues. Start here.
2. **#148** — Topology fixtures (M/Med). Can start in parallel with #141. Use internet research for realistic device layouts.
3. **#142–#146** — Panel wiring (all S or XS). Independent, any order after #141 lands.
4. **#147** — E2e verification (S/Med). Blocked by everything above. Acceptance gate for the epic.

## Known Issues

- **webapp-api build failure** — pre-existing compilation error in webapp-api module. Does not affect scenario, desiredstate, or other modules.
- **webapp-api test compilation** — `WorkItemOutcomeRecorderTest` and `WorkItemPredictionServiceTest` have pre-existing compile errors. Excluded via `<testExcludes>`.
- **QuarkusTest in provider modules** — pre-existing `UnsatisfiedResolutionException: SimulationRuntime`. Unit tests run fine.
- **topology-viewer tests** — pre-existing tsconfig.base.json issue prevents test runner from loading .casehub-packages/packages/topology-viewer. Not in vitest include glob.
- **DeclarationParser null invoke** — `DeclarationParser.parseInvoke()` throws on null. Use `StepCatalogService` for testing step definitions that include metadata-only entries (no invoke binding).

## References

- Design spec for #140: `specs/iot-playbook-console-epic/2026-10-13-iot-playbook-console-design.md` (in workspace)
- Decisions D1-D3: `specs/iot-playbook-console-epic/decisions.md` (in workspace)
- fsitrading ops-centre reference: `fsitrading/app/src/main/webui/src/ops-centre.ts`
- Step catalog entries: `webapp/src/main/resources/steps/iot-steps.yaml`
- Deferred frontend: casehubio/casehub-pages#532
