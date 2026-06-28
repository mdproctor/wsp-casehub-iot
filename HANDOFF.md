# Handoff — casehub-iot

**Head commit (project):** 208ba21 — feat: BridgeAuditStore SPI — structured query/retrieval — Closes #35

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Repo is clean — three open issues (#37, #38, #39), all deferred. Pick from What's Next or start casehub-life integration work.

## What's Left

None — all filed issues are deferred by design (build when a consuming app has the concrete need).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #38 | JPA BridgeAuditStore implementation | M | Med | Build when a consuming app deploys bridge-server with a database |
| #37 | Reactive Uni<> variant for BridgeAuditStore | S | Low | Build when a reactive query endpoint is needed |
| #39 | Integration test for audit observer coexistence | XS | Low | @QuarkusTest verifying StoringBridgeAuditObserver + LoggingBridgeAuditObserver both fire |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver needed |

## Cross-Module

**We delivered** (other modules can now use):
- `BridgeAuditStore` SPI — programmatic query of bridge audit events via composable `BridgeAuditQuery`
- `InMemoryBridgeAuditStore` @DefaultBean — bounded ring buffer, zero-database deployments
- ARC42STORIES.MD Chapter 6 — deployment, discovery, audit trail fully documented
- casehubio/parent#317 filed — PLATFORM.md needs BridgeAuditStore capability row

## Key References

- Design spec: `docs/superpowers/specs/2026-06-28-bridge-audit-store-design.md`
- ARC42 C6 spec: `docs/superpowers/specs/2026-06-27-arc42stories-c6-update-design.md`
- Diary: `blog/2026-06-28-mdp06-store-spi-that-almost-wasnt.md`
- GitHub repo: `casehubio/iot`
