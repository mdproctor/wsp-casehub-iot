# IoT Playbook Console — Epic Design

**Epic:** IoT Playbook Console
**Status:** Design
**Scale:** L | **Complexity:** Med

## Summary

Upgrade the IoT "Scenarios" page to a full operational console called "Playbooks." Compose platform UI components (`dockWorkbench`, `PagesActionCatalog`, `PagesLibraryView`, `PagesPlaybookNarrative`, `OrchestrationWorkbench`) into a dockable multi-panel layout for browsing, executing, and monitoring IoT playbooks against simulated devices.

## Goal

An operator can open the Playbooks page, browse the step catalog and playbook library, select a playbook, run it against simulated devices, watch the narrative unfold, and monitor execution state — all in one integrated console without leaving the page.

## What's In

- Rename "Scenarios" → "Playbooks" across the frontend
- Upgrade layout from `rows(columns(...))` to `dockWorkbench()`
- Wire `PagesActionCatalog` as standalone panel (reads `iot-steps.yaml` via catalog service)
- Wire `PagesLibraryView` as standalone panel (reads bundled playbooks via `/playbook/library`)
- Wire `PagesPlaybookNarrative` for guided walkthroughs
- Wire `OrchestrationWorkbench` for execution monitoring
- Dataset bindings and status bar badges (simulation state, active playbook)
- End-to-end simulation verification against `MockDeviceProvider`

## What's Out

- Visual scenario authoring (`PagesBuilderShell`, `PagesCodeEditor`) — follow-on epic
- Scenario CRUD via ScriptRegistry REST (#135) — follow-on, gated by authoring
- Preset composer (#136) and preset diff panel (#137) — separate presets epic
- Changes to backend playbook execution runtime — already complete

## Architecture

### Composition Pattern

Follow the fsitrading ops-centre pattern:

```
page("Playbooks",
  dockWorkbench({
    storageKey: "iot-playbooks",
    centre:  [playbookController, yamlViewer],
    left:    { zones: 1, panels: [library] },
    right:   { zones: 2, panels: [stepCatalog, narrative] },
    bottom:  { zones: 1, panels: [orchestration] },
    statusBar: split("horizontal", [simulationBadge, playbookBadge]),
  }),
  { datasets, save }
)
```

Each panel uses `hostPanel("logical-name")` backed by `registerPanel()` in the entry point.

### Data Flow

```
Bundled playbooks (META-INF/playbooks/)
       │
       ▼
PagesLibraryView ──select──▶ PagesPlaybookController ──start──▶ /playbook/start
       │                              │                              │
       │                              ▼                              ▼
       │                     PagesPlaybookYamlViewer         PlaybookExecutor
       │                              │                         │        │
       │                              ▼                         ▼        ▼
       │                     PagesPlaybookNarrative    StateChangeEvent  BindingEvent
       │                                                    │              │
       ▼                                                    ▼              ▼
iot-steps.yaml ──▶ PagesActionCatalog          OrchestrationWorkbench (via /push WebSocket)
```

### Panel Inventory

| Panel | Component | Source | Data source |
|-------|-----------|--------|-------------|
| Playbook Controller | `PagesPlaybookController` | `pages-aria` | `/playbook/*` REST, `/push` WebSocket |
| YAML Viewer | `PagesPlaybookYamlViewer` | `pages-aria` | WebSocket `scenario:state` topic |
| Library | `PagesLibraryView` | `pages-aria` | `/playbook/library` REST |
| Step Catalog | `PagesActionCatalog` | `pages-aria` | `StepCatalogService` via GraphQL or REST |
| Narrative | `PagesPlaybookNarrative` | `pages-aria` | Inline `content:` from playbook YAML |
| Orchestration | `OrchestrationWorkbench` | `orchestration-workbench` | `/push` WebSocket |

### Simulation Stack

Already complete — no new backend work:

```
TemporalProfileRegistry → TemporalSimulationDriver → SimulationPayloadConverter
    → StateChangeEvent → DeviceRegistry → SSE + WebSocket push → UI panels
```

Profiles: `morning-routine`, `emergency`, `quiet-night`, `full-demo`. Controlled via `delivery: graphql` steps or MCP domain operations (`iot/simulation`).

### Naming Migration

| Current | New |
|---------|-----|
| `pages/scenarios.ts` | `pages/playbooks.ts` |
| `scenariosPage()` | `playbooksPage()` |
| Sidebar label "Scenarios" | "Playbooks" |
| Any `scenario` references in IoT webapp TS | `playbook` |

Backend stays unchanged — endpoints are already `/playbook/*`.

## Sub-Issues

### 1. Rename Scenarios → Playbooks + upgrade to dockWorkbench layout

**Scale:** S | **Complexity:** Low

Rename the page and replace `rows(columns(...))` with `dockWorkbench()`. Initial layout has centre (controller + YAML viewer) and empty slots for panels wired in subsequent issues. Wire `registerPanel()` calls and `datasets` bindings in `app.ts`.

**Blocked by:** nothing
**Blocks:** all subsequent issues (provides the dock layout)

### 2. Wire PagesActionCatalog panel

**Scale:** S | **Complexity:** Low

Register `PagesActionCatalog` as a dock panel. Connect to `StepCatalogService` via `RestCatalogSource` or `GraphqlCatalogSource`. Verify `iot-steps.yaml` entries appear with correct inputs/outputs, invoke bindings, and "Try It" panel working against MCP tools.

**Blocked by:** #1 (dock layout)
**Blocks:** nothing

### 3. Wire PagesLibraryView panel

**Scale:** S | **Complexity:** Low

Register `PagesLibraryView` as a dock panel. Connect to `/playbook/library` REST endpoint. Verify all 8 bundled playbooks appear. Select-to-load into the controller.

**Blocked by:** #1 (dock layout)
**Blocks:** nothing

### 4. Wire PagesPlaybookNarrative panel

**Scale:** XS | **Complexity:** Low

Register `PagesPlaybookNarrative` as a dock panel. Receives narrative content from the active playbook's `content:` sections. Verify with `morning-routine.yaml` which has inline markdown narratives.

**Blocked by:** #1 (dock layout)
**Blocks:** nothing

### 5. Wire OrchestrationWorkbench panel

**Scale:** S | **Complexity:** Med

Register `OrchestrationWorkbench` as a dock panel. Connect to `/push` WebSocket for live execution state, binding events, and device provisioning progress. Verify the execution monitor shows step-by-step progress during a playbook run.

**Blocked by:** #1 (dock layout)
**Blocks:** nothing

### 6. Dataset bindings and status bar

**Scale:** XS | **Complexity:** Low

Wire `fetchSource()` and `topicSource()` for simulation status and active playbook state. Add status bar badges: simulation state (IDLE/RUNNING), active playbook name, speed multiplier.

**Blocked by:** #1 (dock layout)
**Blocks:** nothing

### 7. End-to-end simulation verification

**Scale:** S | **Complexity:** Med

Run `morning-routine` playbook through the full stack with `MockDeviceProvider` and temporal simulation. Verify: simulation starts, device state changes flow through SSE/WebSocket, topology panel updates, playbook steps advance, narrative renders, orchestration workbench shows progress. Fix integration gaps.

**Blocked by:** #2, #3, #4, #5, #6 (all panels wired)
**Blocks:** nothing — but is the acceptance gate for the epic

## Dependency Graph

```
#1 (dock layout + rename)
 ├──▶ #2 (step catalog)
 ├──▶ #3 (library)
 ├──▶ #4 (narrative)
 ├──▶ #5 (orchestration)
 └──▶ #6 (datasets + status bar)
          │
          └──▶ #7 (e2e verification) ← blocked by #2–#6
```

Issues #2–#6 are independent of each other and can be worked in parallel or any order. #7 requires all panels wired.

## Follow-On Epics

- **Playbook Authoring** — `PagesBuilderShell`, `PagesCodeEditor`, ScriptRegistry REST (#135). Create and edit playbooks from the console.
- **IoT Presets** — Preset composer (#136), diff panel (#137). Device picker, state configuration, before/after comparison.

## References

- `fsitrading/app/src/main/webui/src/ops-centre.ts` — reference composition pattern
- `webapp/src/main/webapp/src/pages/scenarios.ts` — current page implementation
- `webapp/src/main/webapp/src/app.ts` — site structure and sidebar registration
- `.casehub-packages/packages/pages-aria/src/controller/` — available platform controllers
- `.casehub-packages/packages/orchestration-workbench/` — execution monitor component
- `docs/specs/issue-128-evaluate-playbook-ui/2026-10-07-playbook-ui-evaluation.md` — gap analysis
- `webapp/src/main/resources/META-INF/playbooks/` — bundled playbook library
- `webapp/src/main/resources/steps/iot-steps.yaml` — step catalog entries (#134)
- `DefaultIoTSimulationApi.java` — simulation MCP operations
- `IoTPlaybookConfigProducer.java` — playbook config producer
- D1–D3 in `decisions.md` — scope and approach decisions
