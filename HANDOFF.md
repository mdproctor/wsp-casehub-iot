# Handoff — casehub-iot

**Head commit (project):** e43a181 — feat: situation resolution suggestion via CBR (#50)

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up the queue infrastructure pipeline. The dependency chain is:
1. casehubio/platform#175 (generic queue toolkit) — no blockers, start here
2. casehubio/engine#730 (case queue implementation) — blocked by platform#175
3. #62 (CBR-aware case triage) — blocked by engine#730
4. #63 (LLM resolution agent) — blocked by #62

Alternatively, pick up #58 (CBR retention/purge policy) — independent of the queue pipeline.

## What Was Done (2026-07-14)

**Branch:** `issue-50-cbr-situation-resolution` — closed, landed as `e43a181`.

- **IoTCbrRetrievalService** (webapp-api, Tier 1) — queries the CBR case base for similar past resolutions. Uses `CbrConfig` directly from `CaseDefinition` as single source of truth for weights, topK, minSimilarity.
- **ResolutionSuggestion** and **ResolutionConfidence** DTOs — structured results with per-feature similarity breakdown and outcome consistency computation.
- **REST endpoints** on `CaseResource`: `GET /api/cases/{caseId}/suggestions` (on-demand retrieval), `POST /api/cases/{caseId}/suggestions/{pastCaseId}/accept` (idempotent plan pre-fill).
- **Suggestions panel** in case detail TypeScript UI — "Resolution Suggestions" table between Worker Results and Actions.
- **Design spec** covering the full pipeline: CBR retrieval → case queues → triage → LLM resolution (§1-§8). Design review: 3 rounds, 24 issues resolved.
- **6 cross-repo issues filed**: platform#175 (queue toolkit), engine#730 (case queue), #62 (triage), #63 (LLM agent), #64 (temporal recency), #65 (situation surfacing).

## Cross-Repo Changes

None — all work in casehub-iot. Cross-repo issues filed but no code changes in other repos.

## What's Left

- #58 — CBR case base retention and purge policy · S · Low
- #64 — CBR temporal recency weighting · S · Med
- #65 — Situation-level suggestion surfacing · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| platform#175 | Generic queue toolkit | M | Med | Unblocks engine#730 |
| engine#730 | Case queue implementation | M | Med | Blocked by platform#175 |
| #62 | CBR-aware case triage | M | Med | Blocked by engine#730 |
| #63 | LLM resolution agent | L | High | Blocked by #62 |
| #51 | Work item outcome prediction via CBR | M | Med | Independent of queue pipeline |
| #52 | False-positive suppression via CBR | M | Med | Independent of queue pipeline |
| #46 | Evaluate webapp extraction to app tier | M | Med | |

## Key References

- Spec: `docs/superpowers/specs/2026-07-14-cbr-situation-resolution-design.md`
- Garden: GE-20260612-bd3b4d (degenerate CBR), GE-20260713-b879b2 (H2 JSONB)
- Design review: `~/adr/casehub-iot/cbr-situation-resolution-20260714-022613/tracker.md`
