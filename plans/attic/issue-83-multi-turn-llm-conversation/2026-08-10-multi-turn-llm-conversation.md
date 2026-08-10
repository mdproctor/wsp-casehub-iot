# Multi-Turn LLM Conversation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #83 — Multi-turn LLM conversation for complex resolutions
**Issue group:** #83

**Goal:** Add multi-turn conversation support to IoTAiResolutionAgent using
AgentProvider sessions with MCP tools, orchestrated by a blocks loop.

**Architecture:** The agent opens an `AgentSession` (platform API) with
read-only IoT MCP tools attached. A blocks `LoopBuilder` with a custom
`ExecutionBackend` manages conversation turns — the session maintains
conversation memory, the loop handles termination. Configuration-driven
routing (single/multi/auto) preserves backward compatibility.

**Tech Stack:** Java 21, Quarkus, casehub-platform-agent-api
(`AgentProvider`, `AgentSession`, `AgentEvent`), casehub-blocks
(`Patterns.loop()`, `ExecutionBackend`), casehub-iot-mcp (read-only
tool surface), Micrometer, JUnit 5 + Mockito

## Global Constraints

- All SPIs are blocking (virtual-thread-aligned per ADR-0005)
- `webapp-api` is Tier 1: no CDI, no Quarkus runtime deps
- Pre-release project: record field additions are acceptable
- All config properties must use `@WithDefault` to prevent SmallRye startup validation failure
- Single tenancy property: `casehub.iot.tenancy-id`

---

### Task 1: webapp-api records — TurnSignal, MultiTurnResponse, ConversationTurn, ConversationTranscript, ToolCall

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/TurnSignal.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/MultiTurnResponse.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ToolCall.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ConversationTurn.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ConversationTranscript.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/resolution/MultiTurnResponseTest.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/resolution/ConversationTranscriptTest.java`

**Interfaces:**
- Consumes: `PlannedActionSpec` (existing)
- Produces: `TurnSignal` enum, `MultiTurnResponse` record, `ToolCall` record, `ConversationTurn` record, `ConversationTranscript` record — used by Tasks 2-5

- [ ] **Step 1: Write failing tests for MultiTurnResponse**

```java
// MultiTurnResponseTest.java
package io.casehub.iot.webapp.resolution;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class MultiTurnResponseTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void deserializesResolvedSignal() throws Exception {
        String json = """
            {"signal":"RESOLVED","reasoning":"temp stable",
             "actions":[{"actionType":"SET_TEMPERATURE","targetDeviceId":"d1",
                         "parameters":{},"rationale":"cool down"}],
             "escalationReason":null,"informationNeeded":null}""";
        MultiTurnResponse r = MAPPER.readValue(json, MultiTurnResponse.class);
        assertThat(r.signal()).isEqualTo(TurnSignal.RESOLVED);
        assertThat(r.actions()).hasSize(1);
        assertThat(r.actions().get(0).actionType()).isEqualTo("SET_TEMPERATURE");
    }

    @Test
    void deserializesContinueSignal() throws Exception {
        String json = """
            {"signal":"CONTINUE","reasoning":"need more data",
             "actions":[],"escalationReason":null,
             "informationNeeded":"What is the outdoor temperature?"}""";
        MultiTurnResponse r = MAPPER.readValue(json, MultiTurnResponse.class);
        assertThat(r.signal()).isEqualTo(TurnSignal.CONTINUE);
        assertThat(r.informationNeeded()).isEqualTo("What is the outdoor temperature?");
    }

    @Test
    void deserializesEscalateSignal() throws Exception {
        String json = """
            {"signal":"ESCALATE","reasoning":"too complex",
             "actions":[],"escalationReason":"multiple failures",
             "informationNeeded":null}""";
        MultiTurnResponse r = MAPPER.readValue(json, MultiTurnResponse.class);
        assertThat(r.signal()).isEqualTo(TurnSignal.ESCALATE);
        assertThat(r.escalationReason()).isEqualTo("multiple failures");
    }

    @Test
    void missingActionsDefaultsToEmptyList() throws Exception {
        String json = """
            {"signal":"ESCALATE","reasoning":"r","escalationReason":"e"}""";
        MultiTurnResponse r = MAPPER.readValue(json, MultiTurnResponse.class);
        assertThat(r.actions()).isEmpty();
    }

    @Test
    void invalidJsonReturnsNull() {
        MultiTurnResponse r = MultiTurnResponse.tryParse("not json", MAPPER);
        assertThat(r).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn test -f webapp-api/pom.xml -Dtest=MultiTurnResponseTest -pl webapp-api`
Expected: compilation failure — classes don't exist yet

- [ ] **Step 3: Implement the records**

```java
// TurnSignal.java
package io.casehub.iot.webapp.resolution;

public enum TurnSignal { CONTINUE, RESOLVED, ESCALATE }
```

```java
// ToolCall.java
package io.casehub.iot.webapp.resolution;

public record ToolCall(String name, String arguments, String result, boolean isError) {}
```

```java
// MultiTurnResponse.java
package io.casehub.iot.webapp.resolution;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
public record MultiTurnResponse(
        TurnSignal signal,
        String reasoning,
        List<PlannedActionSpec> actions,
        String escalationReason,
        String informationNeeded
) {
    public MultiTurnResponse {
        actions = actions != null ? List.copyOf(actions) : List.of();
    }

    public static MultiTurnResponse tryParse(String json, ObjectMapper mapper) {
        try {
            return mapper.readValue(json, MultiTurnResponse.class);
        } catch (Exception e) {
            return null;
        }
    }
}
```

```java
// ConversationTurn.java
package io.casehub.iot.webapp.resolution;

import java.time.Instant;
import java.util.List;

public record ConversationTurn(
        int turnNumber,
        String query,
        String responseText,
        List<ToolCall> toolCalls,
        TurnSignal signal,
        Instant timestamp,
        int inputTokens,
        int outputTokens
) {
    public ConversationTurn {
        toolCalls = toolCalls != null ? List.copyOf(toolCalls) : List.of();
    }
}
```

```java
// ConversationTranscript.java
package io.casehub.iot.webapp.resolution;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public final class ConversationTranscript {

    private final List<ConversationTurn> turns = new ArrayList<>();
    private int totalInputTokens;
    private int totalOutputTokens;
    private int totalThinkingTokens;
    private long totalDurationMs;
    private Double totalCostUsd;

    public void addTurn(ConversationTurn turn) {
        turns.add(turn);
        totalInputTokens += turn.inputTokens();
        totalOutputTokens += turn.outputTokens();
    }

    public void addInvocationStats(int thinkingTokens, long durationMs, Double costUsd) {
        totalThinkingTokens += thinkingTokens;
        totalDurationMs += durationMs;
        if (costUsd != null) {
            totalCostUsd = (totalCostUsd != null ? totalCostUsd : 0.0) + costUsd;
        }
    }

    public List<ConversationTurn> turns() { return Collections.unmodifiableList(turns); }
    public int turnCount() { return turns.size(); }
    public int totalInputTokens() { return totalInputTokens; }
    public int totalOutputTokens() { return totalOutputTokens; }
    public int totalThinkingTokens() { return totalThinkingTokens; }
    public long totalDurationMs() { return totalDurationMs; }
    public Double totalCostUsd() { return totalCostUsd; }
}
```

- [ ] **Step 4: Write failing tests for ConversationTranscript**

```java
// ConversationTranscriptTest.java
package io.casehub.iot.webapp.resolution;

import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ConversationTranscriptTest {

    @Test
    void accumulatesTurnsAndTokens() {
        var transcript = new ConversationTranscript();
        transcript.addTurn(new ConversationTurn(1, "q1", "r1", List.of(),
                TurnSignal.CONTINUE, Instant.now(), 100, 50));
        transcript.addTurn(new ConversationTurn(2, "q2", "r2", List.of(),
                TurnSignal.RESOLVED, Instant.now(), 120, 60));

        assertThat(transcript.turnCount()).isEqualTo(2);
        assertThat(transcript.totalInputTokens()).isEqualTo(220);
        assertThat(transcript.totalOutputTokens()).isEqualTo(110);
    }

    @Test
    void accumulatesInvocationStats() {
        var transcript = new ConversationTranscript();
        transcript.addInvocationStats(10, 500, 0.01);
        transcript.addInvocationStats(15, 600, 0.02);

        assertThat(transcript.totalThinkingTokens()).isEqualTo(25);
        assertThat(transcript.totalDurationMs()).isEqualTo(1100);
        assertThat(transcript.totalCostUsd()).isEqualTo(0.03);
    }

    @Test
    void turnsAreUnmodifiable() {
        var transcript = new ConversationTranscript();
        transcript.addTurn(new ConversationTurn(1, "q", "r", List.of(),
                TurnSignal.RESOLVED, Instant.now(), 10, 5));
        org.junit.jupiter.api.Assertions.assertThrows(
                UnsupportedOperationException.class,
                () -> transcript.turns().add(null));
    }

    @Test
    void recordsToolCalls() {
        var toolCalls = List.of(
                new ToolCall("iot_get_state", "{\"deviceId\":\"d1\"}", "{\"temp\":22}", false));
        var transcript = new ConversationTranscript();
        transcript.addTurn(new ConversationTurn(1, "q", "r", toolCalls,
                TurnSignal.CONTINUE, Instant.now(), 50, 30));

        assertThat(transcript.turns().get(0).toolCalls()).hasSize(1);
        assertThat(transcript.turns().get(0).toolCalls().get(0).name()).isEqualTo("iot_get_state");
    }
}
```

- [ ] **Step 5: Run all tests to verify they pass**

Run: `/opt/homebrew/bin/mvn test -f webapp-api/pom.xml -Dtest="MultiTurnResponseTest,ConversationTranscriptTest" -pl webapp-api`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add webapp-api/src/main/java/io/casehub/iot/webapp/resolution/TurnSignal.java \
        webapp-api/src/main/java/io/casehub/iot/webapp/resolution/MultiTurnResponse.java \
        webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ToolCall.java \
        webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ConversationTurn.java \
        webapp-api/src/main/java/io/casehub/iot/webapp/resolution/ConversationTranscript.java \
        webapp-api/src/test/java/io/casehub/iot/webapp/resolution/MultiTurnResponseTest.java \
        webapp-api/src/test/java/io/casehub/iot/webapp/resolution/ConversationTranscriptTest.java
git commit -m "feat(#83): add multi-turn conversation records — TurnSignal, MultiTurnResponse, ConversationTranscript"
```

---

### Task 2: AiEscalationContext — add transcript field

**Files:**
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/resolution/AiEscalationContext.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java` (update constructor calls)
- Modify: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java` (update assertions)

**Interfaces:**
- Consumes: `ConversationTranscript` (Task 1)
- Produces: Updated `AiEscalationContext` with 6-arg constructor — used by Tasks 4, 5

- [ ] **Step 1: Add `transcript` field to AiEscalationContext**

Use `ide_edit_member` to modify the record. Add `ConversationTranscript transcript`
as the last field. Since this is pre-release, update all constructor call sites.

New record signature:
```java
public record AiEscalationContext(
        String reason,
        List<ResolutionSuggestion> consideredSuggestions,
        String partialAnalysis,
        List<PlannedActionSpec> partialPlan,
        List<ExecutedActionResult> executedActions,
        ConversationTranscript transcript
) {}
```

- [ ] **Step 2: Update all constructor call sites in IoTAiResolutionAgent**

Search with `ide_find_references` for `AiEscalationContext` constructor usages.
Add `null` as the last argument to every existing call in `writePreLlmContext()`
and `updateEscalationContext()`.

- [ ] **Step 3: Update test assertions if needed**

Run: `/opt/homebrew/bin/mvn test -f webapp/pom.xml -Dtest=IoTAiResolutionAgentTest -pl webapp`
Fix any compilation errors from the constructor change.

- [ ] **Step 4: Verify all existing tests still pass**

Run: `/opt/homebrew/bin/mvn test -f webapp/pom.xml -Dtest=IoTAiResolutionAgentTest -pl webapp`
Expected: all existing tests PASS (no behavioral change)

- [ ] **Step 5: Commit**

```bash
git add webapp-api/src/main/java/io/casehub/iot/webapp/resolution/AiEscalationContext.java \
        webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java \
        webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java
git commit -m "feat(#83): add ConversationTranscript field to AiEscalationContext"
```

---

### Task 3: AgentEventCollector + MultiTurnResolutionState

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/AgentEventCollector.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/CollectedTurn.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/MultiTurnResolutionState.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/AgentEventCollectorTest.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/MultiTurnResolutionStateTest.java`

**Interfaces:**
- Consumes: `MultiTurnResponse`, `ConversationTurn`, `ConversationTranscript`, `ToolCall`, `TurnSignal` (Task 1), `PlannedActionSpec` (existing), `AgentEvent` subtypes (platform API)
- Produces: `AgentEventCollector.collect(Multi<AgentEvent>) → CollectedTurn`, `MultiTurnResolutionState` with `isFirstTurn()`, `isTerminal()`, `withResolution()`, `withEscalation()`, `addTurn()` — used by Tasks 4, 5

- [ ] **Step 1: Write failing tests for AgentEventCollector**

```java
// AgentEventCollectorTest.java
package io.casehub.iot.webapp.app.resolution;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.webapp.resolution.TurnSignal;
import io.casehub.platform.agent.AgentEvent;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class AgentEventCollectorTest {

    private final AgentEventCollector collector = new AgentEventCollector(new ObjectMapper());

    @Test
    void collectsTextDeltasIntoFullText() {
        Multi<AgentEvent> events = Multi.createFrom().items(
                new AgentEvent.TextDelta("{\"signal\":\"RESOLVED\","),
                new AgentEvent.TextDelta("\"reasoning\":\"done\","),
                new AgentEvent.TextDelta("\"actions\":[],\"escalationReason\":null,\"informationNeeded\":null}"),
                new AgentEvent.InvocationComplete(100, 50, 10, 0, 0, 0.01, 500, 480, "s1", 1, false)
        );

        CollectedTurn turn = collector.collect(events);

        assertThat(turn.response()).isNotNull();
        assertThat(turn.response().signal()).isEqualTo(TurnSignal.RESOLVED);
        assertThat(turn.response().reasoning()).isEqualTo("done");
    }

    @Test
    void capturesToolCallsAndResults() {
        Multi<AgentEvent> events = Multi.createFrom().items(
                new AgentEvent.ToolCallComplete(0, "tc1", "iot_get_state", "{\"deviceId\":\"d1\"}"),
                new AgentEvent.ToolResult("tc1", "{\"temp\":22}", false),
                new AgentEvent.TextDelta("{\"signal\":\"RESOLVED\",\"reasoning\":\"r\",\"actions\":[],\"escalationReason\":null,\"informationNeeded\":null}"),
                new AgentEvent.InvocationComplete(100, 50, 0, 0, 0, null, 400, 380, "s1", 1, false)
        );

        CollectedTurn turn = collector.collect(events);

        assertThat(turn.toolCalls()).hasSize(1);
        assertThat(turn.toolCalls().get(0).name()).isEqualTo("iot_get_state");
        assertThat(turn.toolCalls().get(0).result()).isEqualTo("{\"temp\":22}");
        assertThat(turn.toolCalls().get(0).isError()).isFalse();
    }

    @Test
    void capturesTokenCounts() {
        Multi<AgentEvent> events = Multi.createFrom().items(
                new AgentEvent.TextDelta("{\"signal\":\"ESCALATE\",\"reasoning\":\"r\",\"actions\":[],\"escalationReason\":\"e\",\"informationNeeded\":null}"),
                new AgentEvent.InvocationComplete(200, 100, 30, 50, 10, 0.05, 1000, 950, "s1", 1, false)
        );

        CollectedTurn turn = collector.collect(events);

        assertThat(turn.completion().inputTokens()).isEqualTo(200);
        assertThat(turn.completion().outputTokens()).isEqualTo(100);
        assertThat(turn.completion().thinkingTokens()).isEqualTo(30);
    }

    @Test
    void invalidJsonReturnsNullResponse() {
        Multi<AgentEvent> events = Multi.createFrom().items(
                new AgentEvent.TextDelta("not valid json"),
                new AgentEvent.InvocationComplete(10, 5, 0, 0, 0, null, 100, 90, "s1", 1, false)
        );

        CollectedTurn turn = collector.collect(events);

        assertThat(turn.response()).isNull();
        assertThat(turn.rawText()).isEqualTo("not valid json");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn test -f webapp/pom.xml -Dtest=AgentEventCollectorTest -pl webapp`
Expected: compilation failure

- [ ] **Step 3: Add platform-agent-api dependency to webapp pom.xml**

Add to `webapp/pom.xml` `<dependencies>`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
</dependency>
```

And to parent `pom.xml` `<dependencyManagement>`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
    <version>${version.casehub.platform}</version>
</dependency>
```

- [ ] **Step 4: Implement CollectedTurn and AgentEventCollector**

```java
// CollectedTurn.java
package io.casehub.iot.webapp.app.resolution;

import io.casehub.iot.webapp.resolution.MultiTurnResponse;
import io.casehub.iot.webapp.resolution.ToolCall;
import io.casehub.platform.agent.AgentEvent;

import java.util.List;

public record CollectedTurn(
        MultiTurnResponse response,
        String rawText,
        List<ToolCall> toolCalls,
        AgentEvent.InvocationComplete completion
) {}
```

```java
// AgentEventCollector.java
package io.casehub.iot.webapp.app.resolution;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.iot.webapp.resolution.MultiTurnResponse;
import io.casehub.iot.webapp.resolution.ToolCall;
import io.casehub.platform.agent.AgentEvent;
import io.smallrye.mutiny.Multi;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class AgentEventCollector {

    private final ObjectMapper objectMapper;

    public AgentEventCollector(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    public CollectedTurn collect(Multi<AgentEvent> events) {
        StringBuilder text = new StringBuilder();
        Map<String, ToolCall> pendingToolCalls = new HashMap<>();
        List<ToolCall> completedToolCalls = new ArrayList<>();
        AgentEvent.InvocationComplete completion = null;

        for (AgentEvent event : events.subscribe().asIterable()) {
            switch (event) {
                case AgentEvent.TextDelta d -> text.append(d.text());
                case AgentEvent.ToolCallComplete tc -> pendingToolCalls.put(
                        tc.id(), new ToolCall(tc.name(), tc.arguments(), null, false));
                case AgentEvent.ToolResult tr -> {
                    ToolCall pending = pendingToolCalls.remove(tr.toolCallId());
                    if (pending != null) {
                        completedToolCalls.add(new ToolCall(
                                pending.name(), pending.arguments(), tr.content(), tr.isError()));
                    }
                }
                case AgentEvent.InvocationComplete ic -> completion = ic;
                default -> {}
            }
        }

        String rawText = text.toString();
        MultiTurnResponse response = MultiTurnResponse.tryParse(rawText, objectMapper);
        return new CollectedTurn(response, rawText, List.copyOf(completedToolCalls), completion);
    }
}
```

- [ ] **Step 5: Write failing tests for MultiTurnResolutionState**

```java
// MultiTurnResolutionStateTest.java
package io.casehub.iot.webapp.app.resolution;

import io.casehub.iot.webapp.resolution.ConversationTranscript;
import io.casehub.iot.webapp.resolution.MultiTurnResponse;
import io.casehub.iot.webapp.resolution.PlannedActionSpec;
import io.casehub.iot.webapp.resolution.TurnSignal;
import io.casehub.platform.agent.AgentEvent;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class MultiTurnResolutionStateTest {

    @Test
    void startsAsFirstTurnNotTerminal() {
        var state = new MultiTurnResolutionState(new ConversationTranscript());
        assertThat(state.isFirstTurn()).isTrue();
        assertThat(state.isTerminal()).isFalse();
    }

    @Test
    void withResolutionIsTerminal() {
        var state = new MultiTurnResolutionState(new ConversationTranscript());
        var actions = List.of(new PlannedActionSpec("TURN_OFF", "d1", Map.of(), "reason"));
        var resolved = state.withResolution(actions);
        assertThat(resolved.isTerminal()).isTrue();
        assertThat(resolved.resolution()).isEqualTo(actions);
    }

    @Test
    void withEscalationIsTerminal() {
        var state = new MultiTurnResolutionState(new ConversationTranscript());
        var escalated = state.withEscalation("too complex");
        assertThat(escalated.isTerminal()).isTrue();
        assertThat(escalated.escalationReason()).isEqualTo("too complex");
    }

    @Test
    void addTurnIncrementsTurnCount() {
        var state = new MultiTurnResolutionState(new ConversationTranscript());
        assertThat(state.isFirstTurn()).isTrue();

        var response = new MultiTurnResponse(TurnSignal.CONTINUE, "r", List.of(), null, "need data");
        var completion = new AgentEvent.InvocationComplete(50, 30, 0, 0, 0, null, 200, 180, "s", 1, false);
        var turn = new CollectedTurn(response, "{}", List.of(), completion);

        state.addTurn("query", turn);

        assertThat(state.isFirstTurn()).isFalse();
        assertThat(state.turnCount()).isEqualTo(1);
        assertThat(state.lastResponse()).isEqualTo(response);
        assertThat(state.transcript().turnCount()).isEqualTo(1);
    }
}
```

- [ ] **Step 6: Implement MultiTurnResolutionState**

```java
// MultiTurnResolutionState.java
package io.casehub.iot.webapp.app.resolution;

import io.casehub.iot.webapp.resolution.ConversationTranscript;
import io.casehub.iot.webapp.resolution.ConversationTurn;
import io.casehub.iot.webapp.resolution.MultiTurnResponse;
import io.casehub.iot.webapp.resolution.PlannedActionSpec;

import java.time.Instant;
import java.util.List;

public final class MultiTurnResolutionState {

    private final ConversationTranscript transcript;
    private int turnCount;
    private List<PlannedActionSpec> resolution;
    private String escalationReason;
    private MultiTurnResponse lastResponse;

    public MultiTurnResolutionState(ConversationTranscript transcript) {
        this.transcript = transcript;
    }

    public void addTurn(String query, CollectedTurn collectedTurn) {
        turnCount++;
        lastResponse = collectedTurn.response();
        var turn = new ConversationTurn(
                turnCount, query,
                collectedTurn.rawText(),
                collectedTurn.toolCalls(),
                collectedTurn.response() != null ? collectedTurn.response().signal() : null,
                Instant.now(),
                collectedTurn.completion() != null ? collectedTurn.completion().inputTokens() : 0,
                collectedTurn.completion() != null ? collectedTurn.completion().outputTokens() : 0);
        transcript.addTurn(turn);
        if (collectedTurn.completion() != null) {
            transcript.addInvocationStats(
                    collectedTurn.completion().thinkingTokens(),
                    collectedTurn.completion().durationMs(),
                    collectedTurn.completion().totalCostUsd());
        }
    }

    public MultiTurnResolutionState withResolution(List<PlannedActionSpec> actions) {
        this.resolution = actions;
        return this;
    }

    public MultiTurnResolutionState withEscalation(String reason) {
        this.escalationReason = reason;
        return this;
    }

    public boolean isFirstTurn() { return turnCount == 0; }
    public boolean isTerminal() { return resolution != null || escalationReason != null; }
    public int turnCount() { return turnCount; }
    public List<PlannedActionSpec> resolution() { return resolution; }
    public String escalationReason() { return escalationReason; }
    public MultiTurnResponse lastResponse() { return lastResponse; }
    public ConversationTranscript transcript() { return transcript; }
}
```

- [ ] **Step 7: Run all tests to verify they pass**

Run: `/opt/homebrew/bin/mvn test -f webapp/pom.xml -Dtest="AgentEventCollectorTest,MultiTurnResolutionStateTest" -pl webapp`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add webapp/pom.xml pom.xml \
        webapp/src/main/java/io/casehub/iot/webapp/app/resolution/AgentEventCollector.java \
        webapp/src/main/java/io/casehub/iot/webapp/app/resolution/CollectedTurn.java \
        webapp/src/main/java/io/casehub/iot/webapp/app/resolution/MultiTurnResolutionState.java \
        webapp/src/test/java/io/casehub/iot/webapp/app/resolution/AgentEventCollectorTest.java \
        webapp/src/test/java/io/casehub/iot/webapp/app/resolution/MultiTurnResolutionStateTest.java
git commit -m "feat(#83): AgentEventCollector and MultiTurnResolutionState"
```

---

### Task 4: IoTAiResolutionConfig — add multi-turn configuration properties

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionConfig.java`

**Interfaces:**
- Consumes: nothing new
- Produces: `conversationMode()` → `String` (single/multi/auto), `maxConversationTurns()` → `int`, `maxConcurrentSessions()` → `int` — used by Task 5

- [ ] **Step 1: Add config properties**

Use `ide_insert_member` to add three methods to `IoTAiResolutionConfig`:

```java
@WithName("conversation-mode")
@WithDefault("auto")
String conversationMode();

@WithName("max-conversation-turns")
@WithDefault("5")
int maxConversationTurns();

@WithName("max-concurrent-sessions")
@WithDefault("1")
int maxConcurrentSessions();
```

- [ ] **Step 2: Verify compilation**

Run: `/opt/homebrew/bin/mvn compile -f webapp/pom.xml -pl webapp`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionConfig.java
git commit -m "feat(#83): add multi-turn config — conversationMode, maxConversationTurns, maxConcurrentSessions"
```

---

### Task 5: Refactor IoTAiResolutionAgent — conversation mode routing with multi-turn support

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java` (add new tests, keep existing)

**Interfaces:**
- Consumes: `AgentProvider` (platform), `AgentSession`, `AgentSessionInit`, `AgentEvent`, `AgentMcpServer` (platform), `AgentEventCollector`, `CollectedTurn`, `MultiTurnResolutionState` (Task 3), `MultiTurnResponse`, `TurnSignal`, `ConversationTranscript` (Task 1), `IoTAiResolutionConfig.conversationMode/maxConversationTurns/maxConcurrentSessions` (Task 4)
- Produces: Refactored `processEntry()` with conversation mode routing

This is the largest task. It modifies the agent to support three modes
(single, multi, auto) and adds the conversation loop. Existing tests
must continue to pass — the `single` mode path is identical to the
current implementation.

- [ ] **Step 1: Write failing test — auto mode single-turn resolve**

```java
@Test
void autoMode_resolvesOnFirstTurn_noMultiTurn() {
    // Setup: conversation-mode=auto, mock AgentProvider.openSession
    // LLM returns RESOLVED on turn 1
    // Verify: session opened, query called once, session closed,
    //         actions executed via existing pipeline,
    //         aiConversationTranscript written to case context
}
```

Full test implementation uses mocked `AgentProvider`, `AgentSession` returning
`Multi.createFrom().items(...)` of `AgentEvent`s that form a RESOLVED
`MultiTurnResponse` JSON.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Add AgentProvider injection and session semaphore to IoTAiResolutionAgent**

Add fields:
```java
@Inject AgentProvider agentProvider;
private Semaphore sessionSemaphore;
private final ConcurrentHashMap<UUID, Instant> activeSessions = new ConcurrentHashMap<>();
```

In `init()`:
```java
sessionSemaphore = new Semaphore(config.maxConcurrentSessions());
```

- [ ] **Step 4: Implement conversation mode routing in processEntry()**

After claiming the entry and loading the case, route based on config:

```java
String mode = config.conversationMode();
if ("single".equals(mode)) {
    processSingleShot(entry, instance, suggestions);
} else {
    processMultiTurn(entry, instance, suggestions);
}
```

Extract existing single-shot logic into `processSingleShot()` (rename, no
behavior change). Implement `processMultiTurn()` with the session lifecycle:

```java
private void processMultiTurn(CaseQueueEntry entry, CaseInstance instance,
                               List<ResolutionSuggestion> suggestions) {
    if (!sessionSemaphore.tryAcquire()) {
        LOG.info("Session semaphore full — falling back to single-shot");
        registry.counter("casehub.iot.ai.resolution.session.fallback",
                "reason", "semaphore").increment();
        processSingleShot(entry, instance, suggestions);
        return;
    }

    activeSessions.put(entry.getCaseId(), Instant.now());
    AgentSession session = null;
    try {
        session = agentProvider.openSession(buildSessionInit(entry));
        var collector = new AgentEventCollector(objectMapper);
        var state = new MultiTurnResolutionState(new ConversationTranscript());

        int maxTurns = config.maxConversationTurns();
        for (int turn = 0; turn < maxTurns && !state.isTerminal(); turn++) {
            String query = state.isFirstTurn()
                    ? buildInitialQuery(instance, suggestions)
                    : buildFollowUpQuery(state);

            Multi<AgentEvent> events = session.query(query);
            CollectedTurn collected = collector.collect(events);
            state.addTurn(query, collected);

            if (collected.response() == null) {
                state.withEscalation("Failed to parse LLM response: " + collected.rawText());
            } else {
                switch (collected.response().signal()) {
                    case RESOLVED -> state.withResolution(collected.response().actions());
                    case ESCALATE -> state.withEscalation(collected.response().escalationReason());
                    case CONTINUE -> {} // loop continues
                }
            }
        }

        if (!state.isTerminal()) {
            state.withEscalation("Max conversation turns exceeded");
        }

        instance.getCaseContext().set("aiConversationTranscript", state.transcript());

        if (state.resolution() != null) {
            // Same risk-check + status-guard + execute pipeline as single-shot
            handleResolution(state.resolution(), entry, instance, suggestions, state.transcript());
        } else {
            updateEscalationContext(instance, state.escalationReason(),
                    suggestions, null, null, null, state.transcript());
            queueService.escalate(entry.getId(), tenancyId, operatorAssistedViewId);
        }
    } catch (io.casehub.platform.agent.AgentSessionLimitException e) {
        LOG.warn("AgentSession limit reached — falling back to single-shot");
        registry.counter("casehub.iot.ai.resolution.session.fallback",
                "reason", "limit").increment();
        processSingleShot(entry, instance, suggestions);
    } catch (Exception e) {
        LOG.errorf(e, "Multi-turn conversation failed for entry %s", entry.getId());
        escalateWithReason(entry, "Multi-turn error: " + e.getMessage(), instance, suggestions);
    } finally {
        if (session != null) {
            session.close(java.time.Duration.ofSeconds(5));
        }
        activeSessions.remove(entry.getCaseId());
        sessionSemaphore.release();
    }
}
```

- [ ] **Step 5: Update sweepStaleEntries() to skip active sessions**

```java
private void sweepStaleEntries() {
    List<CaseQueueEntry> all = queueService.findByView(aiResolutionViewId, tenancyId);
    Instant threshold = Instant.now().minusSeconds(config.timeoutSeconds());
    for (CaseQueueEntry entry : all) {
        if (activeSessions.containsKey(entry.getCaseId())) {
            continue; // skip entries with active conversations
        }
        // ... existing timeout logic unchanged
    }
}
```

- [ ] **Step 6: Implement buildInitialQuery() and buildFollowUpQuery()**

```java
private String buildInitialQuery(CaseInstance instance, List<ResolutionSuggestion> suggestions) {
    return AiResolutionPromptBuilder.build(
            extractFeatures(instance), suggestions, AUTONOMOUS_ACTIONS)
           + "\n\nRespond with JSON: {\"signal\":\"CONTINUE|RESOLVED|ESCALATE\","
           + "\"reasoning\":\"...\",\"actions\":[...],\"escalationReason\":\"...\","
           + "\"informationNeeded\":\"...\"}";
}

private String buildFollowUpQuery(MultiTurnResolutionState state) {
    var last = state.lastResponse();
    if (last != null && last.informationNeeded() != null) {
        return "You requested: " + last.informationNeeded()
               + "\nUse the available MCP tools to gather this information, "
               + "then respond with your updated assessment as JSON.";
    }
    return "Continue your analysis. Respond with JSON.";
}
```

- [ ] **Step 7: Run existing tests to verify no regressions**

Run: `/opt/homebrew/bin/mvn test -f webapp/pom.xml -Dtest=IoTAiResolutionAgentTest -pl webapp`
Expected: all existing tests PASS (they use `single` mode or the mocked `llmAgent`)

- [ ] **Step 8: Write and run multi-turn tests**

Write tests for:
- Auto mode multi-turn (CONTINUE → CONTINUE → RESOLVED)
- Max turns exceeded (5x CONTINUE → auto-escalate)
- Multi-turn escalation (turn 3 → ESCALATE)
- Session limit fallback
- Session semaphore fallback
- Session cleanup on exception
- Timeout sweep skip for active sessions
- Token tracking across turns

Each test mocks `AgentProvider.openSession()` returning a mock
`AgentSession` whose `query()` returns `Multi.createFrom().items(...)`
with appropriate `AgentEvent`s.

- [ ] **Step 9: Run full webapp test suite**

Run: `/opt/homebrew/bin/mvn test -f webapp/pom.xml -pl webapp`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java \
        webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java
git commit -m "feat(#83): multi-turn conversation support — session lifecycle, mode routing, conversation loop"
```

---

### Task 6: Conversation metrics

**Files:**
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java`
- Modify: `webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java`

**Interfaces:**
- Consumes: `ConversationTranscript` (Task 1), `MultiTurnResolutionState` (Task 3), `MeterRegistry` (existing)
- Produces: Micrometer metrics for conversation turns, duration, tool calls, tokens, fallbacks

- [ ] **Step 1: Write failing test for conversation metrics**

```java
@Test
void multiTurn_recordsConversationMetrics() {
    // Setup: auto mode, 3-turn conversation (CONTINUE → CONTINUE → RESOLVED)
    // Verify: casehub.iot.ai.resolution.conversation.turns recorded with value 3
    //         casehub.iot.ai.resolution.conversation.duration timer recorded
    //         casehub.iot.ai.resolution.conversation.tokens counter incremented
}
```

- [ ] **Step 2: Add metric recording at conversation end**

After the conversation loop exits (in `processMultiTurn`), record:

```java
registry.summary("casehub.iot.ai.resolution.conversation.turns",
        "outcome", state.resolution() != null ? "resolved" : "escalated")
        .record(state.turnCount());

conversationTimer.stop(Timer.builder("casehub.iot.ai.resolution.conversation.duration")
        .tag("outcome", outcome).tag("mode", config.conversationMode())
        .register(registry));

registry.counter("casehub.iot.ai.resolution.conversation.tokens",
        "type", "input").increment(state.transcript().totalInputTokens());
registry.counter("casehub.iot.ai.resolution.conversation.tokens",
        "type", "output").increment(state.transcript().totalOutputTokens());
```

Record tool calls per turn in `addTurn`:
```java
for (ToolCall tc : collected.toolCalls()) {
    registry.counter("casehub.iot.ai.resolution.conversation.tool.calls",
            "tool", tc.name()).increment();
}
```

- [ ] **Step 3: Run tests to verify metrics are recorded**

Run: `/opt/homebrew/bin/mvn test -f webapp/pom.xml -Dtest=IoTAiResolutionAgentTest -pl webapp`
Expected: all tests PASS

- [ ] **Step 4: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgent.java \
        webapp/src/test/java/io/casehub/iot/webapp/app/resolution/IoTAiResolutionAgentTest.java
git commit -m "feat(#83): conversation metrics — turns, duration, tokens, tool calls, fallbacks"
```

---

### Task 7: Full integration verification + consumer guide update

**Files:**
- Modify: `docs/guides/consumer-guide.md` (document conversation-mode config)
- Modify: `docs/guides/contributor-guide.md` (document AgentEventCollector, MultiTurnResolutionState)

- [ ] **Step 1: Run full project build**

Run: `/opt/homebrew/bin/mvn install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 2: Update consumer guide**

Add a section under the AI Resolution Agent documentation describing the
three conversation modes and their configuration properties:

```markdown
### Multi-Turn Conversation

The AI resolution agent supports multi-turn LLM conversations for complex
situations. Configure via:

| Property | Default | Description |
|----------|---------|-------------|
| `casehub.iot.ai-resolution.conversation-mode` | `auto` | `single` (legacy), `multi` (always), `auto` (session with single-turn exit) |
| `casehub.iot.ai-resolution.max-conversation-turns` | `5` | Max turns before auto-escalation |
| `casehub.iot.ai-resolution.max-concurrent-sessions` | `1` | Concurrent multi-turn conversations |
```

- [ ] **Step 3: Commit**

```bash
git add docs/guides/consumer-guide.md docs/guides/contributor-guide.md
git commit -m "docs(#83): document multi-turn conversation configuration and internals"
```
