# Desired-State Scenario Delivery Handler Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #126 — feat: desired-state scenario delivery mode — IoT plugin for DeliveryHandler SPI
**Issue group:** #126

**Goal:** Implement a `DeliveryHandler` that bridges the casehub-pages scenario engine to IoT desired-state reconciliation, supporting inline device config and preset references.

**Architecture:** Single `@ApplicationScoped` CDI bean implementing `DeliveryHandler` from `casehub-pages-scenario`. Uses the same one-shot reconciliation pattern as `DefaultIoTPresetApi.applyPreset()`: compile goals → read actual state → plan transitions → provision devices. Lives in the `casehub-iot-scenario` module alongside existing `IoTCommandPlugin` and `IoTStatePlugin`.

**Tech Stack:** Java 21+, Quarkus (CDI/Arc), JUnit 5, AssertJ, casehub-desiredstate API + runtime

## Global Constraints

- All SPIs are blocking — designed for virtual threads per ADR-0005
- `casehub.iot.tenancy-id` is the single tenancy property — never per-module
- Provider activation uses `@LookupIfProperty` — disabled providers are invisible
- `casehub-iot-api` is a public API surface — no breaking changes
- `casehub-iot-testing` is test-scope only

---

## Batch 1: DesiredStateDeliveryHandler — inline config, preset reference, and reconciliation

### Task 1: Add POM dependencies, write tests, implement handler

**Files:**
- Modify: `scenario/pom.xml` — add 3 new dependencies
- Create: `scenario/src/main/java/io/casehub/iot/scenario/DesiredStateDeliveryHandler.java`
- Create: `scenario/src/test/java/io/casehub/iot/scenario/DesiredStateDeliveryHandlerTest.java`

**Interfaces:**
- Consumes: `DeliveryHandler` (from `io.casehub.pages.scenario`), `DeviceRegistry` (from `io.casehub.iot.api.spi`), `IoTGoalCompiler`, `IoTActualStateAdapter`, `IoTNodeProvisioner`, `IoTPresetResolver` (all from `io.casehub.iot.desiredstate`), `TransitionPlanner`, `DefaultDesiredStateGraphFactory` (from `io.casehub.desiredstate.runtime`)
- Produces: CDI-discovered `DeliveryHandler` bean with `name() = "desired-state"`

- [ ] **Step 1: Add dependencies to scenario/pom.xml**

Add these dependencies after the existing `casehub-platform-yaml-plugin-api` dependency:

```xml
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-scenario</artifactId>
            <version>${casehub.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-desiredstate</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate</artifactId>
        </dependency>
```

Note: `casehub-pages-scenario` uses `${casehub.version}` because it is a cross-repo dependency not managed in the IoT parent POM. `casehub-iot-desiredstate` and `casehub-desiredstate` are already version-managed in the parent.

- [ ] **Step 2: Verify POM compiles**

Run: `mvn --batch-mode -pl scenario -am compile -q`
Expected: BUILD SUCCESS (no compilation errors from new dependencies)

- [ ] **Step 3: Write the test class with all test cases**

Create `scenario/src/test/java/io/casehub/iot/scenario/DesiredStateDeliveryHandlerTest.java`:

```java
package io.casehub.iot.scenario;

import io.casehub.desiredstate.api.ActualState;
import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.desiredstate.api.OrderedStep;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.desiredstate.api.StepAction;
import io.casehub.desiredstate.api.TransitionPlan;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.runtime.TransitionPlanner;
import io.casehub.iot.api.DeviceClass;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.iot.desiredstate.DeviceConfigSpec;
import io.casehub.iot.desiredstate.IoTActualStateAdapter;
import io.casehub.iot.desiredstate.IoTGoalCompiler;
import io.casehub.iot.desiredstate.IoTGoals;
import io.casehub.iot.desiredstate.IoTNodeProvisioner;
import io.casehub.iot.desiredstate.IoTPresetResolver;
import io.casehub.iot.testing.Fixtures;
import io.casehub.iot.testing.MockDeviceProvider;
import io.casehub.iot.testing.MockDeviceRegistry;
import io.casehub.pages.scenario.DeliveryContext;
import io.casehub.pages.scenario.StepOutcome;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class DesiredStateDeliveryHandlerTest {

    private static final String TENANCY_ID = "default-tenant";
    private static final DeliveryContext CTX = mock(DeliveryContext.class);

    private MockDeviceProvider provider;
    private MockDeviceRegistry registry;
    private IoTPresetResolver presetResolver;
    private IoTGoalCompiler compiler;
    private IoTActualStateAdapter actualStateAdapter;
    private IoTNodeProvisioner provisioner;
    private DesiredStateDeliveryHandler handler;

    @BeforeEach
    void setUp() {
        provider = new MockDeviceProvider("test");
        Fixtures.standardHome().forEach(provider::addDevice);
        registry = new MockDeviceRegistry();
        registry.addDevices(provider.discover());

        presetResolver = mock(IoTPresetResolver.class);
        compiler = new IoTGoalCompiler();
        actualStateAdapter = mock(IoTActualStateAdapter.class);
        provisioner = mock(IoTNodeProvisioner.class);

        handler = new DesiredStateDeliveryHandler(
                registry, presetResolver, compiler,
                actualStateAdapter, provisioner, TENANCY_ID);
    }

    @Test
    void name_returns_desired_state() {
        assertThat(handler.name()).isEqualTo("desired-state");
    }

    @Test
    void inline_config_provisions_devices() {
        when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
                .thenReturn(new ActualState(Map.of()));
        when(provisioner.provision(any(), any()))
                .thenReturn(new ProvisionResult.Success());

        Map<String, Object> data = new LinkedHashMap<>();
        data.put("light-living-1", Map.of("on", false));
        data.put("thermostat-living-1", Map.of("mode", "cool", "target", 22));

        StepOutcome outcome = handler.execute("night-mode", data, CTX);

        assertThat(outcome.success()).isTrue();
        assertThat(outcome.result()).containsKey("provisioned");
    }

    @Test
    void preset_reference_resolves_and_provisions() {
        var goals = new IoTGoals(TENANCY_ID, List.of());
        when(presetResolver.resolve("night-mode")).thenReturn(goals);
        when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
                .thenReturn(new ActualState(Map.of()));

        Map<String, Object> data = Map.of("preset", "night-mode");

        StepOutcome outcome = handler.execute("apply-preset", data, CTX);

        assertThat(outcome.success()).isTrue();
        verify(presetResolver).resolve("night-mode");
    }

    @Test
    void unknown_device_fails_step() {
        Map<String, Object> data = Map.of(
                "light-living-1", Map.of("on", false),
                "nonexistent-device", Map.of("on", true));

        StepOutcome outcome = handler.execute("bad-step", data, CTX);

        assertThat(outcome.success()).isFalse();
        assertThat(outcome.error()).contains("nonexistent-device");
        verify(provisioner, never()).provision(any(), any());
    }

    @Test
    void partial_provision_failure_fails_step() {
        when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
                .thenReturn(new ActualState(Map.of()));
        when(provisioner.provision(any(), any()))
                .thenReturn(new ProvisionResult.Success())
                .thenReturn(new ProvisionResult.Failed("provider offline"));

        Map<String, Object> data = new LinkedHashMap<>();
        data.put("light-living-1", Map.of("on", false));
        data.put("thermostat-living-1", Map.of("mode", "cool"));

        StepOutcome outcome = handler.execute("partial-fail", data, CTX);

        assertThat(outcome.success()).isFalse();
        assertThat(outcome.error()).contains("failed");
    }

    @Test
    void already_converged_returns_success() {
        when(actualStateAdapter.readActual(any(), eq(TENANCY_ID)))
                .thenReturn(new ActualState(Map.of(
                        NodeId.of("light-living-1"), NodeStatus.PRESENT)));
        when(provisioner.provision(any(), any()))
                .thenReturn(new ProvisionResult.AlreadyConverged());

        Map<String, Object> data = Map.of(
                "light-living-1", Map.of("on", false));

        StepOutcome outcome = handler.execute("already-done", data, CTX);

        assertThat(outcome.success()).isTrue();
    }

    @Test
    void preset_not_found_fails_step() {
        when(presetResolver.resolve("missing"))
                .thenThrow(new IllegalArgumentException("Preset not found: missing"));

        Map<String, Object> data = Map.of("preset", "missing");

        StepOutcome outcome = handler.execute("bad-preset", data, CTX);

        assertThat(outcome.success()).isFalse();
        assertThat(outcome.error()).contains("Preset not found");
    }
}
```

- [ ] **Step 4: Add Mockito test dependency to scenario/pom.xml**

Add in the test dependencies section (after `assertj-core`):

```xml
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <scope>test</scope>
        </dependency>
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `mvn --batch-mode -pl scenario test -Dtest=DesiredStateDeliveryHandlerTest`
Expected: COMPILATION FAILURE — `DesiredStateDeliveryHandler` does not exist yet

- [ ] **Step 6: Implement DesiredStateDeliveryHandler**

Create `scenario/src/main/java/io/casehub/iot/scenario/DesiredStateDeliveryHandler.java`:

```java
package io.casehub.iot.scenario;

import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.OrderedStep;
import io.casehub.desiredstate.api.ProvisionContext;
import io.casehub.desiredstate.api.ProvisionResult;
import io.casehub.desiredstate.api.StepAction;
import io.casehub.desiredstate.runtime.DefaultDesiredStateGraphFactory;
import io.casehub.desiredstate.runtime.TransitionPlanner;
import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.iot.desiredstate.IoTActualStateAdapter;
import io.casehub.iot.desiredstate.IoTDeviceGoal;
import io.casehub.iot.desiredstate.IoTGoalCompiler;
import io.casehub.iot.desiredstate.IoTGoals;
import io.casehub.iot.desiredstate.IoTNodeProvisioner;
import io.casehub.iot.desiredstate.IoTPresetResolver;
import io.casehub.pages.scenario.DeliveryContext;
import io.casehub.pages.scenario.DeliveryHandler;
import io.casehub.pages.scenario.StepOutcome;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class DesiredStateDeliveryHandler implements DeliveryHandler {

    private final DeviceRegistry registry;
    private final IoTPresetResolver presetResolver;
    private final IoTGoalCompiler compiler;
    private final IoTActualStateAdapter actualStateAdapter;
    private final IoTNodeProvisioner provisioner;
    private final String tenancyId;

    private final DefaultDesiredStateGraphFactory graphFactory =
            new DefaultDesiredStateGraphFactory();
    private final TransitionPlanner planner = new TransitionPlanner();

    @Inject
    public DesiredStateDeliveryHandler(
            DeviceRegistry registry,
            IoTPresetResolver presetResolver,
            IoTGoalCompiler compiler,
            IoTActualStateAdapter actualStateAdapter,
            IoTNodeProvisioner provisioner,
            @ConfigProperty(name = "casehub.iot.tenancy-id") String tenancyId) {
        this.registry = registry;
        this.presetResolver = presetResolver;
        this.compiler = compiler;
        this.actualStateAdapter = actualStateAdapter;
        this.provisioner = provisioner;
        this.tenancyId = tenancyId;
    }

    @Override
    public String name() {
        return "desired-state";
    }

    @Override
    public StepOutcome execute(String stepName, Map<String, Object> data,
                               DeliveryContext ctx) {
        IoTGoals goals;
        try {
            goals = resolveGoals(data);
        } catch (Exception e) {
            return StepOutcome.fail(stepName, e.getMessage());
        }

        try {
            return reconcile(stepName, goals);
        } catch (Exception e) {
            return StepOutcome.fail(stepName, "Reconciliation failed: " + e.getMessage());
        }
    }

    @SuppressWarnings("unchecked")
    private IoTGoals resolveGoals(Map<String, Object> data) {
        if (data.containsKey("preset")) {
            return presetResolver.resolve((String) data.get("preset"));
        }

        List<IoTDeviceGoal> deviceGoals = new ArrayList<>();
        List<String> unknownIds = new ArrayList<>();

        for (var entry : data.entrySet()) {
            String deviceId = entry.getKey();
            var entity = registry.findById(deviceId);
            if (entity.isEmpty()) {
                unknownIds.add(deviceId);
                continue;
            }
            DeviceEntity device = entity.get();
            deviceGoals.add(new IoTDeviceGoal(
                    deviceId,
                    device.deviceClass(),
                    device.label(),
                    false,
                    (Map<String, Object>) entry.getValue(),
                    List.of()));
        }

        if (!unknownIds.isEmpty()) {
            throw new IllegalArgumentException(
                    "Unknown device IDs: " + unknownIds);
        }

        return new IoTGoals(tenancyId, deviceGoals);
    }

    private StepOutcome reconcile(String stepName, IoTGoals goals) {
        var compilationResult = compiler.compile(goals, graphFactory);
        if (!(compilationResult instanceof CompilationResult.SingleGraph sg)) {
            return StepOutcome.fail(stepName,
                    "Unexpected compilation result: "
                            + compilationResult.getClass().getSimpleName());
        }

        DesiredStateGraph graph = sg.graph();
        var actual = actualStateAdapter.readActual(graph, tenancyId);
        var plan = planner.plan(graph, actual);

        int provisioned = 0;
        int failed = 0;
        List<String> failedDetails = new ArrayList<>();

        for (OrderedStep step : plan.flatAdditions()) {
            if (step.action() == StepAction.PROVISION) {
                var result = provisioner.provision(
                        step.node(), new ProvisionContext(tenancyId, graph));
                if (result instanceof ProvisionResult.Success
                        || result instanceof ProvisionResult.AlreadyConverged) {
                    provisioned++;
                } else if (result instanceof ProvisionResult.Failed f) {
                    failed++;
                    failedDetails.add(step.node().id().value() + ": " + f.reason());
                } else {
                    failed++;
                    failedDetails.add(step.node().id().value() + ": "
                            + result.getClass().getSimpleName());
                }
            }
        }

        if (failed > 0) {
            return StepOutcome.fail(stepName,
                    failed + " of " + (provisioned + failed)
                            + " devices failed: " + failedDetails);
        }

        return StepOutcome.ok(stepName, Map.of(
                "provisioned", provisioned,
                "converged", true));
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode -pl scenario test -Dtest=DesiredStateDeliveryHandlerTest`
Expected: All 7 tests PASS

- [ ] **Step 8: Run all scenario module tests to check for regressions**

Run: `mvn --batch-mode -pl scenario test`
Expected: All tests PASS (existing IoTCommandPluginTest, IoTStatePluginTest, IoTDeviceVariableSourceTest + new tests)

- [ ] **Step 9: Run full project build**

Run: `mvn --batch-mode install -DskipTests=false`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 10: Commit**

```bash
git add scenario/pom.xml \
       scenario/src/main/java/io/casehub/iot/scenario/DesiredStateDeliveryHandler.java \
       scenario/src/test/java/io/casehub/iot/scenario/DesiredStateDeliveryHandlerTest.java
git commit -m "feat(#126): desired-state scenario delivery handler

Implements DeliveryHandler for 'desired-state' delivery mode in the
IoT scenario module. Supports inline device config (deviceId → capabilities)
and preset references. Uses one-shot reconciliation following the
applyPreset() pattern.

Refs #126

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-05-desired-state-delivery-design.md] — design spec this plan implements
- [DefaultIoTPresetApi.java:64-89] — existing one-shot reconciliation pattern
- [DeliveryHandler.java] — pages scenario SPI (name + execute)
- [StepOutcome.java] — ok/fail factory methods
- [IoTCommandPluginTest.java] — existing test pattern with MockDeviceProvider + Fixtures
- [IoTGoalCompiler.java] — goal compilation (IoTGoals → DesiredStateGraph)
- [IoTNodeProvisioner.java] — device provisioning (DesiredNode → ProvisionResult)
- [IoTPresetResolver.java] — preset name resolution
- [TransitionPlanner.java:27] — plan(desired, actual) overload
- [GitHub #126] — focal issue
- [GitHub #121] — IoT desired state epic
