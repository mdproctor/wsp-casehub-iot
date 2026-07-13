# Handoff — casehub-iot

**Head commit (project):** 0943115 — feat: CbrConfig wiring and schema registration in IoT CaseHubs (#49)

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Run `work-end` on branch `issue-49-cbr-infrastructure` to complete the close. Pre-close sweep and code review are done (0 findings). Remaining: rebase onto main, squash, push. Main has commit d91e0e8 (#59 fix: work-adapter artifact swap) — rebase will pick it up.

## What Was Done (2026-07-11 → 2026-07-13)

**Branch:** `issue-49-cbr-infrastructure` — implementation complete, work-end in progress.

- **JPA CbrCaseMemoryStore** — `memory-cbr-jpa` module in neocortex (428454a on neocortex main). 111 contract tests.
- **IoT feature schemas** — `IoTCbrFeatureSchemas` with similarity tables. **Extractors** with temporal derivation.
- **CbrConfig wiring** — all four CaseHubs configured. Schema registration at startup.
- **Design reviewed** — 4-round adversarial review, 16 issues, all resolved.
- **#59 fixed** — swapped stale `casehub-engine-work-adapter` → `casehub-work-engine-adapter`. Build green.
- **Garden** — GE-20260713-b879b2 (H2 JSONB gotcha).

## Cross-Repo Changes

- **neocortex** — 428454a on main: `memory-cbr-jpa` module.
- **work** — engine-adapter PlanItemStatus → TaskStatus fix (local, not pushed).
- **engine** — rebuilt locally, installed to ~/.m2.

## What's Left

- Complete work-end for #49 · XS · Low
- #57 — case file population with device metadata · M · Med
- #58 — CBR retention/purge policy · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #50 | Situation resolution suggestion via CBR | M | Med | Consumes #49 |
| #51 | Work item outcome prediction via CBR | M | Med | Consumes #49 |
| #52 | False-positive suppression via CBR | M | Med | Consumes #49 |
| #46 | Evaluate webapp extraction to app tier | M | Med | |

## Key References

- Spec: `docs/superpowers/specs/2026-07-12-cbr-infrastructure-design.md`
- Plan: `docs/plans/2026-07-12-cbr-infrastructure.md`
- Garden: GE-20260713-b879b2
- Review: `~/adr/casehub-iot/cbr-infrastructure-20260712-205510/`
