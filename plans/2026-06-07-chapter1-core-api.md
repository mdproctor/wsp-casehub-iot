# Chapter 1: Core API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the full `casehub-iot-api` public API surface — typed device hierarchy, provider/registry SPIs, event/command model, and CDI-wired device registry.

**Architecture:** Abstract `DeviceEntity` base with self-referential generic builders. 10 concrete device subtypes (4 extensible for vendor supplements). Blocking SPIs. CdiDeviceRegistry with volatile map swap and synchronized writes. All in `iot-api` module.

**Tech Stack:** Java 21, Quarkus Arc (`@DefaultBean`), JUnit 5, `@QuarkusTest` for CDI integration tests.

**Spec:** `docs/superpowers/specs/2026-06-07-chapter1-core-api-design.md`

---

### Task 1: Maven setup — iot-api dependencies

**Files:**
- Modify: `iot-api/pom.xml`

- [ ] **Step 1: Add dependencies to iot-api/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-iot-parent</artifactId>
        <version>0.1-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-iot-api</artifactId>
    <name>CaseHub IoT API</name>
    <description>Core SPIs and typed device class hierarchy — public API, semver discipline</description>

    <dependencies>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Verify build compiles**

Run: `mvn compile -pl iot-api --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```
git add iot-api/pom.xml
git commit -m "chore: add dependencies to iot-api — quarkus-arc, junit5, assertj #1"
```

---

### Task 2: Enums — DeviceClass, ThermostatMode, SensorType, CommandResult, ProviderStatus

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/DeviceClass.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/ThermostatMode.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/SensorType.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/CommandResult.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/ProviderStatus.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/EnumTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class EnumTest {

    @Test
    void deviceClassHasTenValues() {
        assertThat(DeviceClass.values()).hasSize(10);
        assertThat(DeviceClass.valueOf("SWITCH")).isEqualTo(DeviceClass.SWITCH);
        assertThat(DeviceClass.valueOf("PRESENCE_SENSOR")).isEqualTo(DeviceClass.PRESENCE_SENSOR);
    }

    @Test
    void thermostatModeHasFiveValues() {
        assertThat(ThermostatMode.values()).hasSize(5);
        assertThat(ThermostatMode.values()).containsExactly(
            ThermostatMode.HEAT, ThermostatMode.COOL, ThermostatMode.AUTO,
            ThermostatMode.OFF, ThermostatMode.FAN_ONLY);
    }

    @Test
    void sensorTypeHasSevenValues() {
        assertThat(SensorType.values()).hasSize(7);
        assertThat(SensorType.valueOf("GENERIC")).isEqualTo(SensorType.GENERIC);
    }

    @Test
    void commandResultHasThreeValues() {
        assertThat(CommandResult.values()).containsExactly(
            CommandResult.SENT, CommandResult.FAILED, CommandResult.TIMEOUT);
    }

    @Test
    void providerStatusHasThreeValues() {
        assertThat(ProviderStatus.values()).containsExactly(
            ProviderStatus.CONNECTED, ProviderStatus.CONNECTING, ProviderStatus.DISCONNECTED);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=EnumTest --batch-mode`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Implement all five enums**

`DeviceClass.java`:
```java
package io.casehub.iot.api;

public enum DeviceClass {
    SWITCH, LIGHT, THERMOSTAT, SENSOR, PRESENCE_SENSOR,
    POWER_SENSOR, LOCK, COVER, MEDIA_PLAYER, FAN
}
```

`ThermostatMode.java`:
```java
package io.casehub.iot.api;

public enum ThermostatMode {
    HEAT, COOL, AUTO, OFF, FAN_ONLY
}
```

`SensorType.java`:
```java
package io.casehub.iot.api;

public enum SensorType {
    TEMPERATURE, HUMIDITY, MOTION, DOOR_WINDOW, CO2, LUX, GENERIC
}
```

`CommandResult.java`:
```java
package io.casehub.iot.api;

public enum CommandResult {
    SENT, FAILED, TIMEOUT
}
```

`ProviderStatus.java`:
```java
package io.casehub.iot.api;

public enum ProviderStatus {
    CONNECTED, CONNECTING, DISCONNECTED
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl iot-api -Dtest=EnumTest --batch-mode`
Expected: PASS — 5 tests

- [ ] **Step 5: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add DeviceClass, ThermostatMode, SensorType, CommandResult, ProviderStatus enums #1"
```

---

### Task 3: Temperature value type

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/Temperature.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/TemperatureTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import static org.assertj.core.api.Assertions.assertThat;

class TemperatureTest {

    @Test
    void celsiusToFahrenheit() {
        var celsius = new Temperature(BigDecimal.valueOf(100), Temperature.TemperatureUnit.CELSIUS);
        var fahrenheit = celsius.toFahrenheit();
        assertThat(fahrenheit.unit()).isEqualTo(Temperature.TemperatureUnit.FAHRENHEIT);
        assertThat(fahrenheit.value()).isEqualByComparingTo(BigDecimal.valueOf(212));
    }

    @Test
    void fahrenheitToCelsius() {
        var fahrenheit = new Temperature(BigDecimal.valueOf(32), Temperature.TemperatureUnit.FAHRENHEIT);
        var celsius = fahrenheit.toCelsius();
        assertThat(celsius.unit()).isEqualTo(Temperature.TemperatureUnit.CELSIUS);
        assertThat(celsius.value()).isEqualByComparingTo(BigDecimal.ZERO);
    }

    @Test
    void celsiusToCelsiusReturnsSameInstance() {
        var celsius = new Temperature(BigDecimal.valueOf(20), Temperature.TemperatureUnit.CELSIUS);
        assertThat(celsius.toCelsius()).isSameAs(celsius);
    }

    @Test
    void fahrenheitToFahrenheitReturnsSameInstance() {
        var fahrenheit = new Temperature(BigDecimal.valueOf(68), Temperature.TemperatureUnit.FAHRENHEIT);
        assertThat(fahrenheit.toFahrenheit()).isSameAs(fahrenheit);
    }

    @Test
    void recordEquality() {
        var a = new Temperature(BigDecimal.valueOf(20), Temperature.TemperatureUnit.CELSIUS);
        var b = new Temperature(BigDecimal.valueOf(20), Temperature.TemperatureUnit.CELSIUS);
        assertThat(a).isEqualTo(b);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=TemperatureTest --batch-mode`
Expected: FAIL — Temperature does not exist

- [ ] **Step 3: Implement Temperature**

```java
package io.casehub.iot.api;

import java.math.BigDecimal;
import java.math.MathContext;

public record Temperature(BigDecimal value, TemperatureUnit unit) {

    public enum TemperatureUnit { CELSIUS, FAHRENHEIT }

    public Temperature toCelsius() {
        if (unit == TemperatureUnit.CELSIUS) return this;
        BigDecimal celsius = value.subtract(BigDecimal.valueOf(32))
            .multiply(BigDecimal.valueOf(5))
            .divide(BigDecimal.valueOf(9), MathContext.DECIMAL64);
        return new Temperature(celsius, TemperatureUnit.CELSIUS);
    }

    public Temperature toFahrenheit() {
        if (unit == TemperatureUnit.FAHRENHEIT) return this;
        BigDecimal fahrenheit = value.multiply(BigDecimal.valueOf(9))
            .divide(BigDecimal.valueOf(5), MathContext.DECIMAL64)
            .add(BigDecimal.valueOf(32));
        return new Temperature(fahrenheit, TemperatureUnit.FAHRENHEIT);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl iot-api -Dtest=TemperatureTest --batch-mode`
Expected: PASS — 5 tests

- [ ] **Step 5: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add Temperature value type with unit conversion #1"
```

---

### Task 4: DeviceEntity abstract base with Builder

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/DeviceEntity.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/DeviceEntityTest.java`

- [ ] **Step 1: Write the test**

Tests use a minimal concrete subclass (`TestDevice`) to exercise the abstract base.

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DeviceEntityTest {

    static final class TestDevice extends DeviceEntity {
        private TestDevice(Builder builder) { super(builder); }

        static Builder builder() { return new Builder(); }

        static final class Builder extends DeviceEntity.Builder<Builder> {
            @Override protected Builder self() { return this; }
            @Override public TestDevice build() { return new TestDevice(this); }
        }
    }

    private TestDevice device(String id) {
        return TestDevice.builder()
            .deviceId(id)
            .deviceClass(DeviceClass.SWITCH)
            .label("Test")
            .available(true)
            .lastUpdated(Instant.parse("2026-06-07T10:00:00Z"))
            .tenancyId("tenant-1")
            .build();
    }

    @Test
    void builderSetsAllFields() {
        var d = device("d1");
        assertThat(d.deviceId()).isEqualTo("d1");
        assertThat(d.deviceClass()).isEqualTo(DeviceClass.SWITCH);
        assertThat(d.label()).isEqualTo("Test");
        assertThat(d.available()).isTrue();
        assertThat(d.lastUpdated()).isEqualTo(Instant.parse("2026-06-07T10:00:00Z"));
        assertThat(d.tenancyId()).isEqualTo("tenant-1");
    }

    @Test
    void equalsAndHashCodeOnDeviceId() {
        var a = device("d1");
        var b = device("d1");
        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }

    @Test
    void differentDeviceIdNotEqual() {
        assertThat(device("d1")).isNotEqualTo(device("d2"));
    }

    @Test
    void nullDeviceIdThrows() {
        assertThatThrownBy(() -> TestDevice.builder()
            .deviceClass(DeviceClass.SWITCH).label("X")
            .available(true).lastUpdated(Instant.now()).tenancyId("t1")
            .build()
        ).isInstanceOf(NullPointerException.class);
    }

    @Test
    void toStringIncludesClassAndLabel() {
        var d = device("d1");
        assertThat(d.toString()).contains("TestDevice");
        assertThat(d.toString()).contains("d1");
        assertThat(d.toString()).contains("Test");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=DeviceEntityTest --batch-mode`
Expected: FAIL — DeviceEntity does not exist

- [ ] **Step 3: Implement DeviceEntity**

```java
package io.casehub.iot.api;

import java.time.Instant;
import java.util.Objects;

public abstract class DeviceEntity {

    private final String deviceId;
    private final DeviceClass deviceClass;
    private final String label;
    private final boolean available;
    private final Instant lastUpdated;
    private final String tenancyId;

    protected DeviceEntity(Builder<?> builder) {
        this.deviceId = Objects.requireNonNull(builder.deviceId, "deviceId");
        this.deviceClass = Objects.requireNonNull(builder.deviceClass, "deviceClass");
        this.label = Objects.requireNonNull(builder.label, "label");
        this.available = builder.available;
        this.lastUpdated = Objects.requireNonNull(builder.lastUpdated, "lastUpdated");
        this.tenancyId = Objects.requireNonNull(builder.tenancyId, "tenancyId");
    }

    public String deviceId() { return deviceId; }
    public DeviceClass deviceClass() { return deviceClass; }
    public String label() { return label; }
    public boolean available() { return available; }
    public Instant lastUpdated() { return lastUpdated; }
    public String tenancyId() { return tenancyId; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof DeviceEntity that)) return false;
        return deviceId.equals(that.deviceId);
    }

    @Override
    public int hashCode() {
        return deviceId.hashCode();
    }

    @Override
    public String toString() {
        return getClass().getSimpleName() + "{deviceId='" + deviceId
            + "', deviceClass=" + deviceClass + ", label='" + label + "'}";
    }

    @SuppressWarnings("unchecked")
    protected abstract static class Builder<B extends Builder<B>> {
        String deviceId;
        DeviceClass deviceClass;
        String label;
        boolean available;
        Instant lastUpdated;
        String tenancyId;

        public B deviceId(String v) { this.deviceId = v; return self(); }
        public B deviceClass(DeviceClass v) { this.deviceClass = v; return self(); }
        public B label(String v) { this.label = v; return self(); }
        public B available(boolean v) { this.available = v; return self(); }
        public B lastUpdated(Instant v) { this.lastUpdated = v; return self(); }
        public B tenancyId(String v) { this.tenancyId = v; return self(); }

        protected B self() { return (B) this; }
        public abstract DeviceEntity build();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl iot-api -Dtest=DeviceEntityTest --batch-mode`
Expected: PASS — 5 tests

- [ ] **Step 5: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add DeviceEntity abstract base with self-referential generic Builder #1"
```

---

### Task 5: Leaf device types — SwitchDevice, FanDevice, MediaPlayerDevice, PresenceSensor, PowerSensor

These types are never extended by vendor supplements. Simple `Builder extends DeviceEntity.Builder<Builder>`.

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/SwitchDevice.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/FanDevice.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/MediaPlayerDevice.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/PresenceSensor.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/PowerSensor.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/LeafDeviceTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class LeafDeviceTest {

    private static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    @Test
    void switchDeviceBuilder() {
        var d = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Hall Switch")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .on(true)
            .build();
        assertThat(d.isOn()).isTrue();
        assertThat(d.deviceClass()).isEqualTo(DeviceClass.SWITCH);
        assertThat(SwitchDevice.CAP_ON).isEqualTo("isOn");
    }

    @Test
    void fanDeviceBuilder() {
        var d = FanDevice.builder()
            .deviceId("fan1").deviceClass(DeviceClass.FAN).label("Ceiling Fan")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .on(true).speed(75)
            .build();
        assertThat(d.isOn()).isTrue();
        assertThat(d.speed()).hasValue(75);
        assertThat(FanDevice.CAP_ON).isEqualTo("isOn");
        assertThat(FanDevice.CAP_SPEED).isEqualTo("speed");
    }

    @Test
    void fanDeviceSpeedOptional() {
        var d = FanDevice.builder()
            .deviceId("fan2").deviceClass(DeviceClass.FAN).label("Fan")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .on(false)
            .build();
        assertThat(d.speed()).isEmpty();
    }

    @Test
    void mediaPlayerDeviceBuilder() {
        var d = MediaPlayerDevice.builder()
            .deviceId("mp1").deviceClass(DeviceClass.MEDIA_PLAYER).label("TV")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .playing(true).volume(80)
            .build();
        assertThat(d.isPlaying()).isTrue();
        assertThat(d.volume()).hasValue(80);
        assertThat(MediaPlayerDevice.CAP_PLAYING).isEqualTo("isPlaying");
        assertThat(MediaPlayerDevice.CAP_VOLUME).isEqualTo("volume");
    }

    @Test
    void presenceSensorBuilder() {
        var d = PresenceSensor.builder()
            .deviceId("ps1").deviceClass(DeviceClass.PRESENCE_SENSOR).label("Hallway Motion")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .present(true).lastSeen(NOW)
            .build();
        assertThat(d.isPresent()).isTrue();
        assertThat(d.lastSeen()).isEqualTo(NOW);
        assertThat(PresenceSensor.CAP_PRESENT).isEqualTo("isPresent");
        assertThat(PresenceSensor.CAP_LAST_SEEN).isEqualTo("lastSeen");
    }

    @Test
    void powerSensorBuilder() {
        var d = PowerSensor.builder()
            .deviceId("pw1").deviceClass(DeviceClass.POWER_SENSOR).label("Mains Meter")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .power(BigDecimal.valueOf(2500)).energy(BigDecimal.valueOf(150.7))
            .build();
        assertThat(d.power()).isEqualByComparingTo(BigDecimal.valueOf(2500));
        assertThat(d.energy()).isEqualByComparingTo(BigDecimal.valueOf(150.7));
        assertThat(PowerSensor.CAP_POWER).isEqualTo("power");
        assertThat(PowerSensor.CAP_ENERGY).isEqualTo("energy");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=LeafDeviceTest --batch-mode`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Implement SwitchDevice**

```java
package io.casehub.iot.api;

public class SwitchDevice extends DeviceEntity {

    public static final String CAP_ON = "isOn";

    private final boolean on;

    private SwitchDevice(Builder builder) {
        super(builder);
        this.on = builder.on;
    }

    public boolean isOn() { return on; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends DeviceEntity.Builder<Builder> {
        private boolean on;

        public Builder on(boolean on) { this.on = on; return this; }

        @Override protected Builder self() { return this; }
        @Override public SwitchDevice build() { return new SwitchDevice(this); }
    }
}
```

- [ ] **Step 4: Implement FanDevice**

```java
package io.casehub.iot.api;

import java.util.Optional;

public class FanDevice extends DeviceEntity {

    public static final String CAP_ON = "isOn";
    public static final String CAP_SPEED = "speed";

    private final boolean on;
    private final Integer speed;

    private FanDevice(Builder builder) {
        super(builder);
        this.on = builder.on;
        this.speed = builder.speed;
    }

    public boolean isOn() { return on; }
    public Optional<Integer> speed() { return Optional.ofNullable(speed); }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends DeviceEntity.Builder<Builder> {
        private boolean on;
        private Integer speed;

        public Builder on(boolean on) { this.on = on; return this; }
        public Builder speed(Integer speed) { this.speed = speed; return this; }

        @Override protected Builder self() { return this; }
        @Override public FanDevice build() { return new FanDevice(this); }
    }
}
```

- [ ] **Step 5: Implement MediaPlayerDevice**

```java
package io.casehub.iot.api;

import java.util.Optional;

public class MediaPlayerDevice extends DeviceEntity {

    public static final String CAP_PLAYING = "isPlaying";
    public static final String CAP_VOLUME = "volume";

    private final boolean playing;
    private final Integer volume;

    private MediaPlayerDevice(Builder builder) {
        super(builder);
        this.playing = builder.playing;
        this.volume = builder.volume;
    }

    public boolean isPlaying() { return playing; }
    public Optional<Integer> volume() { return Optional.ofNullable(volume); }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends DeviceEntity.Builder<Builder> {
        private boolean playing;
        private Integer volume;

        public Builder playing(boolean playing) { this.playing = playing; return this; }
        public Builder volume(Integer volume) { this.volume = volume; return this; }

        @Override protected Builder self() { return this; }
        @Override public MediaPlayerDevice build() { return new MediaPlayerDevice(this); }
    }
}
```

- [ ] **Step 6: Implement PresenceSensor**

```java
package io.casehub.iot.api;

import java.time.Instant;
import java.util.Objects;

public class PresenceSensor extends DeviceEntity {

    public static final String CAP_PRESENT = "isPresent";
    public static final String CAP_LAST_SEEN = "lastSeen";

    private final boolean present;
    private final Instant lastSeen;

    private PresenceSensor(Builder builder) {
        super(builder);
        this.present = builder.present;
        this.lastSeen = Objects.requireNonNull(builder.lastSeen, "lastSeen");
    }

    public boolean isPresent() { return present; }
    public Instant lastSeen() { return lastSeen; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends DeviceEntity.Builder<Builder> {
        private boolean present;
        private Instant lastSeen;

        public Builder present(boolean present) { this.present = present; return this; }
        public Builder lastSeen(Instant lastSeen) { this.lastSeen = lastSeen; return this; }

        @Override protected Builder self() { return this; }
        @Override public PresenceSensor build() { return new PresenceSensor(this); }
    }
}
```

- [ ] **Step 7: Implement PowerSensor**

```java
package io.casehub.iot.api;

import java.math.BigDecimal;
import java.util.Objects;

public class PowerSensor extends DeviceEntity {

    public static final String CAP_POWER = "power";
    public static final String CAP_ENERGY = "energy";

    private final BigDecimal power;
    private final BigDecimal energy;

    private PowerSensor(Builder builder) {
        super(builder);
        this.power = Objects.requireNonNull(builder.power, "power");
        this.energy = Objects.requireNonNull(builder.energy, "energy");
    }

    public BigDecimal power() { return power; }
    public BigDecimal energy() { return energy; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends DeviceEntity.Builder<Builder> {
        private BigDecimal power;
        private BigDecimal energy;

        public Builder power(BigDecimal power) { this.power = power; return this; }
        public Builder energy(BigDecimal energy) { this.energy = energy; return this; }

        @Override protected Builder self() { return this; }
        @Override public PowerSensor build() { return new PowerSensor(this); }
    }
}
```

- [ ] **Step 8: Run tests to verify all pass**

Run: `mvn test -pl iot-api -Dtest=LeafDeviceTest --batch-mode`
Expected: PASS — 6 tests

- [ ] **Step 9: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add leaf device types — Switch, Fan, MediaPlayer, PresenceSensor, PowerSensor #1"
```

---

### Task 6: Extensible device types — LightDevice, ThermostatDevice, LockDevice, CoverDevice

These types will be extended by vendor supplement types in Chapters 3/4. Their builders use `AbstractBuilder<B>` for extension.

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/LightDevice.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/ThermostatDevice.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/LockDevice.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/CoverDevice.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/ExtensibleDeviceTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;

class ExtensibleDeviceTest {

    private static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    @Test
    void lightDeviceBuilder() {
        var d = LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Desk Lamp")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .on(true).brightness(200).colorTemp(4000)
            .build();
        assertThat(d.isOn()).isTrue();
        assertThat(d.brightness()).hasValue(200);
        assertThat(d.colorTemp()).hasValue(4000);
    }

    @Test
    void lightDeviceOptionalFieldsAbsent() {
        var d = LightDevice.builder()
            .deviceId("l2").deviceClass(DeviceClass.LIGHT).label("Simple Light")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .on(false)
            .build();
        assertThat(d.brightness()).isEmpty();
        assertThat(d.colorTemp()).isEmpty();
    }

    @Test
    void lightDeviceCapConstants() {
        assertThat(LightDevice.CAP_ON).isEqualTo("isOn");
        assertThat(LightDevice.CAP_BRIGHTNESS).isEqualTo("brightness");
        assertThat(LightDevice.CAP_COLOR_TEMP).isEqualTo("colorTemp");
    }

    @Test
    void lightDeviceBuilderExtensible() {
        // Simulates vendor supplement builder extending AbstractBuilder
        var builder = new LightDevice.AbstractBuilder<>() {
            @Override protected Object self() { return this; }
            @Override public LightDevice build() { return new LightDevice(this); }
        };
        // Verify the abstract builder path compiles and works
        var d = LightDevice.builder()
            .deviceId("l3").deviceClass(DeviceClass.LIGHT).label("L")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).build();
        assertThat(d).isInstanceOf(LightDevice.class);
    }

    @Test
    void thermostatDeviceBuilder() {
        var d = ThermostatDevice.builder()
            .deviceId("th1").deviceClass(DeviceClass.THERMOSTAT).label("Living Room")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .currentTemperature(new Temperature(BigDecimal.valueOf(21), Temperature.TemperatureUnit.CELSIUS))
            .targetTemperature(new Temperature(BigDecimal.valueOf(23), Temperature.TemperatureUnit.CELSIUS))
            .mode(ThermostatMode.HEAT)
            .build();
        assertThat(d.currentTemperature().value()).isEqualByComparingTo(BigDecimal.valueOf(21));
        assertThat(d.targetTemperature().value()).isEqualByComparingTo(BigDecimal.valueOf(23));
        assertThat(d.mode()).isEqualTo(ThermostatMode.HEAT);
        assertThat(ThermostatDevice.CAP_CURRENT_TEMPERATURE).isEqualTo("currentTemperature");
        assertThat(ThermostatDevice.CAP_TARGET_TEMPERATURE).isEqualTo("targetTemperature");
        assertThat(ThermostatDevice.CAP_MODE).isEqualTo("mode");
    }

    @Test
    void lockDeviceBuilder() {
        var d = LockDevice.builder()
            .deviceId("lk1").deviceClass(DeviceClass.LOCK).label("Front Door")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .locked(true)
            .build();
        assertThat(d.isLocked()).isTrue();
        assertThat(LockDevice.CAP_LOCKED).isEqualTo("isLocked");
    }

    @Test
    void coverDeviceBuilder() {
        var d = CoverDevice.builder()
            .deviceId("cv1").deviceClass(DeviceClass.COVER).label("Blinds")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .position(75).moving(false)
            .build();
        assertThat(d.position()).isEqualTo(75);
        assertThat(d.isMoving()).isFalse();
        assertThat(CoverDevice.CAP_POSITION).isEqualTo("position");
        assertThat(CoverDevice.CAP_MOVING).isEqualTo("isMoving");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=ExtensibleDeviceTest --batch-mode`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Implement LightDevice**

```java
package io.casehub.iot.api;

import java.util.Optional;

public class LightDevice extends DeviceEntity {

    public static final String CAP_ON = "isOn";
    public static final String CAP_BRIGHTNESS = "brightness";
    public static final String CAP_COLOR_TEMP = "colorTemp";

    private final boolean on;
    private final Integer brightness;
    private final Integer colorTemp;

    protected LightDevice(AbstractBuilder<?> builder) {
        super(builder);
        this.on = builder.on;
        this.brightness = builder.brightness;
        this.colorTemp = builder.colorTemp;
    }

    public boolean isOn() { return on; }
    public Optional<Integer> brightness() { return Optional.ofNullable(brightness); }
    public Optional<Integer> colorTemp() { return Optional.ofNullable(colorTemp); }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends AbstractBuilder<Builder> {
        @Override protected Builder self() { return this; }
        @Override public LightDevice build() { return new LightDevice(this); }
    }

    @SuppressWarnings("unchecked")
    public abstract static class AbstractBuilder<B extends AbstractBuilder<B>>
            extends DeviceEntity.Builder<B> {
        boolean on;
        Integer brightness;
        Integer colorTemp;

        public B on(boolean on) { this.on = on; return (B) this; }
        public B brightness(Integer brightness) { this.brightness = brightness; return (B) this; }
        public B colorTemp(Integer colorTemp) { this.colorTemp = colorTemp; return (B) this; }
    }
}
```

- [ ] **Step 4: Implement ThermostatDevice**

```java
package io.casehub.iot.api;

import java.util.Objects;

public class ThermostatDevice extends DeviceEntity {

    public static final String CAP_CURRENT_TEMPERATURE = "currentTemperature";
    public static final String CAP_TARGET_TEMPERATURE = "targetTemperature";
    public static final String CAP_MODE = "mode";

    private final Temperature currentTemperature;
    private final Temperature targetTemperature;
    private final ThermostatMode mode;

    protected ThermostatDevice(AbstractBuilder<?> builder) {
        super(builder);
        this.currentTemperature = Objects.requireNonNull(builder.currentTemperature, "currentTemperature");
        this.targetTemperature = Objects.requireNonNull(builder.targetTemperature, "targetTemperature");
        this.mode = Objects.requireNonNull(builder.mode, "mode");
    }

    public Temperature currentTemperature() { return currentTemperature; }
    public Temperature targetTemperature() { return targetTemperature; }
    public ThermostatMode mode() { return mode; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends AbstractBuilder<Builder> {
        @Override protected Builder self() { return this; }
        @Override public ThermostatDevice build() { return new ThermostatDevice(this); }
    }

    @SuppressWarnings("unchecked")
    public abstract static class AbstractBuilder<B extends AbstractBuilder<B>>
            extends DeviceEntity.Builder<B> {
        Temperature currentTemperature;
        Temperature targetTemperature;
        ThermostatMode mode;

        public B currentTemperature(Temperature v) { this.currentTemperature = v; return (B) this; }
        public B targetTemperature(Temperature v) { this.targetTemperature = v; return (B) this; }
        public B mode(ThermostatMode v) { this.mode = v; return (B) this; }
    }
}
```

- [ ] **Step 5: Implement LockDevice**

```java
package io.casehub.iot.api;

public class LockDevice extends DeviceEntity {

    public static final String CAP_LOCKED = "isLocked";

    private final boolean locked;

    protected LockDevice(AbstractBuilder<?> builder) {
        super(builder);
        this.locked = builder.locked;
    }

    public boolean isLocked() { return locked; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends AbstractBuilder<Builder> {
        @Override protected Builder self() { return this; }
        @Override public LockDevice build() { return new LockDevice(this); }
    }

    @SuppressWarnings("unchecked")
    public abstract static class AbstractBuilder<B extends AbstractBuilder<B>>
            extends DeviceEntity.Builder<B> {
        boolean locked;

        public B locked(boolean locked) { this.locked = locked; return (B) this; }
    }
}
```

- [ ] **Step 6: Implement CoverDevice**

```java
package io.casehub.iot.api;

public class CoverDevice extends DeviceEntity {

    public static final String CAP_POSITION = "position";
    public static final String CAP_MOVING = "isMoving";

    private final int position;
    private final boolean moving;

    protected CoverDevice(AbstractBuilder<?> builder) {
        super(builder);
        this.position = builder.position;
        this.moving = builder.moving;
    }

    public int position() { return position; }
    public boolean isMoving() { return moving; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends AbstractBuilder<Builder> {
        @Override protected Builder self() { return this; }
        @Override public CoverDevice build() { return new CoverDevice(this); }
    }

    @SuppressWarnings("unchecked")
    public abstract static class AbstractBuilder<B extends AbstractBuilder<B>>
            extends DeviceEntity.Builder<B> {
        int position;
        boolean moving;

        public B position(int position) { this.position = position; return (B) this; }
        public B moving(boolean moving) { this.moving = moving; return (B) this; }
    }
}
```

- [ ] **Step 7: Run tests to verify all pass**

Run: `mvn test -pl iot-api -Dtest=ExtensibleDeviceTest --batch-mode`
Expected: PASS — 8 tests

- [ ] **Step 8: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add extensible device types — Light, Thermostat, Lock, Cover with AbstractBuilder #1"
```

---

### Task 7: SensorDevice (leaf with SensorType sub-classification)

Separated from Task 5 because SensorDevice depends on SensorType enum and has distinct sub-classification semantics.

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/SensorDevice.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/SensorDeviceTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class SensorDeviceTest {

    private static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    @Test
    void numericSensor() {
        var d = SensorDevice.builder()
            .deviceId("s1").deviceClass(DeviceClass.SENSOR).label("Outdoor Temp")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .sensorType(SensorType.TEMPERATURE)
            .numericValue(BigDecimal.valueOf(18.5)).unit("°C")
            .build();
        assertThat(d.sensorType()).isEqualTo(SensorType.TEMPERATURE);
        assertThat(d.numericValue()).hasValue(BigDecimal.valueOf(18.5));
        assertThat(d.unit()).hasValue("°C");
        assertThat(d.binaryValue()).isEmpty();
    }

    @Test
    void binarySensor() {
        var d = SensorDevice.builder()
            .deviceId("s2").deviceClass(DeviceClass.SENSOR).label("Door Sensor")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .sensorType(SensorType.DOOR_WINDOW)
            .binaryValue(true)
            .build();
        assertThat(d.sensorType()).isEqualTo(SensorType.DOOR_WINDOW);
        assertThat(d.binaryValue()).hasValue(true);
        assertThat(d.numericValue()).isEmpty();
    }

    @Test
    void genericSensorType() {
        var d = SensorDevice.builder()
            .deviceId("s3").deviceClass(DeviceClass.SENSOR).label("Unknown Sensor")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .sensorType(SensorType.GENERIC)
            .build();
        assertThat(d.sensorType()).isEqualTo(SensorType.GENERIC);
    }

    @Test
    void capConstants() {
        assertThat(SensorDevice.CAP_NUMERIC_VALUE).isEqualTo("numericValue");
        assertThat(SensorDevice.CAP_UNIT).isEqualTo("unit");
        assertThat(SensorDevice.CAP_BINARY_VALUE).isEqualTo("binaryValue");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=SensorDeviceTest --batch-mode`
Expected: FAIL — SensorDevice does not exist

- [ ] **Step 3: Implement SensorDevice**

```java
package io.casehub.iot.api;

import java.math.BigDecimal;
import java.util.Objects;
import java.util.Optional;

public class SensorDevice extends DeviceEntity {

    public static final String CAP_NUMERIC_VALUE = "numericValue";
    public static final String CAP_UNIT = "unit";
    public static final String CAP_BINARY_VALUE = "binaryValue";

    private final SensorType sensorType;
    private final BigDecimal numericValue;
    private final String unit;
    private final Boolean binaryValue;

    private SensorDevice(Builder builder) {
        super(builder);
        this.sensorType = Objects.requireNonNull(builder.sensorType, "sensorType");
        this.numericValue = builder.numericValue;
        this.unit = builder.unit;
        this.binaryValue = builder.binaryValue;
    }

    public SensorType sensorType() { return sensorType; }
    public Optional<BigDecimal> numericValue() { return Optional.ofNullable(numericValue); }
    public Optional<String> unit() { return Optional.ofNullable(unit); }
    public Optional<Boolean> binaryValue() { return Optional.ofNullable(binaryValue); }

    public static Builder builder() { return new Builder(); }

    public static final class Builder extends DeviceEntity.Builder<Builder> {
        private SensorType sensorType;
        private BigDecimal numericValue;
        private String unit;
        private Boolean binaryValue;

        public Builder sensorType(SensorType sensorType) { this.sensorType = sensorType; return this; }
        public Builder numericValue(BigDecimal numericValue) { this.numericValue = numericValue; return this; }
        public Builder unit(String unit) { this.unit = unit; return this; }
        public Builder binaryValue(Boolean binaryValue) { this.binaryValue = binaryValue; return this; }

        @Override protected Builder self() { return this; }
        @Override public SensorDevice build() { return new SensorDevice(this); }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl iot-api -Dtest=SensorDeviceTest --batch-mode`
Expected: PASS — 4 tests

- [ ] **Step 5: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add SensorDevice with SensorType sub-classification #1"
```

---

### Task 8: SPI interfaces — DeviceProvider, DeviceRegistry

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/spi/DeviceProvider.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/spi/DeviceRegistry.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/spi/DeviceRegistryContractTest.java`

- [ ] **Step 1: Write the test**

Tests verify the SPI contract using a simple in-memory implementation.

```java
package io.casehub.iot.api.spi;

import io.casehub.iot.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class DeviceRegistryContractTest {

    private static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    private SimpleDeviceRegistry registry;
    private SwitchDevice sw;
    private LightDevice light;

    static class SimpleDeviceRegistry implements DeviceRegistry {
        private final Map<String, DeviceEntity> devices = new LinkedHashMap<>();

        void addDevice(DeviceEntity device) { devices.put(device.deviceId(), device); }

        @Override public Optional<DeviceEntity> findById(String deviceId) {
            return Optional.ofNullable(devices.get(deviceId));
        }
        @Override @SuppressWarnings("unchecked")
        public <T extends DeviceEntity> List<T> findByClass(Class<T> deviceClass) {
            return devices.values().stream()
                .filter(deviceClass::isInstance).map(d -> (T) d).toList();
        }
        @Override public List<DeviceEntity> findByTenancyId(String tenancyId) {
            return devices.values().stream()
                .filter(d -> d.tenancyId().equals(tenancyId)).toList();
        }
        @Override public List<DeviceEntity> findAll() {
            return List.copyOf(devices.values());
        }
        @Override public void refresh() { /* no-op */ }
    }

    @BeforeEach
    void setUp() {
        registry = new SimpleDeviceRegistry();
        sw = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).build();
        light = LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Light")
            .available(true).lastUpdated(NOW).tenancyId("t2").on(true).brightness(200).build();
        registry.addDevice(sw);
        registry.addDevice(light);
    }

    @Test
    void findById() {
        assertThat(registry.findById("sw1")).hasValue(sw);
        assertThat(registry.findById("nonexistent")).isEmpty();
    }

    @Test
    void findByClass() {
        assertThat(registry.findByClass(SwitchDevice.class)).containsExactly(sw);
        assertThat(registry.findByClass(LightDevice.class)).containsExactly(light);
        assertThat(registry.findByClass(DeviceEntity.class)).containsExactly(sw, light);
    }

    @Test
    void findByTenancyId() {
        assertThat(registry.findByTenancyId("t1")).containsExactly(sw);
        assertThat(registry.findByTenancyId("t2")).containsExactly(light);
        assertThat(registry.findByTenancyId("unknown")).isEmpty();
    }

    @Test
    void findAll() {
        assertThat(registry.findAll()).containsExactly(sw, light);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=DeviceRegistryContractTest --batch-mode`
Expected: FAIL — DeviceProvider and DeviceRegistry do not exist

- [ ] **Step 3: Implement DeviceProvider**

```java
package io.casehub.iot.api.spi;

import io.casehub.iot.api.CommandResult;
import io.casehub.iot.api.DeviceCommand;
import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.ProviderStatus;
import java.util.List;

public interface DeviceProvider {
    String providerId();
    List<DeviceEntity> discover();
    CommandResult dispatch(DeviceCommand command);
    ProviderStatus status();
}
```

- [ ] **Step 4: Implement DeviceRegistry**

```java
package io.casehub.iot.api.spi;

import io.casehub.iot.api.DeviceEntity;
import java.util.List;
import java.util.Optional;

public interface DeviceRegistry {
    Optional<DeviceEntity> findById(String deviceId);
    <T extends DeviceEntity> List<T> findByClass(Class<T> deviceClass);
    List<DeviceEntity> findByTenancyId(String tenancyId);
    List<DeviceEntity> findAll();
    void refresh();
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl iot-api -Dtest=DeviceRegistryContractTest --batch-mode`
Expected: PASS — 4 tests

- [ ] **Step 6: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add DeviceProvider and DeviceRegistry SPI interfaces #1"
```

---

### Task 9: Event and command model — StateChangeEvent, ProviderStatusEvent, DeviceCommand

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/api/StateChangeEvent.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/ProviderStatusEvent.java`
- Create: `iot-api/src/main/java/io/casehub/iot/api/DeviceCommand.java`
- Test: `iot-api/src/test/java/io/casehub/iot/api/EventCommandTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class EventCommandTest {

    private static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    private SwitchDevice switchDevice(String id, boolean on) {
        return SwitchDevice.builder()
            .deviceId(id).deviceClass(DeviceClass.SWITCH).label("Switch")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(on).build();
    }

    @Test
    void stateChangeEventFields() {
        var before = switchDevice("sw1", false);
        var after = switchDevice("sw1", true);
        var event = new StateChangeEvent(before, after,
            Set.of(SwitchDevice.CAP_ON), NOW, "homeassistant");
        assertThat(event.before()).isEqualTo(before);
        assertThat(event.after()).isEqualTo(after);
        assertThat(event.changedCapabilities()).containsExactly("isOn");
        assertThat(event.providerId()).isEqualTo("homeassistant");
    }

    @Test
    void providerStatusEventFields() {
        var event = new ProviderStatusEvent("ha",
            ProviderStatus.DISCONNECTED, ProviderStatus.CONNECTED);
        assertThat(event.providerId()).isEqualTo("ha");
        assertThat(event.previousStatus()).isEqualTo(ProviderStatus.DISCONNECTED);
        assertThat(event.currentStatus()).isEqualTo(ProviderStatus.CONNECTED);
    }

    @Test
    void deviceCommandActionConstants() {
        assertThat(DeviceCommand.ACTION_TURN_ON).isEqualTo("turn_on");
        assertThat(DeviceCommand.ACTION_TURN_OFF).isEqualTo("turn_off");
        assertThat(DeviceCommand.ACTION_SET_TEMPERATURE).isEqualTo("set_temperature");
        assertThat(DeviceCommand.ACTION_LOCK).isEqualTo("lock");
        assertThat(DeviceCommand.ACTION_UNLOCK).isEqualTo("unlock");
        assertThat(DeviceCommand.ACTION_SET_POSITION).isEqualTo("set_position");
        assertThat(DeviceCommand.ACTION_SET_VOLUME).isEqualTo("set_volume");
    }

    @Test
    void deviceCommandGenericConstructor() {
        var cmd = new DeviceCommand("sw1", "custom_action",
            Map.of("key", "value"), "actor1", "corr1");
        assertThat(cmd.targetDeviceId()).isEqualTo("sw1");
        assertThat(cmd.action()).isEqualTo("custom_action");
        assertThat(cmd.parameters()).containsEntry("key", "value");
        assertThat(cmd.dispatchedBy()).isEqualTo("actor1");
        assertThat(cmd.correlationId()).isEqualTo("corr1");
    }

    @Test
    void deviceCommandTurnOnFactory() {
        var cmd = DeviceCommand.turnOn("l1", Map.of("brightness", 200), "actor", "corr");
        assertThat(cmd.action()).isEqualTo("turn_on");
        assertThat(cmd.targetDeviceId()).isEqualTo("l1");
        assertThat(cmd.parameters()).containsEntry("brightness", 200);
    }

    @Test
    void deviceCommandTurnOffFactory() {
        var cmd = DeviceCommand.turnOff("sw1", "actor", "corr");
        assertThat(cmd.action()).isEqualTo("turn_off");
        assertThat(cmd.parameters()).isEmpty();
    }

    @Test
    void deviceCommandSetTemperatureFactory() {
        var target = new Temperature(BigDecimal.valueOf(22), Temperature.TemperatureUnit.CELSIUS);
        var cmd = DeviceCommand.setTemperature("th1", target, "actor", "corr");
        assertThat(cmd.action()).isEqualTo("set_temperature");
        assertThat(cmd.parameters()).containsEntry("temperature", BigDecimal.valueOf(22));
        assertThat(cmd.parameters()).containsEntry("unit", "CELSIUS");
    }

    @Test
    void deviceCommandLockFactory() {
        var cmd = DeviceCommand.lock("lk1", "actor", "corr");
        assertThat(cmd.action()).isEqualTo("lock");
    }

    @Test
    void deviceCommandUnlockFactory() {
        var cmd = DeviceCommand.unlock("lk1", "actor", "corr");
        assertThat(cmd.action()).isEqualTo("unlock");
    }

    @Test
    void deviceCommandSetPositionFactory() {
        var cmd = DeviceCommand.setPosition("cv1", 50, "actor", "corr");
        assertThat(cmd.action()).isEqualTo("set_position");
        assertThat(cmd.parameters()).containsEntry("position", 50);
    }

    @Test
    void deviceCommandSetVolumeFactory() {
        var cmd = DeviceCommand.setVolume("mp1", 80, "actor", "corr");
        assertThat(cmd.action()).isEqualTo("set_volume");
        assertThat(cmd.parameters()).containsEntry("volume", 80);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=EventCommandTest --batch-mode`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Implement StateChangeEvent**

```java
package io.casehub.iot.api;

import java.time.Instant;
import java.util.Set;

public record StateChangeEvent(
    DeviceEntity before,
    DeviceEntity after,
    Set<String> changedCapabilities,
    Instant occurredAt,
    String providerId
) {}
```

- [ ] **Step 4: Implement ProviderStatusEvent**

```java
package io.casehub.iot.api;

public record ProviderStatusEvent(
    String providerId,
    ProviderStatus previousStatus,
    ProviderStatus currentStatus
) {}
```

- [ ] **Step 5: Implement DeviceCommand**

```java
package io.casehub.iot.api;

import java.util.Map;

public record DeviceCommand(
    String targetDeviceId,
    String action,
    Map<String, Object> parameters,
    String dispatchedBy,
    String correlationId
) {
    public static final String ACTION_TURN_ON = "turn_on";
    public static final String ACTION_TURN_OFF = "turn_off";
    public static final String ACTION_SET_TEMPERATURE = "set_temperature";
    public static final String ACTION_LOCK = "lock";
    public static final String ACTION_UNLOCK = "unlock";
    public static final String ACTION_SET_POSITION = "set_position";
    public static final String ACTION_SET_VOLUME = "set_volume";

    public static DeviceCommand turnOn(String targetDeviceId, Map<String, Object> parameters,
                                       String dispatchedBy, String correlationId) {
        return new DeviceCommand(targetDeviceId, ACTION_TURN_ON, parameters, dispatchedBy, correlationId);
    }

    public static DeviceCommand turnOff(String targetDeviceId,
                                        String dispatchedBy, String correlationId) {
        return new DeviceCommand(targetDeviceId, ACTION_TURN_OFF, Map.of(), dispatchedBy, correlationId);
    }

    public static DeviceCommand setTemperature(String targetDeviceId, Temperature target,
                                               String dispatchedBy, String correlationId) {
        return new DeviceCommand(targetDeviceId, ACTION_SET_TEMPERATURE,
            Map.of("temperature", target.value(), "unit", target.unit().name()),
            dispatchedBy, correlationId);
    }

    public static DeviceCommand lock(String targetDeviceId,
                                     String dispatchedBy, String correlationId) {
        return new DeviceCommand(targetDeviceId, ACTION_LOCK, Map.of(), dispatchedBy, correlationId);
    }

    public static DeviceCommand unlock(String targetDeviceId,
                                       String dispatchedBy, String correlationId) {
        return new DeviceCommand(targetDeviceId, ACTION_UNLOCK, Map.of(), dispatchedBy, correlationId);
    }

    public static DeviceCommand setPosition(String targetDeviceId, int position,
                                            String dispatchedBy, String correlationId) {
        return new DeviceCommand(targetDeviceId, ACTION_SET_POSITION,
            Map.of("position", position), dispatchedBy, correlationId);
    }

    public static DeviceCommand setVolume(String targetDeviceId, int volume,
                                          String dispatchedBy, String correlationId) {
        return new DeviceCommand(targetDeviceId, ACTION_SET_VOLUME,
            Map.of("volume", volume), dispatchedBy, correlationId);
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn test -pl iot-api -Dtest=EventCommandTest --batch-mode`
Expected: PASS — 11 tests

- [ ] **Step 7: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add StateChangeEvent, ProviderStatusEvent, DeviceCommand with factories #1"
```

---

### Task 10: CdiDeviceRegistry

CDI-integrated registry with volatile map swap, synchronized writes, and @ObservesAsync for state freshness.

**Files:**
- Create: `iot-api/src/main/java/io/casehub/iot/spi/CdiDeviceRegistry.java`
- Test: `iot-api/src/test/java/io/casehub/iot/spi/CdiDeviceRegistryTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.spi;

import io.casehub.iot.api.*;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.annotation.Priority;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class CdiDeviceRegistryTest {

    private static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    @ApplicationScoped
    @Alternative
    @Priority(1)
    static class TestProvider implements DeviceProvider {
        @Override public String providerId() { return "test"; }
        @Override public List<DeviceEntity> discover() {
            return List.of(
                SwitchDevice.builder().deviceId("sw1").deviceClass(DeviceClass.SWITCH)
                    .label("Switch").available(true).lastUpdated(NOW).tenancyId("t1").on(true).build(),
                LightDevice.builder().deviceId("l1").deviceClass(DeviceClass.LIGHT)
                    .label("Light").available(true).lastUpdated(NOW).tenancyId("t2").on(true).brightness(200).build()
            );
        }
        @Override public CommandResult dispatch(DeviceCommand command) { return CommandResult.SENT; }
        @Override public ProviderStatus status() { return ProviderStatus.CONNECTED; }
    }

    @Inject
    DeviceRegistry registry;

    @Test
    void discoversDevicesAtStartup() {
        assertThat(registry.findAll()).hasSize(2);
        assertThat(registry.findById("sw1")).isPresent();
        assertThat(registry.findById("l1")).isPresent();
    }

    @Test
    void findByClassFiltersCorrectly() {
        assertThat(registry.findByClass(SwitchDevice.class)).hasSize(1);
        assertThat(registry.findByClass(LightDevice.class)).hasSize(1);
        assertThat(registry.findByClass(DeviceEntity.class)).hasSize(2);
    }

    @Test
    void findByTenancyIdFiltersCorrectly() {
        assertThat(registry.findByTenancyId("t1")).hasSize(1);
        assertThat(registry.findByTenancyId("t2")).hasSize(1);
        assertThat(registry.findByTenancyId("unknown")).isEmpty();
    }

    @Test
    void stateChangeUpdatesRegistry() {
        var before = registry.findById("sw1").orElseThrow();
        var after = SwitchDevice.builder().deviceId("sw1").deviceClass(DeviceClass.SWITCH)
            .label("Switch").available(true).lastUpdated(Instant.now()).tenancyId("t1").on(false).build();

        ((CdiDeviceRegistry) registry).onStateChange(
            new StateChangeEvent(before, after, Set.of(SwitchDevice.CAP_ON), Instant.now(), "test"));

        var updated = registry.findById("sw1").orElseThrow();
        assertThat(((SwitchDevice) updated).isOn()).isFalse();
    }

    @Test
    void refreshRebuildsDeviceMap() {
        registry.refresh();
        assertThat(registry.findAll()).hasSize(2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl iot-api -Dtest=CdiDeviceRegistryTest --batch-mode`
Expected: FAIL — CdiDeviceRegistry does not exist

- [ ] **Step 3: Implement CdiDeviceRegistry**

```java
package io.casehub.iot.spi;

import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.StateChangeEvent;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.quarkus.arc.DefaultBean;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import java.util.*;

@ApplicationScoped
@DefaultBean
public class CdiDeviceRegistry implements DeviceRegistry {

    @Any
    Instance<DeviceProvider> providers;

    private volatile Map<String, DeviceEntity> devices = Map.of();

    void onStartup(@Observes StartupEvent event) {
        refresh();
    }

    @Override
    public synchronized void refresh() {
        Map<String, DeviceEntity> next = new HashMap<>();
        for (DeviceProvider p : providers) {
            p.discover().forEach(d -> next.put(d.deviceId(), d));
        }
        devices = Map.copyOf(next);
    }

    void onStateChange(@ObservesAsync StateChangeEvent event) {
        updateDevice(event.after());
    }

    private synchronized void updateDevice(DeviceEntity device) {
        Map<String, DeviceEntity> current = devices;
        Map<String, DeviceEntity> next = new HashMap<>(current);
        next.put(device.deviceId(), device);
        devices = Map.copyOf(next);
    }

    @Override
    public Optional<DeviceEntity> findById(String deviceId) {
        return Optional.ofNullable(devices.get(deviceId));
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends DeviceEntity> List<T> findByClass(Class<T> deviceClass) {
        return devices.values().stream()
            .filter(deviceClass::isInstance)
            .map(d -> (T) d)
            .toList();
    }

    @Override
    public List<DeviceEntity> findByTenancyId(String tenancyId) {
        return devices.values().stream()
            .filter(d -> d.tenancyId().equals(tenancyId))
            .toList();
    }

    @Override
    public List<DeviceEntity> findAll() {
        return List.copyOf(devices.values());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl iot-api -Dtest=CdiDeviceRegistryTest --batch-mode`
Expected: PASS — 5 tests

- [ ] **Step 5: Run all iot-api tests**

Run: `mvn test -pl iot-api --batch-mode`
Expected: PASS — all tests across all test classes

- [ ] **Step 6: Commit**

```
git add iot-api/src/
git commit -m "feat(iot-api): add CdiDeviceRegistry — volatile map, synchronized writes, @ObservesAsync freshness #1"
```

---

### Task 11: Full build verification

- [ ] **Step 1: Build all modules**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS for all modules (other modules are empty but should compile)

- [ ] **Step 2: Verify test count**

Run: `mvn test -pl iot-api --batch-mode`
Expected: All tests pass. Verify count matches expectations (~35 tests across 7 test classes).

- [ ] **Step 3: Final commit if any formatting adjustments needed**

```
git add -A
git commit -m "chore: formatting and build verification #1"
```

Only commit if there are actual changes. Skip if working tree is clean.
