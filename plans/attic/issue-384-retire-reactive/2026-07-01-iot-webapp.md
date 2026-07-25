# IoT Webapp Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone Quarkus application that surfaces the full IoT platform as an operational console — device management, situational awareness via RAS, case orchestration, human WorkItems — with composable TypeScript pages.

**Architecture:** Three new modules in casehub-iot: `webapp-api` (reusable ganglia, case descriptors, worker functions, risk classifier — pure domain logic), `webapp-drools` (DroolsCEP temporal pattern ganglia — heavy Drools dependency isolated), `webapp` (standalone Quarkus app wiring IoT + RAS + engine + work + ledger with REST endpoints, SSE, Flyway migrations, and TypeScript pages via Quinoa). Follows the AML pattern: YAML case definitions + `*CaseDescriptor` POJOs + `YamlCaseHub` subclasses.

**Tech Stack:** Java 21, Quarkus 3.x, SmallRye Mutiny, Drools CEP, Flyway, PostgreSQL (H2 for tests), Jackson, TypeScript, casehub-pages DSL (`@casehubio/ui`, `@casehubio/data`), Quinoa.

## Global Constraints

- **Spec:** `docs/superpowers/specs/2026-07-01-iot-webapp-design.md` — the authoritative design. Consult before implementing every task.
- **Platform coherence:** Read `casehub-parent/docs/PLATFORM.md` and relevant protocols in `casehub/garden/docs/protocols/` before implementing. Do a final coherence review before completion.
- **Module tier structure:** `webapp-api` is Tier 1 (pure-Java SPI + CDI/Mutiny `provided`-scope). No JPA, no Quarkus runtime deps. `webapp-drools` is Tier 2 (Drools runtime). `webapp` is Tier 3 (full Quarkus app). Per `module-tier-structure` protocol.
- **Case definition layers:** YAML (structure) + `*CaseDescriptor` POJO (business logic) + `YamlCaseHub` subclass (CDI entry point). Per `case-definition-layers` protocol.
- **Persistence CDI priority:** `@DefaultBean` (no-op) → `@ApplicationScoped` → `@Alternative @Priority(N)`. Per `persistence-backend-cdi-priority` protocol.
- **Auth retrofit readiness:** `@RolesAllowed` annotations on all REST resources. Inert without `casehub-platform-oidc`. No auth logic in service layers. Per `auth-retrofit-readiness` protocol.
- **Flyway scoping:** All webapp migrations at `db/iot-webapp/migration/` starting at V500. Three datasources: default (bridge + webapp), `iot-work` (work + ledger), `iot-ras` (RAS). Per `flyway-repo-scoped-migration-path` protocol.
- **Submodule folder naming:** Short names — `webapp-api`, `webapp-drools`, `webapp`. Not `casehub-iot-webapp-api`. Per `maven-submodule-folder-naming` protocol.
- **TDD:** Use `superpowers:test-driven-development` before writing implementation. Tests first, always.
- **Java dev:** Use `java-dev` skill for all Java implementation.
- **TypeScript dev:** Use `ts-dev` skill for all TypeScript implementation.
- **Code review:** Use `superpowers:requesting-code-review` before committing each task.
- **Package base:** `io.casehub.iot.webapp` for all new classes.
- **IoT API types:** `DeviceEntity`, `DeviceProvider`, `DeviceRegistry`, `DeviceCommand`, `CommandResult`, `StateChangeEvent`, `BridgeAuditStore`, `BridgeAuditQuery` — from `casehub-iot-api`. Import, do not redefine.
- **RAS types:** `Ganglion`, `JavaSwitchGanglion`, `DetectionResult`, `DetectionSignal`, `SituationContext`, `SituationDefinition`, `CaseTriggerConfig`, `ChainMode`, `TriggerMode` — from `casehub-ras-api`. Import, do not redefine.
- **Engine types:** `YamlCaseHub`, `CaseDefinition`, `CaseHub` — from `casehub-engine-api`. `Worker`, `Capability`, `WorkerFunction`, `WorkerResult`, `PlannedAction` — from `casehub-worker-api`. `ActionRiskClassifier`, `RiskDecision`, `ClassificationContext` — from `casehub-engine-api`.

---

### Task 1: Module Scaffolding

**Files:**
- Create: `webapp-api/pom.xml`
- Create: `webapp-drools/pom.xml`
- Create: `webapp/pom.xml`
- Modify: `pom.xml` (parent — add 3 modules)
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/package-info.java`
- Create: `webapp-drools/src/main/java/io/casehub/iot/webapp/drools/package-info.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/package-info.java`

**Interfaces:**
- Consumes: Parent POM `casehub-iot-parent` version `0.2-SNAPSHOT`
- Produces: Three buildable modules — `mvn install` passes on all three

- [ ] **Step 1: Read existing parent pom.xml to understand module pattern**

Read `pom.xml` at project root. Note the `<modules>` section, `<dependencyManagement>`, and existing module artifact IDs.

- [ ] **Step 2: Create `webapp-api/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-iot-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-iot-webapp-api</artifactId>
    <name>CaseHub IoT Webapp API</name>
    <description>Reusable IoT ganglia, case descriptors, worker functions, and REST interfaces</description>

    <dependencies>
        <!-- IoT domain types -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-api</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- RAS SPI (Ganglion, SituationDefinition, DetectionResult) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ras-api</artifactId>
        </dependency>

        <!-- RAS runtime (SituationDefinitionProvider — lives in runtime, not api) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ras</artifactId>
        </dependency>

        <!-- Engine API (CaseDefinition, YamlCaseHub, ActionRiskClassifier) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-api</artifactId>
        </dependency>

        <!-- Worker primitives (Worker, Capability, WorkerFunction, PlannedAction) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-worker-api</artifactId>
        </dependency>

        <!-- Connectors (HouseholdNotificationWorkerFunction uses ConnectorService) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-core</artifactId>
        </dependency>

        <!-- CDI annotations — provided scope, inert without runtime -->
        <dependency>
            <groupId>jakarta.enterprise</groupId>
            <artifactId>jakarta.enterprise.cdi-api</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- JAX-RS annotations — provided scope for REST interfaces -->
        <dependency>
            <groupId>jakarta.ws.rs</groupId>
            <artifactId>jakarta.ws.rs-api</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- Mutiny — provided scope -->
        <dependency>
            <groupId>io.smallrye.reactive</groupId>
            <artifactId>mutiny</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- CloudEvents SDK -->
        <dependency>
            <groupId>io.cloudevents</groupId>
            <artifactId>cloudevents-core</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

Note: Check the parent POM `<dependencyManagement>` for exact versions of `casehub-ras-api`, `casehub-engine-api`, `casehub-worker-api`, `casehub-connectors-core`. If not managed, add them to the parent's `<dependencyManagement>` section with `${casehub.version}` property. These are cross-repo dependencies — verify they are published to GitHub Packages.

- [ ] **Step 3: Create `webapp-drools/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-iot-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-iot-webapp-drools</artifactId>
    <name>CaseHub IoT Webapp Drools</name>
    <description>DroolsCEP temporal pattern ganglia for IoT situational awareness</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-webapp-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ras-drools</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 4: Create `webapp/pom.xml`**

Full Quarkus application module. Dependencies from the spec's Foundation Layer Wiring table. Include:
- `casehub-iot-webapp-api`, `casehub-iot-webapp-drools`
- `casehub-iot-homeassistant`, `casehub-iot-openhab` (runtime scope)
- `casehub-iot-bridge-server`, `casehub-iot-bridge-persistence-jpa`
- `casehub-ras` (runtime), `casehub-ras-persistence-jpa`
- `casehub-engine`, `casehub-engine-scheduler-quartz`
- `casehub-engine-work-adapter`, `casehub-engine-blackboard`
- `casehub-engine-persistence-memory` (in-memory engine SPIs — same pattern as AML)
- `casehub-work`
- `casehub-ledger`
- `casehub-platform`, `casehub-platform-expression`
- `casehub-connectors-core`
- `quarkus-rest`, `quarkus-rest-jackson`, `quarkus-rest-sse` (for SSE endpoints)
- `quarkus-hibernate-orm`, `quarkus-jdbc-postgresql` (runtime), `quarkus-flyway`
- `quarkus-scheduler` (for retention job)
- `quarkus-quinoa` (for TypeScript pages)
- Test: `quarkus-junit5`, `quarkus-jdbc-h2`, `assertj-core`, `casehub-iot-testing`

Include `quarkus-maven-plugin` with `<goal>build</goal>` for production augmentation.

- [ ] **Step 5: Update parent `pom.xml` — add three modules**

Add to the `<modules>` section:
```xml
<module>webapp-api</module>
<module>webapp-drools</module>
<module>webapp</module>
```

Order: `webapp-api` before `webapp-drools` before `webapp` (dependency order).

Also add to `<dependencyManagement>` if not already present:
- `casehub-ras-api`, `casehub-ras`, `casehub-ras-drools`, `casehub-ras-persistence-jpa`
- `casehub-engine-api`, `casehub-engine`, `casehub-engine-scheduler-quartz`, `casehub-engine-work-adapter`, `casehub-engine-blackboard`, `casehub-engine-persistence-memory`
- `casehub-worker-api`
- `casehub-work`
- `casehub-ledger`
- `casehub-connectors-core`
- `casehub-platform`, `casehub-platform-expression`

- [ ] **Step 6: Create package-info.java for each module**

Create the base package directories and `package-info.java` files:
- `webapp-api/src/main/java/io/casehub/iot/webapp/package-info.java`
- `webapp-drools/src/main/java/io/casehub/iot/webapp/drools/package-info.java`
- `webapp/src/main/java/io/casehub/iot/webapp/app/package-info.java`

- [ ] **Step 7: Verify build**

Run: `mvn --batch-mode install -pl webapp-api,webapp-drools,webapp -am`
Expected: BUILD SUCCESS on all three modules (empty modules with correct dependency resolution)

- [ ] **Step 8: Commit**

```bash
git add webapp-api/ webapp-drools/ webapp/ pom.xml
git commit -m "chore: scaffold webapp-api, webapp-drools, webapp modules (#TBD)"
```

---

### Task 2: JavaSwitch Ganglia (webapp-api)

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/ganglion/TemperatureThresholdGanglion.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/ganglion/MotionAtTimeGanglion.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/ganglion/DeviceUnavailableGanglion.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/ganglion/LockStateGanglion.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/ganglion/PowerAnomalyGanglion.java`
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/ganglion/TemperatureThresholdGanglionTest.java`
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/ganglion/MotionAtTimeGanglionTest.java`
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/ganglion/DeviceUnavailableGanglionTest.java`
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/ganglion/LockStateGanglionTest.java`
- Create: `webapp-api/src/test/java/io/casehub/iot/webapp/ganglion/PowerAnomalyGanglionTest.java`

**Interfaces:**
- Consumes: `JavaSwitchGanglion` (abstract class from `casehub-ras-api`), `CloudEvent`, `SituationContext`, `DetectionResult`, `DetectionSignal`. IoT types: `StateChangeEvent`, `DeviceEntity`, `SensorDevice`, `ThermostatDevice`, `PresenceSensor`, `LockDevice`, `PowerSensor`, `Temperature`.
- Produces: 5 `@ApplicationScoped` ganglion beans. Each handles specific CloudEvent types and returns `DetectionResult` via `evaluate(CloudEvent, SituationContext)`.

**Pattern (shown fully for TemperatureThresholdGanglion — same structure for all 5):**

- [ ] **Step 1: Write failing test for TemperatureThresholdGanglion**

```java
package io.casehub.iot.webapp.ganglion;

import io.casehub.ras.api.DetectionSignal;
import io.casehub.ras.api.SituationContext;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.net.URI;
import java.time.Instant;
import java.time.OffsetDateTime;

import static org.assertj.core.api.Assertions.assertThat;

class TemperatureThresholdGanglionTest {

    private final TemperatureThresholdGanglion ganglion =
            new TemperatureThresholdGanglion(new BigDecimal("35.0"), new BigDecimal("5.0"));

    private static final SituationContext CTX = SituationContext.initial(
            "fire-risk", "home-1", "default-tenant", Instant.now());

    @Test
    void detectsTemperatureAboveUpperThreshold() {
        var event = buildCloudEvent("io.casehub.iot.state_change.sensor",
                """
                {"after":{"deviceClass":"SENSOR","sensorType":"TEMPERATURE","numericValue":38.5,"unit":"CELSIUS"}}
                """);

        var result = ganglion.evaluate(event, CTX);

        assertThat(result.signal()).isEqualTo(DetectionSignal.DETECTED);
        assertThat(result.confidence()).isGreaterThan(0.5);
        assertThat(result.ganglionId()).isEqualTo("temperature-threshold");
    }

    @Test
    void returnsNoiseForNormalTemperature() {
        var event = buildCloudEvent("io.casehub.iot.state_change.sensor",
                """
                {"after":{"deviceClass":"SENSOR","sensorType":"TEMPERATURE","numericValue":22.0,"unit":"CELSIUS"}}
                """);

        var result = ganglion.evaluate(event, CTX);

        assertThat(result.signal()).isEqualTo(DetectionSignal.NOISE);
    }

    @Test
    void detectsTemperatureBelowLowerThreshold() {
        var event = buildCloudEvent("io.casehub.iot.state_change.sensor",
                """
                {"after":{"deviceClass":"SENSOR","sensorType":"TEMPERATURE","numericValue":3.0,"unit":"CELSIUS"}}
                """);

        var result = ganglion.evaluate(event, CTX);

        assertThat(result.signal()).isEqualTo(DetectionSignal.DETECTED);
    }

    @Test
    void handlesNonTemperatureSensorAsNoise() {
        var event = buildCloudEvent("io.casehub.iot.state_change.sensor",
                """
                {"after":{"deviceClass":"SENSOR","sensorType":"HUMIDITY","numericValue":85.0,"unit":"%"}}
                """);

        var result = ganglion.evaluate(event, CTX);

        assertThat(result.signal()).isEqualTo(DetectionSignal.NOISE);
    }

    @Test
    void handledEventTypesIncludesSensorAndThermostat() {
        assertThat(ganglion.handledEventTypes()).containsExactlyInAnyOrder(
                "io.casehub.iot.state_change.sensor",
                "io.casehub.iot.state_change.thermostat");
    }

    private static CloudEvent buildCloudEvent(String type, String data) {
        return CloudEventBuilder.v1()
                .withId("test-" + System.nanoTime())
                .withSource(URI.create("/casehub-iot"))
                .withType(type)
                .withTime(OffsetDateTime.now())
                .withData("application/json", data.getBytes())
                .withExtension("tenancyid", "default-tenant")
                .build();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -pl webapp-api -Dtest=TemperatureThresholdGanglionTest`
Expected: FAIL — `TemperatureThresholdGanglion` class does not exist.

- [ ] **Step 3: Implement TemperatureThresholdGanglion**

```java
package io.casehub.iot.webapp.ganglion;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.ras.api.DetectionResult;
import io.casehub.ras.api.JavaSwitchGanglion;
import io.casehub.ras.api.SituationContext;
import io.cloudevents.CloudEvent;
import jakarta.enterprise.context.ApplicationScoped;

import java.math.BigDecimal;
import java.util.Map;
import java.util.Set;

@ApplicationScoped
public class TemperatureThresholdGanglion extends JavaSwitchGanglion {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final BigDecimal upperThreshold;
    private final BigDecimal lowerThreshold;

    public TemperatureThresholdGanglion() {
        this(new BigDecimal("40.0"), new BigDecimal("2.0"));
    }

    public TemperatureThresholdGanglion(BigDecimal upperThreshold, BigDecimal lowerThreshold) {
        super("temperature-threshold", Set.of(
                "io.casehub.iot.state_change.sensor",
                "io.casehub.iot.state_change.thermostat"));
        this.upperThreshold = upperThreshold;
        this.lowerThreshold = lowerThreshold;
    }

    @Override
    protected DetectionResult evaluate(CloudEvent event, SituationContext context) {
        try {
            var data = event.getData();
            if (data == null) return noise();

            var root = MAPPER.readTree(data.toBytes());
            var after = root.path("after");
            if (after.isMissingNode()) return noise();

            var sensorType = after.path("sensorType").asText("");
            if (!"TEMPERATURE".equals(sensorType) && !isThermostateEvent(event)) return noise();

            BigDecimal tempValue = extractTemperatureCelsius(after);
            if (tempValue == null) return noise();

            if (tempValue.compareTo(upperThreshold) > 0) {
                double overshoot = tempValue.subtract(upperThreshold)
                        .divide(upperThreshold, 2, java.math.RoundingMode.HALF_UP)
                        .doubleValue();
                double confidence = Math.min(1.0, 0.6 + overshoot);
                return detected(confidence, Map.of(
                        "temperature", tempValue, "threshold", upperThreshold, "direction", "ABOVE"));
            }

            if (tempValue.compareTo(lowerThreshold) < 0) {
                double undershoot = lowerThreshold.subtract(tempValue)
                        .divide(lowerThreshold.abs().max(BigDecimal.ONE), 2, java.math.RoundingMode.HALF_UP)
                        .doubleValue();
                double confidence = Math.min(1.0, 0.6 + undershoot);
                return detected(confidence, Map.of(
                        "temperature", tempValue, "threshold", lowerThreshold, "direction", "BELOW"));
            }

            return noise();
        } catch (Exception e) {
            return noise();
        }
    }

    private boolean isThermostateEvent(CloudEvent event) {
        return "io.casehub.iot.state_change.thermostat".equals(event.getType());
    }

    private BigDecimal extractTemperatureCelsius(JsonNode after) {
        var numericValue = after.path("numericValue");
        var currentTemperature = after.path("currentTemperature");

        if (numericValue.isNumber()) {
            var unit = after.path("unit").asText("CELSIUS");
            var value = numericValue.decimalValue();
            return "FAHRENHEIT".equals(unit) ? fahrenheitToCelsius(value) : value;
        }

        if (currentTemperature.isObject()) {
            var value = currentTemperature.path("value").decimalValue();
            var unit = currentTemperature.path("unit").asText("CELSIUS");
            return "FAHRENHEIT".equals(unit) ? fahrenheitToCelsius(value) : value;
        }

        return null;
    }

    private static BigDecimal fahrenheitToCelsius(BigDecimal f) {
        return f.subtract(new BigDecimal("32"))
                .multiply(new BigDecimal("5"))
                .divide(new BigDecimal("9"), 2, java.math.RoundingMode.HALF_UP);
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn test -pl webapp-api -Dtest=TemperatureThresholdGanglionTest`
Expected: PASS — all 5 tests green.

- [ ] **Step 5: Implement remaining 4 ganglia + tests (same TDD cycle each)**

For each ganglion, write the test first, verify it fails, implement, verify it passes:

**MotionAtTimeGanglion** (`motion-at-time`):
- Handles: `io.casehub.iot.state_change.presence_sensor`
- Detection: `PresenceSensor.isPresent == true` AND current time outside configured hours (default 23:00–06:00)
- Tests: motion during restricted hours → DETECTED, motion during day → NOISE, no-motion during night → NOISE

**DeviceUnavailableGanglion** (`device-unavailable`):
- Handles: all `io.casehub.iot.state_change.*` (use `Set.of("io.casehub.iot.state_change.*")` — verify RAS wildcard support, otherwise enumerate all device class event types)
- Detection: `after.available == false` in the CloudEvent data
- Tests: device goes unavailable → DETECTED, device stays available → NOISE

**LockStateGanglion** (`lock-state`):
- Handles: `io.casehub.iot.state_change.lock`
- Detection: `before.isLocked == true && after.isLocked == false` (unexpected unlock)
- Tests: locked→unlocked → DETECTED, unlocked→locked → NOISE (securing), already unlocked → NOISE

**PowerAnomalyGanglion** (`power-anomaly`):
- Handles: `io.casehub.iot.state_change.power_sensor`
- Detection: `PowerSensor.power` exceeds configurable threshold (default 5000W)
- Tests: power spike above threshold → DETECTED, normal power → NOISE, null power → NOISE

- [ ] **Step 6: Commit**

```bash
git add webapp-api/src/
git commit -m "feat: 5 JavaSwitch IoT ganglia — temperature, motion, device-offline, lock, power (#TBD)"
```

---

### Task 3: Worker Functions + ActionRiskClassifier (webapp-api)

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/worker/DeviceCommandWorkerFunction.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/worker/HouseholdNotificationWorkerFunction.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/worker/HumanDecisionWorkerFunction.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/risk/IoTActionRiskClassifier.java`
- Create: test files for each

**Interfaces:**
- Consumes: `WorkerFunction` (from `casehub-worker-api`), `DeviceProvider`, `DeviceCommand`, `CommandResult` (from `casehub-iot-api`), `ActionRiskClassifier`, `RiskDecision`, `ClassificationContext`, `PlannedAction` (from `casehub-engine-api`), `ConnectorService` (from `casehub-connectors-core`)
- Produces: 3 worker function classes usable by case descriptors, 1 `ActionRiskClassifier` implementation

- [ ] **Step 1: Write failing test for IoTActionRiskClassifier**

```java
package io.casehub.iot.webapp.risk;

import io.casehub.api.spi.ClassificationContext;
import io.casehub.api.spi.RiskDecision;
import io.casehub.worker.api.PlannedAction;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class IoTActionRiskClassifierTest {

    private final IoTActionRiskClassifier classifier = new IoTActionRiskClassifier();

    private static ClassificationContext ctx(String caseDefName) {
        return new ClassificationContext(
                "iot-worker", UUID.randomUUID(), "default-tenant",
                caseDefName, "device-command-dispatch", "dispatch");
    }

    @Test
    void safetyCommandsDuringSafetyAlertAreAutonomous() {
        var action = PlannedAction.of("Unlock doors", "TURN_OFF",
                Map.of("action", "TURN_OFF", "context", "safety-alert"));
        var result = classifier.classify(action, ctx("safety-alert"));
        assertThat(result).isInstanceOf(RiskDecision.Autonomous.class);
    }

    @Test
    void lockCommandsRequireGate() {
        var action = PlannedAction.of("Lock front door", "LOCK",
                Map.of("action", "LOCK"));
        var result = classifier.classify(action, ctx("security-alert"));
        assertThat(result).isInstanceOf(RiskDecision.GateRequired.class);
    }

    @Test
    void hvacAdjustmentsAreAutonomous() {
        var action = PlannedAction.of("Set temperature", "SET_TEMPERATURE",
                Map.of("action", "SET_TEMPERATURE"));
        var result = classifier.classify(action, ctx("hvac-anomaly"));
        assertThat(result).isInstanceOf(RiskDecision.Autonomous.class);
    }

    @Test
    void unknownCommandsRequireGate() {
        var action = PlannedAction.of("Unknown action", "CUSTOM_ACTION",
                Map.of("action", "CUSTOM_ACTION"));
        var result = classifier.classify(action, ctx("generic-response"));
        assertThat(result).isInstanceOf(RiskDecision.GateRequired.class);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

- [ ] **Step 3: Implement IoTActionRiskClassifier**

```java
package io.casehub.iot.webapp.risk;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.ClassificationContext;
import io.casehub.api.spi.RiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.casehub.worker.api.PlannedAction;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.List;
import java.util.Set;

@RiskClassifier
@ApplicationScoped
public class IoTActionRiskClassifier implements ActionRiskClassifier {

    private static final Set<String> SAFETY_CASE_TYPES = Set.of("safety-alert");
    private static final Set<String> AUTONOMOUS_ACTIONS = Set.of(
            "TURN_ON", "TURN_OFF", "SET_TEMPERATURE", "SET_POSITION", "SET_VOLUME");
    private static final Set<String> GATED_ACTIONS = Set.of("LOCK", "UNLOCK");

    @Override
    public RiskDecision classify(PlannedAction action, ClassificationContext context) {
        var actionType = action.actionType();

        if (SAFETY_CASE_TYPES.contains(context.caseDefinitionName())) {
            return new RiskDecision.Autonomous();
        }

        if (AUTONOMOUS_ACTIONS.contains(actionType)) {
            return new RiskDecision.Autonomous();
        }

        if (GATED_ACTIONS.contains(actionType)) {
            return new RiskDecision.GateRequired(
                    "Lock/unlock commands require human approval",
                    true,
                    List.of("iot-operator"),
                    null,
                    "casehubio/iot/oversight");
        }

        return new RiskDecision.GateRequired(
                "Unknown IoT command — manual review required",
                true,
                List.of("iot-admin"),
                null,
                "casehubio/iot/oversight");
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

- [ ] **Step 5: Write tests + implement DeviceCommandWorkerFunction**

This is a `WorkerFunction.Sync` implementation that:
- Extracts `targetDeviceId`, `action`, `parameters` from the case context input map
- Looks up the device via `DeviceRegistry.findById()`
- Resolves the owning provider via `DeviceProvider` CDI beans
- Calls `provider.dispatch(DeviceCommand)` and awaits the result
- Returns `WorkerResult` with `CommandResult` or `PlannedAction` if the action type requires gating

Constructor-injected: `Instance<DeviceProvider>`, `DeviceRegistry`

- [ ] **Step 6: Write tests + implement HouseholdNotificationWorkerFunction**

This calls `ConnectorService.send()` with the tenancy and message template from case context. Constructor-injected: `ConnectorService`

- [ ] **Step 7: Write tests + implement HumanDecisionWorkerFunction**

This creates a WorkItem via the casehub-work API with the situation context. It returns `WorkerResult` that blocks the case until the WorkItem is completed. Constructor-injected: `WorkBroker` or `WorkItemService`

Note: Verify the exact casehub-work API for programmatic WorkItem creation by reading the AML example and `casehub-work-api` source before implementing.

- [ ] **Step 8: Commit**

```bash
git add webapp-api/src/
git commit -m "feat: IoT worker functions + ActionRiskClassifier (#TBD)"
```

---

### Task 4: Case Descriptors + REST Interfaces + Situation YAML (webapp-api)

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/SafetyAlertCaseDescriptor.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/SecurityAlertCaseDescriptor.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/HvacAnomalyCaseDescriptor.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/engine/GenericResponseCaseDescriptor.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/rest/DeviceResponse.java` (+ other REST records)
- Create: `webapp-api/src/main/resources/META-INF/ras-iot-situations.yaml`
- Create: test files for descriptors

**Interfaces:**
- Consumes: `Worker.builder()`, `Capability.builder()`, `WorkerFunction`, `FlowWorkerFunction`, worker functions from Task 3
- Produces: 4 case descriptors with `registerWorkers(CaseDefinition)`, REST request/response records, 2 situation definitions YAML

- [ ] **Step 1: Write failing test for SafetyAlertCaseDescriptor**

```java
package io.casehub.iot.webapp.engine;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SafetyAlertCaseDescriptorTest {

    @Test
    void workersContainsExpectedCapabilities() {
        var descriptor = new SafetyAlertCaseDescriptor(null, null, null);
        var workers = descriptor.workers();
        assertThat(workers).hasSize(3);
        assertThat(workers.stream().flatMap(w -> w.capabilities().stream()).map(c -> c.name()))
                .containsExactlyInAnyOrder(
                        "device-command-dispatch",
                        "household-notification",
                        "human-acknowledgement");
    }
}
```

- [ ] **Step 2: Implement SafetyAlertCaseDescriptor**

Follow the AML `AmlInvestigationCaseDescriptor` pattern: plain POJO, constructor takes CDI-managed dependencies, `workers()` returns `List<Worker>`, `registerWorkers(CaseDefinition)` calls `definition.addWorkers(workers())`. Each worker uses `Worker.builder().name(...).capabilities(...).function(...)`.

- [ ] **Step 3: Implement remaining 3 descriptors (same pattern)**

Each follows the same structure. Key differences:
- `SecurityAlertCaseDescriptor` — 4 capabilities, includes camera activation
- `HvacAnomalyCaseDescriptor` — 3 capabilities, setpoint correction before escalation
- `GenericResponseCaseDescriptor` — 1 capability (`human-triage`), no device commands

- [ ] **Step 4: Create REST request/response records**

In `webapp-api/src/main/java/io/casehub/iot/webapp/rest/`:
- `DeviceResponse.java` — record mapping `DeviceEntity` to REST JSON
- `CommandRequest.java` — record for `POST /api/devices/{id}/commands`
- `CommandResponse.java` — record wrapping `CommandResult`
- `SituationDefinitionRequest.java` — record for create/update situation definitions
- `HealthOverviewResponse.java` — composite record for health endpoint

- [ ] **Step 5: Create `META-INF/ras-iot-situations.yaml`**

```yaml
situations:
  - situationId: unexpected-unlock
    eventTypes:
      - io.casehub.iot.state_change.lock
    chainMode:
      type: or
      ganglia:
        - lock-state
    triggerMode:
      type: repeating
      cooldown: PT30M
    triggerConfig:
      caseNamespace: io.casehub.iot
      caseName: security-alert
      caseVersion: "1.0"

  - situationId: device-offline
    eventTypes:
      - io.casehub.iot.state_change.switch
      - io.casehub.iot.state_change.light
      - io.casehub.iot.state_change.thermostat
      - io.casehub.iot.state_change.sensor
      - io.casehub.iot.state_change.presence_sensor
      - io.casehub.iot.state_change.power_sensor
      - io.casehub.iot.state_change.lock
      - io.casehub.iot.state_change.cover
      - io.casehub.iot.state_change.media_player
      - io.casehub.iot.state_change.fan
      - io.casehub.iot.state_change.camera
    eventBufferDelay: PT5S
    chainMode:
      type: or
      ganglia:
        - device-unavailable
    triggerMode:
      type: fireOnce
    triggerConfig:
      caseNamespace: io.casehub.iot
      caseName: generic-response
      caseVersion: "1.0"
```

Note: Verify the YAML format matches `YamlSituationDefinitionProvider`'s parser by reading `casehub-ras/runtime/src/main/java/io/casehub/ras/runtime/YamlSituationDefinitionProvider.java`. The event type list for `device-offline` enumerates all known device class event types because RAS may not support wildcards.

- [ ] **Step 6: Commit**

```bash
git add webapp-api/src/
git commit -m "feat: case descriptors, REST interfaces, default situation definitions (#TBD)"
```

---

### Task 5: DroolsCEP Ganglia (webapp-drools)

**Files:**
- Create: `webapp-drools/src/main/java/io/casehub/iot/webapp/drools/SustainedTemperatureRiseGanglion.java`
- Create: `webapp-drools/src/main/java/io/casehub/iot/webapp/drools/MultiRoomMotionGanglion.java`
- Create: `webapp-drools/src/main/resources/META-INF/ras-iot-drools-situations.yaml`
- Create: test files for both ganglia

**Interfaces:**
- Consumes: `DroolsGanglion` (from `casehub-ras-drools`), CloudEvent, SituationContext
- Produces: 2 DroolsCEP ganglion beans, 3 Drools-dependent situation definitions

- [ ] **Step 1: Read `DroolsGanglion` source from casehub-ras-drools**

Understand the constructor, session mode, rule loading, `ResultCollectorChannel`, and `ResultCollectionStrategy`. Read test examples in `casehub-ras-drools/src/test/`.

- [ ] **Step 2: Write tests + implement SustainedTemperatureRiseGanglion**

CEP rule: temperature readings (from the `TemperatureThresholdGanglion`'s event types) within a sliding window of M minutes (configurable, default 30). If N consecutive readings (default 5) show monotonic increase ≥ threshold delta (default 3C), fire DETECTED.

Uses STATEFUL session mode (maintains state across events in the same correlation key).

- [ ] **Step 3: Write tests + implement MultiRoomMotionGanglion**

CEP rule: motion events (`io.casehub.iot.state_change.presence_sensor`) from 3+ distinct `deviceId` values within a configurable time window (default 2 minutes). Uses `distinct` aggregation over deviceId.

Uses EPHEMERAL session mode (no state between correlation keys).

- [ ] **Step 4: Create `META-INF/ras-iot-drools-situations.yaml`**

```yaml
situations:
  - situationId: fire-risk
    eventTypes:
      - io.casehub.iot.state_change.sensor
      - io.casehub.iot.state_change.thermostat
    chainMode:
      type: and
      requiredGanglia:
        - temperature-threshold
        - sustained-rise
    triggerMode:
      type: repeating
      cooldown: PT5M
    triggerConfig:
      caseNamespace: io.casehub.iot
      caseName: safety-alert
      caseVersion: "1.0"

  - situationId: intrusion
    eventTypes:
      - io.casehub.iot.state_change.presence_sensor
    eventBufferDelay: PT2S
    chainMode:
      type: threshold
      ganglia:
        - motion-at-time
        - multi-room-motion
      minConfidence: 0.7
    triggerMode:
      type: repeating
      cooldown: PT10M
    triggerConfig:
      caseNamespace: io.casehub.iot
      caseName: security-alert
      caseVersion: "1.0"

  - situationId: hvac-failure
    eventTypes:
      - io.casehub.iot.state_change.thermostat
    chainMode:
      type: count
      ganglionId: sustained-rise
      requiredCount: 3
    triggerMode:
      type: fireOnce
    triggerConfig:
      caseNamespace: io.casehub.iot
      caseName: hvac-anomaly
      caseVersion: "1.0"
```

- [ ] **Step 5: Commit**

```bash
git add webapp-drools/src/
git commit -m "feat: DroolsCEP ganglia — sustained temperature rise + multi-room motion (#TBD)"
```

---

### Task 6: Flyway Migrations + JPA Entities + Retention (webapp)

**Files:**
- Create: `webapp/src/main/resources/db/iot-webapp/migration/V500__create_iot_situation_definition.sql`
- Create: `webapp/src/main/resources/db/iot-webapp/migration/V501__create_iot_case_command_log.sql`
- Create: `webapp/src/main/resources/db/iot-webapp/migration/V502__create_iot_device_state_history.sql`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/persistence/IoTSituationDefinitionEntity.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/persistence/IoTCaseCommandLogEntity.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/persistence/IoTDeviceStateHistoryEntity.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/persistence/StateHistoryRetentionConfig.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/persistence/StateHistoryRetentionJob.java`
- Create: test files

**Interfaces:**
- Consumes: Flyway, JPA/Hibernate, `@ConfigMapping`, `@Scheduled`
- Produces: 3 JPA entities with JSONB support, 3 Flyway migrations, configurable retention job

- [ ] **Step 1: Create Flyway migrations**

V500 — `iot_situation_definition` table (see spec Data Model section for exact DDL):
```sql
CREATE TABLE iot_situation_definition (
    id              UUID NOT NULL PRIMARY KEY,
    situation_id    VARCHAR(255) NOT NULL,
    tenancy_id      VARCHAR(255) NOT NULL,
    definition      JSONB NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL,
    CONSTRAINT uq_iot_sit_def_tenant UNIQUE (situation_id, tenancy_id)
);
CREATE INDEX idx_iot_situation_def_tenancy ON iot_situation_definition (tenancy_id);
```

V501 — `iot_case_command_log` table (see spec).
V502 — `iot_device_state_history` table (see spec).

- [ ] **Step 2: Create JPA entities**

Follow the `BridgeAuditJpaEntity` pattern — immutable, constructor injection, no setters, `@JdbcTypeCode(SqlTypes.JSON)` for JSONB columns. `GenerationType.UUID` for primary keys.

- [ ] **Step 3: Create StateHistoryRetentionJob**

Follow the `JpaAuditRetentionJob` pattern exactly:
- `@ConfigMapping(prefix = "casehub.iot.webapp.state-history")` with `Optional<Integer> retentionDays()` (default absent = disabled) and `Duration purgeInterval()` (default `PT1H`)
- `@Scheduled(every = "${...purge-interval:1h}", concurrentExecution = SKIP)`
- Bulk JPQL DELETE, warn threshold at 10K rows

- [ ] **Step 4: Write `@QuarkusTest` for entity persistence + retention**

Use H2 with `MODE=PostgreSQL` for tests. Test save/query round-trip for all 3 entities and retention purge logic.

- [ ] **Step 5: Commit**

```bash
git add webapp/src/
git commit -m "feat: Flyway migrations, JPA entities, state history retention (#TBD)"
```

---

### Task 7: StateChangeHistoryObserver + JpaRuntimeSituationDefinitionProvider (webapp)

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/observer/StateChangeHistoryObserver.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/situation/JpaRuntimeSituationDefinitionProvider.java`
- Create: test files

**Interfaces:**
- Consumes: `StateChangeEvent` (CDI async), `SituationDefinitionProvider` (from `casehub-ras-runtime`), `EntityManager`, JPA entities from Task 6
- Produces: CDI observer that persists state changes, `SituationDefinitionProvider` that merges classpath + database definitions

- [ ] **Step 1: Write tests + implement StateChangeHistoryObserver**

`@ObservesAsync StateChangeEvent` — maps `StateChangeEvent.after()` to `IoTDeviceStateHistoryEntity`, serializes the `DeviceEntity` to JSONB via Jackson, persists via `EntityManager.persist()`. `@Transactional`.

- [ ] **Step 2: Write tests + implement JpaRuntimeSituationDefinitionProvider**

Implements `SituationDefinitionProvider`. At startup:
1. Load classpath definitions from all `META-INF/ras-iot-situations.yaml` and `META-INF/ras-iot-drools-situations.yaml` resources
2. Query `iot_situation_definition` table for the current tenant
3. Merge: database definitions with matching `situationId` override classpath ones
4. Return the merged list as `SituationRegistration` objects

Note: Read the `SituationDefinitionProvider` and `SituationRegistration` types from `casehub-ras/runtime/` before implementing. Understand how the provider maps to `SituationDefinitionRegistry`.

- [ ] **Step 3: Commit**

```bash
git add webapp/src/
git commit -m "feat: state change observer + runtime situation definition provider (#TBD)"
```

---

### Task 8: YamlCaseHub Subclasses + YAML Case Definitions + Config (webapp)

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/SafetyAlertCaseHub.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/SecurityAlertCaseHub.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/HvacAnomalyCaseHub.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/engine/GenericResponseCaseHub.java`
- Create: `webapp/src/main/resources/iot/safety-alert.yaml`
- Create: `webapp/src/main/resources/iot/security-alert.yaml`
- Create: `webapp/src/main/resources/iot/hvac-anomaly.yaml`
- Create: `webapp/src/main/resources/iot/generic-response.yaml`
- Create: `webapp/src/main/resources/application.properties`
- Create: `webapp/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `YamlCaseHub` (from `casehub-engine-api`), case descriptors from Task 4
- Produces: 4 `@ApplicationScoped` CaseHub beans, 4 YAML case definitions, app configuration

- [ ] **Step 1: Create YAML case definitions**

Each YAML file declares metadata (namespace, name, version) and capabilities with binding conditions. Follow the format used by AML's `aml-investigation.yaml`. Read that file first to understand the exact YAML schema.

```yaml
# iot/safety-alert.yaml
metadata:
  namespace: io.casehub.iot
  name: safety-alert
  version: "1.0"

capabilities:
  - name: device-command-dispatch
  - name: household-notification
    prerequisites:
      - device-command-dispatch
  - name: human-acknowledgement
    prerequisites:
      - household-notification
```

Create similar YAMLs for security-alert, hvac-anomaly, and generic-response.

- [ ] **Step 2: Implement YamlCaseHub subclasses**

Follow the pattern from the spec exactly (see §Case Definition Pattern):

```java
@ApplicationScoped
public class SafetyAlertCaseHub extends YamlCaseHub {
    @Inject SafetyAlertCaseDescriptor descriptor;

    public SafetyAlertCaseHub() { super("iot/safety-alert.yaml"); }

    @Override
    protected void augment(CaseDefinition definition) {
        descriptor.registerWorkers(definition);
    }
}
```

Same pattern for all 4.

- [ ] **Step 3: Create `application.properties`**

Configure three datasources, Flyway locations, provider activation, scheduler:

```properties
# Default datasource (bridge + webapp tables)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/iot
quarkus.datasource.username=iot
quarkus.datasource.password=iot
quarkus.hibernate-orm.packages=io.casehub.iot.bridge.persistence.jpa,io.casehub.iot.webapp.app.persistence
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=classpath:db/iot-bridge/migration,classpath:db/iot-webapp/migration

# iot-work datasource (work + ledger)
quarkus.datasource.iot-work.db-kind=postgresql
quarkus.datasource.iot-work.jdbc.url=jdbc:postgresql://localhost:5432/iot
quarkus.datasource.iot-work.username=iot
quarkus.datasource.iot-work.password=iot
quarkus.hibernate-orm.iot-work.packages=io.casehub.work.runtime.model
quarkus.flyway.iot-work.migrate-at-start=true
quarkus.flyway.iot-work.locations=classpath:db/work/migration,classpath:db/ledger/migration

# iot-ras datasource (RAS isolated)
quarkus.datasource.iot-ras.db-kind=postgresql
quarkus.datasource.iot-ras.jdbc.url=jdbc:postgresql://localhost:5432/iot
quarkus.datasource.iot-ras.username=iot
quarkus.datasource.iot-ras.password=iot
quarkus.flyway.iot-ras.migrate-at-start=true
quarkus.flyway.iot-ras.locations=classpath:db/ras/migration

# IoT providers (disabled by default — enable per deployment)
casehub.iot.homeassistant.enabled=false
casehub.iot.openhab.enabled=false

# State history retention
casehub.iot.webapp.state-history.retention-days=30
casehub.iot.webapp.state-history.purge-interval=PT1H

# Tenancy
casehub.iot.tenancy-id=default-tenant
```

- [ ] **Step 4: Create test `application.properties`**

H2 with `MODE=PostgreSQL` for all three datasources. Scheduler disabled. Providers disabled.

- [ ] **Step 5: Write `@QuarkusTest` verifying CaseHub beans load**

Test that all 4 CaseHub beans are resolvable and `getDefinition()` returns non-null with expected capabilities.

- [ ] **Step 6: Commit**

```bash
git add webapp/src/
git commit -m "feat: CaseHub subclasses, YAML definitions, 3-datasource config (#TBD)"
```

---

### Task 9: REST Resources + SSE (webapp)

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/DeviceResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/ProviderResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/BridgeResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SituationResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/CaseResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/WorkItemResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/HealthResource.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/DeviceSseResource.java`
- Create: test files for each resource

**Interfaces:**
- Consumes: `DeviceRegistry`, `Instance<DeviceProvider>`, `BridgeAuditStore`, `BridgeConnectionRegistry`, `EntityManager` (for situation definitions, state history, case command log), `CurrentPrincipal`, `CaseHubRuntime` (for case queries), `WorkItemService` (for WorkItem queries)
- Produces: 8 REST resource classes with `@RolesAllowed` annotations, SSE endpoint

- [ ] **Step 1: Implement DeviceResource**

```java
@Path("/api/devices")
@ApplicationScoped
public class DeviceResource {

    @Inject DeviceRegistry deviceRegistry;
    @Inject Instance<DeviceProvider> providers;
    @Inject CurrentPrincipal principal;
    @Inject EntityManager em;

    @GET
    @RolesAllowed("iot-viewer")
    public List<DeviceResponse> list(
            @QueryParam("deviceClass") String deviceClass,
            @QueryParam("providerId") String providerId,
            @QueryParam("available") Boolean available) {
        return deviceRegistry.findAll().stream()
                .filter(d -> filterByTenancy(d))
                .filter(d -> deviceClass == null || d.deviceClass().name().equals(deviceClass))
                .filter(d -> providerId == null || d.providerId().equals(providerId))
                .filter(d -> available == null || d.available() == available)
                .map(DeviceResponse::from)
                .toList();
    }

    @GET @Path("/{deviceId}")
    @RolesAllowed("iot-viewer")
    public DeviceResponse get(@PathParam("deviceId") String deviceId) { ... }

    @POST @Path("/{deviceId}/commands")
    @RolesAllowed("iot-operator")
    public CommandResponse dispatch(@PathParam("deviceId") String deviceId, CommandRequest request) { ... }

    @GET @Path("/{deviceId}/history")
    @RolesAllowed("iot-viewer")
    public List<StateHistoryResponse> history(
            @PathParam("deviceId") String deviceId,
            @QueryParam("from") Instant from,
            @QueryParam("to") Instant to,
            @QueryParam("limit") @DefaultValue("100") int limit) { ... }

    private boolean filterByTenancy(DeviceEntity d) {
        return d.tenancyId() == null || d.tenancyId().equals(principal.tenancyId());
    }
}
```

Follow this pattern for all 7 resources per the spec's REST API section. Each resource gets `@RolesAllowed` per the spec's Authentication and Authorization table.

- [ ] **Step 2: Implement DeviceSseResource**

```java
@Path("/api/devices/stream")
@ApplicationScoped
public class DeviceSseResource {

    @Inject Event<StateChangeEvent> stateChangeEvents;

    @GET
    @Produces(MediaType.SERVER_SENT_EVENTS)
    @RolesAllowed("iot-viewer")
    public Multi<SseEvent<String>> stream() {
        // Observe StateChangeEvent via CDI async, map to SSE events
        // First event: full snapshot via DeviceRegistry.findAll()
        // Subsequent events: append/replace per the pages SSE protocol
        ...
    }
}
```

Note: Read the pages SSE protocol from `docs/CASEHUB-PAGES.md` §Push Sources. The SSE endpoint sends `snapshot` on connect, then `replace` on each state change. Use Quarkus REST SSE support (`@Produces(MediaType.SERVER_SENT_EVENTS)` returning `Multi<SseEvent<String>>`).

- [ ] **Step 3: Implement remaining resources**

`ProviderResource`, `BridgeResource`, `SituationResource`, `CaseResource`, `WorkItemResource`, `HealthResource` — each follows the same pattern with appropriate `@RolesAllowed` and tenant filtering.

For `SituationResource`: CRUD on `iot_situation_definition` table via `EntityManager`. `GET` merges classpath + runtime definitions. `DELETE` removes the runtime override (restoring classpath default).

For `CaseResource` and `WorkItemResource`: read the casehub-engine and casehub-work APIs for querying case instances and WorkItems. Use `CaseInstanceRepository` and `WorkItemService` (verify exact types by reading the engine/work source).

- [ ] **Step 4: Write `@QuarkusTest` for each resource**

Use `@TestHTTPEndpoint` for REST tests. Mock `DeviceRegistry`, `DeviceProvider`, etc. via `@InjectMock` or test profiles. Test tenant isolation, role enforcement (when OIDC is absent annotations are inert — test the filtering logic directly).

- [ ] **Step 5: Commit**

```bash
git add webapp/src/
git commit -m "feat: REST resources + SSE — devices, providers, situations, cases, workitems, health (#TBD)"
```

---

### Task 10: TypeScript Pages via Quinoa (webapp)

**Files:**
- Create: `webapp/src/main/webapp/package.json`
- Create: `webapp/src/main/webapp/tsconfig.json`
- Create: `webapp/src/main/webapp/src/app.ts`
- Create: `webapp/src/main/webapp/src/components/device-kpi-row.ts`
- Create: `webapp/src/main/webapp/src/components/device-table.ts`
- Create: `webapp/src/main/webapp/src/pages/health.ts`
- Create: `webapp/src/main/webapp/src/pages/devices.ts`
- Create: `webapp/src/main/webapp/src/pages/situations.ts`
- Create: `webapp/src/main/webapp/src/pages/cases.ts`
- Create: `webapp/src/main/webapp/src/pages/workitems.ts`
- Create: `webapp/src/main/webapp/src/pages/audit.ts`
- Create: `webapp/src/main/webapp/src/pages/providers.ts`
- Create: `webapp/src/main/webapp/index.html`

**Interfaces:**
- Consumes: `@casehubio/pages-runtime`, `@casehubio/pages-ui` (aliased as `@casehubio/ui`), `@casehubio/pages-data` (aliased as `@casehubio/data`)
- Produces: 7 composable pages in a sidebar-navigated app shell

- [ ] **Step 1: Set up Quinoa project**

```json
{
  "name": "@casehubio/iot-webapp",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build"
  },
  "dependencies": {
    "@casehubio/pages-runtime": "workspace:*",
    "@casehubio/pages-ui": "workspace:*",
    "@casehubio/pages-data": "workspace:*",
    "@casehubio/pages-viz": "workspace:*"
  },
  "devDependencies": {
    "typescript": "^5.4",
    "vite": "^5.0"
  }
}
```

Note: Verify the exact dependency versions and workspace protocol by reading casehub-pages' existing consumers (e.g., claudony, drafthouse, life). The pages packages may be published to npm rather than using workspace protocol. Check `casehub-pages/package.json` for the publish setup.

- [ ] **Step 2: Create `index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>IoT Console</title>
</head>
<body>
    <div id="app" style="width: 100vw; height: 100vh;"></div>
    <script type="module" src="/src/app.ts"></script>
</body>
</html>
```

- [ ] **Step 3: Create reusable components**

`src/components/device-kpi-row.ts`:
```typescript
import { columns, metric } from "@casehubio/ui";
import { lookup, groupBy, count, distinct, filterBy } from "@casehubio/data";

export function deviceKpiRow(datasetId: string) {
    return columns([3, 3, 3, 3],
        [metric({ title: "Total Devices", lookup: lookup(datasetId, groupBy(null, count("deviceId"))), subtype: "card" })],
        [metric({ title: "Online", lookup: lookup(datasetId, filterBy("available", "EQUALS_TO", "true"), groupBy(null, count("deviceId"))), subtype: "card" })],
        [metric({ title: "Providers", lookup: lookup(datasetId, groupBy(null, distinct("providerId"))), subtype: "card" })],
        [metric({ title: "Active Alerts", lookup: lookup("situations-active", groupBy(null, count("situationId"))), subtype: "card" })],
    );
}
```

`src/components/device-table.ts`:
```typescript
import { table } from "@casehubio/ui";
import { lookup, sortBy } from "@casehubio/data";
import { withId } from "@casehubio/ui";

export function deviceTable(datasetId: string) {
    return withId("device-table", table({
        title: "Devices",
        sortable: true,
        pageSize: 20,
        csvExport: true,
        filter: { enabled: true, listening: true },
        lookup: lookup(datasetId, sortBy("lastUpdated", "DESCENDING")),
    }));
}
```

- [ ] **Step 4: Create all 7 page functions**

Each page function returns a `Component`. Follow the spec's Page Descriptions section for content:

`src/pages/health.ts` — metric cards + provider status table, 10s refresh
`src/pages/devices.ts` — KPI row + device table + device detail sub-page with action buttons
`src/pages/situations.ts` — tabs (Active | Definitions), active table + definitions form
`src/pages/cases.ts` — case table + case detail with event log timeline
`src/pages/workitems.ts` — WorkItem table + action buttons + countdown
`src/pages/audit.ts` — audit table with filters + expandable rows
`src/pages/providers.ts` — provider table + bridge connections

- [ ] **Step 5: Create app shell**

`src/app.ts`:
```typescript
import { page, sidebar, dataset } from "@casehubio/ui";
import { loadSite } from "@casehubio/pages-runtime";
import { healthPage } from "./pages/health";
import { devicesPage } from "./pages/devices";
import { situationsPage } from "./pages/situations";
import { casesPage } from "./pages/cases";
import { workItemsPage } from "./pages/workitems";
import { auditPage } from "./pages/audit";
import { providersPage } from "./pages/providers";

dataset("devices", "/api/devices");
dataset("device-events", "sse:///api/devices/stream");
dataset("providers", "/api/providers");
dataset("situations-active", "/api/situations/active");
dataset("situation-defs", "/api/situations/definitions");
dataset("cases", "/api/cases");
dataset("workitems", "/api/workitems");
dataset("health", "/api/health/overview");
dataset("audit", "/api/bridge/audit");

const app = page("IoT Console",
    sidebar(
        ["Health", healthPage()],
        ["Devices", devicesPage()],
        ["Situations", situationsPage()],
        ["Cases", casesPage()],
        ["Work Items", workItemsPage()],
        ["Audit", auditPage()],
        ["Providers", providersPage()],
    ),
    { settings: { mode: "dark" } },
);

loadSite(document.getElementById("app")!, app);
```

- [ ] **Step 6: Configure Quinoa in `application.properties`**

Add to `webapp/src/main/resources/application.properties`:
```properties
quarkus.quinoa.dev-server.port=3000
quarkus.quinoa.build-dir=dist
quarkus.quinoa.enable-spa-routing=true
```

- [ ] **Step 7: Verify pages build**

Run: `cd webapp/src/main/webapp && yarn install && yarn build`
Expected: TypeScript compiles, Vite produces `dist/` with bundled output.

Run: `mvn quarkus:dev -pl webapp`
Expected: Quarkus starts, pages accessible at `http://localhost:8080/`

- [ ] **Step 8: Commit**

```bash
git add webapp/src/main/webapp/
git commit -m "feat: TypeScript pages — 7-page IoT console with sidebar nav (#TBD)"
```

---

## Post-Implementation Checklist

After all tasks are complete:

- [ ] **Platform coherence review** — re-read `PLATFORM.md` and verify:
  - Module tier boundaries are correct (webapp-api is Tier 1, webapp-drools is Tier 2, webapp is Tier 3)
  - No domain logic leaked into foundation modules
  - Dependency directions are correct (application → foundation, never reverse)
  - All protocols referenced in Global Constraints were followed

- [ ] **Update CLAUDE.md** — add 3 new modules to the Module Structure table

- [ ] **Update PLATFORM.md** — update the `casehub-iot` entry in the Repository Map and Capability Ownership tables to reflect the webapp, RAS integration, and case orchestration

- [ ] **Create GitHub issue** — before implementation begins, create the tracking issue in `casehubio/iot` replacing `#TBD` references throughout

- [ ] **Create deferred issues** — for any design review findings marked DEFERRED:
  - R1-02: Application-tier domain logic placement (if the decision is to extract later)
  - iot#43: Per-provider refresh API (already noted in spec)
  - casehub-ras#23: Promote `SituationDefinitionProvider` to `casehub-ras-api`

- [ ] **Run `implementation-doc-sync`** — sync documentation after committing
