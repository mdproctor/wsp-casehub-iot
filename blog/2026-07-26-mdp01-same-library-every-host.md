---
layout: post
title: "Same Library, Every Host"
date: 2026-07-26
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub IoT]
tags: [mcp, device-provider, architecture]
series: issue-69-mcp-device-provider-tools
---

The IoT repo already has the full device stack — typed entities, providers for HA and OpenHAB, a registry that caches everything in memory, command dispatch. What it didn't have was a way for LLM agents to reach any of it.

The connectors repo solved this for messaging months ago: `ChatPlatformMcpTool` sits in a library module, depends on `quarkus-mcp-server-core` (not `-http`), and any Quarkus app that adds the JAR plus the HTTP transport gets `send_chat` at `/mcp`. The host app decides which providers are active. The tool doesn't care.

We followed the same pattern for IoT. New module `casehub-iot-mcp`, three tools: `iot_get_devices`, `iot_get_state`, `iot_send_command`. The interesting part isn't the tools themselves — device list, device detail, command dispatch — but where they run.

Drop the module into the bridge app and the tools see local HA and OpenHAB devices directly. Drop it into the webapp and they see remote devices through `BridgeDeviceProvider`, which makes bridge-connected devices look local. Drop it into both and you get a federated view. The MCP tool class injects `DeviceRegistry` and `Instance<DeviceProvider>` — whatever the host configures is what the agent sees. No conditional logic, no host detection.

Three design choices worth noting. First, `iot_get_devices` returns a summary projection — device ID, class, label, availability, last-updated timestamp — not the full `DeviceEntity` with all its typed state fields. An agent scanning 50 devices to find the one that's offline doesn't need thermostat temperatures clogging its context window. `iot_get_state` returns the full Jackson-serialized entity when the agent wants depth.

Second, `iot_send_command` takes `Map<String, Object>` as a native `@ToolArg` parameter rather than a JSON string. The Quarkus MCP framework deserializes the map directly via `ObjectMapper.convertValue()` — undocumented but verified in the framework source. This avoids forcing the LLM to double-encode JSON (a string containing escaped JSON inside the already-JSON tool call payload). One less thing to go wrong.

Third, dispatch has a 30-second safety timeout that the connectors tools don't need. `send_chat` hits an HTTP API with its own timeout. Device command dispatch goes through `DeviceProvider.dispatch()`, which is a `Uni<CommandResult>` — and the bridge provider waits on a WebSocket response correlation that could hang if the bridge agent disconnects mid-flight. The MCP worker thread pool can't afford that, so the tool wraps dispatch in `await().atMost(Duration.ofSeconds(30))` as a circuit breaker.

The deferred items — RBAC, tenancy filtering, audit integration — are all blocked on the same prerequisite: principal propagation from OpenClaw to CaseHub MCP endpoints. When that lands, `dispatchedBy` stops being `"mcp-agent"` and becomes the actual household user identity. The tools are ready for it; the plumbing isn't.
