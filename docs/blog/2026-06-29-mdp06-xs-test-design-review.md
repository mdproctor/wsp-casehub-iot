---
layout: post
title: "CaseHub IoT — The XS Test That Needed a Design Review"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [cdi, testing, quarkus, design-review]
---

# CaseHub IoT — The XS Test That Needed a Design Review

**Date:** 2026-06-29
**Type:** phase-update

---

## What I was trying to achieve: verify CDI wires both audit observers

Issue #39 asked for one `@QuarkusTest` — fire a `BridgeAuditEvent` via CDI, assert both `StoringBridgeAuditObserver` and `LoggingBridgeAuditObserver` receive it. XS scale, Low complexity. The kind of task you'd expect to ship in twenty minutes.

## What we believed going in: "just assert not-empty on both side effects"

The initial design was straightforward: inject `Event<BridgeAuditEvent>`, call `fireAsync()`, check the store has the event, check the logger received a record. Assert presence, not format. The interesting question was whether to bother with log capture at all — injecting the `LoggingBridgeAuditObserver` bean proves CDI discovered it, so why verify it actually ran?

The answer turned out to matter: injecting a bean proves *discovery*, not *invocation*. If someone removes `@ObservesAsync` from the parameter, the bean is still injectable but the observer method never fires. We needed the side-effect check.

## JBoss Logging and the handler that almost didn't work

The existing unit test (`LoggingBridgeAuditObserverTest`) captures log output by adding a JUL `Handler` to `java.util.logging.Logger.getLogger("io.casehub.iot.bridge.audit")` and reading `getMessage()` as strings. Straightforward in plain JUnit.

In `@QuarkusTest`, JBoss Log Manager replaces the JUL LogManager. When the observer calls `LOG.infof("[AUDIT] type=%s tenancy=%s ...")`, JBoss Logging creates an `ExtLogRecord` with the format template as the message and the arguments as parameters. `getMessage()` returns `"[AUDIT] type=%s tenancy=%s device=%s correlation=%s"` — the unformatted template, not the rendered string.

The fix: capture `LogRecord` objects, not `getMessage()` strings. Assert the list is non-empty. Format verification belongs in the unit test — the integration test only needs to know the observer fired.

## `hasSize(1)` over `isNotEmpty`

An adversarial review caught this: `isNotEmpty()` on the log records passes even if `LoggingBridgeAuditObserver` is registered twice by a Jandex or Arc bug. One event, one observer, one record — `hasSize(1)` catches duplicate registration. Same reasoning applies to the store assertion. The review also caught that `CopyOnWriteArrayList` is the right collection for the handler — the observer runs on the CDI async executor thread, not the test thread.

The most load-bearing assumption turned out to be the one I hadn't documented: CDI 4.0 §10.3.2 guarantees that `fireAsync()` completes only after all `@ObservesAsync` handlers finish. That's what makes `.get()` a valid synchronization point. Without it, both assertions would be racy.
