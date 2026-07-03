# Handoff — casehub-iot

**Head commit (project):** d3c358f — feat: IoT webapp — operational console with situational awareness — Closes #44

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Resolve upstream blockers for #47 — engine blocking SPI (casehubio/engine#640), work tenancy query (casehubio/work#286), ras artifact name (casehubio/casehub-ras#24). Until those ship, #47 is blocked. PLATFORM.md update (casehubio/parent#342) is independent and ready now.

## What's Left

- casehubio/iot#45 — minor review findings: SSE tenancy filtering, platform scope, stub cleanup · S · Low
- casehubio/iot#46 — evaluate extracting webapp domain logic to application-tier repo · M · Med
- casehubio/iot#47 — wire CaseResource + WorkItemResource to real APIs · M · Med · **blocked by** engine#640, work#286, casehub-ras#24
- casehub-ras#23 — promote SituationDefinitionProvider to casehub-ras-api · S · Low
- ARC42STORIES.MD update for webapp modules — skipped at session end · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| parent#342 | PLATFORM.md update for webapp capabilities | XS | Low | Ready now — no blockers |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver needed |
| — | Situation definition editor form in pages | M | Med | Runtime CRUD via REST already wired |

## Cross-Module

**We're blocking** (other modules waiting on us): none

**Blocked by** (can't proceed until):
- `casehub-engine` — blocking CaseInstanceRepository SPI (engine#640) · gates #47
- `casehub-work` — tenancy-aware query via WorkItemService (work#286) · gates #47
- `casehub-ras` — artifact name alignment (casehub-ras#24) · gates #47 build

**We delivered** (available to consumers once deps are published):
- Standalone IoT operational console — 7 REST resource groups, SSE, 7 TypeScript pages
- `webapp-api` reusable by casehub-life for IoT ganglia and case descriptors

## Key References

- Spec: `docs/superpowers/specs/2026-07-01-iot-webapp-design.md`
- Design review: `~/adr/casehub-iot/iot-webapp-*/tracker.md`
- Implementation plan: workspace `plans/attic/issue-44-iot-webapp-console/2026-07-01-iot-webapp.md`
- GitHub repo: `casehubio/iot`
