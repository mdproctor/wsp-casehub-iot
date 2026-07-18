# Handoff — casehub-iot

**Head commit (project):** 96fe952 — feat: work item outcome prediction via CBR (#51)

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up the queue infrastructure pipeline. The dependency chain is:
1. casehubio/platform#175 (generic queue toolkit) — no blockers, start here
2. casehubio/engine#730 (case queue implementation) — blocked by platform#175
3. #62 (CBR-aware case triage) — blocked by engine#730
4. #63 (LLM resolution agent) — blocked by #62

Alternatively: #52 (false-positive suppression via CBR) — independent.

## What Was Done (2026-07-18)

**Branch:** `issue-51-work-item-outcome-prediction` — closed, landed as `96fe952`.
**Covers:** #51.

- **WorkItemPredictionService** — aggregation layer computing weighted outcome distribution, resolution time percentiles (COMPLETED-only), and assignee rankings with controllable success rates. Uses `FeatureVectorCbrCase` (existing CBR type) with case type `"iot-work-item"`.
- **WorkItemOutcomeRecorder** — `WorkItemObserver` implementation storing completed work items as CBR cases. Reads IoT context from work item payload; falls back to `CaseInstanceCache` for pre-enrichment work items.
- **WorkItemFeatureExtractor** — shared extraction between retain and retrieve paths. Reuses `IoTCbrFeatureExtractors.deriveTemporalFeatures()` via new `Instant` overload.
- **HumanDecisionWorkerFunction** — replaced stub with real `WorkItemCreator` integration. Embeds IoT context (caseId, deviceClass, roomType, eventTimestamp) in work item payload.
- **REST endpoint** — `GET /api/workitems/{id}/prediction` on `WorkItemResource`.
- **Design review:** 4 rounds, 21 issues, 20 verified, 1 accepted. $16.69.
- **Garden entry:** GE-20260718-207fde (WorkItemService constructor NPE blocks subclassing for tests).

## Cross-Repo Changes

None. Used existing `FeatureVectorCbrCase` from neocortex — no cross-repo changes needed.

## What's Left

*Nothing trailing from this branch.*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| platform#175 | Generic queue toolkit | M | Med | Unblocks engine#730 |
| engine#730 | Case queue implementation | M | Med | Blocked by platform#175 |
| #62 | CBR-aware case triage | M | Med | Blocked by engine#730 |
| #63 | LLM resolution agent | L | High | Blocked by #62 |
| #52 | False-positive suppression via CBR | M | Med | Independent |
| #46 | Evaluate webapp extraction to app tier | M | Med | |

## Key References

- Spec: `docs/superpowers/specs/2026-07-18-work-item-outcome-prediction-design.md`
- Design review: `~/adr/casehub-iot/work-item-outcome-prediction-20260718-013523/tracker.md`
- Blog: `2026-07-18-mdp01-case-type-already-there.md`
- Garden: GE-20260718-207fde (WorkItemService constructor NPE), GE-20260717-34ba5f (IntelliJ MCP lifecycle timeout)
