---
layout: post
title: "The Store SPI That Almost Wasn't"
date: 2026-06-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [arc42, store-spi, audit, design-review]
series: issue-36-arc42-and-audit-store
---

*Part of a series on [#36 — ARC42STORIES.MD update](https://github.com/casehubio/iot/issues/36) and [#35 — BridgeAuditStore SPI](https://github.com/casehubio/iot/issues/35).*

## Documenting what you already built

The ARC42STORIES.MD update for Chapter 6 was supposed to be the quick task — take what #32, #33, #34 delivered and write it into the architecture doc. Eight sections to update, two new mermaid diagrams, a Mode 3 rewrite. Straightforward documentation.

It wasn't. The document review surfaced errors that had been hiding since C5: BridgeMessage has 7 sealed variants, not 6 — ReplayedStateChange was added after the original spec was written but the ARC42 never caught up. The §9.1 flowchart styled C3 and C4 as gray (incomplete) despite both being green (done). The self-assessment claimed three ADRs when four exist in §10. Small errors, individually harmless, collectively corrosive. They undermine trust in the document.

The Chapter 6 entry itself landed cleanly once the corrections were in. The four new crosscutting concepts — provider activation, tenancyId consolidation, programmatic REST clients, and the dual-trail audit pattern — are genuine architectural concerns, not padding. Each affects multiple modules and each was missing from the architecture record.

## Where the Store SPI gets interesting

Issue #35 was explicitly deferred from #34: "build when a consuming app has a concrete query need." I expected a mechanical implementation — follow the Store SPI pattern from the module-tier-structure protocol, add the classes, write the tests.

The first design question changed my expectations. The issue specified four individual query methods: `findByTenancyId`, `findByTimeRange`, `findByEventType`, `findByDeviceId`. But audit queries compose — "show me COMMAND_SENT events for device X in tenant Y during the last hour" hits all four criteria. Individual methods create a combinatorial problem that gets worse with every field added. A `BridgeAuditQuery` record with nullable criteria and a builder gives composable filtering through a single `query()` method. Adding `correlationId` later (which the review correctly demanded — it links command-sent to command-response, the primary audit investigation use case) was one field and one filter line, not a new method signature.

The more substantive design question was module placement. The protocol says `persistence-memory/` is mandatory for every Store SPI. But casehub-iot doesn't own the JPA implementation — consuming apps provide that. The in-memory store lives in bridge-server because bridge-server IS the deployment target. Creating a one-class `bridge-persistence-memory/` module would satisfy the letter of the protocol while violating its intent. The protocol exists because `persistence-memory/` enables test isolation AND production ephemeral installs. When bridge-server already serves both purposes (the Pi deployment scenario is ephemeral, the test scenario uses the package-private constructor), a separate module adds Maven overhead without architectural benefit.

I documented this as a protocol deviation with the full reasoning — not just a checkmark.

## What the review caught

Claude's code review came back clean with two improvements worth making. The ordering contract — "results ordered by receivedAt descending" — was documented in the SPI Javadoc but depended on an assumption the implementation didn't state: that events are saved in chronological order. The in-memory store uses `addFirst` to maintain newest-at-head ordering, which works when the sole caller is a real-time CDI event observer. But a future JPA implementation would use `ORDER BY received_at DESC` regardless of insertion order. Without documenting the assumption, the in-memory implementation looks like a reference that a JPA implementation should mirror — when actually the JPA approach is more correct.

The contradictory time range test was the other catch. The spec explicitly said "from > to produces empty results" as a design decision, but the test suite didn't verify it. Five lines of test code that codifies a contract boundary.

Both of these are the kind of thing that costs nothing to add now and saves a debugging session later.