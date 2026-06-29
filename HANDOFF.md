# Handoff — casehub-iot

**Head commit (project):** 33cc737 — chore: slim dependencyManagement — inherit from casehub-parent BOM (#319)

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Start #37 (Reactive `Uni<>` variant for `BridgeAuditStore`), then #38 (JPA `BridgeAuditStore` implementation). Both are deferred-by-design — build when a consuming app has the concrete need.

## What's Left

None — all filed issues are deferred by design.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #37 | Reactive Uni<> variant for BridgeAuditStore | S | Low | Build when a reactive query endpoint is needed |
| #38 | JPA BridgeAuditStore implementation | M | Med | Build when a consuming app deploys bridge-server with a database |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver needed |

## Cross-Module

**We delivered** (other modules can now use):
- `AuditObserverCoexistenceTest` — CDI wiring verification for both audit observers
- casehubio/parent#317 filed — PLATFORM.md needs BridgeAuditStore capability row

## Key References

- Spec: `docs/superpowers/specs/2026-06-28-audit-observer-coexistence-test-design.md`
- Diary: `blog/2026-06-29-mdp06-xs-test-design-review.md`
- GitHub repo: `casehubio/iot`
