# LLM Resolution Agent — IoTAiResolutionAgent

**Date:** 2026-07-29
**Issue:** casehubio/iot#63
**Parent spec:** `docs/superpowers/specs/2026-07-14-cbr-situation-resolution-design.md` §7

## Summary

When the triage pipeline (#62) routes a case to the `iot-ai-resolution` queue
view (high CBR similarity + consistent outcomes), an LLM agent claims the entry,
loads CBR suggestions as context, calls the LLM to produce a resolution plan,
risk-classifies each action, and either executes directly or escalates to a
human operator.

## Architecture Deviation from Parent Spec

The parent spec (§7) designed against an assumed architecture that diverged from
what was built:

| Parent spec assumes | Actual architecture |
|---|---|
| `CaseQueueEntryCreated` CDI event | No such event. `CaseQueueEntryManager` creates entries on `@Observes CaseQueueEvent(ADDED)` |
| `CaseQueueEntry.metadata` field | No metadata field. Case working layer stores operational state |
| Status: PENDING/CLAIMED/COMPLETED/ESCALATED | PENDING/CLAIMED/REVOKED. Completion implicit via label lifecycle |
| `CaseQueueRoutingStrategy` SPI | Label rules → SubjectViews → CaseQueueEvent pipeline |
| Direct LLM client (Anthropic SDK) | Engine has full `Agent` infrastructure: langchain4j `ChatModel`, `AgentBuilder`, structured output |

This spec adapts §7 to the actual architecture. One engine modification is
required: adding `findByView(UUID viewId, String tenancyId)` to
`CaseQueueService` to expose all entries regardless of status (see §8).

---

## §1 Component Architecture

### Module placement

| Module | Component | Purpose |
|--------|-----------|---------|
| `webapp-api` (Tier 1) | `AiResolutionPlan`, `AiEscalationContext`, `PlannedActionSpec` | Pure data records. CDI-free, testable without container |
| `webapp-api` (Tier 1) | `AiResolutionPromptBuilder` | Builds LLM prompt from case context + suggestions + available actions |
| `webapp` | `IoTAiResolutionAgent` | CDI bean. `@Scheduled` polling, queue ops, LLM calls, action execution |
| `webapp` | `IoTAiResolutionConfig` | `@ConfigMapping` for agent configuration |

### Key classes

**`IoTAiResolutionAgent`** — `@ApplicationScoped`. Single `@Scheduled` method
polls for PENDING entries in the `iot-ai-resolution` view, claims them, and
processes each on a virtual thread bounded by a `Semaphore` (see §2
Concurrency). Same method handles the timeout sweep for stale CLAIMED entries.

**`AiResolutionPromptBuilder`** — constructs the LLM user message from three
inputs: case context (device class, room type, readings), CBR suggestions
(similarity scores, outcomes, plan steps), and available actions (action types
that `IoTActionRiskClassifier` classifies as `Autonomous`).

**`AiResolutionPlan`** — structured LLM output record:

```java
record AiResolutionPlan(
    Decision decision,              // EXECUTE or ESCALATE
    String reasoning,
    List<PlannedActionSpec> actions, // empty if ESCALATE
    String escalationReason         // null if EXECUTE
)

enum Decision { EXECUTE, ESCALATE }
```

**`PlannedActionSpec`** — a single action proposed by the LLM:

```java
record PlannedActionSpec(
    String actionType,          // e.g. SET_TEMPERATURE, TURN_OFF
    String targetDeviceId,
    Map<String, Object> parameters,
    String rationale
)
```

**`ExecutedActionResult`** — tracks actual execution outcome per action:

```java
record ExecutedActionResult(
    PlannedActionSpec action,
    boolean succeeded,
    String outcome              // success description or failure reason
)
```

**`AiEscalationContext`** — written to the case working layer before the LLM
call (survives timeout/crash):

```java
record AiEscalationContext(
    String reason,
    List<ResolutionSuggestion> consideredSuggestions,
    String partialAnalysis,              // null pre-LLM, populated post-LLM
    List<PlannedActionSpec> partialPlan,  // null pre-LLM, populated post-LLM
    List<ExecutedActionResult> executedActions  // null until execution, populated on partial failure
)
```

---

## §2 Event Flow

```
Case starts
  → engine injects CBR stats to working context
  → CaseLabelEvaluator fires → iot-triage:ai-resolution label added
  → SubjectViewOrchestrator → CaseQueueEvent(ADDED)
  → CaseQueueEntryManager creates CaseQueueEntry(PENDING)

  ... ≤10s ...

IoTAiResolutionAgent @Scheduled poll
  → CaseQueueService.findPending(aiResolutionViewId, tenancyId)
  → for each PENDING entry (acquire semaphore permit first):
      1. CaseQueueService.claim(entryId, tenancyId, "iot-ai-agent")
      2. CaseInstanceCache.get(caseId) → load case
      3. IoTCbrRetrievalService.retrieve() → CBR suggestions
      4. Write AiEscalationContext to working layer (pre-LLM snapshot)
      5. Agent.execute(Map.of("prompt", promptString))
         → WorkerResult<Map<String, Object>>
         → deserialize map to AiResolutionPlan via ObjectMapper
         (Agent.execute() throws AgentException on failure — it never
         returns WorkerResult.failed())
      6. If decision=ESCALATE → CaseQueueService.escalate(→ operator-assisted)
      7. If decision=EXECUTE:
         a. IoTActionRiskClassifier.classify() each action
         b. ANY GateRequired → escalate whole case
         c. ALL Autonomous → execute via worker functions
         d. Write results to case context
         e. Entry stays CLAIMED. Case resolution is driven by the
            real-world feedback loop: device responds → sensor detects
            improvement → situation resolves → case resolves → labels
            change → entry revoked naturally
```

### Why polling over CDI event observation

`CaseQueueEntryManager` uses `@Observes CaseQueueEvent` to create the entry.
A second `@Observes` handler on the agent has no guaranteed ordering — the agent
may fire before the entry exists. Since `CaseQueueEntryManager` is in the
engine jar, its priority cannot be modified.

`@Scheduled` polling via `CaseQueueService.findPending()` avoids this entirely.
10s latency is imperceptible for AI resolution (not safety-critical).

### Concurrency control

A `Semaphore(maxConcurrentLlmCalls)` bounds concurrent LLM API calls. Each
virtual thread acquires a permit before calling `Agent.execute()` and releases
it in a `finally` block. Default: 3 concurrent calls. This prevents rate-limit
cascades and cost spikes when many cases queue up between polls. The semaphore
bounds the LLM API bottleneck — virtual threads make the concurrency itself
cheap.

### Timeout sweep

The same `@Scheduled` method handles stale claims:
`CaseQueueService.findByView(aiResolutionViewId, tenancyId)` returns all entries
regardless of status. Filter for `status == CLAIMED` where
`claimedAt + timeout < now()`, escalate each to `iot-operator-assisted`. The
`AiEscalationContext` was written to the working layer before the LLM call, so
partial context survives.

### View ID resolution

The agent resolves the `iot-ai-resolution` and `iot-operator-assisted` view IDs
once at startup from `SubjectViewStore.findByTenancy()` and caches them.
Tenancy ID is injected via `@ConfigProperty(name = "casehub.iot.tenancy-id")`,
consistent with the platform-wide single tenancy property convention
(ARC42STORIES §8).

---

## §3 LLM Integration

Uses the engine's existing `Agent` infrastructure (langchain4j):

```java
// Built once at startup — userMessage is a PromptTemplate with {{prompt}} placeholder
Agent agent = Agent.builder()
    .systemPrompt(SYSTEM_PROMPT)
    .userMessage("{{prompt}}")
    .model(ModelType.ANTHROPIC)
    .responseSchema(AiResolutionPlan.class)
    .build();

// Per invocation — prompt string built per case, passed via execute input map
String prompt = AiResolutionPromptBuilder.build(context, suggestions, actions);
WorkerResult<Map<String, Object>> result = agent.execute(Map.of("prompt", prompt));
AiResolutionPlan plan = objectMapper.convertValue(result.output(), AiResolutionPlan.class);
```

The `Agent` instance is built once at startup and reused. `ChatModel` is
stateless. The `userMessage` is set as a `PromptTemplate` with a `{{prompt}}`
placeholder — `Agent.execute()` substitutes it from the input map via
langchain4j `PromptTemplate.apply()`. `AiResolutionPromptBuilder` constructs
the full prompt string per case; this is passed through `execute()`, not
through the builder.

`Agent.execute()` returns `WorkerResult<Map<String, Object>>`. The map is
deserialized to `AiResolutionPlan` via `ObjectMapper.convertValue()`. On
failure (empty response, invalid JSON, template error), `Agent.execute()`
throws `AgentException` — it never returns a failed `WorkerResult`.

### Prompt structure

`AiResolutionPromptBuilder` constructs the user message from:

1. **Case context** — device class, room type, event timestamp, current
   readings, situation description (from working layer)
2. **CBR suggestions** — top-N past resolutions with similarity scores,
   outcomes, and plan steps
3. **Available actions** — action types that `IoTActionRiskClassifier`
   classifies as `Autonomous` for this case type (so the LLM doesn't propose
   actions that will trigger escalation)

### Autonomy levels

The parent spec defined three levels (plan selection, adaptation, generation).
These are not separate code paths — the LLM receives high-quality CBR context
(≥0.85 similarity is the queue entry threshold) and decides naturally whether
to reuse a past plan, adapt parameters, or generate a new one based on context
fit. The prompt does not mention autonomy levels.

---

## §4 Execution Model

The agent IS the execution owner for AI-resolved cases. It does not submit
to the engine's plan execution pipeline — that pipeline is designed for
CaseHub-defined plans (static, defined at case type design time), not
dynamically-submitted action plans.

### Execution flow

For each `PlannedActionSpec` in the LLM's plan:

1. Build `PlannedAction(rationale, actionType, Map.of())` for risk classification
   (`targetDeviceId` and `parameters` are not needed for risk classification —
   `IoTActionRiskClassifier` uses only `actionType` and `context.caseDefinitionName()`)
2. Build `ClassificationContext` from the case instance
3. Call `IoTActionRiskClassifier.classify(action, context)`
4. If ANY action is `GateRequired` → **stop, escalate entire case**. The
   rationale: if the AI proposes an action needing human approval, the whole
   resolution attempt should go to a human.
5. **Status guard** — re-read the queue entry via
   `CaseQueueEntryStore.findById(entryId)` and verify
   `status == CLAIMED && viewId == aiResolutionViewId`. If the entry has been
   moved (by timeout sweep or manual intervention), **abort without executing**.
   This prevents the race where the timeout sweep escalates the entry between
   the LLM call completing and action execution beginning.
6. If ALL actions are `Autonomous` → execute each sequentially via the
   appropriate worker function. Build the worker input map directly from
   `PlannedActionSpec`:
   ```java
   Map.of(
       "targetDeviceId", spec.targetDeviceId(),  // top level — DeviceCommandWorkerFunction reads it here
       "action", spec.actionType(),               // top level
       "parameters", spec.parameters()            // nested map
   )
   ```
   Track each result as an `ExecutedActionResult`. On any
   `WorkerResult.failed()`, **stop executing remaining actions** — don't
   compound the problem with further device commands on a partially-applied
   plan.
7. **On full success:** write all results to `aiResolutionResults` in the
   case working layer. Entry stays CLAIMED.
8. **On partial failure:** write executed results (both successes and the
   failure) to `aiResolutionResults` immediately. Update
   `AiEscalationContext.executedActions` with the full execution record so the
   operator sees both what the AI planned and what actually happened. Escalate
   to `iot-operator-assisted`. The operator can see: which actions took
   real-world effect, which failed, and which were never attempted.

### Worker function resolution

The agent directly injects the dependencies for the device command worker:

```java
@Inject Instance<DeviceProvider> deviceProviders;
@Inject DeviceRegistry deviceRegistry;

// Constructed at startup
DeviceCommandWorkerFunction deviceCommandFn;
```

Action type → function mapping via `switch`:

```java
Function<Map<String, Object>, WorkerResult<Map<String, Object>>> resolveWorker(String actionType) {
    return switch (actionType) {
        case "SET_TEMPERATURE", "TURN_ON", "TURN_OFF",
             "SET_POSITION", "SET_VOLUME", "LOCK", "UNLOCK" -> deviceCommandFn;
        default -> throw new IllegalArgumentException("Unknown action type: " + actionType);
    };
}
```

Only action types with a real worker function backing are included. The `default`
branch catches unknown types. `NOTIFY_HOUSEHOLD` is excluded because
`HouseholdNotificationWorkerFunction` is currently a stub and `NOTIFY_HOUSEHOLD`
is not in `IoTActionRiskClassifier.AUTONOMOUS_ACTIONS` — the risk gate would
escalate the case before the switch is reached. Add it when the worker function
is implemented and the risk classifier permits autonomous notification.

This is IoT-specific by design. `WorkerFunctionProvider.handles(JsonNode)` takes
a full worker definition JSON node from the case model — it is designed for
engine-level routing, not action-type dispatch. `DeviceCommandWorkerFunction` is
a standalone `Function<Map, WorkerResult<Map>>`, not registered through
`WorkerFunctionProvider`.

### Case resolution

The agent does not resolve the case directly. It executes corrective actions
(e.g., set temperature, turn off device). The real-world feedback loop handles
resolution: the device responds → sensors detect improvement → the situation
clears → the case resolves through the normal lifecycle → labels change → the
queue entry is revoked naturally.

The entry stays CLAIMED after successful action execution. This is accurate —
the agent did its work. `CbrCaseRetainObserver` stores the resolution in the
case base when the case completes (feedback loop).

---

## §5 Escalation

Three paths, all using `CaseQueueService.escalate(entryId, tenancyId,
operatorAssistedViewId)`:

1. **LLM decides to escalate** — `decision == ESCALATE`. The LLM determined
   it can't resolve confidently. `escalationReason` stored in
   `AiEscalationContext`.

2. **Risk gate** — LLM proposed actions but one or more classified as
   `GateRequired`. Entire case escalates.

3. **Timeout** — `@Scheduled` sweep finds CLAIMED entries past deadline.
   `AiEscalationContext` was written pre-LLM.

4. **Partial worker failure** — one or more actions executed successfully but
   a subsequent action's `WorkerResult.failed()`. Remaining actions skipped.
   `AiEscalationContext.executedActions` shows the operator what took
   real-world effect.

### AiEscalationContext lifecycle

Written to case working layer (key `aiEscalationContext`) in two phases:

- **Pre-LLM:** `reason = "ai-resolution-in-progress"`, suggestions populated,
  analysis/plan null. Ensures context survives timeout or crash.
- **Post-LLM:** updated with reasoning and proposed actions before escalation.

### LLM failure — retry then escalate

Two distinct exception paths from `Agent.execute()`:

1. **`AgentException`** — thrown by `Agent.execute()` itself for template
   errors, invalid JSON, or empty response. These are **deterministic** —
   escalate immediately, no retry.
2. **Other exceptions** — `model.chat()` is called without a try-catch inside
   `Agent.execute()`, so HTTP-level failures (timeouts, 429, 5xx, connection
   errors) propagate as raw exceptions from the langchain4j `ChatModel`
   implementation. Classify these as **transient** and retry.

```java
try {
    result = agent.execute(input);
} catch (AgentException e) {
    // template error, JSON parse, empty response — deterministic, escalate
    escalateImmediately(e);
} catch (Exception e) {
    // model.chat() HTTP failure — classify and retry if transient
    if (isTransient(e)) retryWithBackoff(e);
    else escalateImmediately(e);
}
```

Transient classification: exception or cause chain contains
`HttpTimeoutException`, `ConnectException`, or HTTP status 429/5xx.
Up to 2 retries with exponential backoff (5s, 15s).

After retries are exhausted, write failure reason to `AiEscalationContext`
and escalate. Never silently drop a case.

---

## §6 Configuration

```properties
casehub.iot.ai-resolution.enabled=true
casehub.iot.ai-resolution.poll-interval=10s
casehub.iot.ai-resolution.timeout-seconds=300
casehub.iot.ai-resolution.model-type=ANTHROPIC
casehub.iot.ai-resolution.agent-id=iot-ai-agent
casehub.iot.ai-resolution.max-concurrent-llm-calls=3
```

```java
@ConfigMapping(prefix = "casehub.iot.ai-resolution")
public interface IoTAiResolutionConfig {
    @WithDefault("true")
    boolean enabled();

    @WithDefault("10s")
    Duration pollInterval();

    @WithDefault("300")
    int timeoutSeconds();

    @WithDefault("ANTHROPIC")
    ModelType modelType();

    @WithDefault("iot-ai-agent")
    String agentId();

    @WithDefault("3")
    int maxConcurrentLlmCalls();
}
```

When `enabled = false`, the `@Scheduled` poll is a no-op. Cases sit in the
AI resolution view until either the config is flipped or an operator manually
claims them.

All properties use `@WithDefault` — no SmallRye startup validation failures.

---

## §7 Testing Strategy

### webapp-api (unit, no CDI)

- **`AiResolutionPromptBuilderTest`** — prompt construction from case
  context + suggestions + available actions. Boundaries: empty suggestions,
  missing context fields, large suggestion lists.
- **`AiResolutionPlanTest`** — JSON deserialization of structured LLM output.
  Boundaries: missing fields, invalid action types.

### webapp (integration, CDI)

- **`IoTAiResolutionAgentTest`** — full flow with mock `ChatModel`:
  - Happy path: case → queue → claim → LLM EXECUTE → Autonomous actions →
    worker functions → case resolved
  - LLM escalation: decision=ESCALATE → entry to operator-assisted →
    `AiEscalationContext` in working layer
  - Risk gate: LLM EXECUTE but GateRequired action → escalation
  - LLM transient failure + retry success: mock `ChatModel` throws
    connection error on first call, succeeds on second → no escalation
  - LLM transient failure + retry exhaustion: all 3 attempts throw
    transient error → escalate with final error context
  - LLM deterministic failure: `AgentException` (invalid JSON) → escalate
    immediately, no retry
  - Timeout sweep vs active thread race: entry escalated by sweep before
    action execution → status guard aborts, no device commands dispatched
  - Partial worker failure: 3-action plan, action 1 succeeds, action 2
    returns `WorkerResult.failed()` → action 3 not attempted, results
    written with executed state, `AiEscalationContext.executedActions`
    shows completed/failed/skipped, case escalated
  - Timeout: stale CLAIMED entry → sweep escalates
  - Concurrent claim: two cycles claim same entry → `claimIfPending`
    optimistic locking → only one succeeds
  - Disabled: `enabled=false` → no processing

### Not tested here

Label routing (covered by #62 `IoTTriageLabelRulesTest`), CBR retrieval
(covered by #50), risk classification (covered by `IoTActionRiskClassifierTest`).

---

## §8 Dependencies

### New dependency for webapp

`casehub-engine-queue` (already in local maven, 0.2-SNAPSHOT). Per
GE-20260721-076719, also add `casehub-platform-view` to the IoT parent POM
`<dependencyManagement>` — transitive resolution fails without it.

### Engine modification

One addition to `CaseQueueService` in `casehub-engine-queue`:

```java
public List<CaseQueueEntry> findByView(UUID viewId, String tenancyId) {
    return this.store.findByView(viewId, tenancyId);
}
```

`findPending()` already calls `store.findByView()` and filters to PENDING.
This method exposes the unfiltered list through the service layer, maintaining
tenancy scoping (the store's `findByView` takes `tenancyId`). Required by the
timeout sweep, which needs CLAIMED entries where `claimedAt + timeout < now()`.

No other engine modifications required.

---

## §9 Scope Boundaries

**In scope:**
- `IoTAiResolutionAgent` with `@Scheduled` polling, claim, LLM call, execution
- `AiResolutionPromptBuilder` for prompt construction
- `AiEscalationContext` for human handoff context
- Configuration, timeout sweep, error handling
- Tests for all paths

**Out of scope** (tracked as GitHub issues):
- Queue listing REST endpoints (#81)
- Re-routing on context changes / CBR re-evaluation (#82)
- Multi-turn LLM conversation (#83)
- Custom model fine-tuning or prompt versioning (#84)
- Agent performance metrics/observability (#85)
