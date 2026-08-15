---
layout: post
title: "The Webapp's Phantom Dependency"
date: 2026-08-15
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [webapp, blocks-ui, pages-ui, ui-audit]
---

The IoT webapp declares `casehub-blocks-ui-npm` as a Maven dependency, unpacks it to `.casehub-packages/`, configures portal resolutions in `package.json` — and then never imports a single component from it. All seven pages build their UI manually with pages-ui DSL primitives: `columns()`, `metric()`, `panel()`, `table()`. The blocks-ui web components sit in the unpacked directory, fully resolved, completely unused.

I found this while checking whether the webapp was correctly wired against the latest pages and blocks-ui. The pages-ui integration is fine — every page uses the DSL correctly, design tokens flow from `@casehubio/pages-ui-tokens`, no custom CSS. But blocks-ui is a phantom: present on the dependency graph, absent from the source.

The most obvious duplication is `device-kpi-row.ts`. It builds a four-column KPI metric row using `columns()` + `metric()` — the same thing `<blocks-kpi-metric-row>` already does, plus sparklines, trend indicators, status colours, density modes, and proper ARIA. The webapp's version has none of that.

Claude mapped all 40+ blocks-ui components against the seven webapp pages. Three replacements stood out: the KPI rows (devices and health pages), the audit page (where `<blocks-audit-trail-viewer>` would replace ~30 lines of filter-plus-table construction), and the work items page (where `<blocks-split-workbench>` with `<blocks-work-item-detail>` would give proper lifecycle actions — claim, escalate, delegate, complete — instead of inline HTML buttons). Case and device detail sub-pages could also use `<blocks-detail-pane>` for tabbed views with lazy loading.

Filed as #95. About 40% of the manual construction can be replaced, and `device-kpi-row.ts` can be deleted entirely.

Separately, an ARC42STORIES.MD stale scan turned up three references that hadn't been updated: #35 (BridgeAuditStore SPI) still marked as "deferred" though it shipped, #20 and #22 still in the "Not closed here" list though both are done, and the OpenHAB semantic model risk still saying "Thing-scoped discovery fallback deferred" when #11 delivered it months ago. All three fixed and committed.

The phantom dependency is a good reminder that declaring a dependency and actually using it are different things. The webapp was built correctly against pages-ui; it just never took the next step with blocks-ui. Now there's an issue for it.
