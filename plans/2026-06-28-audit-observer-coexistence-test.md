# Audit Observer Coexistence Test Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `@QuarkusTest` that fires `BridgeAuditEvent` via CDI and verifies both `StoringBridgeAuditObserver` and `LoggingBridgeAuditObserver` receive it.

**Architecture:** Single integration test class in bridge-server. Fires event via injected `Event<BridgeAuditEvent>.fireAsync()`, awaits completion, asserts store has exactly 1 matching event and JUL logger received exactly 1 record.

**Tech Stack:** Quarkus 3.x, CDI (ArC), JUnit 5, AssertJ

**Spec:** `docs/superpowers/specs/2026-06-28-audit-observer-coexistence-test-design.md`

## Global Constraints

- Handler captures `LogRecord` objects into `CopyOnWriteArrayList` — safe for cross-thread writes from CDI async executor; JBoss Log Manager returns unformatted `printf` template from `getMessage()` in `@QuarkusTest`
- Assert exact count (`hasSize(1)`) — catches duplicate observer registration, correlationId collision, logger category reuse
- Use `correlationId` filter to isolate from `@ApplicationScoped` store shared across test classes

---

### Task 1: Audit Observer Coexistence Integration Test

**Files:**
- Create: `bridge-server/src/test/java/io/casehub/iot/bridge/server/audit/AuditObserverCoexistenceTest.java`

**Interfaces:**
- Consumes: `BridgeAuditEvent` (api module), `BridgeAuditStore` (api module), `BridgeAuditQuery` (api module), `BridgeAuditEventType` (api module)
- Produces: nothing — leaf test

**Context for implementer:**

Existing classes to understand:
- `LoggingBridgeAuditObserver` at `bridge-server/src/main/java/io/casehub/iot/bridge/server/audit/LoggingBridgeAuditObserver.java` — uses `@ObservesAsync`, logs via `org.jboss.logging.Logger.infof()` to category `io.casehub.iot.bridge.audit`
- `StoringBridgeAuditObserver` at `bridge-server/src/main/java/io/casehub/iot/bridge/server/audit/StoringBridgeAuditObserver.java` — uses `@ObservesAsync`, delegates to injected `BridgeAuditStore.save()`
- `InMemoryBridgeAuditStore` at `bridge-server/src/main/java/io/casehub/iot/bridge/server/audit/InMemoryBridgeAuditStore.java` — `@DefaultBean @ApplicationScoped`, ring buffer, `query()` supports `correlationId` filter
- `BridgeAuditEvent` at `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEvent.java` — record with fields `tenancyId`, `receivedAt`, `eventType`, `correlationId`, `deviceId`, `message`
- Existing test pattern: `LoggingBridgeAuditObserverTest` at `bridge-server/src/test/java/io/casehub/iot/bridge/server/audit/LoggingBridgeAuditObserverTest.java` — plain JUnit (no CDI), uses `TestLogHandler` that captures `getMessage()` as strings. Our handler differs intentionally: `LogRecord` objects in `CopyOnWriteArrayList` (different shape AND threading model).

- [ ] **Step 1: Write the test**

```java
package io.casehub.iot.bridge.server.audit;

import io.casehub.iot.api.bridge.BridgeAuditEvent;
import io.casehub.iot.api.bridge.BridgeAuditEventType;
import io.casehub.iot.api.bridge.BridgeAuditQuery;
import io.casehub.iot.api.bridge.BridgeAuditStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.TimeUnit;
import java.util.logging.Handler;
import java.util.logging.LogRecord;
import java.util.logging.Logger;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class AuditObserverCoexistenceTest {

    @Inject Event<BridgeAuditEvent> auditEvents;
    @Inject BridgeAuditStore store;

    private TestLogHandler logHandler;

    @BeforeEach
    void installLogHandler() {
        logHandler = new TestLogHandler();
        Logger.getLogger("io.casehub.iot.bridge.audit").addHandler(logHandler);
    }

    @AfterEach
    void removeLogHandler() {
        Logger.getLogger("io.casehub.iot.bridge.audit").removeHandler(logHandler);
    }

    @Test
    void bothObserversReceiveAsyncEvent() throws Exception {
        var event = new BridgeAuditEvent(
            "coexistence-tenant", Instant.now(), BridgeAuditEventType.STATE_CHANGE,
            "coexistence-test", "coexistence-tenant/light-1", null);

        auditEvents.fireAsync(event).toCompletableFuture().get(5, TimeUnit.SECONDS);

        assertThat(store.query(BridgeAuditQuery.builder()
            .correlationId("coexistence-test").build()))
            .as("StoringBridgeAuditObserver should have persisted the event")
            .hasSize(1)
            .containsExactly(event);

        assertThat(logHandler.records)
            .as("LoggingBridgeAuditObserver should have logged the event")
            .hasSize(1);
    }

    private static class TestLogHandler extends Handler {
        final List<LogRecord> records = new CopyOnWriteArrayList<>();

        @Override
        public void publish(LogRecord record) {
            records.add(record);
        }

        @Override public void flush() {}
        @Override public void close() {}
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `mvn --batch-mode test -pl bridge-server -Dtest=AuditObserverCoexistenceTest -Dsurefire.useFile=false`

Expected: PASS — this is an integration test verifying existing production code, not new functionality. Both observers already exist and work; the test confirms CDI wires them correctly. If it fails, the failure reveals a real CDI wiring issue.

- [ ] **Step 3: Commit**

```bash
git add bridge-server/src/test/java/io/casehub/iot/bridge/server/audit/AuditObserverCoexistenceTest.java
git commit -m "test: integration test for audit observer coexistence — Closes #39"
```
