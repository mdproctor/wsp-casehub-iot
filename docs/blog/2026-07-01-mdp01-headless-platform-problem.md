---
layout: post
title: "The Headless Platform Problem"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [webapp, ras, situational-awareness, pages, flyway]
series: issue-44-iot-webapp-console
---

*Part of a series on [#44 — IoT webapp — operational console with situational awareness](https://github.com/casehubio/iot/issues/44). Previous: [XS Test: The Design Review That Found a Real Bug](2026-06-29-mdp06-xs-test-design-review.md).*

I came in planning to do table partitioning. PostgreSQL range partitioning by month on `bridge_audit_event` — the kind of work that looks productive because it's technical and measurable. Then I stopped and asked what problem it actually solved.

At ~876K rows per year, the existing index-based purge handles the volume in seconds. Partitioning becomes relevant at multi-year retention with millions of rows, and no deployment is anywhere near that. I was building infrastructure for a problem that doesn't exist yet.

The real problem was staring at me from the handoff: we've built a complete IoT platform — typed device hierarchy, provider SPIs, bridge relay, audit persistence, CloudEvent adapters — and nobody can use any of it without writing code. A headless thing. What use is that to anyone?

That question opened up something bigger than a device dashboard. IoT feeds events into RAS for situation detection. RAS fires cases through the engine. Cases create WorkItems for human decisions. The full automation loop — temperature spike → fire-risk situation → safety-alert case → unlock doors + kill HVAC + notify household — existed in code as wired CDI events, but with no face.

The webapp wires the entire loop into a standalone Quarkus application. Three new modules: `webapp-api` carries the reusable domain logic (five JavaSwitch ganglia for threshold detection, two DroolsCEP ganglia for temporal patterns, four case descriptors following the AML `*CaseDescriptor` pattern, an `ActionRiskClassifier` with deliberate safety asymmetry — during a fire, doors unlock without asking). `webapp-drools` isolates the Drools dependency so consumers wanting only JavaSwitch ganglia aren't forced to pull CEP. `webapp` wires everything together with REST endpoints, SSE for live device state, Flyway migrations, and seven composable TypeScript pages via Quinoa.

The pages use casehub-pages' TypeScript DSL — type-safe, composable as functions, designed so casehub-life can pick them up later. Health dashboard, device inventory with command controls, situation monitor, case viewer, WorkItem inbox, audit trail, provider status. All dark-mode, all with live refresh.

The interesting technical decision was the three-datasource Flyway layout. RAS ships migrations at V1–V4, bridge at V1–V2, work at V1–V39 — all independently published artifacts with fixed version numbers I can't renumber. Three modules, all starting at V1, all needing to share one PostgreSQL database. The solution: three named Quarkus datasources pointing at the same JDBC URL, each with its own `flyway_schema_history` table. Logical isolation, not physical. It feels wrong — why open three connections to the same database? — but the overhead is negligible and the alternative (forking upstream migrations to renumber them) is a permanent maintenance burden.

The webapp can't compile yet. Cross-repo dependencies — `casehub-ras`, `casehub-engine`, `casehub-work`, `casehub-ledger` — aren't published to GitHub Packages. The `webapp-api` and `webapp-drools` modules build and test cleanly (53 tests between them), and the webapp's structure follows the AML pattern exactly. Once those artifacts are published, the full build will work. The design review caught a version mismatch in the YAML case definitions that would have silently prevented RAS from finding the CaseHub beans at runtime — `DefaultCaseTrigger` does exact string matching on the `(namespace, name, version)` triple.

This started as "do #42" and ended as "make the platform usable." Sometimes the right answer to "what should I build next?" is "step back and ask what anyone can actually do with what I've already built."
