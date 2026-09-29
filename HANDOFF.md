# Handover — casehub-iot

## Last Completed

**#114 — Add vitest unit tests for audit trail renderers and correlation (closed, landed as 78deef2)**
**#115 — Shared filter adapter for cross-tab audit filter sync (closed, landed as 1e384f1)**

Branch `issue-114-vitest-audit-tests` landed on main (3 commits after squash). Both issues closed.

### #114 — Vitest tests
- Unblocked `yarn install` — 28 missing portal resolutions added to `package.json`
- Added `vitest`, `jsdom`, `lit` as devDependencies; `yarn test` script
- 23 tests across `audit-detail.test.ts` and `audit-correlation.test.ts`
- All 6 BridgeMessage payload variant renderers, lifecycle events, correlation cache/error-eviction, auditRowKey, renderAuditDetail

### #115 — Shared filter adapter
- `audit-filter-sync.ts` (19 lines) — document-level `pages-filter` event listener bridges DSL filter group to Trail endpoint URL params via `syncEndpoint()`
- Filters moved above tabs for shared visibility
- 5 tests covering apply/reset/combine/isolation/graceful-null

### Compilation fixes (pre-existing)
- `driver.state()` → `driver.lifecycle().currentState()` in IoTSimulationBeansTest + DefaultIoTSimulationApi
- Deleted 3 orphaned test files (IoTDeviceMcpToolTest, KpiResourceTest, IoTScenarioYamlTest) for deleted production classes

## Known issues

- **Quarkus webapp build failure** — CDI ambiguity: `casehub-engine-persistence-memory` at compile scope produces `@Default` beans (`DefaultTestPrincipal`, `InMemoryPlanItemStore`) that conflict with JPA and work-runtime implementations. Needs `@Alternative @Priority(100)` upstream in engine `PersistenceMemoryBeans` and platform `DefaultBeans`. Not fixable in this repo.

## Upstream issues filed this session

| Repo | # | Title |
|------|---|-------|
| platform | 486 | Map scenario concurrency primitives to distributed command dispatch — execution model, drift policy, SWF delegation, desired state as primary model |
| casehub-desiredstate | 153 | IoT consumer requirements — 7 foundation gaps (drift policy, flat graph, idempotency, step generation, presets, convergence reporting, ordering) |

## IoT issues filed this session

| # | Title | Scale | Status |
|---|-------|-------|--------|
| 105 | Unified API generation — @McpDomain migration | — | Closed (already landed) |
| 116 | Device topology + simulation + playbooks | XL | Closed (superseded by #121) |
| 117 | Distributed command orchestration | L / High | Open — soft dep on platform#486 |
| 118 | IoT NodeProvisioner — desiredstate adapter | M / Med | Open — start immediately |
| 119 | Dual-view topology + drift visualization | M / Med | Open — after icon registry + location API |
| 120 | Desired state + YAML + playbook UI | XL | Closed (superseded by #121) |
| 121 | **Consolidated epic: IoT desired state, topology, and scenario-driven automation** | XL / High | Open — 15 items, 5 phases |

## What's Next

Start with **#121 Phase 1** — no upstream dependencies:

1. **#118 — `casehub-iot-desiredstate` module** (M / Med) — IoTNodeProvisioner, IoTGoalCompiler, IoTActualStateAdapter against current desiredstate-api SPIs
2. **Device icon registry** (S / Low) — `Record<DeviceClass, SVGPath>` mapping Matter device types to icons with status badges
3. **Location hierarchy API** (M / Med) — expose spatial tree from DeviceRegistry

Also: build out a **sample house catalogue** — pre-built house/building configurations with device inventories and matching simulation profiles for demos and testing.

## Key Context

- IoT has full `@McpDomain` coverage — 8 domains, 24 queries, 14 mutations, 1 stream. Every public API is annotated; no standalone REST resources.
- The YAML scenario language is functional and declarative with concurrency control — maps directly to IoT device orchestration (fan-out, join gates, conditional routing).
- Desired state is the primary execution model — commands are single-node desired states. Most IoT automation is flat convergence (no ordering). Ordering declared as structural metadata when needed.
- Reactive triggers produce drift. Drift policies govern permitted overrides with auto-revert (duration, schedule, event, never).
- `casehub/fsitrading` has extensive playbook UI support — evaluate before building IoT-specific playbook features.
- 1 garden entry captured: GE-20260928-05352b (lit TemplateResult `.map()` array gotcha).
