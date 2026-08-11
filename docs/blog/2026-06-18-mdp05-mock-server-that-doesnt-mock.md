---
layout: post
title: "CaseHub IoT — The Mock Server That Doesn't Mock"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [ci, testing, quarkus, vertx]
---

# CaseHub IoT — The Mock Server That Doesn't Mock

**Date:** 2026-06-18
**Type:** phase-update

---

## What I was trying to fix: CI has never been green

casehub-iot's GitHub Actions build has been red since the repo was created. Two independent problems, both invisible locally.

## The easy one: Maven couldn't find its own parent

The root POM declared `casehub-parent:0.2-SNAPSHOT` as its parent but had no `<repositories>` section pointing at GitHub Packages. Locally it works — the parent is cached in `~/.m2/` from sibling repos. CI starts from a clean Maven cache every time and has nowhere to look.

The fix took five minutes: add `<repositories>`, `<distributionManagement>`, and `<relativePath/>` to the root POM, matching the pattern casehub-life already uses. The error message was clear, the fix was obvious.

## The hard one: 68 tests, 68 timeouts

Every Home Assistant integration test timed out. Not flaky — 100% consistent, every test, every run. The REST client connected to OkHttp MockWebServer on the right port (confirmed in the error messages), enqueued responses were waiting, but nothing came back. Ten seconds of silence, then `NoStackTraceTimeoutException`.

I started with the obvious: maybe MockWebServer 5.3.2 (an alpha Kotlin rewrite) had broken something. Downgraded to 4.12.0 — same timeout. Tried disabling HTTP/2 and ALPN negotiation on the Quarkus REST client — same timeout. Changed the WebSocket client URL to rule out port collisions with the Quarkus test server — same timeout. Ran a single test in complete isolation — same timeout.

The symptom was misleading. It looked like a configuration problem — wrong port, wrong protocol, wrong headers. But the port was right, the protocol was right, and the headers never got a chance to matter because the response was never received.

The diagnostic that cracked it was simple: write a ten-line Java program that creates a MockWebServer, sends a request with `java.net.http.HttpClient`, and prints the response. It worked instantly. Same JVM, same MockWebServer, same classpath. The plain Java HTTP client got the response; the Vert.x HTTP client inside Quarkus didn't.

## JDK to the rescue

The fix was to replace MockWebServer entirely. Not with WireMock (which would work but brings Jetty and a large dependency tree), but with `com.sun.net.httpserver.HttpServer` — a stable JDK API that's been there since Java 6. It's about 90 lines of code: a `ConcurrentLinkedQueue` for response stubs, another for recorded requests, and a single `HttpExchange` handler that connects the two.

All 68 tests pass in 3 seconds. The build went from perpetually red to green across all seven modules.

The irony is that `com.sun.net.httpserver` gets overlooked because of the `com.sun` namespace — developers assume it's internal API. It's not. It's an exported JDK module (`jdk.httpserver`), stable, and guaranteed to work with any HTTP client because it implements the actual HTTP/1.1 spec rather than whatever protocol dance MockWebServer's internal server does with Netty-based clients.

I still don't know exactly what goes wrong at the wire level between Vert.x and MockWebServer. The TCP connection succeeds, the request is sent and accepted, but the response never materialises. Something in how MockWebServer dispatches responses doesn't mesh with how Vert.x's Netty-based client reads them. The failure mode is completely silent — no error, no warning, just a timeout.
