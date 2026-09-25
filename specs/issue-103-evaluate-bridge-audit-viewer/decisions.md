## D1: Overall approach — tab-toggled dual view

**Choice:** Replace the single DSL table audit page with a tab-toggled page offering both the existing DSL table ("Table") and a new `pages-event-trail` component ("Trail"). Operators choose the view that fits their task — flat table for scanning, trail for drilling into payloads and correlations.
**Alternatives:**
- Build only the event trail, drop the DSL table — loses the fast-scan workflow that a flat table serves; the DSL table is 34 lines and already works
- Keep DSL table only, add features incrementally — the expandable detail and correlation requirements push beyond what DSL table primitives support
- Build an IoT-specific `<iot-audit-viewer>` component — unnecessary when `pages-event-trail` already provides expandable details, chip/entity filtering, and `configure()` for hostPanel integration
**Rationale:** `pages-event-trail` is a platform component in pages-ui-components that was unavailable when #103 was filed. It provides all four revisit triggers (expandable details, filtering, correlation via detail pane, export via pages-table CSV). Keeping both views costs almost nothing — the DSL table is already written and the tabs() DSL function handles the toggle.
**Trade-offs:** Two views to maintain instead of one. Both independently fetch from the same `/api/bridge/audit` endpoint — acceptable since the endpoint is simple and the data is the same.
**Sources:** `webapp/src/main/webapp/src/pages/audit.ts` (34-line DSL table), `pages-ui-components/src/event-trail/pages-event-trail.ts` (platform event trail component), #95 D1 (rationale for keeping DSL table), #95 D2 (hostPanel integration pattern)
**Exploration:** quick
**Status:** captured

## D2: Integration pattern — hostPanel via tabs DSL

**Choice:** Use the `tabs()` DSL function to create "Table" and "Trail" tabs. Trail tab uses `hostPanel("event-trail", { ...config })` to host `pages-event-trail`. Requires `registerPanel("event-trail", "pages-event-trail")` in the webapp setup.
**Alternatives:**
- Inline Lit template via `html()` — cannot pass complex properties (functions like `getRowDetail`, objects like column configs); `html()` uses `innerHTML` which only supports string attributes
- Custom DSL function `eventTrail()` — no DSL wrapper exists; building one is unnecessary when `hostPanel()` already handles external component hosting
**Rationale:** `hostPanel()` is the validated integration pattern from #95 D2. It calls `configure(panelProps)` before `connectedCallback()`, enabling complex property delivery including the `getRowDetail` callback function. `pages-event-trail` already implements `configure()`.
**Trade-offs:** Requires a `registerPanel()` call in the webapp setup code. This is a one-liner following the established pattern.
**Depends on:** D1 (tab-toggled page)
**Sources:** `pages-ui/src/dsl/builders.ts:574` (hostPanel), `pages-ui/src/dsl/builders.ts:245` (tabs), `pages-event-trail.ts:91` (configure method), #95 D2 (hostPanel validation)
**Exploration:** quick
**Status:** captured

## D3: Detail pane content — tailored rendering per BridgeMessage variant

**Choice:** Each `BridgeMessage` variant gets a purpose-built detail rendering when a row is expanded in the Trail view. A TypeScript module maps `@type` discriminator to renderer functions returning Lit `TemplateResult`.
**Alternatives:**
- Pretty-printed JSON dump — universal but operators must parse structure themselves; defeats the purpose of expandable details
- Key-value summary + collapsible JSON — middle ground but still requires knowing which fields matter per type
**Rationale:** The 7 `BridgeMessage` variants carry fundamentally different information (StateChange: device/property/old-value/new-value; Command: target device/action/parameters; CommandResponse: result status/error). Tailored rendering surfaces the operationally relevant fields immediately. Variants with similar structure (StateChange and ReplayedStateChange) can share renderers.
**Trade-offs:** More implementation work (one renderer per variant). New message variants added to the sealed interface will need a corresponding renderer — but the TypeScript `switch` on `@type` will catch missing cases at compile time.
**Depends on:** D1 (trail view exists), D2 (hostPanel delivers getRowDetail callback)
**Sources:** `api/src/main/java/io/casehub/iot/api/bridge/BridgeMessage.java` (7 variants), `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEventType.java` (8 event types)
**Exploration:** quick
**Status:** captured

## D4: Correlation display — detail pane siblings

**Choice:** When a row with a non-null `correlationId` is expanded, the detail pane shows the payload rendering (D3) plus a "Related Events" section listing all other audit events sharing that `correlationId`. This shows the full correlation chain (e.g. COMMAND_SENT → COMMAND_RESPONSE) in-context.
**Alternatives:**
- Clickable correlationId filter — clicking filters the Trail to that correlationId; simpler but requires manual unfiltering to return to the full view
- Visual grouping in table rows — correlated events grouped with connecting indicators; richer but requires extending pages-table rendering beyond what pages-event-trail supports
**Rationale:** The detail pane siblings approach uses the existing `getRowDetail` callback — no changes to pages-table or pages-event-trail needed. The correlated events are fetched client-side from the already-loaded data (matching on `correlationId`). For COMMAND_SENT → COMMAND_RESPONSE, the operator expands the command row and immediately sees its response status.
**Trade-offs:** Correlation is only visible when a row is expanded — not in the table scan view. This is acceptable because correlation tracking is a drill-down activity, not a scan activity. Requires the Trail view to hold enough data in memory to find siblings (the current page of results).
**Depends on:** D3 (detail pane rendering)
**Sources:** `BridgeAuditEvent.java:10` (correlationId field), `BridgeAuditQuery.java:13` (correlationId query support), `pages-event-trail.ts:25` (getRowDetail callback)
**Exploration:** quick
**Status:** captured

## D5: Filter strategy — independent per tab

**Choice:** Each tab manages its own filtering independently. The Table tab keeps its existing DSL dropdown selectors and date pickers. The Trail tab uses `pages-event-trail`'s built-in filter bar with chip filtering (eventType), entity filtering (deviceId), and date range.
**Alternatives:**
- Shared page-level filters above the tabs — both views respect the same filter state; preserves filter selection across tab switches but requires bridging DSL filter model with pages-event-trail's FilterState, adding coupling between the two views
**Rationale:** Independent filtering keeps each view self-contained. The Table and Trail serve different workflows — an operator scanning in Table mode likely applies different filters than one drilling down in Trail mode. No cross-tab state synchronisation is needed, which simplifies both implementation and mental model.
**Trade-offs:** Switching tabs loses filter state — the other tab starts with its default (unfiltered) view. This is acceptable since tab switching indicates a workflow change, not a continuation.
**Depends on:** D1 (tab-toggled page)
**Sources:** `audit.ts` (current DSL filter setup), `pages-event-trail.ts:173` (built-in filter bar rendering)
**Exploration:** quick
**Status:** captured
