# Command Binding with Execution Context Threading — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #131 — Command binding with execution context threading
**Issue group:** #131

**Goal:** Thread execution context (executionId) through the plugin SPI and command dispatch chain so that `iot.command` step executions fire `PlaybookBindingEvent` for topology view highlighting.

**Architecture:** Three-layer approach. Layer 1: upstream SPI additions (`PluginExecutionContext` in yaml-plugin-api, `executionId()` in `DeliveryContext`, runtime wiring in `PlaybookExecutor`). Layer 2: IoT plugin changes (`IoTCommandPlugin` accepts context, `DeviceCommandDispatcher` fires binding events). Layer 3: unify `DesiredStateDeliveryHandler` to use runtime-provided executionId.

**Tech Stack:** Java 21, Quarkus (CDI), casehub-platform-yaml-plugin-api, casehub-pages-playbook, JUnit 5, AssertJ, Mockito

## Global Constraints

- All SPIs are blocking (virtual-thread-aligned per ADR-0005). No `Uni<>` return types.
- `casehub-iot-api` is a public API surface — no breaking changes without major version bump.
- Binding events use synchronous `Event.fire()` (not `fireAsync()`), same as DesiredStateDeliveryHandler.
- Upstream changes require explicit user approval before modifying platform or pages repos.
- Backward-compatible overloads must be provided for existing callers (MCP, REST API, bridge).

---

## Batch 1: Upstream SPI additions (platform + pages)

**After this batch:** `PluginExecutionContext` exists in yaml-plugin-api, `DeliveryContext` has `executionId()`, and the playbook runtime generates and populates execution IDs per step. All changes are backward-compatible — existing code compiles unchanged.

### Task 1: Add PluginExecutionContext to yaml-plugin-api

**Files:**
- Create: `platform/yaml-plugin-api/src/main/java/io/casehub/yaml/plugin/api/PluginExecutionContext.java`
- Test: `platform/yaml-plugin-api/src/test/java/io/casehub/yaml/plugin/api/PluginExecutionContextTest.java`

**Interfaces:**
- Produces: `PluginExecutionContext` interface with `String executionId()` — consumed by IoTCommandPlugin (Task 4) via `services.lookup(PluginExecutionContext.class)`

**Repos:** `~/claude/casehub/platform` — **requires user approval before execution**

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.yaml.plugin.api;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PluginExecutionContextTest {

    @Test
    void executionId_returns_provided_value() {
        PluginExecutionContext ctx = () -> "exec-123";
        assertThat(ctx.executionId()).isEqualTo("exec-123");
    }

    @Test
    void lookup_via_service_registry_returns_context() {
        PluginExecutionContext ctx = () -> "exec-456";
        ServiceRegistry registry = new MapServiceRegistry(
                java.util.Map.of(PluginExecutionContext.class, ctx));
        PluginExecutionContext resolved = registry.lookup(PluginExecutionContext.class);
        assertThat(resolved).isNotNull();
        assertThat(resolved.executionId()).isEqualTo("exec-456");
    }

    @Test
    void lookup_returns_null_when_not_populated() {
        ServiceRegistry registry = new MapServiceRegistry(java.util.Map.of());
        PluginExecutionContext resolved = registry.lookup(PluginExecutionContext.class);
        assertThat(resolved).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl yaml-plugin-api test -Dtest=PluginExecutionContextTest -f ~/claude/casehub/platform/pom.xml`
Expected: FAIL — `PluginExecutionContext` does not exist

- [ ] **Step 3: Write minimal implementation**

```java
package io.casehub.yaml.plugin.api;

public interface PluginExecutionContext {
    String executionId();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode -pl yaml-plugin-api test -Dtest=PluginExecutionContextTest -f ~/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 5: Install SNAPSHOT and commit**

Run: `mvn --batch-mode -pl yaml-plugin-api install -DskipTests -f ~/claude/casehub/platform/pom.xml`

```bash
git -C ~/claude/casehub/platform add yaml-plugin-api/src
git -C ~/claude/casehub/platform commit -m "feat(#131): add PluginExecutionContext to yaml-plugin-api

Minimal interface with executionId() for threading scenario execution
context through plugin execution. Plugins opt in by adding the
parameter to their @Execute method — APT generates the lookup.

Refs casehubio/iot#131

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Add executionId() to DeliveryContext and wire RuntimeDeliveryContext

**Files:**
- Modify: `pages/backend/playbook/src/main/java/io/casehub/pages/playbook/DeliveryContext.java` — add default method
- Modify: `pages/backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/RuntimeDeliveryContext.java` — add executionId field
- Modify: `pages/backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runtime/PlaybookExecutor.java:106` — generate executionId, pass to RuntimeDeliveryContext
- Test: `pages/backend/playbook-runtime/src/test/java/io/casehub/pages/playbook/runtime/PlaybookExecutorTest.java` — verify executionId flows

**Interfaces:**
- Consumes: None (backward-compatible additions)
- Produces: `DeliveryContext.executionId()` returning `String` (nullable) — consumed by DesiredStateDeliveryHandler (Task 5)

**Repos:** `~/claude/casehub/pages` — **requires user approval before execution**

- [ ] **Step 1: Write the failing test**

Add to `PlaybookExecutorTest.java`:

```java
@Test
void executeStep_passes_executionId_to_delivery_context() {
    var capturedCtx = new AtomicReference<DeliveryContext>();
    var handler = new DeliveryHandler() {
        @Override public String name() { return "test-delivery"; }
        @Override public StepOutcome execute(String stepName, Map<String, Object> data,
                                              DeliveryContext ctx) {
            capturedCtx.set(ctx);
            return StepOutcome.ok(stepName, Map.of());
        }
    };

    var executor = new PlaybookExecutor(List.of(handler));
    var step = new CompactStep("test-delivery", Map.of(), null, null, null, Map.of());
    executor.execute(List.of(step), new PlaybookConfig("", "", Map.of(), Map.of()));

    assertThat(capturedCtx.get()).isNotNull();
    assertThat(capturedCtx.get().executionId()).isNotNull();
    assertThat(capturedCtx.get().executionId()).matches("[0-9a-f-]{36}");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode -pl backend/playbook-runtime test -Dtest=PlaybookExecutorTest#executeStep_passes_executionId_to_delivery_context -f ~/claude/casehub/pages/pom.xml`
Expected: FAIL — `executionId()` method does not exist on `DeliveryContext`

- [ ] **Step 3: Add default method to DeliveryContext**

In `DeliveryContext.java`, add after line 8:

```java
    default String executionId() {
        return null;
    }
```

- [ ] **Step 4: Add executionId field to RuntimeDeliveryContext**

Replace the constructor and add the field + override in `RuntimeDeliveryContext.java`:

```java
class RuntimeDeliveryContext implements DeliveryContext {

    private final PlaybookConfig config;
    private final VariableContext variables;
    private final String executionId;

    RuntimeDeliveryContext(PlaybookConfig config, VariableContext variables) {
        this(config, variables, null);
    }

    RuntimeDeliveryContext(PlaybookConfig config, VariableContext variables,
                           String executionId) {
        this.config = config;
        this.variables = variables;
        this.executionId = executionId;
    }

    @Override
    public String executionId() {
        return executionId;
    }

    // ... existing config(), resolve(), resolveMap() unchanged
```

- [ ] **Step 5: Generate executionId in PlaybookExecutor.executeStep()**

In `PlaybookExecutor.java`, change line 106 from:

```java
        DeliveryContext ctx   = new RuntimeDeliveryContext(config, context);
```

to:

```java
        String executionId = java.util.UUID.randomUUID().toString();
        DeliveryContext ctx   = new RuntimeDeliveryContext(config, context, executionId);
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode -pl backend/playbook-runtime test -Dtest=PlaybookExecutorTest -f ~/claude/casehub/pages/pom.xml`
Expected: PASS (all existing tests and the new one)

- [ ] **Step 7: Install SNAPSHOTs and commit**

Run: `mvn --batch-mode -pl backend/playbook,backend/playbook-runtime install -DskipTests -f ~/claude/casehub/pages/pom.xml`

```bash
git -C ~/claude/casehub/pages add backend/playbook/src backend/playbook-runtime/src
git -C ~/claude/casehub/pages commit -m "feat(#131): thread executionId through DeliveryContext

Add DeliveryContext.executionId() default method (returns null for
backward compatibility). RuntimeDeliveryContext carries the value.
PlaybookExecutor generates a UUID per step and passes it through.

Refs casehubio/iot#131

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: IoT command binding events

**After this batch:** `DeviceCommandDispatcher` fires `PlaybookBindingEvent` when executionId is present. `IoTCommandPlugin` threads execution context through. `DesiredStateDeliveryHandler` uses runtime-provided executionId. Command steps highlight devices in the topology view.

### Task 3: DeviceCommandDispatcher fires binding events

**Files:**
- Modify: `scenario/src/main/java/io/casehub/iot/scenario/DeviceCommandDispatcher.java` — add executionId parameter, inject Event<PlaybookBindingEvent>, fire binding events
- Test: `scenario/src/test/java/io/casehub/iot/scenario/DeviceCommandDispatcherTest.java` — new tests for binding events

**Interfaces:**
- Consumes: `PlaybookBindingEvent` (casehub-iot-api, already exists)
- Produces: `dispatch(deviceId, action, params, correlationId, executionId)` — consumed by IoTCommandPlugin (Task 4). Backward-compatible overload `dispatch(deviceId, action, params, correlationId)` preserved.

- [ ] **Step 1: Write failing tests for binding events**

Add to `DeviceCommandDispatcherTest.java`.

First, update imports and add field:

```java
import io.casehub.iot.api.PlaybookBindingEvent;
import java.util.ArrayList;
```

Add field alongside existing fields:

```java
private List<PlaybookBindingEvent> capturedBindingEvents;
```

Replace the entire `setUp()` method — the constructor gains `tenancyId` and `bindingEvent` parameters. Existing tests continue to work because the 4-arg `dispatch()` delegates to the same logic, and the old 4-arg `dispatch(deviceId, action, params, correlationId)` still exists as a backward-compatible overload:

```java
@BeforeEach
void setUp() {
    provider = new MockDeviceProvider("test");
    Fixtures.standardHome().forEach(provider::addDevice);
    var registry = new MockDeviceRegistry();
    registry.addDevices(provider.discover());
    capturedBindingEvents = new ArrayList<>();
    dispatcher = new DeviceCommandDispatcher(
            registry, List.of(provider), "test-tenant", capturedBindingEvents::add);
}
```

@Test
void dispatch_with_executionId_fires_binding_events_on_sent() {
    provider.setDispatchResult(CommandResult.SENT);
    dispatcher.dispatch("light-living-1", "turn_on", Map.of(), "corr-1", "exec-1");

    assertThat(capturedBindingEvents).hasSize(3);
    assertThat(capturedBindingEvents.get(0))
            .isInstanceOf(PlaybookBindingEvent.StepStart.class);
    var start = (PlaybookBindingEvent.StepStart) capturedBindingEvents.get(0);
    assertThat(start.executionId()).isEqualTo("exec-1");
    assertThat(start.deviceIds()).containsExactly("light-living-1");

    assertThat(capturedBindingEvents.get(1))
            .isInstanceOf(PlaybookBindingEvent.DeviceProvisioned.class);

    assertThat(capturedBindingEvents.get(2))
            .isInstanceOf(PlaybookBindingEvent.StepComplete.class);
    var complete = (PlaybookBindingEvent.StepComplete) capturedBindingEvents.get(2);
    assertThat(complete.provisioned()).isEqualTo(1);
    assertThat(complete.failed()).isZero();
}

@Test
void dispatch_with_executionId_fires_binding_events_on_failed() {
    provider.setDispatchResult(CommandResult.FAILED);
    dispatcher.dispatch("light-living-1", "turn_on", Map.of(), "corr-1", "exec-2");

    assertThat(capturedBindingEvents).hasSize(3);
    assertThat(capturedBindingEvents.get(0))
            .isInstanceOf(PlaybookBindingEvent.StepStart.class);
    assertThat(capturedBindingEvents.get(1))
            .isInstanceOf(PlaybookBindingEvent.DeviceFailed.class);
    assertThat(capturedBindingEvents.get(2))
            .isInstanceOf(PlaybookBindingEvent.StepFailed.class);
}

@Test
void dispatch_with_executionId_fires_binding_events_on_timeout() {
    provider.setDispatchResult(CommandResult.TIMEOUT);
    dispatcher.dispatch("light-living-1", "turn_on", Map.of(), "corr-1", "exec-3");

    assertThat(capturedBindingEvents).hasSize(3);
    assertThat(capturedBindingEvents.get(1))
            .isInstanceOf(PlaybookBindingEvent.DeviceFailed.class);
    var failed = (PlaybookBindingEvent.DeviceFailed) capturedBindingEvents.get(1);
    assertThat(failed.reason()).contains("timed out");
}

@Test
void dispatch_without_executionId_fires_no_binding_events() {
    provider.setDispatchResult(CommandResult.SENT);
    dispatcher.dispatch("light-living-1", "turn_on", Map.of(), "corr-1", null);

    assertThat(capturedBindingEvents).isEmpty();
}

@Test
void dispatch_backward_compatible_overload_fires_no_binding_events() {
    provider.setDispatchResult(CommandResult.SENT);
    dispatcher.dispatch("light-living-1", "turn_on", Map.of(), "corr-1");

    assertThat(capturedBindingEvents).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl scenario test -Dtest=DeviceCommandDispatcherTest -f ~/claude/casehub/iot/pom.xml`
Expected: FAIL — constructor and method signature mismatch

- [ ] **Step 3: Implement DeviceCommandDispatcher changes**

Replace the full `DeviceCommandDispatcher.java` content:

```java
package io.casehub.iot.scenario;

import io.casehub.iot.api.CommandResult;
import io.casehub.iot.api.DeviceCommand;
import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.PlaybookBindingEvent;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.function.Consumer;

@ApplicationScoped
public class DeviceCommandDispatcher {

    private final DeviceRegistry registry;
    private final Map<String, DeviceProvider> providers;
    private final String tenancyId;
    private final Consumer<PlaybookBindingEvent> bindingEvent;

    @Inject
    public DeviceCommandDispatcher(
            DeviceRegistry registry,
            @Any Instance<DeviceProvider> providerBeans,
            @ConfigProperty(name = "casehub.iot.tenancy-id") String tenancyId,
            jakarta.enterprise.event.Event<PlaybookBindingEvent> bindingEvent) {
        this(registry, providerBeans.stream().toList(), tenancyId, bindingEvent::fire);
    }

    DeviceCommandDispatcher(DeviceRegistry registry, List<DeviceProvider> providerList,
                            String tenancyId, Consumer<PlaybookBindingEvent> bindingEvent) {
        this.registry = registry;
        this.providers = new HashMap<>();
        providerList.forEach(p -> providers.put(p.providerId(), p));
        this.tenancyId = tenancyId;
        this.bindingEvent = bindingEvent;
    }

    public CommandResult dispatch(String deviceId, String action,
                                  Map<String, Object> params, String correlationId) {
        return dispatch(deviceId, action, params, correlationId, null);
    }

    public CommandResult dispatch(String deviceId, String action,
                                  Map<String, Object> params,
                                  String correlationId,
                                  String executionId) {
        var optDevice = registry.findById(deviceId);
        if (optDevice.isEmpty()) {
            throw new IllegalArgumentException("Device not found: " + deviceId);
        }
        var entity = optDevice.get();

        var provider = providers.get(entity.providerId());
        if (provider == null) {
            throw new IllegalArgumentException("No provider for: " + entity.providerId());
        }

        var command = new DeviceCommand(
                deviceId, action, params != null ? params : Map.of(),
                "iot-scenario", correlationId);

        String stepLabel = "iot.command:" + action;

        if (executionId != null) {
            bindingEvent.accept(new PlaybookBindingEvent.StepStart(
                    executionId, tenancyId, stepLabel, Set.of(deviceId)));
        }

        CommandResult result = provider.dispatch(command);

        if (executionId != null) {
            switch (result) {
                case SENT -> {
                    bindingEvent.accept(new PlaybookBindingEvent.DeviceProvisioned(
                            executionId, tenancyId, stepLabel, deviceId));
                    bindingEvent.accept(new PlaybookBindingEvent.StepComplete(
                            executionId, tenancyId, stepLabel, 1, 0));
                }
                case FAILED, TIMEOUT -> {
                    String reason = result == CommandResult.TIMEOUT
                            ? "Command timed out" : "Command failed";
                    bindingEvent.accept(new PlaybookBindingEvent.DeviceFailed(
                            executionId, tenancyId, stepLabel, deviceId, reason));
                    bindingEvent.accept(new PlaybookBindingEvent.StepFailed(
                            executionId, tenancyId, stepLabel,
                            0, 1, List.of(deviceId + ": " + reason)));
                }
            }
        }

        return result;
    }

    public Optional<DeviceEntity> findDevice(String deviceId) {
        return registry.findById(deviceId);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -pl scenario test -Dtest=DeviceCommandDispatcherTest -f ~/claude/casehub/iot/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/iot add scenario/src
git -C ~/claude/casehub/iot commit -m "feat(#131): DeviceCommandDispatcher fires binding events with executionId

Add executionId parameter to dispatch(). When present, fires
PlaybookBindingEvent.StepStart before dispatch and StepComplete/
StepFailed after. Backward-compatible overload without executionId
preserved for non-scenario callers (MCP, REST, bridge).

Refs #131

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: IoTCommandPlugin threads PluginExecutionContext

**Files:**
- Modify: `scenario/src/main/java/io/casehub/iot/scenario/IoTCommandPlugin.java` — add `PluginExecutionContext` parameter to `@Execute` method
- Test: `scenario/src/test/java/io/casehub/iot/scenario/IoTCommandPluginTest.java` — test context threading and null handling

**Interfaces:**
- Consumes: `PluginExecutionContext` (from yaml-plugin-api, Task 1), `DeviceCommandDispatcher.dispatch(deviceId, action, params, correlationId, executionId)` (from Task 3)
- Produces: Updated `@Execute` method signature — APT generates `services.lookup(PluginExecutionContext.class)`

- [ ] **Step 1: Write failing tests**

Add to `IoTCommandPluginTest.java`:

```java
import io.casehub.iot.api.PlaybookBindingEvent;
import io.casehub.yaml.plugin.api.PluginExecutionContext;
import java.util.ArrayList;

// Replace setUp to use binding-aware dispatcher:
private List<PlaybookBindingEvent> capturedBindingEvents;

@BeforeEach
void setUp() {
    provider = new MockDeviceProvider("test");
    Fixtures.standardHome().forEach(provider::addDevice);
    var registry = new MockDeviceRegistry();
    registry.addDevices(provider.discover());
    capturedBindingEvents = new ArrayList<>();
    dispatcher = new DeviceCommandDispatcher(
            registry, List.of(provider), "test-tenant", capturedBindingEvents::add);
}

@Test
void run_with_execution_context_fires_binding_events() {
    provider.setDispatchResult(CommandResult.SENT);
    PluginExecutionContext ctx = () -> "exec-abc";
    var plugin = new IoTCommandPlugin("light-living-1", "turn_on", null, null);
    Result result = plugin.run(dispatcher, ctx);

    assertThat(result.isSuccess()).isTrue();
    assertThat(capturedBindingEvents).hasSize(3);
    assertThat(capturedBindingEvents.get(0))
            .isInstanceOf(PlaybookBindingEvent.StepStart.class);
}

@Test
void run_without_execution_context_fires_no_binding_events() {
    provider.setDispatchResult(CommandResult.SENT);
    var plugin = new IoTCommandPlugin("light-living-1", "turn_on", null, null);
    Result result = plugin.run(dispatcher, null);

    assertThat(result.isSuccess()).isTrue();
    assertThat(capturedBindingEvents).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl scenario test -Dtest=IoTCommandPluginTest -f ~/claude/casehub/iot/pom.xml`
Expected: FAIL — `run()` method signature mismatch (only takes `DeviceCommandDispatcher`)

- [ ] **Step 3: Update IoTCommandPlugin**

Replace the full `IoTCommandPlugin.java` content:

```java
package io.casehub.iot.scenario;

import io.casehub.iot.api.CommandResult;
import io.casehub.yaml.plugin.api.Execute;
import io.casehub.yaml.plugin.api.Optional;
import io.casehub.yaml.plugin.api.Plugin;
import io.casehub.yaml.plugin.api.PluginExecutionContext;
import io.casehub.yaml.plugin.api.Required;
import io.casehub.yaml.plugin.api.Result;

import java.util.Map;
import java.util.UUID;

@Plugin(value = "iot.command",
        description = "Dispatches a command to an IoT device via DeviceProvider")
public record IoTCommandPlugin(
        @Required String device,
        @Required String action,
        @Optional Map<String, Object> params,
        @Optional String correlationId) {

    @Execute
    public Result run(DeviceCommandDispatcher dispatcher,
                      PluginExecutionContext executionContext) {
        String corrId = correlationId != null
                ? correlationId : UUID.randomUUID().toString();
        String execId = executionContext != null
                ? executionContext.executionId() : null;

        CommandResult result;
        try {
            result = dispatcher.dispatch(device, action, params, corrId, execId);
        } catch (IllegalArgumentException e) {
            return Result.failed(e.getMessage());
        }

        return switch (result) {
            case SENT -> Result.of(Map.of(
                    "result", "SENT",
                    "device", device,
                    "action", action,
                    "correlationId", corrId));
            case FAILED -> Result.failed(
                    "Command " + action + " failed for device " + device);
            case TIMEOUT -> Result.failed(
                    "Command " + action + " timed out for device " + device);
        };
    }
}
```

- [ ] **Step 4: Update existing tests to match new signature**

Update all existing test calls from `plugin.run(dispatcher)` to `plugin.run(dispatcher, null)` — six occurrences in tests: `sent_returns_success_with_device_and_action`, `failed_returns_failure`, `timeout_returns_failure`, `unknown_device_returns_failure`, `custom_correlation_id_is_used`, `params_forwarded_to_command`.

- [ ] **Step 5: Run all plugin tests to verify they pass**

Run: `mvn --batch-mode -pl scenario test -Dtest=IoTCommandPluginTest -f ~/claude/casehub/iot/pom.xml`
Expected: PASS (all 8 tests)

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/casehub/iot add scenario/src
git -C ~/claude/casehub/iot commit -m "feat(#131): IoTCommandPlugin threads PluginExecutionContext to dispatcher

Add PluginExecutionContext as second @Execute parameter. APT generates
services.lookup(PluginExecutionContext.class). Null-safe — when context
is absent (non-playbook callers), no binding events fire.

Refs #131

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 5: Unify DesiredStateDeliveryHandler executionId source

**Files:**
- Modify: `scenario/src/main/java/io/casehub/iot/scenario/DesiredStateDeliveryHandler.java:91` — read from `ctx.executionId()` with UUID fallback
- Test: `scenario/src/test/java/io/casehub/iot/scenario/DesiredStateDeliveryHandlerTest.java` — verify context-provided executionId is used

**Interfaces:**
- Consumes: `DeliveryContext.executionId()` (from Task 2)
- Produces: No API change — same binding events, but executionId now comes from the runtime

- [ ] **Step 1: Write failing test**

Add to `DesiredStateDeliveryHandlerTest.java`:

```java
@Test
void usesExecutionIdFromDeliveryContext() {
    DeliveryContext ctxWithId = mock(DeliveryContext.class);
    when(ctxWithId.executionId()).thenReturn("runtime-exec-id");

    when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
            .thenReturn(new ActualState(Map.of()));
    when(provisioner.provision(any(), any()))
            .thenReturn(new ProvisionResult.Success());

    Map<String, Object> data = new LinkedHashMap<>();
    data.put("light-living-1", Map.of("on", false));

    handler.execute("ctx-step", data, ctxWithId);

    assertThat(capturedEvents).isNotEmpty();
    var start = (PlaybookBindingEvent.StepStart) capturedEvents.get(0);
    assertThat(start.executionId()).isEqualTo("runtime-exec-id");
}

@Test
void fallsBackToUuidWhenContextHasNoExecutionId() {
    when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
            .thenReturn(new ActualState(Map.of()));
    when(provisioner.provision(any(), any()))
            .thenReturn(new ProvisionResult.Success());

    Map<String, Object> data = new LinkedHashMap<>();
    data.put("light-living-1", Map.of("on", false));

    handler.execute("fallback-step", data, CTX);

    assertThat(capturedEvents).isNotEmpty();
    var start = (PlaybookBindingEvent.StepStart) capturedEvents.get(0);
    assertThat(start.executionId()).isNotNull();
    assertThat(start.executionId()).matches("[0-9a-f-]{36}");
}
```

- [ ] **Step 2: Run tests to verify the first one fails**

Run: `mvn --batch-mode -pl scenario test -Dtest=DesiredStateDeliveryHandlerTest#usesExecutionIdFromDeliveryContext -f ~/claude/casehub/iot/pom.xml`
Expected: FAIL — handler ignores `ctx.executionId()` and always generates its own UUID

- [ ] **Step 3: Update execute() method**

In `DesiredStateDeliveryHandler.java`, change line 91 from:

```java
        String   executionId = UUID.randomUUID().toString();
```

to:

```java
        String   executionId = ctx.executionId() != null
                ? ctx.executionId()
                : UUID.randomUUID().toString();
```

- [ ] **Step 4: Run all delivery handler tests**

Run: `mvn --batch-mode -pl scenario test -Dtest=DesiredStateDeliveryHandlerTest -f ~/claude/casehub/iot/pom.xml`
Expected: PASS (all tests including existing binding event tests)

- [ ] **Step 5: Run full scenario module test suite**

Run: `mvn --batch-mode -pl scenario test -f ~/claude/casehub/iot/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/casehub/iot add scenario/src
git -C ~/claude/casehub/iot commit -m "feat(#131): unify executionId source — use DeliveryContext when available

DesiredStateDeliveryHandler reads ctx.executionId() instead of always
generating UUID. Falls back to UUID for non-playbook callers (e.g.,
direct MCP invocation). Both delivery paths now use the same
runtime-generated identity.

Refs #131

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 6: Full build verification

**Files:** None — verification only

- [ ] **Step 1: Full build**

Run: `mvn --batch-mode install -f ~/claude/casehub/iot/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 2: Verify generated IoTCommandPluginAction**

Read `scenario/target/generated-sources/annotations/io/casehub/iot/scenario/IoTCommandPluginAction.java` and verify it contains:

```java
services.lookup(io.casehub.yaml.plugin.api.PluginExecutionContext.class)
```

as the second parameter to `spec.run()`.

- [ ] **Step 3: Commit any remaining changes**

If the full build produces any changes (e.g., regenerated sources), commit them.

## References

- `2026-10-07-command-binding-exec-context-design.md` — design spec this plan implements
- `IoTCommandPlugin.java:13-45` — current plugin without execution context
- `DeviceCommandDispatcher.java:19-61` — current dispatcher without binding events
- `DesiredStateDeliveryHandler.java:88-104` — current execute() with self-generated UUID
- `DeviceCommandDispatcherTest.java:16-76` — existing dispatcher tests
- `IoTCommandPluginTest.java:16-89` — existing plugin tests
- `DesiredStateDeliveryHandlerTest.java:34-220` — existing delivery handler tests with binding event verification
- `PlaybookBindingEvent.java:1-39` — sealed interface (unchanged)
- `Action.java:6-13` (platform yaml-plugin-api) — plugin execution contract
- `DeliveryContext.java:1-9` (pages-playbook) — delivery handler context interface
- `RuntimeDeliveryContext.java:7-42` (pages-playbook-runtime) — runtime context implementation
- `PlaybookExecutor.java:90-123` (pages-playbook-runtime) — step execution loop
- GitHub casehubio/iot#131 — focal issue
- GitHub casehubio/iot#129 — parent issue (scenario-topology binding)
