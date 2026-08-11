---
layout: post
title: "The Case Type That Was Already There"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [cbr, work-items, prediction, feature-extraction, architecture]
series: issue-51-work-item-outcome-prediction
---

The CBR system had three case types: `TextualCbrCase` for text-only matching, `FeatureVectorCbrCase` for structured feature matching, and `PlanCbrCase` for plan resolution with trace data. Every consumer used `PlanCbrCase`. The stores — Qdrant, JPA, in-memory — had hardcoded serialization for each type in switch statements. Adding a new type meant modifying three deserializers across neocortex.

I was about to propose a `WorkItemCbrCase` when I traced the serializer code and found the answer sitting in plain sight. `FeatureVectorCbrCase` is `PlanCbrCase` without the plan trace — problem, solution, outcome, confidence, and a rich feature map. It's already in every switch statement. The `FeatureValue` sealed interface supports strings, numbers, structs, and struct lists — expressive enough to carry everything work item prediction needs. The `caseType` string on `CbrQuery` handles segregation: `"iot-work-item"` never mixes with `"plan"` cases.

The real contribution of #51 isn't a new case type. It's the prediction aggregation layer — something the CBR system didn't have. The existing `IoTCbrRetrievalService` returns individual similar cases. Work item prediction needs aggregate analysis: outcome distribution weighted by similarity score, resolution time percentiles filtered to COMPLETED-only items (CANCELLED durations measure abandonment, not resolution), assignee rankings with controllable success rates that exclude external cancellations from the denominator.

`WorkItemPredictionService` takes scored CBR results and computes all of this. Confidence combines mean similarity with a log-scaled sample size factor — a single high-match case gives low confidence because the statistics are thin. The aggregation is stateless computation over `List<ScoredCbrCase<FeatureVectorCbrCase>>`, testable with a trivial stub store.

Feature extraction turned out cleaner than I expected. `IoTCbrFeatureExtractors` already had the temporal derivation logic (hour of day, weekday/weekend, season from timestamps) and the device-class/room-type extraction from case contexts. I added an `Instant` overload to `deriveTemporalFeatures` so both the existing `ReadableLayer`-based extractors and the new `WorkItemFeatureExtractor` share the same derivation. One extractor, two methods: `extractForRetain` includes output data (resolution duration, assignee, terminal status) for storage; `extractForRetrieve` omits them for queries.

The `HumanDecisionWorkerFunction` had been a stub since the webapp was built — returning a mock "approved" outcome without creating any work items. Replacing it was straightforward: inject `WorkItemCreator`, build `WorkItemCreateRequest` with title, priority mapping, candidate groups, and a JSON payload carrying the IoT context (case ID, device class, room type, event timestamp). That payload is what makes the retain path work — when a work item completes, the observer reads context straight from the work item's own payload instead of traversing back to the case through the instance cache.

That traversal is still there as a fallback for work items created before this feature ships. The `CaseInstanceCache` scan filters by tenancy first to prevent cross-tenant context leakage — a detail the design review caught. The review ran four rounds, surfaced 21 issues. The most impactful: `FeatureVectorCbrCase` rejects blank `solution` strings, and CANCELLED/EXPIRED work items have no resolution text. The fix is a fallback chain: `resolution ?? outcome ?? event.detail ?? status.name()`. The status name is always non-blank for terminal statuses — it's the safety net that prevents the recorder from crashing on edge cases.

The stores didn't need refactoring for this feature, but the closed type set is a design flaw waiting for its third consumer. When the next CBR domain appears — incident patterns, agent routing — someone will trace the same serializer switch statements I did and reach the same conclusion: `FeatureVectorCbrCase` works, but the stores should be type-agnostic. That's a neocortex issue, not an IoT one.
