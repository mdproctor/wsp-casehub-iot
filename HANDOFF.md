# Handoff — casehub-iot

**Head commit (project):** d3c358f — feat: IoT webapp — operational console with situational awareness — Closes #44

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Publish cross-repo dependencies (`casehub-ras`, `casehub-engine`, `casehub-work`, `casehub-ledger`, `casehub-platform`, `casehub-connectors`) to GitHub Packages so the `webapp` module can compile. Then run `mvn install` to verify the full build. The `webapp-api` and `webapp-drools` modules already build and test cleanly (53 tests).

## What's Left

- casehubio/parent#325 — PLATFORM.md sync for BridgeAuditStore retention capability · XS · Low
- casehubio/iot#45 — minor review findings: SSE tenancy filtering with @RequestScoped, platform scope, stub cleanup · S · Low
- casehubio/iot#46 — evaluate extracting webapp domain logic to application-tier repo · M · Med
- casehub-ras#23 — promote SituationDefinitionProvider to casehub-ras-api · S · Low
- Cross-repo dependency publishing — webapp module can't compile until upstream artifacts are available · M · Low (mechanical)
- ARC42STORIES.MD update for webapp modules — skipped at session end, do in next session · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver needed |
| — | Wire CaseResource + WorkItemResource to real engine/work APIs | S | Med | Placeholder implementations, requires cross-repo deps |
| — | Situation definition editor form in pages | M | Med | Runtime CRUD via REST already wired |
| — | PLATFORM.md update for webapp capabilities | XS | Low | Capability ownership table needs webapp entry |

## Cross-Module

**We delivered** (available to consumers once deps are published):
- Standalone IoT operational console — 7 REST resource groups, SSE, 7 TypeScript pages
- 5 JavaSwitch + 2 DroolsCEP ganglia for IoT situation detection
- 4 case definitions with safety-first ActionRiskClassifier
- Three-datasource Flyway layout for upstream version collision isolation (garden: GE-20260701-107e7b)
- `webapp-api` reusable by casehub-life for IoT ganglia and case descriptors

## Key References

- Spec: `docs/superpowers/specs/2026-07-01-iot-webapp-design.md`
- Design review: `~/adr/casehub-iot/iot-webapp-*/tracker.md`
- Implementation plan: workspace `plans/attic/issue-44-iot-webapp-console/2026-07-01-iot-webapp.md`
- GitHub repo: `casehubio/iot`
