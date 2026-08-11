---
layout: post
title: "Queue Endpoints and the Data That Isn't There"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [rest, queue, ai-resolution, design-review]
---

The AI resolution agent (#63) runs autonomously — polling, claiming, calling the LLM, executing device commands. What it doesn't do is tell anyone what it's up to. The queue has entries, but the only way to see them is to read the database. #81 adds the read surface: two REST endpoints that let a dashboard show what's waiting, what's being processed, and what got escalated.

The design question was how much to put on the wire. A queue entry by itself is thin — an ID, a status, a timestamp. The value is in joining it with what the IoT layer knows: which device, what room, what the CBR engine thinks is going on, what the AI decided (if it got that far). We settled on a list + detail split. The list carries enough device context to populate a dashboard row. The detail loads CBR suggestions and escalation context on demand.

The design review caught several things we'd missed. `CaseQueueService.escalate()` sets `viewName` to null on the target entry — the method takes a `UUID targetViewId` and simply doesn't resolve the name. Not documented anywhere in the engine API. The fix is a cached viewId-to-name mapping built at startup from `SubjectViewStore`. The review also flagged REVOKED entries silently appearing in unfiltered results, and the detail endpoint doing an O(N) scan across both views instead of using `findById`.

The most useful catch was during plan self-review, not the design review. I'd written the detail endpoint to read `aiResolutionResults` from the case context and cast it to `AiResolutionPlan`. Tracing the actual write path in `IoTAiResolutionAgent.processEntry()` showed the plan is ephemeral — computed by the LLM, used for execution, then discarded. What gets stored under `aiResolutionResults` is `List<ExecutedActionResult>`, not the plan. For escalated cases, `AiEscalationContext.partialPlan` captures the planned actions. For successful cases, only the execution results survive. The spec had `resolutionPlan` as a field — we removed it.

The pattern of enriching generic infrastructure with domain context is one that'll recur. The queue is an engine concept. The device class, the CBR similarity score, the escalation reason — those are IoT concepts. The endpoint lives at the join. When #77 adds SSE streaming for queue changes, the same response records serve both the REST snapshot and the event stream.
