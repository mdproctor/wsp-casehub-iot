# Handoff — casehub-iot

**Head commit (project):** 4186748 — fix: docker-build job — add GitHub Packages credentials for Maven

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up #45 (minor review findings) or parent#342 (PLATFORM.md update) — both are S/Low with no blockers.

## What's Left

- casehubio/iot#45 — minor review findings: SSE tenancy filtering, platform scope, stub cleanup · S · Low
- casehubio/iot#46 — evaluate extracting webapp domain logic to application-tier repo · M · Med
- casehub-ras#23 — promote SituationDefinitionProvider to casehub-ras-api · S · Low
- 3 `@Disabled` webapp tests — JSONB deserialization of sealed `TriggerAction`/`ChainMode` needs Jackson `@JsonTypeInfo` in casehub-ras API · S · Med
- ARC42STORIES.MD update for webapp modules · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| parent#342 | PLATFORM.md update for webapp capabilities | XS | Low | Ready now |
| #48 | CBR epic — case-based reasoning for IoT | — | — | Design-first; start with #49 |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver |
| — | Situation definition editor form in pages | M | Med | Runtime CRUD already wired |

## Key References

- Garden: GE-20260707-50052f — Quarkiverse extension version mismatch gotcha
- CI: green (build + docker-build) as of 4186748
- GitHub repo: `casehubio/iot`
