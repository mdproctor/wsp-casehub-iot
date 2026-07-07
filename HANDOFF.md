# Handoff — casehub-iot

**Head commit (project):** dcdf737 — fix(#47): align webapp with upstream API changes — imports, methods, quinoa bump

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up #45 (minor review findings) or parent#342 (PLATFORM.md update) — both are S/Low with no blockers. The CBR epic (#48–52) is ready for design work on #49 (infrastructure).

## What's Left

- casehubio/iot#45 — minor review findings: SSE tenancy filtering, platform scope, stub cleanup · S · Low
- casehubio/iot#46 — evaluate extracting webapp domain logic to application-tier repo · M · Med
- casehub-ras#23 — promote SituationDefinitionProvider to casehub-ras-api · S · Low
- ARC42STORIES.MD update for webapp modules — skipped at session end · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| parent#342 | PLATFORM.md update for webapp capabilities | XS | Low | Ready now |
| #48 | CBR epic — case-based reasoning for IoT | — | — | Design-first; start with #49 (infrastructure) |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver |
| — | Situation definition editor form in pages | M | Med | Runtime CRUD already wired |
| — | Webapp Quarkus augmentation datasource config | S | Med | 3 persistence units need runtime config |

## Cross-Module

**Resolved this session:**
- ~~engine#640~~ — blocking CaseInstanceRepository SPI → closed
- ~~work#286~~ — tenancy-aware query → closed
- ~~casehub-ras#24~~ — artifact name alignment → closed
- parent#355 — quarkus-quinoa 2.8.3 added to BOM → pushed

## Key References

- Spec: `docs/superpowers/specs/2026-07-01-iot-webapp-design.md`
- Garden: GE-20260707-50052f — Quarkiverse extension version mismatch gotcha
- GitHub repo: `casehubio/iot`
