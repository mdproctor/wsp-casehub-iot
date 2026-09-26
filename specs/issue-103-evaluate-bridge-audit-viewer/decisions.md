## D1: Overall approach — tab-toggled dual view

**Choice:** Replace the single DSL table audit page with a tab-toggled page offering both the existing DSL table ("Table") and a new `pages-event-trail` component ("Trail"). Operators choose the view that fits their task — flat table for scanning, trail for drilling into payloads and correlations.
**Alternatives:**
- Build only the event trail, drop the DSL table — loses the fast-scan workflow that a flat table serves; the DSL table is 34 lines and already works
- Keep DSL table only, add features incrementally — the expandable detail and correlation requirements push beyond what DSL table primitives support
- Build an IoT-specific `<iot-audit-viewer>` component — bypasses the opportunity to harden `pages-event-trail` for all consumers; the app tier exists to exercise and improve foundation components
**Rationale:** `pages-event-trail` is a platform component in pages-ui-components that was unavailable when #103 was filed. It provides expandable details, chip/entity filtering, and date range. Decision review (R1-02, R1-03, R1-06) identified gaps in the component — these are upstream fixes that harden the foundation for all consumers, not IoT workarounds. The DSL table is kept as a lightweight alternative for fast scanning.
**Trade-offs:** Two views to maintain. Cross-repo changes required in casehub-pages (~15 lines total — see D2). Both views share filters and the same endpoint.
**Sources:** `webapp/src/main/webapp/src/pages/audit.ts` (34-line DSL table), `pages-ui-components/src/event-trail/pages-event-trail.ts` (platform event trail component), #95 D1 (rationale for keeping DSL table), #95 D2 (hostPanel integration pattern)
**Exploration:** quick → revised round 1
**Status:** revised — acknowledged cross-repo pages-event-trail fixes as upstream hardening per reviewer R1-02/R1-03/R1-06; reframed IoT-local alternative as bypassing foundation improvement opportunity

## D2: Integration pattern — hostPanel via tabs DSL, with upstream pages-event-trail fixes

**Choice:** Use the `tabs()` DSL function to create "Table" and "Trail" tabs. Trail tab uses `hostPanel("event-trail", { ...config })` to host `pages-event-trail`. Requires `registerPanel("event-trail", "pages-event-trail")` in the webapp setup. Three targeted upstream fixes in casehub-pages harden the component:
1. `configure()` gap (~3 lines): add `getRowDetail`, `getRowKey`, `columnRenderers` to property passthrough
2. Data shape (~10 lines): add `recordsPath` option so `createSourceFactory()` can extract an array from a wrapped response (e.g. `recordsPath: "records"` for `AuditTrailView`)
3. CSV export (~2 lines): pass `csvExport` configure option through to internal `pages-table`
**Alternatives:**
- IoT-local wrapper around `pages-data-table` — avoids cross-repo changes but leaves `pages-event-trail` gaps unfixed for all future consumers; the app tier exists to harden foundation
- Inline Lit template via `html()` — cannot pass complex properties (functions like `getRowDetail`); `html()` uses `innerHTML` which only supports string attributes
**Rationale:** `hostPanel()` is the validated integration pattern from #95 D2. The configure() gap (R1-02) is real — `PagesEventTrail.configure()` handles only data/columnDefs/columnConfig/chipField/chipValues/entityField/entityLabel/showDateRange, silently dropping `getRowDetail`/`getRowKey`/`columnRenderers`. The data shape mismatch (R1-03) is also real — `createSourceFactory()` casts the response as `unknown[]` but the endpoint returns `{ records, totalCount, offset, limit }`. Both are small, targeted fixes that make the component robust for any hostPanel consumer.
**Trade-offs:** Cross-repo dependency on casehub-pages. Changes are small (~15 lines total) and scoped to `pages-event-trail.ts` only. Issue against casehub-pages should be filed before implementation begins.
**Depends on:** D1 (tab-toggled page)
**Sources:** `pages-ui/src/dsl/builders.ts:574` (hostPanel), `pages-ui/src/dsl/builders.ts:245` (tabs), `pages-event-trail.ts:91` (configure method — gaps at getRowDetail/getRowKey/columnRenderers), `pages-event-trail.ts:40` (createSourceFactory — missing dataPath/recordsPath), #95 D2 (hostPanel validation), review R1-02/R1-03/R1-04/R1-06
**Exploration:** quick → revised round 1
**Status:** revised — acknowledged cross-repo configure() and data shape fixes per reviewer R1-02/R1-03; reframed as upstream hardening

## D3: Detail pane content — tailored rendering per event type, with TypeScript type model

**Choice:** Detail rendering branches on `BridgeAuditEventType` first (always present on the audit event), then on `BridgeMessage.@type` for message-bearing events. A TypeScript discriminated union type models the `BridgeMessage` variants for compile-time exhaustiveness. Renderers return Lit `TemplateResult`.
**Alternatives:**
- Pretty-printed JSON dump — universal but operators must parse structure themselves
- Key-value summary + collapsible JSON — middle ground but still requires knowing which fields matter per type
- Switch on `@type` alone (original D3) — fails for AGENT_CONNECTED/DISCONNECTED which have null message (no `@type`); NPE or silent miss
**Rationale:** `BridgeAuditEventType` has 8 values but `BridgeMessage` has only 7 variants — `AGENT_CONNECTED` and `AGENT_DISCONNECTED` fire with `message = null` (protocol lifecycle events). Branching on `eventType` first handles the null-message case cleanly. A TypeScript discriminated union type (`type BridgeMessagePayload = { "@type": "STATE_CHANGE"; event: StateChangeEvent } | ...`) gives real compile-time exhaustiveness — without it, a `switch` on a plain `string` compiles without a `default` case and misses new variants silently (R1-09). Variants with similar structure (StateChange and ReplayedStateChange) share renderers.
**Trade-offs:** Requires maintaining a TypeScript type model that mirrors the Java sealed interface. When a new `BridgeMessage` variant is added in Java, the TypeScript types must be updated — but the discriminated union's exhaustive switch will flag the gap at compile time.
**Depends on:** D1 (trail view exists), D2 (hostPanel delivers getRowDetail callback)
**Sources:** `api/src/main/java/io/casehub/iot/api/bridge/BridgeMessage.java` (7 variants), `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEventType.java` (8 event types — AGENT_CONNECTED/DISCONNECTED have null message), `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEvent.java:12` (@Nullable BridgeMessage message), review R1-09/R1-10/R1-18
**Exploration:** quick → revised round 1
**Status:** revised — branching on eventType first per R1-10; added TypeScript discriminated union type model per R1-09/R1-18

## D4: Correlation display — detail pane siblings via server-side query

**Choice:** When a row with a non-null `correlationId` is expanded, the detail pane issues a targeted server-side query (`/api/bridge/audit?correlationId=X`) to fetch the complete correlation group, then renders the payload (D3) plus a "Related Events" section listing all sibling events. This shows the full correlation chain (e.g. COMMAND_SENT → COMMAND_RESPONSE) in-context.
**Alternatives:**
- Client-side matching from loaded data (original D4) — fails across page boundaries; if COMMAND_SENT is on page 1 and COMMAND_RESPONSE on page 2, the match silently fails (R1-13)
- Clickable correlationId filter — clicking filters the Trail to that correlationId; simpler but requires manual unfiltering to return to the full view
- Visual grouping in table rows — correlated events grouped with connecting indicators; requires extending pages-table rendering beyond current support
**Rationale:** The `BridgeAuditQuery` already supports `correlationId` as a query parameter. A single REST call on expand guarantees complete correlation chains regardless of pagination state. The fetch is lightweight — correlation groups are typically 2-4 events (command + response, sometimes with state changes). Client-side matching was unreliable because the Trail view's pagination means correlated events may span different pages.
**Trade-offs:** One additional REST call per row expand (when correlationId is non-null). Acceptable latency — the query is indexed and returns 2-4 rows. Correlation is only visible when a row is expanded — not in the table scan view. This is acceptable because correlation tracking is a drill-down activity, not a scan activity.
**Depends on:** D3 (detail pane rendering)
**Sources:** `BridgeAuditEvent.java:10` (correlationId field), `BridgeAuditQuery.java:13` (correlationId query support), `DefaultIoTOperationsApi.java:104` (correlationId filter in REST endpoint), review R1-13
**Exploration:** quick → revised round 1
**Status:** revised — switched from client-side matching to server-side correlationId query per R1-13

## D5: Filter strategy — shared common filters across tabs

**Choice:** Common filter dimensions (deviceId, eventType, date range) are shared across both tabs. When an operator applies a device filter in the Table tab and switches to Trail, the filter persists. Tab-specific features (CSV export in Table, chip display in Trail) remain independent.
**Alternatives:**
- Independent filters per tab (original D5) — each tab manages its own state; simpler implementation but filter state resets on tab switch, defeating the drill-down workflow where operators scan in Table then expand in Trail
**Rationale:** The primary use case for dual views is that operators want different *views* of the *same filtered data*. An operator investigating a specific device's audit trail applies a `deviceId` filter in Table for overview, then switches to Trail for expanded details. Losing the filter on switch forces re-application, which is friction. The shared dimensions (deviceId, eventType, dates) exist in both views' filter models. Implementation: page-level filter state drives URL params on both views' endpoints.
**Trade-offs:** Requires a filter state synchronisation mechanism between the DSL filter model and pages-event-trail's FilterState. More complex than independent filters, but the shared dimensions are simple scalar values (deviceId string, eventType enum, date range).
**Depends on:** D1 (tab-toggled page)
**Sources:** `audit.ts` (current DSL filter setup), `pages-event-trail.ts:103-110` (FilterState handling), review R1-16
**Exploration:** quick → revised round 1
**Status:** revised — switched from independent to shared common filters per R1-16; operator drill-down workflow requires filter persistence across tab switches
