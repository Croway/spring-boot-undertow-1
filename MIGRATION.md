# Undertow Spring Boot Migration Guide

This guide helps you migrate from Spring Boot's Undertow integration or an earlier
version of this project's starter to `io.undertow:undertow-spring-boot-starter`.
Apply each section that affects your application, including changes introduced by
intermediate versions when upgrading across several releases.

Spring Boot 4 removed its Undertow integration. This project provides a separately
maintained replacement using Undertow EE and Jakarta Servlet 6.1. Follow the
[Spring Boot 4 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
alongside this guide for changes to Spring Framework, Spring Security, testing, and
other Spring Boot features.

## Upgrade paths and requirements

The `1.0.0.Alpha1` code targets Java 17+, Spring Boot 4.1.1, and Undertow EE
2.0.2.Final. These are the build baselines for this version, not a compatibility
guarantee for every Spring Boot 4.x release. Check the target release's
[README](README.md), [changelog](CHANGELOG.md), and dependency versions before upgrading.
The starter has its own version; do not use the Spring Boot or Camel version as its version.

| Starting point | Migration path |
|---|---|
| Spring Boot 3.x with `org.springframework.boot:spring-boot-starter-undertow` | Follow Spring's guidance to upgrade to the latest 3.5.x first, then upgrade to Boot 4 and replace the Undertow dependency, properties, and Java APIs described below. |
| Spring Boot 2.x with the official Undertow starter | Complete the Boot 2 to 3 migration first, including the move from `javax.servlet` to `jakarta.servlet`, then follow the Boot 3 to 4 path. |
| Spring Boot 4 with `com.redhat.integration:spring-boot-starter-undertow` | Replace the earlier downstream starter and its Java package references, then migrate configuration properties. |
| An application already using `io.undertow:undertow-spring-boot-starter` | Review the target release's changelog and remove deprecated property prefixes before their compatibility support is removed. |

## Dependency changes

### Starter replacement

Remove the previous Undertow starter and add:

```xml
<dependency>
    <groupId>io.undertow</groupId>
    <artifactId>undertow-spring-boot-starter</artifactId>
    <version>${undertow-starter.version}</version>
</dependency>
```

Set `undertow-starter.version` to the compatible release you intend to use. Omit
`version` only if your application's dependency management already manages this exact
coordinate. A Spring Boot BOM does not by itself manage this third-party starter.
For a product-managed application, follow its BOM and supported dependency set.

Applications that directly depended on the integration module must also replace it:

| Previous dependency | Replacement |
|---|---|
| `com.redhat.integration:spring-boot-undertow` | `io.undertow:undertow-spring-boot-autoconfigure` |

Prefer the starter unless you deliberately manage its runtime dependencies yourself.
Do not keep both the old and new integration modules on the classpath.

### Spring MVC

Keep the application's web starter and exclude its default Tomcat server starter.
For example, when using `spring-boot-starter-web`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

Apply the same exclusion if migrating to Boot 4's `spring-boot-starter-webmvc`.
Remove other embedded server starters, such as Jetty, when Undertow is to serve the
application. An existing web server factory bean can cause Undertow's factory
auto-configuration to back off.

### WebFlux

Keep `spring-boot-starter-webflux` and exclude its default server starter:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-reactor-netty</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

Review other dependency paths that may bring a competing server back onto the
classpath. Keep any libraries your application needs independently, for example
for an HTTP client. Avoid adding the MVC starter just to obtain Servlet classes:
the Undertow starter supplies its Servlet runtime, and adding MVC can change
Spring Boot's application-type detection.

### Undertow libraries and logging

The Servlet and WebSocket implementations now use
`io.undertow.ee:undertow-servlet` and `io.undertow.ee:undertow-websockets`.
Undertow core remains `io.undertow:undertow-core`; the Java packages of the
Undertow libraries still use `io.undertow.*`. Review old direct dependencies,
exclusions, and version overrides rather than replacing every Undertow group ID.

The starter no longer depends on `spring-boot-starter` and therefore no longer
supplies that starter's logging dependencies itself. Applications using a regular
Spring Boot web starter normally already obtain them. Minimal applications must
declare the application infrastructure and logging implementation they need.
The `tomcat-embed-el` dependency supplies expression-language support; it does not
start a Tomcat server.

## Java API changes

Applications using only standard controllers and configuration properties normally
need no Undertow-specific Java changes. Applications with customizers, factory
subclasses, or explicit auto-configuration imports/exclusions must update them.

| Spring Boot 3 API | Replacement |
|---|---|
| `org.springframework.boot.web.embedded.undertow.UndertowBuilderCustomizer` | `io.undertow.springboot.UndertowBuilderCustomizer` |
| `org.springframework.boot.web.embedded.undertow.ConfigurableUndertowWebServerFactory` | `io.undertow.springboot.ConfigurableUndertowWebServerFactory` |
| `org.springframework.boot.web.embedded.undertow.UndertowServletWebServerFactory` | `io.undertow.springboot.servlet.UndertowServletWebServerFactory` |
| `org.springframework.boot.web.embedded.undertow.UndertowDeploymentInfoCustomizer` | `io.undertow.springboot.servlet.UndertowDeploymentInfoCustomizer` |
| `org.springframework.boot.web.embedded.undertow.UndertowReactiveWebServerFactory` | `io.undertow.springboot.reactive.UndertowReactiveWebServerFactory` |
| `org.springframework.boot.autoconfigure.web.ServerProperties.Undertow` / `ServerProperties.getUndertow()` | Inject `io.undertow.springboot.autoconfigure.UndertowServerProperties` directly. |

For example, a builder customizer now uses:

```java
import io.undertow.UndertowOptions;
import io.undertow.springboot.UndertowBuilderCustomizer;
import org.springframework.context.annotation.Bean;

// Inside an application configuration class
@Bean
UndertowBuilderCustomizer undertowBuilderCustomizer() {
    return builder -> builder.setServerOption(UndertowOptions.ALWAYS_SET_KEEP_ALIVE, false);
}
```

For the earlier downstream integration, replace `com.redhat.integration.undertow`
with `io.undertow.springboot`, retaining the appropriate `servlet`, `reactive`, or
`autoconfigure` subpackage. The Maven group ID is `io.undertow`, while the
integration's Java package is `io.undertow.springboot`.

Recompile customizations and review factory API usage against Spring Boot 4's web
server abstractions. This is a source migration, not a binary-compatible replacement
for libraries compiled against Spring Boot's old Undertow classes.

## Property prefix migration

As of the first release under `io.undertow`, Undertow-specific properties have moved
out of the `server.*` and `management.*` namespaces reserved by Spring Boot. The old
prefixes are deprecated, still work for supported properties, and will be removed
in the next feature release.

Only Undertow-specific properties move. Keep general Spring Boot properties such
as `server.port`, `server.ssl.*`, `server.servlet.*`, `server.compression.*`,
`server.forward-headers-strategy`, and `management.server.port` in their existing namespaces.

## Old → New Key Table

### Server properties (`server.undertow.*` → `undertow.server.*`)

| Deprecated key | New key |
|---|---|
| `server.undertow.max-http-post-size` | `undertow.server.max-http-post-size` |
| `server.undertow.buffer-size` | `undertow.server.buffer-size` |
| `server.undertow.direct-buffers` | `undertow.server.direct-buffers` |
| `server.undertow.eager-filter-init` | `undertow.server.eager-filter-init` |
| `server.undertow.max-parameters` | `undertow.server.max-parameters` |
| `server.undertow.max-headers` | `undertow.server.max-headers` |
| `server.undertow.max-cookies` | `undertow.server.max-cookies` |
| `server.undertow.decode-slash` | `undertow.server.decode-slash` |
| `server.undertow.decode-url` | `undertow.server.decode-url` |
| `server.undertow.url-charset` | `undertow.server.url-charset` |
| `server.undertow.always-set-keep-alive` | `undertow.server.always-set-keep-alive` |
| `server.undertow.no-request-timeout` | `undertow.server.no-request-timeout` |
| `server.undertow.preserve-path-on-forward` | `undertow.server.preserve-path-on-forward` |
| `server.undertow.accesslog.enabled` | `undertow.server.accesslog.enabled` |
| `server.undertow.accesslog.pattern` | `undertow.server.accesslog.pattern` |
| `server.undertow.accesslog.prefix` | `undertow.server.accesslog.prefix` |
| `server.undertow.accesslog.suffix` | `undertow.server.accesslog.suffix` |
| `server.undertow.accesslog.dir` | `undertow.server.accesslog.dir` |
| `server.undertow.accesslog.rotate` | `undertow.server.accesslog.rotate` |
| `server.undertow.threads.io` | `undertow.server.threads.io` |
| `server.undertow.threads.worker` | `undertow.server.threads.worker` |
| `server.undertow.options.server.*` | `undertow.server.options.server.*` |
| `server.undertow.options.socket.*` | `undertow.server.options.socket.*` |

### Management properties (`management.server.undertow.*` → `undertow.management.*`)

| Deprecated key | New key |
|---|---|
| `management.server.undertow.accesslog.prefix` | `undertow.management.accesslog.prefix` |

## Behavior during deprecation period

- Old keys are automatically mapped to new keys at startup via an `EnvironmentPostProcessor`.
- If both old and new keys are set for the same property, the **new key wins**.
- A consolidated `WARN` is logged at application startup listing every deprecated key
  in use and its replacement.
- Environment variable style (`SERVER_UNDERTOW_THREADS_IO`) is also supported.

Rename deployment configuration as well as application files. For example,
`SERVER_UNDERTOW_THREADS_IO` becomes `UNDERTOW_SERVER_THREADS_IO`.
Avoid mixing old and new prefixes across profiles, environment variables, command-line
arguments, and external configuration: the new key wins even if the old key comes
from a normally higher-priority source.

The compatibility mapper runs during environment preparation and reads enumerable
property sources. Do not rely on it to translate properties added later or exposed
only by a non-enumerable custom property source; use the canonical keys there.

## Example migration

**Before:**

```yaml
server:
  undertow:
    threads:
      io: 4
      worker: 32
    accesslog:
      enabled: true
```

**After:**

```yaml
undertow:
  server:
    threads:
      io: 4
      worker: 32
    accesslog:
      enabled: true
```

## Removed properties

`server.undertow.allow-encoded-slash`, deprecated in Spring Boot 3, has no matching
property in this integration. Renaming its prefix does not restore its behavior.
If the application needs encoded slashes to be decoded, explicitly evaluate
`undertow.server.decode-slash` and test the resulting paths through the front-end
proxy and application. Do not enable decoding solely to silence a migration warning.

The obsolete `server.undertow.buffers-per-region` property has no replacement and
is excluded from the dotted-key compatibility mapping. Remove it if present.

## Runtime behavior and deployment

### Reactive HTTP and WebSockets

Spring Boot 3's reactive integration used `UndertowHttpHandlerAdapter`. This
integration uses `ServletHttpHandlerAdapter` and an Undertow Servlet deployment.
Applications can continue using WebFlux, but should verify streaming responses,
server-sent events, cancellation, error handling, and requests in flight during shutdown.

Servlet WebSocket support is configured by the servlet auto-configuration. Do not
infer reactive WebSocket compatibility from servlet WebSocket support or a successful
reactive HTTP request; verify the application's reactive upgrade strategy and a
complete WebSocket handshake/message exchange on the target dependency set.

### Persisted HTTP sessions

When using `server.servlet.session.persistent=true`, container session files contain
Java-serialized instances of the integration's internal
`FileSessionPersistence.SerializablePersistentSession` class. Its package has changed
from the Spring Boot implementation and from the earlier downstream integration.
There is no translation of old serialized class names when loading these files.

Do not expect existing `.session` files to survive this migration. Plan for session
invalidation and user re-authentication, or validate a separate session conversion
before upgrading. Use a separate session-store directory for the new deployment if
old files must be retained for rollback. This applies to container file persistence;
Spring Session with Redis or JDBC requires its own compatibility review, including
the serialization of application session attributes.

### Request limits

The default maximum HTTP POST size remains 2 MiB, as in Spring Boot 3.5. If the
application sets an explicit value, migrate it to `undertow.server.max-http-post-size`.
This implementation maps negative values to Undertow's unlimited setting; retain
an explicit bound unless unlimited requests are required. Verify uploads against
both the server limit and any multipart or proxy limits.

### Server lifecycle

The servlet deployment is destroyed when the web server is destroyed, rather than
when it is stopped. Applications that explicitly stop/restart the server or depend
on `ServletContextListener.contextDestroyed` must review that timing. Verify
graceful shutdown with active requests and the application's configured shutdown
timeout; see the [changelog](CHANGELOG.md) for the lifecycle fixes in this version.

### External containers

For a WAR deployed to an external application server, the container supplies the
server. Adding this starter does not upgrade the external container or replace its
Undertow implementation. Verify Servlet 6.1 compatibility for Boot 4, follow the
container's deployment instructions, and keep embedded server dependencies in the
appropriate provided scope. Test executable WAR and external deployment modes
separately if both are used.

## Camel applications and migration assistance

Camel's `camel-undertow` component and `camel-undertow-starter` configure Camel
Undertow endpoints. This starter configures Spring Boot's embedded web server.
Identify which server handles each endpoint before changing dependencies; an
application can use both integrations.

Follow the applicable [Camel upgrade guides](https://camel.apache.org/manual/camel-4x-upgrade-guide.html)
and [Camel Upgrade Recipes documentation](https://camel.apache.org/manual/camel-upgrade-recipes-tool.html)
when upgrading Camel alongside Spring Boot. The Camel 4.19 migration recipe in the
matching downstream recipe set replaces the official Spring Boot Undertow starter
coordinate with `io.undertow:undertow-spring-boot-starter`. This does not migrate the
Undertow properties, custom Java imports, or persisted sessions described above.
Review generated dependency versions against your target BOM and complete the manual steps.

## Verifying the migration

1. Inspect the effective dependency graph. Confirm that the chosen starter version
   and Undertow EE libraries resolve, the old integration is absent, and competing
   embedded server starters have been removed. Use `mvn dependency:tree` or the
   equivalent Gradle dependency report to inspect the graph, and resolve the actual
   runtime artifacts with `mvn dependency:resolve` or a Gradle build. POM validation
   alone does not resolve dependencies.
2. Recompile customizers and confirm that Undertow starts and serves the application's
   actual endpoints. Check startup diagnostics for deprecated property warnings.
3. Verify settings in every deployment profile, including environment-variable
   overrides, TLS, forwarded headers, context paths, access logs, and request limits.
4. Exercise authentication, session cookies, uploads, encoded paths, and WebSockets
   used by the application. For WebFlux, include streaming and cancellation tests.
5. If Actuator uses a separate management port, verify its listener, TLS, access-log
   prefix, and metrics independently. Session metrics now report actual statistics;
   unsupported or unlimited metric values can be `NaN` rather than `-1`.
6. Verify shutdown, restart, and the planned session/rollback behavior. If producing
   a native image, build and exercise the actual native application as well.
