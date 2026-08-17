---
layout: post
title: "Five Bugs, One Branch, and a Quarkus Version That Fixed Itself"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [openhab, quarkus, bug-fixes, device-mapping, command-dispatch]
series: issue-96-split-package-fix
---

The IoT repo had five open bugs clustering around the OpenHAB provider — device class mapping gaps, command dispatch failing silently on Color items, a persistence unit that was never configured, and the headline issue: `quarkus:dev` crashing on duplicate synthetic beans from split packages.

The split-package issue turned out to be the most interesting for the least effort. The reported root cause was Quarkus discovering `@ConfigMapping` interfaces from both the provider JAR and the webapp's classpath, registering them as synthetic beans twice. We dug into it expecting package restructuring. Instead, running `quarkus:dev` against Quarkus 3.32.2 produced... no error. The `workspaceDiscovery=false` config that a previous session had added to suppress the problem now triggers a warning: `Parameter 'workspaceDiscovery' is unknown`. The Quarkus upgrade had resolved it silently. We removed the dead config and moved on.

The OpenHAB mapping bugs were more satisfying to fix because they had a clear chain of consequences. `MotionDetector` Equipment tags mapped to `DeviceClass.SENSOR` instead of `PRESENCE_SENSOR`. The `PresenceSensor` tag wasn't mapped at all. This meant `IoTCloudEventAdapter` generated `state_change.sensor` events for motion detectors — and `MotionAtTimeGanglion`, which listens exclusively for `state_change.presence_sensor`, never fired. The entire night-motion detection chain was broken by two missing lines in a switch statement.

The ON/OFF command dispatch had a subtler problem. `resolveRequiredTags` required `{Control, Switch}` for `turn_on`/`turn_off`, but OpenHAB Color items (LIFX bulbs, for example) carry `{Control, Color}` — no `Switch` tag. Color items accept ON/OFF commands perfectly well in OpenHAB; the semantic tag just differs. The fix follows the same fallback pattern already used for `set_position`: try `{Control, Switch}` first, then `{Control, Color}`, then `{Control, Light}`.

The command validation gap was simpler but worth closing. An invalid action string like `TURN_OFF` (upper-case) reached `buildCommandValue()`, returned `null`, and `dispatch()` silently returned `CommandResult.FAILED` with HTTP 200. No indication of what went wrong. We added a `VALID_ACTIONS` set to `DeviceCommand` and validate at the REST boundary — `BadRequestException` with the valid action list.

The persistence unit fix was a single missing configuration block. The `iot-ras` datasource existed but had no `quarkus.hibernate-orm.iot-ras.*` config, leaving RAS entities (`SituationEntity`, `GanglionStateEntity`) orphaned with no persistence unit. Three lines of `application.properties` resolved it.

The upstream RAS API had also moved to blocking returns per ADR-0005 since the last time anyone touched the Drools ganglion tests. Every `detect()` call had `.await().indefinitely()` appended — fifty-odd occurrences across two test files. A regex script handled that in seconds.

What made this session satisfying is how small the fixes were relative to the impact. Two lines in `resolveDeviceClass` reconnect the entire night-motion detection chain. A tag fallback list restores Color device control. A `VALID_ACTIONS` set turns silent failures into actionable errors. None of these required design work — just reading the issue descriptions, tracing the code paths, and applying the obvious fix.
