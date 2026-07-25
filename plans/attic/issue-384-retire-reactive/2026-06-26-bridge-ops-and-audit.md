# Bridge Operations and Audit — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Docker Compose deployment, provider auto-discovery (mDNS/SSDP), and server-side audit event logging to the bridge ecosystem.

**Architecture:** Cross-cutting config changes first (tenancyId consolidation, Optional config, REST client programmatic creation), then the three features (discovery, audit, Docker) each built on the cleaned-up foundation. TDD throughout — tests before implementation.

**Tech Stack:** Quarkus 3.32, SmallRye Config, JmDNS (mDNS), raw UDP (SSDP), CDI events, MicroProfile REST Client Builder, Docker buildx

## Global Constraints

- `casehub-iot-api` is Tier 1 (pure Java + CDI annotations, no JPA, no Quarkus runtime)
- Spec: `docs/superpowers/specs/2026-06-25-bridge-ops-and-audit-design.md`
- Issues: #32, #33, #34; deferred: #35
- Every commit references an issue
- Build: `mvn --batch-mode install` must pass after every task
- Provider modules are consumed only via `Instance<DeviceProvider>` — never direct injection

---

### Task 1: TenancyId Consolidation

**Files:**
- Modify: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantConfig.java`
- Modify: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantEntityMapper.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabConfig.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabEntityMapper.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabSseClient.java`
- Modify: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeAgentConfig.java`
- Modify: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeConnectionManager.java`
- Modify: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeEventObserver.java`
- Modify: `bridge/src/main/java/io/casehub/iot/bridge/agent/BridgeCloudClient.java`
- Modify: `homeassistant/src/test/resources/application.properties`
- Modify: `openhab/src/test/resources/application.properties`
- Modify: `bridge/src/main/resources/application.properties`
- Modify: all test files that reference tenancyId config properties

**Interfaces:**
- Consumes: nothing (first task)
- Produces: single `casehub.iot.tenancy-id` config property consumed by all downstream tasks

- [ ] **Step 1: Update test application.properties files**

Replace per-module tenancyId properties with the root property in all three test config files:

`homeassistant/src/test/resources/application.properties` — replace `casehub.iot.homeassistant.tenancy-id=test-tenant` with `casehub.iot.tenancy-id=test-tenant`

`openhab/src/test/resources/application.properties` — replace `casehub.iot.openhab.tenancy-id=test-tenant` with `casehub.iot.tenancy-id=test-tenant`

`bridge/src/main/resources/application.properties` — replace `casehub.iot.bridge.tenancy-id=default` with `casehub.iot.tenancy-id=default`

- [ ] **Step 2: Remove tenancyId() from all three @ConfigMapping interfaces**

`HomeAssistantConfig` — remove `String tenancyId();` (line 10)

`OpenHabConfig` — remove `String tenancyId();` (line 11)

`BridgeAgentConfig` — remove `String tenancyId();` (line 12)

- [ ] **Step 3: Add @ConfigProperty injection in all 6 consumer files**

In each file, add `@Inject @ConfigProperty(name = "casehub.iot.tenancy-id") String tenancyId;` as a field, then replace all `config.tenancyId()` calls with `tenancyId`.

Files and usage counts:
- `HomeAssistantEntityMapper` — 11 usages of `config.tenancyId()`. Also update the test constructor to accept `String tenancyId` parameter.
- `OpenHabEntityMapper` — field `tenancyId` already exists, reads from `config.tenancyId()` in constructor. Change to injected field.
- `OpenHabSseClient` — 2 usages passing `config.tenancyId()` to `OpenHabThingResolver`. Change to injected field.
- `BridgeConnectionManager` — `config.tenancyId()` used for `X-Tenancy-ID` header and snapshot creation.
- `BridgeEventObserver` — `config.tenancyId()` for BridgeMessage stamping (3 usages).
- `BridgeCloudClient` — `config.tenancyId()` for Heartbeat stamping (1 usage).

- [ ] **Step 4: Run full build to verify**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all existing tests pass with the root property.

- [ ] **Step 5: Commit**

```
feat: consolidate tenancyId to single root property — #33

Three redundant tenancyId properties (bridge, HA, OpenHAB) replaced by
casehub.iot.tenancy-id. Eliminates divergence risk in bridge deployments.
```

---

### Task 2: Provider Config Safety + @LookupIfProperty

**Files:**
- Modify: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantConfig.java`
- Modify: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantProvider.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabConfig.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabProvider.java`
- Modify: `homeassistant/src/test/resources/application.properties`
- Modify: `openhab/src/test/resources/application.properties`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/ProviderActivationTest.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/ProviderActivationTest.java`

**Interfaces:**
- Consumes: tenancyId consolidation from Task 1
- Produces: `enabled` config property, `Optional<String> url()`, `Optional<String> token()` — used by Tasks 3-6

- [ ] **Step 1: Write activation test for HA**

Create `homeassistant/src/test/java/io/casehub/iot/homeassistant/ProviderActivationTest.java`:

```java
package io.casehub.iot.homeassistant;

import io.casehub.iot.api.spi.DeviceProvider;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(ProviderActivationTest.DisabledProfile.class)
class ProviderActivationTest {

    @Inject @Any Instance<DeviceProvider> providers;

    @Test
    void disabledProviderNotDiscoverable() {
        assertThat(providers.stream().count()).isZero();
    }

    public static class DisabledProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of(
                "casehub.iot.homeassistant.enabled", "false",
                "casehub.iot.tenancy-id", "test-tenant"
            );
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl homeassistant -Dtest=ProviderActivationTest`
Expected: FAIL — `enabled` property doesn't exist yet, `@LookupIfProperty` not applied.

- [ ] **Step 3: Make HomeAssistantConfig properties Optional, add enabled**

```java
@ConfigMapping(prefix = "casehub.iot.homeassistant")
public interface HomeAssistantConfig {
    @WithDefault("false") boolean enabled();
    Optional<String> url();
    Optional<String> token();
    @WithDefault("5")   int reconnectBaseSeconds();
    @WithDefault("300") int reconnectMaxSeconds();
    @WithDefault("30")  int pingIntervalSeconds();
    @WithDefault("10")  int pongTimeoutSeconds();
    @WithDefault("5")   int discoveryTimeoutSeconds();
}
```

- [ ] **Step 4: Add @LookupIfProperty to HomeAssistantProvider**

```java
@ApplicationScoped
@LookupIfProperty(name = "casehub.iot.homeassistant.enabled", stringValue = "true")
public class HomeAssistantProvider implements DeviceProvider {
```

- [ ] **Step 5: Update HA test application.properties**

Add `casehub.iot.homeassistant.enabled=true` so existing tests still discover the provider.

- [ ] **Step 6: Run ProviderActivationTest to verify it passes**

Run: `mvn --batch-mode test -pl homeassistant -Dtest=ProviderActivationTest`
Expected: PASS

- [ ] **Step 7: Repeat steps 1-6 for OpenHab**

Same pattern: `ProviderActivationTest` with `DisabledProfile`, `OpenHabConfig` gets `enabled`, `Optional<String> url()`, `@LookupIfProperty` on `OpenHabProvider`, test `application.properties` gets `enabled=true`.

OpenHabConfig already has `Optional<Bearer>` and `Optional<Basic>` for auth — no change needed there.

- [ ] **Step 8: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```
feat: provider activation via @LookupIfProperty + Optional config — #33

Disabled providers are invisible to Instance<DeviceProvider>. All config
properties Optional to prevent SmallRye startup validation failure when
a provider module is on the classpath but not enabled.
```

---

### Task 3: HA REST Client → Programmatic Creation

**Files:**
- Modify: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantRestClient.java`
- Modify: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantProvider.java`
- Modify: `homeassistant/src/main/resources/application.properties`
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/BearerAuthFilter.java`
- Modify: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantProviderTest.java`

**Interfaces:**
- Consumes: `Optional<String> url()` and `Optional<String> token()` from Task 2
- Produces: REST client created in `@PostConstruct` with runtime-resolved URL — needed by Task 5 (discovery)

- [ ] **Step 1: Create BearerAuthFilter**

Create `homeassistant/src/main/java/io/casehub/iot/homeassistant/BearerAuthFilter.java`:

```java
package io.casehub.iot.homeassistant;

import jakarta.ws.rs.client.ClientRequestContext;
import jakarta.ws.rs.client.ClientRequestFilter;

class BearerAuthFilter implements ClientRequestFilter {

    private final String authHeader;

    BearerAuthFilter(String token) {
        this.authHeader = "Bearer " + token;
    }

    @Override
    public void filter(ClientRequestContext ctx) {
        ctx.getHeaders().putSingle("Authorization", authHeader);
    }
}
```

- [ ] **Step 2: Strip CDI annotations from HomeAssistantRestClient**

Remove `@RegisterRestClient(configKey = "homeassistant")`, `@ClientHeaderParam(name = "Authorization", value = "{lookupToken}")`, and the `lookupToken()` default method. Keep `@GET`, `@POST`, `@Path` annotations — they work with `RestClientBuilder`.

```java
package io.casehub.iot.homeassistant;

import io.casehub.iot.homeassistant.internal.HaServiceCallDto;
import io.casehub.iot.homeassistant.internal.HaStateDto;
import io.smallrye.mutiny.Uni;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.core.Response;
import java.util.List;

public interface HomeAssistantRestClient {

    @GET
    @Path("/api/states")
    Uni<List<HaStateDto>> getStates();

    @POST
    @Path("/api/services/{domain}/{service}")
    Uni<Response> callService(@PathParam("domain") String domain,
                              @PathParam("service") String service,
                              HaServiceCallDto body);
}
```

- [ ] **Step 3: Remove property expression from application.properties**

Remove all `quarkus.rest-client."homeassistant".*` lines from `homeassistant/src/main/resources/application.properties`. The file becomes empty (or contains only non-rest-client properties if any exist).

- [ ] **Step 4: Update HomeAssistantProvider to create REST client programmatically**

Replace `@Inject @RestClient HomeAssistantRestClient restClient;` with a plain field. In `@PostConstruct`, resolve URL from config (discovery comes in Task 5), create the client with `RestClientBuilder`:

```java
@ApplicationScoped
@LookupIfProperty(name = "casehub.iot.homeassistant.enabled", stringValue = "true")
public class HomeAssistantProvider implements DeviceProvider {

    private static final Logger LOG = Logger.getLogger(HomeAssistantProvider.class);

    @Inject HomeAssistantConfig config;
    @Inject HomeAssistantWebSocketClient wsClient;
    @Inject HomeAssistantEntityMapper mapper;

    private HomeAssistantRestClient restClient;

    @PostConstruct
    void start() {
        String resolvedUrl = config.url()
            .orElseThrow(() -> new IllegalStateException(
                "casehub.iot.homeassistant.url is required when discovery is not available"));
        String token = config.token()
            .orElseThrow(() -> new IllegalStateException(
                "casehub.iot.homeassistant.token is required"));

        this.restClient = RestClientBuilder.newBuilder()
            .baseUri(URI.create(resolvedUrl))
            .register(new BearerAuthFilter(token))
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .build(HomeAssistantRestClient.class);

        wsClient.connect().subscribe().with(
            v -> {},
            e -> LOG.warnf(e, "HA initial connect failed")
        );
    }
    // ... rest unchanged
}
```

- [ ] **Step 5: Update HomeAssistantProviderTest**

The test uses `@QuarkusTest` with `TestHttpServerResource` which stubs HTTP responses. The test `application.properties` already sets `casehub.iot.homeassistant.url` and `casehub.iot.homeassistant.token`. With programmatic client creation, the provider reads these from config in `@PostConstruct` and creates the client. The test HTTP server URL must match.

Verify existing test still passes — the `@Inject HomeAssistantProvider provider` injection should work since `@LookupIfProperty` is satisfied by `enabled=true` in test config.

Run: `mvn --batch-mode test -pl homeassistant`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```
feat: HA REST client programmatic creation — #33

Remove @RegisterRestClient and property expression binding. REST client
created via RestClientBuilder in @PostConstruct with runtime-resolved URL.
Auth via BearerAuthFilter instead of lookupToken()/ConfigProvider.
```

---

### Task 4: OH REST Client → Programmatic Creation

**Files:**
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabRestClient.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabSseRestClient.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabProvider.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabSseClient.java`
- Delete: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabAuthHeadersFactory.java`
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabAuthFilter.java`
- Modify: `openhab/src/main/resources/application.properties`
- Delete: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabAuthHeadersFactoryTest.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabAuthFilterTest.java`

**Interfaces:**
- Consumes: `Optional<String> url()` from Task 2
- Produces: REST clients created in `@PostConstruct` — needed by Task 6 (discovery)

- [ ] **Step 1: Write OpenHabAuthFilter test**

Create `openhab/src/test/java/io/casehub/iot/openhab/OpenHabAuthFilterTest.java` — same logic as `OpenHabAuthHeadersFactoryTest` but testing a `ClientRequestFilter`:

```java
package io.casehub.iot.openhab;

import jakarta.ws.rs.client.ClientRequestContext;
import jakarta.ws.rs.core.MultivaluedHashMap;
import org.junit.jupiter.api.Test;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class OpenHabAuthFilterTest {

    @Test
    void bearerTokenAddsAuthorizationHeader() throws Exception {
        var auth = authWith(Optional.of(() -> "my-token"), Optional.empty());
        var filter = new OpenHabAuthFilter(auth);
        var ctx = mockContext();
        filter.filter(ctx);
        assertThat(ctx.getHeaders().getFirst("Authorization"))
            .isEqualTo("Bearer my-token");
    }

    @Test
    void basicAuthAddsBase64Header() throws Exception {
        var basic = mock(OpenHabConfig.Auth.Basic.class);
        when(basic.username()).thenReturn("user");
        when(basic.password()).thenReturn("pass");
        var auth = authWith(Optional.empty(), Optional.of(basic));
        var filter = new OpenHabAuthFilter(auth);
        var ctx = mockContext();
        filter.filter(ctx);
        assertThat(ctx.getHeaders().getFirst("Authorization"))
            .startsWith("Basic ");
    }

    @Test
    void bothBearerAndBasicThrows() {
        var auth = authWith(
            Optional.of(() -> "tok"),
            Optional.of(mock(OpenHabConfig.Auth.Basic.class)));
        assertThatThrownBy(() -> new OpenHabAuthFilter(auth))
            .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void noAuthSetsNoHeader() throws Exception {
        var auth = authWith(Optional.empty(), Optional.empty());
        var filter = new OpenHabAuthFilter(auth);
        var ctx = mockContext();
        filter.filter(ctx);
        assertThat(ctx.getHeaders().containsKey("Authorization")).isFalse();
    }

    private OpenHabConfig.Auth authWith(
            Optional<OpenHabConfig.Auth.Bearer> bearer,
            Optional<OpenHabConfig.Auth.Basic> basic) {
        var auth = mock(OpenHabConfig.Auth.class);
        when(auth.bearer()).thenReturn(bearer);
        when(auth.basic()).thenReturn(basic);
        return auth;
    }

    private ClientRequestContext mockContext() {
        var ctx = mock(ClientRequestContext.class);
        when(ctx.getHeaders()).thenReturn(new MultivaluedHashMap<>());
        return ctx;
    }
}
```

- [ ] **Step 2: Create OpenHabAuthFilter**

Create `openhab/src/main/java/io/casehub/iot/openhab/OpenHabAuthFilter.java`:

```java
package io.casehub.iot.openhab;

import jakarta.ws.rs.client.ClientRequestContext;
import jakarta.ws.rs.client.ClientRequestFilter;
import java.nio.charset.StandardCharsets;
import java.util.Base64;

class OpenHabAuthFilter implements ClientRequestFilter {

    private final String authHeader;

    OpenHabAuthFilter(OpenHabConfig.Auth auth) {
        boolean hasBearer = auth.bearer().isPresent();
        boolean hasBasic = auth.basic().isPresent();
        if (hasBearer && hasBasic) {
            throw new IllegalStateException(
                "Configure either casehub.iot.openhab.auth.bearer or .auth.basic, not both");
        }
        if (hasBasic) {
            var basic = auth.basic().get();
            authHeader = "Basic " + Base64.getEncoder().encodeToString(
                (basic.username() + ":" + basic.password()).getBytes(StandardCharsets.UTF_8));
        } else if (hasBearer) {
            authHeader = "Bearer " + auth.bearer().get().token();
        } else {
            authHeader = null;
        }
    }

    @Override
    public void filter(ClientRequestContext ctx) {
        if (authHeader != null) {
            ctx.getHeaders().putSingle("Authorization", authHeader);
        }
    }
}
```

- [ ] **Step 3: Run OpenHabAuthFilterTest**

Run: `mvn --batch-mode test -pl openhab -Dtest=OpenHabAuthFilterTest`
Expected: PASS

- [ ] **Step 4: Strip CDI annotations from both REST client interfaces**

`OpenHabRestClient` — remove `@RegisterRestClient(configKey = "openhab")` and `@RegisterClientHeaders(OpenHabAuthHeadersFactory.class)`. Keep JAX-RS annotations.

`OpenHabSseRestClient` — remove `@RegisterRestClient(configKey = "openhab-sse")` and `@RegisterClientHeaders(OpenHabAuthHeadersFactory.class)`. Keep JAX-RS annotations.

- [ ] **Step 5: Remove property expressions from openhab/application.properties**

Remove all `quarkus.rest-client."openhab".*` and `quarkus.rest-client."openhab-sse".*` lines.

- [ ] **Step 6: Update OpenHabProvider and OpenHabSseClient for programmatic creation**

`OpenHabProvider.start()` — resolve URL from config, create REST client, create SSE client:

```java
@PostConstruct
void start() {
    String resolvedUrl = config.url()
        .orElseThrow(() -> new IllegalStateException(
            "casehub.iot.openhab.url is required when discovery is not available"));
    var authFilter = new OpenHabAuthFilter(config.auth());

    this.restClient = RestClientBuilder.newBuilder()
        .baseUri(URI.create(resolvedUrl))
        .register(authFilter)
        .connectTimeout(5, TimeUnit.SECONDS)
        .readTimeout(10, TimeUnit.SECONDS)
        .build(OpenHabRestClient.class);

    sseClient.init(resolvedUrl, authFilter);
    sseClient.connect().subscribe().with(
        v -> {},
        e -> LOG.warnf(e, "OpenHAB initial connect failed")
    );
}
```

`OpenHabSseClient` — add `init(String url, OpenHabAuthFilter authFilter)` method that creates the SSE REST client programmatically. The SSE client currently uses `@Inject @RestClient OpenHabSseRestClient sseRestClient;` — replace with a field set in `init()`.

- [ ] **Step 7: Delete OpenHabAuthHeadersFactory and its test**

Delete `openhab/src/main/java/io/casehub/iot/openhab/OpenHabAuthHeadersFactory.java` and `openhab/src/test/java/io/casehub/iot/openhab/OpenHabAuthHeadersFactoryTest.java`.

- [ ] **Step 8: Run full OpenHAB tests**

Run: `mvn --batch-mode test -pl openhab`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```
feat: OH REST client programmatic creation — #33

Remove @RegisterRestClient and property expression binding for both
openhab and openhab-sse clients. Delete OpenHabAuthHeadersFactory
(ClientHeadersFactory is not a JAX-RS provider — silently ignored by
RestClientBuilder.register()). Replace with OpenHabAuthFilter
(ClientRequestFilter).
```

---

### Task 5: HA Auto-Discovery (mDNS)

**Files:**
- Modify: `homeassistant/pom.xml` — add JmDNS dependency
- Create: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantDiscovery.java`
- Modify: `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantProvider.java`
- Create: `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantDiscoveryTest.java`

**Interfaces:**
- Consumes: `Optional<String> url()`, `discoveryTimeoutSeconds()` from Task 2; programmatic REST client from Task 3
- Produces: runtime-resolved URL passed to `RestClientBuilder` in `@PostConstruct`

- [ ] **Step 1: Add JmDNS dependency to homeassistant/pom.xml**

```xml
<dependency>
    <groupId>org.jmdns</groupId>
    <artifactId>jmdns</artifactId>
    <version>3.6.0</version>
</dependency>
```

- [ ] **Step 2: Write discovery test**

Create `homeassistant/src/test/java/io/casehub/iot/homeassistant/HomeAssistantDiscoveryTest.java`:

```java
package io.casehub.iot.homeassistant;

import org.junit.jupiter.api.Test;
import javax.jmdns.ServiceInfo;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class HomeAssistantDiscoveryTest {

    @Test
    void constructsUrlFromServiceInfo() {
        var info = ServiceInfo.create(
            "_home-assistant._tcp.local.",
            "Home Assistant",
            8123, "");
        // ServiceInfo needs host set — use reflection or test helper
        String url = HomeAssistantDiscovery.buildUrl("192.168.1.50", 8123);
        assertThat(url).isEqualTo("http://192.168.1.50:8123");
    }

    @Test
    void throwsOnTimeoutWithClearMessage() {
        assertThatThrownBy(() ->
            HomeAssistantDiscovery.resolve(0))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("_home-assistant._tcp");
    }
}
```

- [ ] **Step 3: Implement HomeAssistantDiscovery**

Create `homeassistant/src/main/java/io/casehub/iot/homeassistant/HomeAssistantDiscovery.java`:

```java
package io.casehub.iot.homeassistant;

import org.jboss.logging.Logger;
import javax.jmdns.JmDNS;
import javax.jmdns.ServiceInfo;
import java.net.InetAddress;

class HomeAssistantDiscovery {

    private static final Logger LOG = Logger.getLogger(HomeAssistantDiscovery.class);
    private static final String SERVICE_TYPE = "_home-assistant._tcp.local.";

    static String resolve(int timeoutSeconds) {
        try (JmDNS jmdns = JmDNS.create(InetAddress.getLocalHost())) {
            ServiceInfo[] services = jmdns.list(SERVICE_TYPE, timeoutSeconds * 1000);
            if (services.length == 0) {
                throw new IllegalStateException(
                    "No Home Assistant instance found via mDNS (" + SERVICE_TYPE
                    + ") within " + timeoutSeconds + "s. Set casehub.iot.homeassistant.url explicitly.");
            }
            if (services.length > 1) {
                LOG.infof("Found %d HA instances via mDNS, using first: %s:%d",
                    services.length,
                    services[0].getHostAddresses()[0],
                    services[0].getPort());
            }
            ServiceInfo info = services[0];
            return buildUrl(info.getHostAddresses()[0], info.getPort());
        } catch (IllegalStateException e) {
            throw e;
        } catch (Exception e) {
            throw new IllegalStateException("mDNS discovery failed: " + e.getMessage(), e);
        }
    }

    static String buildUrl(String host, int port) {
        return "http://" + host + ":" + port;
    }
}
```

- [ ] **Step 4: Integrate discovery into HomeAssistantProvider.start()**

Update the URL resolution in `@PostConstruct`:

```java
String resolvedUrl = config.url()
    .orElseGet(() -> HomeAssistantDiscovery.resolve(config.discoveryTimeoutSeconds()));
```

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl homeassistant`
Expected: All tests pass. The mDNS discovery is not exercised in CI (requires multicast); existing tests provide URL explicitly.

- [ ] **Step 6: Commit**

```
feat: HA auto-discovery via mDNS — #33

JmDNS discovers _home-assistant._tcp.local. when
casehub.iot.homeassistant.url is not configured. Falls back to
explicit URL when set. Timeout configurable via discovery-timeout-seconds.
```

---

### Task 6: OH Auto-Discovery (mDNS + SSDP)

**Files:**
- Modify: `openhab/pom.xml` — add JmDNS dependency
- Create: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabDiscovery.java`
- Modify: `openhab/src/main/java/io/casehub/iot/openhab/OpenHabProvider.java`
- Create: `openhab/src/test/java/io/casehub/iot/openhab/OpenHabDiscoveryTest.java`

**Interfaces:**
- Consumes: `Optional<String> url()`, `discoveryTimeoutSeconds()` from Task 2; programmatic REST client from Task 4
- Produces: runtime-resolved URL for OpenHAB REST + SSE clients

- [ ] **Step 1: Add JmDNS dependency to openhab/pom.xml**

Same as Task 5 Step 1.

- [ ] **Step 2: Write discovery test**

Create `openhab/src/test/java/io/casehub/iot/openhab/OpenHabDiscoveryTest.java`:

```java
package io.casehub.iot.openhab;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class OpenHabDiscoveryTest {

    @Test
    void buildUrlFromHostAndPort() {
        assertThat(OpenHabDiscovery.buildUrl("192.168.1.101", 8080, false))
            .isEqualTo("http://192.168.1.101:8080");
    }

    @Test
    void buildSslUrl() {
        assertThat(OpenHabDiscovery.buildUrl("192.168.1.101", 8443, true))
            .isEqualTo("https://192.168.1.101:8443");
    }

    @Test
    void throwsOnTimeoutWithClearMessage() {
        assertThatThrownBy(() -> OpenHabDiscovery.resolve(0))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("mDNS")
            .hasMessageContaining("SSDP");
    }

    @Test
    void parseSsdpLocationHeader() {
        String response = "HTTP/1.1 200 OK\r\nLOCATION: http://192.168.1.101:8080/rest\r\n\r\n";
        assertThat(OpenHabDiscovery.parseSsdpLocation(response))
            .isEqualTo("http://192.168.1.101:8080");
    }
}
```

- [ ] **Step 3: Implement OpenHabDiscovery**

Create `openhab/src/main/java/io/casehub/iot/openhab/OpenHabDiscovery.java`:

mDNS first (`_openhab-server-ssl._tcp.local.` preferred, `_openhab-server._tcp.local.` fallback), then SSDP M-SEARCH on `239.255.255.250:1900` with raw UDP. Parse LOCATION header from responses. Timeout split: 60% mDNS, 40% SSDP.

- [ ] **Step 4: Integrate into OpenHabProvider.start()**

```java
String resolvedUrl = config.url()
    .orElseGet(() -> OpenHabDiscovery.resolve(config.discoveryTimeoutSeconds()));
```

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl openhab`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```
feat: OH auto-discovery via mDNS + SSDP — #33

JmDNS discovers _openhab-server[-ssl]._tcp.local. first (prefers SSL),
falls back to SSDP M-SEARCH on 239.255.255.250:1900. Raw UDP, no library.
Timeout configurable via discovery-timeout-seconds (default 10).
```

---

### Task 7: Bridge Audit Event Types

**Files:**
- Create: `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEvent.java`
- Create: `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEventType.java`
- Create: `api/src/test/java/io/casehub/iot/api/bridge/BridgeAuditEventTest.java`

**Interfaces:**
- Consumes: `BridgeMessage` (existing in `api/bridge/`)
- Produces: `BridgeAuditEvent` record and `BridgeAuditEventType` enum — consumed by Tasks 8

- [ ] **Step 1: Write test for BridgeAuditEvent construction**

Create `api/src/test/java/io/casehub/iot/api/bridge/BridgeAuditEventTest.java`:

```java
package io.casehub.iot.api.bridge;

import io.casehub.iot.api.StateChangeEvent;
import io.casehub.iot.testing.Fixtures;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class BridgeAuditEventTest {

    @Test
    void stateChangeEventHasMessageAndDeviceId() {
        var device = Fixtures.light("light.kitchen", true);
        var sce = new StateChangeEvent(null, device, device.capabilities().keySet(), Instant.now(), "ha");
        var msg = new BridgeMessage.StateChange("tenant-1", Instant.now(), sce);
        var audit = new BridgeAuditEvent(
            "tenant-1", Instant.now(), BridgeAuditEventType.STATE_CHANGE,
            null, "light.kitchen", msg);
        assertThat(audit.tenancyId()).isEqualTo("tenant-1");
        assertThat(audit.eventType()).isEqualTo(BridgeAuditEventType.STATE_CHANGE);
        assertThat(audit.deviceId()).isEqualTo("light.kitchen");
        assertThat(audit.message()).isNotNull();
        assertThat(audit.correlationId()).isNull();
    }

    @Test
    void connectionEventHasNullMessage() {
        var audit = new BridgeAuditEvent(
            "tenant-1", Instant.now(), BridgeAuditEventType.AGENT_CONNECTED,
            null, null, null);
        assertThat(audit.message()).isNull();
        assertThat(audit.deviceId()).isNull();
    }

    @Test
    void commandSentHasCorrelationId() {
        var audit = new BridgeAuditEvent(
            "tenant-1", Instant.now(), BridgeAuditEventType.COMMAND_SENT,
            "corr-123", "switch.hall", null);
        assertThat(audit.correlationId()).isEqualTo("corr-123");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=BridgeAuditEventTest`
Expected: FAIL — classes don't exist yet.

- [ ] **Step 3: Create BridgeAuditEventType enum**

Create `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEventType.java`:

```java
package io.casehub.iot.api.bridge;

public enum BridgeAuditEventType {
    STATE_CHANGE,
    REPLAYED_STATE_CHANGE,
    STATE_SNAPSHOT,
    PROVIDER_STATUS_CHANGE,
    COMMAND_SENT,
    COMMAND_RESPONSE,
    AGENT_CONNECTED,
    AGENT_DISCONNECTED
}
```

- [ ] **Step 4: Create BridgeAuditEvent record**

Create `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEvent.java`:

```java
package io.casehub.iot.api.bridge;

import jakarta.annotation.Nullable;
import java.time.Instant;

public record BridgeAuditEvent(
    String tenancyId,
    Instant receivedAt,
    BridgeAuditEventType eventType,
    @Nullable String correlationId,
    @Nullable String deviceId,
    @Nullable BridgeMessage message
) {}
```

- [ ] **Step 5: Run test**

Run: `mvn --batch-mode test -pl api -Dtest=BridgeAuditEventTest`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat: BridgeAuditEvent + BridgeAuditEventType — #34

CDI event record for bridge protocol audit. Placed in api/bridge/
(Tier 1, pure Java). Supports dual-trail pattern via multiple
independent CDI observers.
```

---

### Task 8: Audit Firing Points + Logging Observer

**Files:**
- Modify: `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeWebSocketEndpoint.java`
- Modify: `bridge-server/src/main/java/io/casehub/iot/bridge/server/BridgeDeviceProvider.java`
- Create: `bridge-server/src/main/java/io/casehub/iot/bridge/server/audit/LoggingBridgeAuditObserver.java`
- Create: `bridge-server/src/test/java/io/casehub/iot/bridge/server/audit/LoggingBridgeAuditObserverTest.java`
- Modify: `bridge-server/src/test/java/io/casehub/iot/bridge/server/BridgeDeviceProviderTest.java`

**Interfaces:**
- Consumes: `BridgeAuditEvent`, `BridgeAuditEventType` from Task 7
- Produces: audit events fired for all bridge protocol messages

- [ ] **Step 1: Write LoggingBridgeAuditObserver test**

Create `bridge-server/src/test/java/io/casehub/iot/bridge/server/audit/LoggingBridgeAuditObserverTest.java`:

Test that the observer handles each `BridgeAuditEventType` without throwing. Use a captured log handler to verify structured output.

- [ ] **Step 2: Create LoggingBridgeAuditObserver**

Create `bridge-server/src/main/java/io/casehub/iot/bridge/server/audit/LoggingBridgeAuditObserver.java`:

```java
package io.casehub.iot.bridge.server.audit;

import io.casehub.iot.api.bridge.BridgeAuditEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import org.jboss.logging.Logger;

@ApplicationScoped
public class LoggingBridgeAuditObserver {

    private static final Logger LOG = Logger.getLogger("io.casehub.iot.bridge.audit");

    void onAudit(@ObservesAsync BridgeAuditEvent event) {
        LOG.infof("[AUDIT] type=%s tenancy=%s device=%s correlation=%s",
            event.eventType(), event.tenancyId(),
            event.deviceId(), event.correlationId());
    }
}
```

- [ ] **Step 3: Add audit event injection to BridgeWebSocketEndpoint**

Add `@Inject Event<BridgeAuditEvent> auditEvents;` field. Fire audit in `onOpen()`, `onClose()`, and in the `onMessage()` switch — for each of the 5 handled variants (excluding Heartbeat and anomalous Command):

```java
// In onOpen():
auditEvents.fireAsync(new BridgeAuditEvent(
    tenancyId, Instant.now(), BridgeAuditEventType.AGENT_CONNECTED,
    null, null, null));

// In onMessage(), after deserializing and before each case:
// Fire audit for STATE_CHANGE, REPLAYED_STATE_CHANGE, STATE_SNAPSHOT,
// PROVIDER_STATUS_CHANGE, COMMAND_RESPONSE
```

- [ ] **Step 4: Add audit event firing to BridgeDeviceProvider.dispatch()**

After constructing the `BridgeMessage.Command`, fire `COMMAND_SENT`:

```java
auditEvents.fireAsync(new BridgeAuditEvent(
    tenancyId, Instant.now(), BridgeAuditEventType.COMMAND_SENT,
    correlationId, localId, bridgeCommand));
```

- [ ] **Step 5: Run bridge-server tests**

Run: `mvn --batch-mode test -pl bridge-server`
Expected: All tests pass.

- [ ] **Step 6: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat: bridge audit firing + logging observer — #34

Audit fires from BridgeWebSocketEndpoint (@OnOpen, @OnClose,
@OnTextMessage for 5 of 7 variants) and BridgeDeviceProvider.dispatch()
(COMMAND_SENT). LoggingBridgeAuditObserver provides always-on
operational trail via structured logging.
```

---

### Task 9: Docker Compose + Deployment Guide

**Files:**
- Create: `bridge/src/main/docker/Dockerfile.jvm`
- Create: `bridge/docker-compose.yml`
- Create: `bridge/.env.example`
- Create: `bridge/DEPLOYMENT.md`
- Modify: `.github/workflows/publish.yml`

**Interfaces:**
- Consumes: all prior tasks (provider activation, discovery, audit)
- Produces: deployable Docker image and compose configuration

- [ ] **Step 1: Create Dockerfile.jvm**

Create `bridge/src/main/docker/Dockerfile.jvm`:

```dockerfile
FROM eclipse-temurin:21-jre-alpine

ARG RUN_JAVA_VERSION=1.3.8
ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en'

RUN addgroup -S quarkus && adduser -S -G quarkus -u 1001 quarkus
COPY --chown=1001 target/quarkus-app/lib/ /app/lib/
COPY --chown=1001 target/quarkus-app/*.jar /app/
COPY --chown=1001 target/quarkus-app/app/ /app/app/
COPY --chown=1001 target/quarkus-app/quarkus/ /app/quarkus/

EXPOSE 8080
USER 1001

ENTRYPOINT ["java", "-jar", "/app/quarkus-run.jar", \
  "-Dquarkus.http.host=0.0.0.0"]
```

- [ ] **Step 2: Create docker-compose.yml**

Create `bridge/docker-compose.yml` — per spec Section 1.

- [ ] **Step 3: Create .env.example**

Create `bridge/.env.example` — per spec Section 1.

- [ ] **Step 4: Write DEPLOYMENT.md**

Create `bridge/DEPLOYMENT.md` covering: prerequisites, configuration reference (all env vars with defaults and descriptions), deploy workflow (`docker compose up -d`), verify health (`/q/health/ready`), update procedure, troubleshooting.

- [ ] **Step 5: Add multi-arch build to CI workflow**

Modify `.github/workflows/publish.yml` — add a `docker-build` job after `publish` that uses `docker/build-push-action` with `platforms: linux/amd64,linux/arm64`, pushing to `ghcr.io/casehubio/iot-bridge`.

- [ ] **Step 6: Build Docker image locally to verify**

Run: `mvn --batch-mode package -pl bridge -DskipTests && docker build -f bridge/src/main/docker/Dockerfile.jvm bridge/`
Expected: Image builds successfully.

- [ ] **Step 7: Commit**

```
feat: Docker Compose + deployment guide — #32

Dockerfile.jvm (Temurin 21-jre-alpine, fast-jar), docker-compose.yml
(host network, health check, event store volume), .env.example,
DEPLOYMENT.md, and multi-arch CI build for ARM64 + x86_64.
```
