# IoT Step Catalog Entries Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #134 — feat: define IoT step catalog entries for yaml-core

**Goal:** Create a YAML step catalog file defining IoT step types (`command`, `state`, `desired-state`) so the playbook visual editor and library view can display them.

**Architecture:** Single YAML definition file in the webapp's classpath, parsed by `DeclarationParser` from `casehub-platform-yaml-core`. The file follows the same format as fsitrading's `trading-steps.yaml`. A unit test validates correct parsing.

**Tech Stack:** Java 21, Maven, SnakeYAML/Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- YAML format must match the `DeclarationFile` schema: `namespace` + `actions` map
- Action names must match existing `@Plugin` qualified names when namespace-prefixed (`iot.command`, `iot.state`)
- `casehub-platform-yaml-jackson` version: `${casehub.version}`
- No changes to existing `@Plugin` Java implementations

---

## Batch 1: Step catalog YAML and test

### Task 1: Write failing test for IoT step definitions

**Files:**
- Create: `webapp/src/test/java/io/casehub/iot/webapp/steps/IoTStepDefinitionTest.java`
- Modify: `webapp/pom.xml` (add test dependency)

**Interfaces:**
- Consumes: `DeclarationParser.parse(Map)` from `io.casehub.yaml.core.step`, `YamlMappers.create()` from `io.casehub.yaml.jackson`
- Produces: test class validating `steps/iot-steps.yaml` on classpath

- [ ] **Step 1: Add yaml-jackson test dependency to webapp pom.xml**

In `webapp/pom.xml`, add inside the `<dependencies>` section, after the existing test dependencies:

```xml
        <!-- Step catalog YAML parsing — test only -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-yaml-jackson</artifactId>
            <version>${casehub.version}</version>
            <scope>test</scope>
        </dependency>
```

- [ ] **Step 2: Write the test class**

Create `webapp/src/test/java/io/casehub/iot/webapp/steps/IoTStepDefinitionTest.java`:

```java
package io.casehub.iot.webapp.steps;

import io.casehub.yaml.core.step.DeclarationFile;
import io.casehub.yaml.core.step.DeclarationParser;
import io.casehub.yaml.core.step.InvokeBinding;
import io.casehub.yaml.jackson.YamlMappers;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class IoTStepDefinitionTest {

    private static DeclarationFile stepFile;

    @BeforeAll
    static void loadStepDefinitions() throws IOException {
        var mapper = YamlMappers.create();
        try (var stream = IoTStepDefinitionTest.class.getClassLoader()
                .getResourceAsStream("steps/iot-steps.yaml")) {
            assertThat(stream).as("steps/iot-steps.yaml must exist on classpath").isNotNull();
            @SuppressWarnings("unchecked")
            Map<String, Object> yaml = mapper.readValue(stream, Map.class);
            stepFile = DeclarationParser.parse(yaml);
        }
    }

    @Test
    void namespaceIsIot() {
        assertThat(stepFile.namespace()).isEqualTo("iot");
    }

    @Test
    void allThreeActionsPresent() {
        List<String> expected = List.of("command", "state", "desired-state");
        assertThat(stepFile.actions()).containsKeys(expected.toArray(String[]::new));
        assertThat(stepFile.actions()).hasSize(3);
    }

    @Test
    void commandHasMcpBinding() {
        var action = stepFile.actions().get("command");
        assertThat(action.invoke()).isInstanceOf(InvokeBinding.Mcp.class);
        assertThat(((InvokeBinding.Mcp) action.invoke()).tool()).isEqualTo("iot_send_command");
    }

    @Test
    void stateHasMcpBinding() {
        var action = stepFile.actions().get("state");
        assertThat(action.invoke()).isInstanceOf(InvokeBinding.Mcp.class);
        assertThat(((InvokeBinding.Mcp) action.invoke()).tool()).isEqualTo("iot_get_state");
    }

    @Test
    void desiredStateHasNoInvokeBinding() {
        var action = stepFile.actions().get("desired-state");
        assertThat(action.invoke()).isNull();
    }

    @Test
    void allActionsHaveDescriptions() {
        for (var entry : stepFile.actions().entrySet()) {
            assertThat(entry.getValue().description())
                    .as("Action '%s' should have a description", entry.getKey())
                    .isNotBlank();
        }
    }

    @Test
    void commandHasRequiredInputs() {
        var action = stepFile.actions().get("command");
        assertThat(action.inputs()).containsKey("device");
        assertThat(action.inputs().get("device").required()).isTrue();
        assertThat(action.inputs()).containsKey("action");
        assertThat(action.inputs().get("action").required()).isTrue();
    }

    @Test
    void commandHasOptionalInputs() {
        var action = stepFile.actions().get("command");
        assertThat(action.inputs()).containsKey("params");
        assertThat(action.inputs().get("params").required()).isFalse();
        assertThat(action.inputs()).containsKey("correlation-id");
        assertThat(action.inputs().get("correlation-id").required()).isFalse();
    }

    @Test
    void stateHasRequiredDeviceInput() {
        var action = stepFile.actions().get("state");
        assertThat(action.inputs()).containsKey("device");
        assertThat(action.inputs().get("device").required()).isTrue();
        assertThat(action.inputs()).hasSize(1);
    }

    @Test
    void desiredStateHasOptionalPresetInput() {
        var action = stepFile.actions().get("desired-state");
        assertThat(action.inputs()).containsKey("preset");
        assertThat(action.inputs().get("preset").required()).isFalse();
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `mvn --batch-mode -pl webapp -Dtest=IoTStepDefinitionTest test`
Expected: FAIL — `steps/iot-steps.yaml must exist on classpath` assertion fails (file doesn't exist yet).

- [ ] **Step 4: Commit failing test**

```bash
git add webapp/pom.xml webapp/src/test/java/io/casehub/iot/webapp/steps/IoTStepDefinitionTest.java
git commit -m "test(#134): add IoTStepDefinitionTest — failing, YAML not yet created"
```

### Task 2: Create iot-steps.yaml and make the test pass

**Files:**
- Create: `webapp/src/main/resources/steps/iot-steps.yaml`

**Interfaces:**
- Consumes: nothing
- Produces: YAML file on classpath at `steps/iot-steps.yaml`, parseable by `DeclarationParser`

- [ ] **Step 1: Create the YAML step catalog file**

Create `webapp/src/main/resources/steps/iot-steps.yaml`:

```yaml
namespace: iot

actions:
  command:
    description: Dispatches a command to an IoT device via DeviceProvider
    inputs:
      device:
        type: STRING
        required: true
        description: Device identifier (e.g. light-living-1, lock-front)
      action:
        type: STRING
        required: true
        description: Command action (e.g. turn_on, turn_off, lock, unlock, set_temperature, set_brightness, arm, disarm)
      params:
        type: OBJECT
        description: Additional command parameters (e.g. { temperature: 22, unit: CELSIUS })
      correlation-id:
        type: STRING
        description: Correlation identifier for command tracking
    outputs:
      result:
        type: STRING
        description: "Command result: SENT, FAILED, or TIMEOUT"
      device:
        type: STRING
        description: Device identifier echo
      action:
        type: STRING
        description: Action echo
      correlationId:
        type: STRING
        description: Correlation identifier used
    invoke:
      mcp: "iot_send_command"

  state:
    description: Reads current state of an IoT device from DeviceRegistry
    inputs:
      device:
        type: STRING
        required: true
        description: Device identifier
    outputs:
      deviceId:
        type: STRING
        description: Device identifier
      deviceClass:
        type: STRING
        description: Matter device class (e.g. LIGHT, THERMOSTAT, LOCK, CONTACT_SENSOR)
      providerId:
        type: STRING
        description: Provider identifier (e.g. homeassistant, openhab)
      capabilities:
        type: OBJECT
        description: Current device capabilities and state values
      location:
        type: STRING
        description: Hierarchical location path (nullable)
    invoke:
      mcp: "iot_get_state"

  desired-state:
    description: >-
      Applies a desired device state configuration via the desired-state
      convergence engine. Used with 'delivery: desired-state' in playbooks.
      Accepts either a named preset or inline per-device goal specifications.
      When preset is not specified, remaining keys are treated as inline device
      goals — each key is a device ID mapping to a desired capability state
      object.
    inputs:
      preset:
        type: STRING
        description: Named preset to apply (e.g. nighttime, morning, away). Resolved via IoTPresetResolver with import composition and last-wins merge.
    outputs:
      provisioned:
        type: INTEGER
        description: Number of devices successfully provisioned
      converged:
        type: BOOLEAN
        description: Whether all devices reached desired state
```

- [ ] **Step 2: Run the test to verify it passes**

Run: `mvn --batch-mode -pl webapp -Dtest=IoTStepDefinitionTest test`
Expected: all 10 tests PASS.

- [ ] **Step 3: Commit**

```bash
git add webapp/src/main/resources/steps/iot-steps.yaml
git commit -m "feat(#134): define IoT step catalog entries for yaml-core

Define three step catalog entries in iot-steps.yaml:
- iot.command — device command dispatch via MCP
- iot.state — device state query via MCP
- iot.desired-state — desired-state delivery (metadata-only)

Refs #134"
```

## References

- [2026-10-12-iot-step-catalog-design.md] — design spec this plan implements
- `fsitrading/app/src/main/resources/steps/trading-steps.yaml` — reference YAML format
- `fsitrading/app/src/test/java/io/casehub/fsitrading/app/steps/TradingStepDefinitionTest.java` — reference test
- `io.casehub.yaml.core.step.DeclarationParser` — YAML parser
- `io.casehub.yaml.core.step.InvokeBinding` — invoke binding types
- `io.casehub.yaml.jackson.YamlMappers` — YAML ObjectMapper factory
- GitHub #134 — focal issue
