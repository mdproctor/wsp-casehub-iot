---
layout: post
title: "From Retrieval to Resolution — CBR Gets Its Pipeline"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [cbr, retrieval, case-queues, architecture]
series: issue-50-cbr-situation-resolution
---

The CBR infrastructure work (#49) left a capable engine sitting idle. The case base could store resolved situations. The similarity scorer could compare them. The feature schemas could describe what matters. But nothing asked the question: "has this happened before?"

That's what #50 builds — the retrieval service. `IoTCbrRetrievalService` takes a case type, a feature map extracted from the current situation's working layer, and the `CbrConfig` already wired into each CaseHub's `augment()` method. It builds a `CbrQuery`, calls the store, and maps `ScoredCbrCase<PlanCbrCase>` results to `ResolutionSuggestion` records. The operator sees past resolutions ranked by similarity in the case detail UI.

Using `CbrConfig` directly from the `CaseDefinition` was a design review finding that eliminated an entire class I'd planned — `IoTCbrRetrievalConfigs`. The original spec had a separate static registry for retrieval weights, mirroring the retain-path weights. The reviewer pointed out the obvious: both paths read from the same config, so make them use the same object. One source of truth, zero synchronisation risk. The `SuggestedPlanStep` wrapper went the same way — `PlanTrace` from neocortex-memory-api was already the right type.

The bigger design conversation was about what happens after retrieval. The issue says "surface suggestions to the operator." But I started asking: what about situations that could be auto-resolved? The system already classifies some actions as autonomous (`IoTActionRiskClassifier` marks TURN_ON, SET_TEMPERATURE, and all safety-alert actions as `Autonomous`). If past cases consistently resolved the same way, and the current situation matches at high confidence — why show it to a human?

This led to case queues. Work items already have queue-like behaviour — assignment, candidate groups, priority, status lifecycle. Cases need the same thing, but cases live in the engine and work items live in the work module. The question was whether to make queues generic.

The answer is a platform toolkit. `AbstractQueueEntity` as a `@MappedSuperclass` and `AbstractQueueService` with JPA criteria queries — both in a new `platform-queue` module. Each domain extends these with its own entity and its own table. The engine gets `CaseQueueEntry` with a real FK to `case_instance`. No polymorphic table, no persistence mixing, no drift. Each domain's queue is a full table implementation, not a shared one.

The triage layer sits in the IoT webapp. `IoTCbrCaseQueueRoutingStrategy` computes `ResolutionConfidence` — not just similarity score but outcome consistency. A 0.90 match where past cases diverged on resolution is medium confidence, not high. Safety-critical case types never go to the AI queue regardless of score. Everything has a fallback: if CBR retrieval fails, cases land in the manual operator queue, not in limbo.

The LLM resolution agent claims from the AI queue, loads CBR suggestions, and decides how much autonomy to exercise. High confidence with consistent outcomes → pick the past plan. High confidence but context differs → adapt the plan. Medium confidence → reason from scratch with CBR as background. The design review caught that both the `CaseQueueRouter` and the LLM agent need `@ObservesAsync` — the engine fires lifecycle events via `fireAsync()`, and synchronous observers would never see them.

The implementation this session is the first phase: the retrieval service, REST endpoints, and a suggestions panel in the case detail UI. The queue infrastructure, triage, and LLM agent are tracked as cross-repo issues — `casehubio/platform#175` for the generic queue toolkit, `casehubio/engine#730` for the case queue implementation, and #62/#63 in this repo for triage and the LLM agent.

The retrieval service is Tier 1 — pure Java, no CDI annotations. A `IoTCbrRetrievalServiceProducer` in the webapp module creates the CDI bean. This keeps the service testable without a container and consumable by future non-CDI consumers. The REST endpoints sit on `CaseResource`, not `SituationResource`, because CBR operates on case context — features come from the case's working layer, and the case base stores resolved cases. Situation-level surfacing is a follow-up (#65) once `SituationResource.listActive()` integration exists.

The pipeline from retrieval to automated resolution is a single architectural arc across four repos. This session designed all of it and built the first piece. The remaining pieces — queue toolkit, case queue, triage, LLM agent — can proceed in parallel once the platform work lands.
