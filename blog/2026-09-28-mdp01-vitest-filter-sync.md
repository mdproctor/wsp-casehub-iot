---
title: "Testing lit templates without the DOM, and bridging filter events across tabs"
date: 2026-09-28
author: mdp
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [vitest, lit, testing, pages-ui, filter-sync]
issues: [114, 115]
---

# Testing lit templates without the DOM, and bridging filter events across tabs

The audit trail viewer landed last session — a tab-toggled page with a DSL table
and a `pages-event-trail` hosted panel, eight payload-variant renderers, and
correlation tracking. All untested. Issue #114 was the vitest gap; #115 was the
filter sync between tabs.

## The portal resolution chain

Vitest config existed. Vitest itself didn't — `yarn install` failed because two
transitive `@casehubio` packages (`pages-schema`, `yaml-core`) weren't declared
as portal resolutions in `package.json`. Adding those two surfaced a third
(`pages-property-palette`). Adding that surfaced twenty-five more.

The Yarn PnP model doesn't resolve transitive `@casehubio` packages from the
npm registry — every internal package reference in the dependency tree needs an
explicit portal resolution pointing to the vendored `.casehub-packages/` directory.
I wrote a script to diff the full set of `@casehubio` references across all
portal packages against the declared resolutions. Twenty-eight were missing. After
adding them all, `yarn install` succeeded for the first time on this project.

## Testing TemplateResults directly

The renderers return lit `TemplateResult` objects — `html` tagged template
literals with `strings` and `values` arrays. The conventional approach is to
render to a jsdom DOM and query elements. That requires element registration,
lifecycle management, and a full DOM setup for what are essentially pure
functions: data in, template out.

I went with direct `TemplateResult` inspection instead. The `strings` array
holds the static template fragments; `values` holds the interpolated expressions.
A recursive walk of `strings` and `values` flattens the entire template tree
into a single string — good enough for `toContain()` assertions on CSS classes,
content text, and structural elements.

It broke on the first real test. The `renderStateChange` function maps over
`changedCapabilities` inside the template:

```typescript
${event.changedCapabilities.map(cap => html`
  <tr><td>${cap}</td>...
`)}
```

Lit's `html` doesn't inline `.map()` results — it produces an `Array<TemplateResult>`
in the `values` array. My traversal checked for `strings` and `values` properties
but never checked for arrays. The mapped content was silently skipped; tests
passed with empty assertions. Adding `if (Array.isArray(result)) return result.map(collectText).join('')`
to the recursive flattener fixed it — all 23 tests green.

## Bridging DSL filters to hosted panels

The audit page has two tabs: Table (standard DSL table with filter groups) and
Trail (`hostPanel("event-trail", { endpoint: "..." })`). The selectors use the
DSL filter pipeline — `filter: { enabled: true, group: "audit" }`. The table
listens via `filter: { listening: true, group: "audit" }`. The Trail doesn't
participate in the DSL filter pipeline at all — it fetches from its own endpoint
and has its own chip/entity/date filters.

Moving the selectors above the `tabs()` call was the easy part — the Table
continues listening to the "audit" filter group. But the Trail needs a bridge.

I explored three approaches. Template variables (`#{filter.eventType}` in the
endpoint URL) won't work — `hostPanel` defers `configure()` until *all* template
variables resolve, so an unset filter would prevent the Trail from ever rendering.
Data property binding (`data` on the Trail) isn't reactive from the DSL layer.

The approach that works: a document-level `pages-filter` event listener. The
runtime dispatches `pages-filter` from selectors, which bubbles up through the
DOM. A nineteen-line module listens on `document`, tracks filter state in a `Map`,
rebuilds the Trail endpoint URL with query params, and calls `syncEndpoint()`.
The REST endpoint already accepts `eventType`, `deviceId`, `correlationId`,
`from`, and `to` as query parameters — the bridge just constructs the URL.

```typescript
const filters = new Map<string, string>();

function onFilter(e: Event) {
  const { columnId, value, reset, group } = (e as CustomEvent).detail;
  if (group !== 'audit') return;
  if (reset) filters.delete(columnId);
  else filters.set(columnId, value);

  const trail = document.querySelector('blocks-event-trail') as
    HTMLElement & { endpoint: string; syncEndpoint(): void } | null;
  if (!trail) return;

  const params = new URLSearchParams({ limit: '500' });
  for (const [k, v] of filters) params.set(k, v);
  trail.endpoint = `/api/bridge/audit?${params}`;
  trail.syncEndpoint();
}

document.addEventListener('pages-filter', onFilter);
```

Imported as a side-effect module from `audit.ts` — no custom element registration,
no lifecycle management. The listener is harmless on non-audit pages (no
`blocks-event-trail` element to find, no "audit" group events to match).

## Upstream drift

The Maven build had four pre-existing compilation errors, all from upstream
SNAPSHOT changes that hadn't been absorbed. `TemporalSimulationDriver.state()`
was renamed to `lifecycle().currentState()` in the platform simulation library.
Three test classes referenced deleted production classes — `IoTDeviceMcpTool`
(replaced by `DefaultIoTDeviceApi`), `KpiResource` (replaced by `@McpDomain`
SPIs), and scenario descriptor classes removed from `pages-scenario`.

The fixes were mechanical — two method renames, three dead test deletions.
But they surfaced a deeper issue: `casehub-engine-persistence-memory` at
compile scope in the webapp pom causes CDI ambiguity with three
`CurrentPrincipal` providers and three `PlanItemStore` providers. The memory
module's `PersistenceMemoryBeans` produces `@Default` beans that compete with
the JPA and work-runtime implementations. That needs `@Alternative @Priority(100)`
upstream — the same pattern this repo already uses for `casehub-iot-bridge-persistence-memory`.

## What this opens up

Twenty-eight tests and a functioning vitest pipeline means frontend TypeScript
in this repo now has the same test-before-commit discipline as the Java side.
The `collectText` technique generalises to any lit template testing — no DOM
setup, no element registration, no jsdom lifecycle. And the filter bridge
pattern (`pages-filter` event → endpoint URL injection → `syncEndpoint()`)
works for any `hostPanel` component that loads data from a REST endpoint.
