# Handoff — casehub-iot

**Head commit (project):** 7ce9837 — fix: webapp review findings, per-provider refresh, disabled tests, ARC42 update

---

## What This Repo Is

Foundation IoT device abstraction layer for CaseHub. Typed device hierarchy, provider SPIs, Home Assistant/OpenHAB implementations, bridge runtime, and operational webapp console.

## Immediate Next Step

Pick up #48/#49 (CBR epic — design-first) or #46 (evaluate webapp extraction to app tier).

## What Was Done (2026-07-10)

**Branch:** `issue-45-webapp-review-and-cleanup` — closed, landed as 7ce9837 on main.
**Closed:** #45, #43, #53, #54

- **SSE tenancy filtering (#45)** — moved from broadcast-time `@ObservesAsync` (broken with `@RequestScoped` principal) to per-client `stream()`. Broadcaster changed from `String` to `DeviceResponse` for per-client filtering.
- **Platform scope (#45)** — `casehub-platform` set to `runtime` in webapp pom.xml.
- **Stub cleanup (#45)** — removed unused `EntityManager` injection from `CaseResource` and `WorkItemResource`.
- **Per-provider refresh (#43)** — `DeviceRegistry.refresh(String)` added to SPI. `CdiDeviceRegistry` implementation rediscovers only the target provider. `POST /api/providers/{providerId}/refresh` endpoint added.
- **Disabled tests (#53)** — fixed upstream in `casehub-ras-api` (branch `issue-34-jackson-type-info` on casehubio/casehub-ras). Added `@JsonTypeInfo`/`@JsonSubTypes` to `TriggerAction`, `ChainMode`, `TriggerMode` sealed interfaces. All 3 tests now pass.
- **ARC42STORIES (#54)** — added L7 Webapp API, L8 Webapp Drools, L9 Webapp Console layers and Chapter 7 entry.

## Cross-Repo Changes

- **casehubio/casehub-ras** — branch `issue-34-jackson-type-info` pushed (not merged). Issue #34. Adds Jackson `@JsonTypeInfo` to sealed types in `casehub-ras-api`. Installed to local Maven repo.

## What's Left

- casehubio/iot#46 — evaluate extracting webapp domain logic to application-tier repo · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #48 | CBR epic — case-based reasoning for IoT | — | — | Design-first; start with #49 |
| #46 | Evaluate webapp extraction to app tier | M | Med | PLATFORM.md boundary question |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver |
| — | Situation definition editor form in pages | M | Med | Runtime CRUD already wired |
| — | Merge casehub-ras#34 to ras main | XS | Low | Jackson type info — currently on branch |

## Key References

- Garden: GE-20260707-50052f — Quarkiverse extension version mismatch gotcha
- CI: green as of 7ce9837
- GitHub repo: `casehubio/iot`
