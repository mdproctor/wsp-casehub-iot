---
layout: post
title: "Securing the MCP Surface"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [mcp, rbac, tenancy, security]
series: issue-74-rbac-tenancy-mcp
---

The MCP tools shipped in #69 were deliberately unauthenticated — a placeholder `"mcp-agent"` string where the caller identity should go, `findAll()` returning every device regardless of who's asking. That was fine for proving the tool surface worked. #74 closes the gap.

The interesting constraint is that the MCP module is a library consumed by two hosts with opposite security postures. The webapp runs OIDC with `CurrentPrincipal` from casehub-work. The bridge runs on a local network with no auth mechanism at all. Anything I wired into the MCP module had to work in both — enforcing tenancy where a principal exists, falling back silently where it doesn't.

The design centres on `McpIdentityContext`, an `@ApplicationScoped` bean that wraps `Instance<CurrentPrincipal>` with a three-guard scope check: `isResolvable()`, `Arc.container() != null`, `requestContext().isActive()`. When all three pass, you get the authenticated caller's tenancy and actor ID. When any fails — bridge deployment, background thread, unit test without CDI — it falls back to the `casehub.iot.tenancy-id` config property and `"mcp-agent"`. The tool class injects `McpIdentityContext` and never touches CDI introspection directly.

The `@RolesAllowed` annotations are the simpler half. `IoTRoles` constants in the API module (`VIEWER`, `OPERATOR`, `ADMIN`) replace the string literals that were already scattered across `DeviceResource` and `SituationResource`. When the host app has a security extension, the annotations enforce access. When it doesn't, they're inert. The bridge sees no change.

The tenancy filtering went deeper than expected. `DeviceRegistry` got a `findById(deviceId, tenancyId)` default method — backward compatible, post-filters via `DeviceEntity.tenancyId()`. But `DeviceStateHistoryProvider.findHistory()` needed a proper `tenancyId` parameter added to the SPI signature — a breaking change, because `HistoryEntry` doesn't carry tenancy and a default method couldn't post-filter. The JPA implementation now has `AND h.tenancyId = :tenancyId` in the query, which means history for deprovisioned devices (no longer in the live registry) is still accessible to the right tenant. That's defense-in-depth the original device-lookup approach wouldn't have caught.

The adversarial design review — coherence, structure, and robustness in parallel, then cross-cutting synthesis — surfaced the `findHistory` gap. The structure reviewer assumed the SPI would stay tenancy-unaware. The robustness reviewer independently argued for data-layer enforcement. The cross-cutting pass reconciled them. It also caught that `IoTCommandAuditEvent` needed a `tenancyId` field — without it, correlating commands to tenants after device deprovisioning requires joining through a volatile registry.

Three new GitHub issues came out of the review: #86 (cross-tenant admin access via MCP), #87 (deprovisioned device history regression in `DeviceResource`), and #88 (webapp integration tests for the three-guard pattern).

The delivery epic structure I set up before starting #74 maps the remaining backlog into four parallel tracks. #74 was the first item on the critical path — MCP production readiness. Next is CBR completion (#60, #61), then AI resolution agent maturity (#81, #85). The critical path runs seven sessions if taken sequentially; the parallel lanes can compress that.
