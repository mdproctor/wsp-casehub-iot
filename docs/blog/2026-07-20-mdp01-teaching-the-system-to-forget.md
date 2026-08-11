# Teaching the System to Forget

The suppression problem is one of those things that sounds simple until you trace the data flow. An operator dismisses a situation — motion sensor in the hallway at shift change, temperature blip during HVAC cycling — and the system should learn from that. Next time the same pattern fires, don't escalate it. Or at least flag it as probably noise.

The hard part isn't the CBR query. The hard part is where in the pipeline you intercept, and what you do with the answer.

## The interception question

The ras situation pipeline has a clean SPI for this: `RasTriggerPolicy`. It sits between ganglia detection ("something happened") and execution ("create a case about it"). The `DefaultRasTriggerPolicy` evaluates chain modes — And, Or, Threshold, Sequence — and returns TRIGGER or CONTINUE_ACCUMULATING. Pure signal processing, no historical context.

The obvious move is to override it. IoT provides its own policy that delegates to the default for chain evaluation, then checks CBR before returning the decision. If the pattern has been dismissed 90% of the time in similar contexts, suppress it. If 70%, demote it. If lower, annotate and let the operator decide.

But `RasTriggerPolicy` returned a bare `TriggerDecision` enum — TRIGGER, DISCARD, CONTINUE_ACCUMULATING. No room for metadata. And there was no SUPPRESS value distinct from DISCARD. DISCARD means "garbage signal." SUPPRESS means "valid signal, historically not worth acting on." Different audit trail, different dashboard treatment, different operator expectation.

So we changed ras. Added `PolicyDecision` — wraps the decision with a metadata map. Added `TriggerDecision.SUPPRESS`. Added `SituationChangeEvent.ChangeType.SUPPRESSED` and `DISMISSED`. The `SituationEvaluator` now threads policy metadata through to the case creation path, merging it into `baseCaseData` so the case carries its suppression annotation natively.

The ripple was contained: zero cross-repo references to these types outside ras itself.

## The feedback loop problem

The CBR case base stores operator evidence — dismissals, actioned situations, overrides. The system's own suppression decisions never enter the case base. This matters. If suppression decisions fed back as evidence, you'd get a self-reinforcing loop: suppress 10 situations → those 10 become evidence for future suppression → rate climbs → more suppressions. The system convinces itself to suppress everything, compounding away from operator ground truth.

Overrides are the correction mechanism. Operator says "this suppression was wrong" → counter-evidence enters the case base → future suppression rate drops for that pattern. The `SuppressionLogEntry` records every suppression decision with the full `SituationContext` snapshot, so the override can reconstruct and fire the case that was suppressed.

## The safety gate

Safety-critical situations — smoke, CO, water leak — are exempt from all suppression tiers regardless of dismissal history. The gate checks at two levels: situation ID first (catches NotifyOnly actions like fire alarm notifications), then case type as defense-in-depth for CreateCase actions. Both reference `IoTSafetyCaseTypes`, the same constant set used by `IoTActionRiskClassifier`. One source of truth.

## What landed

The graduated model works across three tiers. Annotate tags the case with suppression metadata — the operator sees it but knows the history. Demote fires a SUPPRESSED event without creating a case — the dashboard shows it prominently for override. Full suppress does the same with lower visibility. Both tiers 2 and 3 produce SUPPRESS at the ras level; the distinction is in the metadata, governing operator attention rather than system behavior.

The feature extraction reuses the existing CBR common fields — deviceClass, roomType, hourOfDay, dayType, season — plus detectionConfidence. The `situationId` partitions the case base via the caseType parameter, so dismissing motion-at-night can't affect temperature-threshold suppression. Cross-situation learning is a different problem (device health) that belongs at a different abstraction level.
