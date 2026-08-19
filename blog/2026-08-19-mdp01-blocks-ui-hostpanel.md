---
layout: post
title: "hostPanel, Not html — How the Decision Review Saved a Wrong Integration"
date: 2026-08-19
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [blocks-ui, pages-ui, hostPanel, web-components, design-review]
series: issue-95-blocks-ui-integration
---

The IoT webapp had been building every page by hand — four-column KPI metric cards assembled from `columns()` + `metric()` DSL, work item actions as inline HTML buttons with `onclick` handlers, device detail as stacked panels. blocks-ui already had components that did all of this better: `<blocks-kpi-metric-row>` with sparklines and trend indicators, `<blocks-work-item-detail>` with full lifecycle management (twelve action types with proper dialogs), `<blocks-detail-pane>` with tabbed navigation and keyboard support.

The integration looked simple. Embed blocks-ui components via `html()` blocks in the pages-ui DSL, pass endpoints as attributes, done. We built a spec around this, captured four decisions, and sent them through a standard decision review.

The review came back and rewrote two of the four decisions.

The first correction was fundamental. `html()` uses `innerHTML` — it can only pass string attributes. A `<blocks-kpi-metric-row>` needs `MetricDefinition[]`. A `<blocks-work-item-detail>` needs `WorkIdentity` objects and `UserSearchProvider` callbacks. None of these survive innerHTML serialisation. The reviewer found `hostPanel()` — a platform mechanism that calls `configure(panelProps)` with JS object references before `connectedCallback()`. It had been in pages-ui the whole time; we just didn't know it existed.

The second correction dropped the audit page entirely. We'd proposed building a generic `<blocks-event-trail>` component by extracting the common pattern from `blocks-audit-trail-viewer`. The reviewer read all 695 lines of the audit-trail-viewer and proved the extraction was hollow — every layer is saturated with ledger-specific concerns (Merkle verification, attestation display, `LedgerEntry` data model with digests and sequence numbers). Strip those and you're left with a table plus filters, which pages-ui already provides. The 30-line DSL audit page stays.

With the corrected design, the implementation was straightforward. New `KpiResource` REST endpoints compute device metrics server-side. Six custom tab content elements render device state, history, and actions inside `<blocks-detail-pane>`. The work items page uses `split()` with `hostPanel("blocks-work-item-detail")` — replacing three inline HTML buttons with twelve lifecycle actions, proper dialogs, and a relations tab.

The review caught a null safety bug too. `DeviceCommand.VALID_ACTIONS` is an immutable `Set.of()` — calling `contains(null)` throws `NullPointerException`, not `false`. A `request.action()` arriving as null from malformed JSON would produce a 500 instead of the intended 400. One null check fixed it.

The real lesson is about `hostPanel()`. Every blocks-ui component in the platform already implements `configure()` via `DataSourceMixin`. The integration mechanism was designed, built, tested, and deployed — we just reached for `html()` first because it was the obvious choice. The decision review's value wasn't catching a bug. It was catching a design that would have worked for the first component and failed silently for every subsequent one.
