# Handoff — casehub-iot

**Head commit (project):** 357928e — feat: CBR temporal recency weighting (#64)

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up the queue infrastructure pipeline. The dependency chain is:
1. casehubio/platform#175 (generic queue toolkit) — no blockers, start here
2. casehubio/engine#730 (case queue implementation) — blocked by platform#175
3. #62 (CBR-aware case triage) — blocked by engine#730
4. #63 (LLM resolution agent) — blocked by #62

Alternatively: #51 (work item outcome prediction) or #52 (false-positive suppression) — both independent of the queue pipeline.

## What Was Done (2026-07-17)

**Branch:** `issue-58-cbr-refinements` — closed, landed as `357928e`.
**Covers:** #58, #64, #65 — all three CBR refinements on one branch.

- **CbrRetentionJob** (#58) — `@Scheduled` purge calling `store.purge(CbrRetentionPolicy)`. Config-driven `max-age-days` and `max-cases-per-type`, skip-if-unconfigured. Depends on neocortex#150.
- **Temporal decay** (#64) — wired `CbrConfig.temporalDecayHalfLifeDays()` through `IoTCbrRetrievalService` into `CbrQuery.withTemporalDecay(HalfLife)`. Depends on engine#733 + neocortex#151.
- **Situation surfacing** (#65) — `GET /api/situations/{situationId}/suggestions` on `SituationResource`. Fixed `IoTCaseInputContributor` to propagate `situationId` unconditionally (design review catch).
- **Cross-repo issues filed and completed:** neocortex#150 (retention), neocortex#151 (temporal decay), engine#733 (CbrConfig field).
- **Design review:** 3 rounds, 13 issues, 12 verified, 1 accepted. $12.31.
- **Also fixed:** `CbrQuery.of()` and `ScoredCbrCase` constructor changes from upstream neocortex SNAPSHOT.

## Cross-Repo Changes

Issues filed and completed in neocortex (#150, #151) and engine (#733). No direct code changes — those repos implemented their own issues.

## What's Left

*Nothing trailing from this branch.*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| platform#175 | Generic queue toolkit | M | Med | Unblocks engine#730 |
| engine#730 | Case queue implementation | M | Med | Blocked by platform#175 |
| #62 | CBR-aware case triage | M | Med | Blocked by engine#730 |
| #63 | LLM resolution agent | L | High | Blocked by #62 |
| #51 | Work item outcome prediction via CBR | M | Med | Independent |
| #52 | False-positive suppression via CBR | M | Med | Independent |
| #46 | Evaluate webapp extraction to app tier | M | Med | |

## Key References

- Spec: `docs/superpowers/specs/2026-07-16-cbr-refinements-design.md`
- Design review: `~/adr/casehub-iot/cbr-refinements-20260716-145251/tracker.md`
- Garden: GE-20260717-34ba5f (IntelliJ MCP lifecycle timeout)
