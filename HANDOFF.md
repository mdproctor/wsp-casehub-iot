# Handoff — casehub-iot

**Head commit (project):** 52b5fde — feat: CBR-aware case triage (#62)

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

The queue infrastructure pipeline is now complete through triage. Next:
1. ~~casehubio/platform#175 (generic queue toolkit)~~ — **CLOSED**
2. ~~casehubio/engine#730 (case queue implementation)~~ — **CLOSED**
3. ~~#62 (CBR-aware case triage)~~ — **CLOSED** (this session)
4. #63 (LLM resolution agent) — **unblocked**, start here

Alternatively: #46 (evaluate webapp extraction to app tier) — independent.

## What Was Done (2026-07-21)

**Branch:** `issue-62-cbr-case-triage` — closed, landed as `52b5fde`.
**Covers:** #62.

- **IoTTriageLabelRules** — mutually exclusive label rules routing cases to
  `iot-triage:{ai-resolution, operator-assisted, operator-manual}` based on
  `cbrBestSimilarity` and `cbrOutcomeConsistency` context scalars.
- **Safety override** — `SafetyAlertCaseHub` and `SecurityAlertCaseHub` set
  unconditional `iot-triage:immediate` labels, bypassing CBR.
- **IoTQueueViewInitializer** — creates four `SubjectViewSpec`s at startup,
  idempotent across restarts.
- **IoTTriageConfig** — `@ConfigMapping` for AI-resolution thresholds (0.85
  similarity, 0.80 consistency).
- **Engine cross-repo** — `CaseStartedEventHandler.injectCbrExperiences()`
  writes `cbrBestSimilarity`, `cbrMatchCount`, `cbrOutcomeConsistency` to
  working context. Engine branch: `issue-62-cbr-summary-stats`.
- **IoTActionRiskClassifier** — updated for changed `ActionRiskClassifier`
  SPI (PlannedAction + ClassificationContext), safety-alert autonomous override.
- **Design review:** 4 rounds, 15 issues, 14 verified, 1 accepted. $15.42.

## Cross-Repo Changes

Engine branch `issue-62-cbr-summary-stats` (1 commit) — not yet merged to engine main. Needs merging before next engine consumer updates.

ras branch `issue-52-policy-decision-suppress` (2 commits, from #52) — still not merged to ras main.

## What's Left

- #68 — DismissalGangliaObserver: close ganglia on situation dismissal · XS · Low
- #66 — sync CBR situation resolution spec §4-5 (queue → subject view toolkit) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #63 | LLM resolution agent | L | High | **Unblocked** — #62 landed |
| #46 | Evaluate webapp extraction to app tier | M | Med | |

## Key References

- Spec: `docs/superpowers/specs/2026-07-21-cbr-case-triage-design.md`
- Design review: `~/adr/casehub-iot/cbr-case-triage-20260721-171734/tracker.md`
- Plan: `plans/attic/issue-62-cbr-case-triage/2026-07-21-cbr-case-triage.md`
