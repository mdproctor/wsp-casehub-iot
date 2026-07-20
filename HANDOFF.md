*Updated: engine#730 closed — removed from backlog, #62 now unblocked.*

# Handoff — casehub-iot

**Head commit (project):** 9f51997 — feat: false-positive suppression via CBR (#52)

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up the queue infrastructure pipeline. The dependency chain is:
1. ~~casehubio/platform#175 (generic queue toolkit)~~ — **CLOSED**
2. ~~casehubio/engine#730 (case queue implementation)~~ — **CLOSED**
3. #62 (CBR-aware case triage) — **unblocked**, start here
4. #63 (LLM resolution agent) — blocked by #62

Alternatively: #46 (evaluate webapp extraction to app tier) — independent.

## What Was Done (2026-07-20)

**Branch:** `issue-52-false-positive-suppression-cbr` — closed, landed as `9f51997`.
**Covers:** #52.

- **Cross-repo ras enhancement** — `PolicyDecision` (metadata on trigger decisions), `TriggerDecision.SUPPRESS`, `SituationChangeEvent` metadata field + `SUPPRESSED`/`DISMISSED` change types, `SuppressionMetadataKeys`. ras branch: `issue-52-policy-decision-suppress`.
- **IoTSuppressionTriggerPolicy** — overrides `DefaultRasTriggerPolicy`, delegates chain-mode evaluation then checks CBR for dismissal history. Three-tier graduated model: annotate, demote, full suppress.
- **SuppressionEvaluator** — queries CBR case base for similar dismissed situations, computes dismissal rate + tier.
- **DismissalRecorder** — records operator dismissals and case outcomes as CBR evidence. Two pathways: situation-level and case-level.
- **SuppressionLogEntry** — JPA entity + Flyway V503 for audit trail. CDI observer persists on `SituationChangeEvent(SUPPRESSED)`.
- **REST endpoints** — dismiss, suppression history, override, suppression stats on `SituationResource`.
- **Safety gate** — two-level check (situation ID + case type) via shared `IoTSafetyCaseTypes`. Never suppresses safety-critical situations.
- **Design review:** 4 rounds, 21 issues, 19 verified, 2 accepted. $15.48.
- **Follow-up:** #68 (DismissalGangliaObserver — close ganglia on situation dismissal).

## Cross-Repo Changes

ras branch `issue-52-policy-decision-suppress` (2 commits) — not yet merged to ras main. Needs merging before next ras consumer updates.

## What's Left

- #68 — DismissalGangliaObserver: close ganglia on situation dismissal · XS · Low
- #66 — sync CBR situation resolution spec §4-5 (queue → subject view toolkit) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #62 | CBR-aware case triage | M | Med | **Unblocked** — engine#730 landed |
| #63 | LLM resolution agent | L | High | Blocked by #62 |
| #46 | Evaluate webapp extraction to app tier | M | Med | |

## Key References

- Spec: `docs/superpowers/specs/2026-07-20-false-positive-suppression-design.md`
- Design review: `~/adr/casehub-iot/false-positive-suppression-20260720-012536/tracker.md`
- Blog: `2026-07-20-mdp01-teaching-the-system-to-forget.md`
