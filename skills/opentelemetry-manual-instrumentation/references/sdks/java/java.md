# Java SDK Rule

Java specifics for manual OTel instrumentation — apply alongside the neutral rules in
SKILL.md's Rule routing (fundamentals, traces, metrics, semantic-review,
cardinality, …). Java's most common deployment is the **javaagent** (zero-code —
see **opentelemetry-auto-instrumentation**); this file is the **manual** API/SDK you write,
whether it runs _alongside_ the agent (enrich its spans) or _standalone_ (you build the SDK).

## Files to inspect

- build files: `pom.xml` (Maven) or `build.gradle`(`.kts`) + `settings.gradle` (Gradle),
  and the lock/versions files; a `gradle/libs.versions.toml` version catalog
- multi-module layout (parent POM / included builds)
- entrypoints (`public static void main`, Spring `@SpringBootApplication`)
- run/build commands (`Dockerfile`, `JAVA_TOOL_OPTIONS`, `-javaagent:` flags,
  `application.yaml`/`application.properties`, `bootRun` config)
- existing OTel setup (a `Telemetry`/`Otel` config class, `OpenTelemetrySdk.builder()`,
  `AutoConfiguredOpenTelemetrySdk`, or an attached `opentelemetry-javaagent.jar`)
- existing vendor instrumentation (Datadog `dd-java-agent`, New Relic, `sentry`)
- existing logging (`logback.xml`, `log4j2.xml`, `slf4j`) and metrics (Micrometer)

These feed the environment checkpoint (SKILL.md → Workflow step 1): JDK version, build tool,
framework, whether the javaagent is attached, existing OTel deps, run commands.

## Dependency guidance

- **Applications** may use: `io.opentelemetry:opentelemetry-api`,
  `-sdk`, `-exporter-otlp`, the semconv artifact
  (`io.opentelemetry.semconv:opentelemetry-semconv`), and optionally
  `-sdk-extension-autoconfigure` (env-driven SDK). **Import the BOM**
  (`io.opentelemetry:opentelemetry-bom`) and omit per-artifact versions so the API/SDK/
  exporter stay mutually compatible.
- **Libraries** should depend **only** on `opentelemetry-api` — never the SDK or an
  exporter. Get a tracer via `GlobalOpenTelemetry.getTracer("stable.scope", "x.y.z")`, or
  accept an `OpenTelemetry` instance; the host app owns the SDK.
- **Agent relationship.** If the javaagent is attached, it installs the SDK and a global
  `OpenTelemetry` — your manual code should call `GlobalOpenTelemetry.get*()` and **not**
  build its own `OpenTelemetrySdk` (that would double-init). Only build the SDK yourself in
  a standalone (no-agent) app.
- **Inspect installed versions** before editing — semconv constant names and the metrics/
  logs APIs shift across majors.

## Environment variables

Read by the javaagent, and by `AutoConfiguredOpenTelemetrySdk` if you use it:

| Variable                                                                | Notes                                                                                     |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `OTEL_SERVICE_NAME`                                                     | Required; else `unknown_service:java`.                                                    |
| `OTEL_TRACES_EXPORTER` / `OTEL_METRICS_EXPORTER` / `OTEL_LOGS_EXPORTER` | Default `otlp` under the agent/autoconfigure (all three). `console`/`none` for local/off. |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                           | Default `http://localhost:4318`.                                                          |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                           | Default `http/protobuf` (port 4318); `grpc` is port 4317.                                 |
| `OTEL_RESOURCE_ATTRIBUTES`                                              | `deployment.environment.name`, `service.version`, …                                       |

Every var has a JVM system-property twin (`-Dotel.service.name=…`,
`-Dotel.exporter.otlp.endpoint=…`). **Protocol/port:** match the protocol to the endpoint port
(`http/protobuf` ↔ 4318, `grpc` ↔ 4317); a mismatch fails to connect. See
`../../verification/validation-and-debugging.md`. A **hand-built** `OpenTelemetrySdk` does not read these vars
unless you wire autoconfigure or read them yourself.

## App vs library (Java)

- **Application/service** → may build the SDK (or let the agent/autoconfigure build it),
  attach OTLP exporters + resource, set it global (`GlobalOpenTelemetry.set(...)` or agent),
  and own shutdown.
- **Library/package** → import `opentelemetry-api` only; acquire tracers/meters via
  `GlobalOpenTelemetry` with a stable scope; **never** build an `OpenTelemetrySdk`, an
  exporter, or call `GlobalOpenTelemetry.set(...)`.

Full rule: `../../app-vs-library.md`.

## Startup and bootstrap ordering

- **Agent** (`java -javaagent:opentelemetry-javaagent.jar -jar app.jar`, or via
  `JAVA_TOOL_OPTIONS`) — installs the SDK + auto-instrumentation before your `main`. Your
  manual code uses `GlobalOpenTelemetry`; do **not** also build an SDK.
- **Programmatic** — build/autoconfigure the SDK **once, first thing in `main`**, before
  starting the server or touching DB clients, and set it global. See **Bootstrap file**.

`JAVA_TOOL_OPTIONS="-javaagent:…"` applies the agent to **every** JVM in that shell
(including Maven/Gradle) — scope it to the app's launch command in production.

## Manual tracing

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

public class OrderService {

    // Acquire a tracer ONCE with a stable scope name + version.
    private static final Tracer TRACER =
        GlobalOpenTelemetry.getTracer("orders.service", "1.0.0");

    public Order processOrder(String orderId, int itemCount, String region) {
        // Span name = operation class, NOT "process order " + orderId (high-cardinality).
        Span span = TRACER.spanBuilder("orders.process").startSpan();
        // makeCurrent() puts the span in context so children attach; the Scope MUST be
        // closed on the SAME thread (try-with-resources). Forgetting it leaks context.
        try (Scope scope = span.makeCurrent()) {
            // SAFE attributes: bounded, operation-describing, no PII. Business dimensions
            // under a bounded namespace (app.*), documented.
            span.setAttribute("app.order.item_count", itemCount);
            span.setAttribute("app.order.region", region);   // bounded enum us/eu/apac
            span.addEvent("validation.completed");            // notable moment, no extra span
            return saveOrder(orderId);
            // On success leave status UNSET; set OK only when logic CONFIRMS success.
        } catch (Exception e) {
            span.recordException(e);
            // ERROR needs a short message: type + reason, NOT the stack trace (log that).
            span.setStatus(StatusCode.ERROR, e.getClass().getSimpleName() + ": " + e.getMessage());
            throw e;                                          // re-throw — do NOT swallow
        } finally {
            span.end();                                       // ALWAYS end, on every path
        }
    }
}
```

**The two independent lifecycles are the #1 Java pitfall:** `Scope` (from `makeCurrent()`)
controls what "current" means and must be `close()`d on the same thread; `span.end()` records
the span. `try (Scope …)` closes the scope; `finally { span.end(); }` ends the span. Message
text may contain user input — see `../../review/cardinality.md`.

## Manual metrics

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.*;

// Acquire a meter ONCE; create each instrument ONCE (static), then reuse.
static final Meter METER = GlobalOpenTelemetry.getMeter("orders.service");

static final LongCounter ordersProcessed = METER.counterBuilder("app.orders.processed")
    .setUnit("{order}").setDescription("Total orders processed").build();
static final LongUpDownCounter ordersInFlight = METER.upDownCounterBuilder("app.orders.in_flight")
    .setUnit("{order}").setDescription("Orders being processed").build();
static final DoubleHistogram orderDuration = METER.histogramBuilder("app.orders.duration")
    .setUnit("s").setDescription("Order processing duration").build();

// ObservableGauge: a current value sampled via a callback (registered once).
static final ObservableDoubleGauge queueDepth = METER.gaugeBuilder("app.orders.queue_depth")
    .setUnit("{order}")
    .buildWithCallback(m -> m.record(getQueueDepth(),
        Attributes.builder().put("app.queue.name", "orders").build()));

void recordOrder(String region, Runnable run) {
    Attributes base = Attributes.builder().put("app.order.region", region).build(); // bounded
    ordersInFlight.add(1, base);
    long startNanos = System.nanoTime();
    String outcome = "ok";
    try {
        run();
    } catch (RuntimeException e) {
        outcome = "error";
        throw e;
    } finally {
        Attributes labels = Attributes.builder()
            .put("app.order.region", region).put("app.order.outcome", outcome).build(); // bounded enum
        ordersProcessed.add(1, labels);
        orderDuration.record((System.nanoTime() - startNanos) / 1e9, labels);
        ordersInFlight.add(-1, base);
        // ❌ NEVER put order_id / user_id / url as attributes — cardinality blowup.
    }
}
```

## Context propagation

The active span lives in an `io.opentelemetry.context.Context`, made current via a `Scope`
(thread-local). The neutral model is `../../signals/context-propagation.md`; Java specifics:

- **Scope discipline.** Every `makeCurrent()` returns a `Scope` you must `close()` on the
  **same thread**, in reverse order (try-with-resources). A leaked scope corrupts the current
  context for later work on that thread.
- **Executors / thread pools.** Context is thread-local and does **not** cross to a pool
  thread. Wrap the executor with `io.opentelemetry.context.Context.taskWrapping(executor)`,
  or capture `Context current = Context.current()` and re-activate inside the task
  (`try (Scope s = current.makeCurrent()) { … }`).
- **`CompletableFuture` / reactive.** Callbacks may run on other threads — propagate context
  the same way (wrap or capture/re-activate).
- **Cross-service.** The global propagators (`W3CTraceContextPropagator`, baggage) inject/
  extract via a `TextMapPropagator` with a getter/setter over headers. HTTP/gRPC
  instrumentation (agent or library) does this for you — don't double-propagate.

## Enriching auto-instrumentation (don't duplicate)

When a path is already covered (the agent owns the HTTP `SERVER` span, JDBC spans, etc.),
enrich the existing span instead of wrapping it (neutral decision order in
`../../instrumentation-rules.md`):

```java
Span.current().setAttribute("app.tenant.id", tenantId);   // bounded
```

`Span.current()` returns a **non-recording** span when none is active, and
`setAttribute`/`setStatus` on it are no-ops — so **no null check is needed**. The server span
leaves the error _message_ for your code to fill (see `../../signals/traces.md`).

## SDK wiring pitfalls (Java)

- **Agent + manual SDK = double init.** If the javaagent is attached, do **not** build your
  own `OpenTelemetrySdk` — use `GlobalOpenTelemetry`. Building both gives duplicate/again
  exported telemetry.
- **`GlobalOpenTelemetry.set(...)` is once-only.** A second call throws. Find existing setup;
  don't re-register.
- **Metric reader interval for short-lived runs.** The periodic reader defaults to ~60s; a
  CLI/job exits before the first flush. Shorten the interval (or call
  `SdkMeterProvider.forceFlush()`), and `close()` the SDK before exit.
- **Resource from env.** Autoconfigure merges `OTEL_RESOURCE_ATTRIBUTES`; a hand-built
  `Resource` should merge `Resource.getDefault()` + env, then code-set attributes on top —
  see `../../discovery/resolve-values.md`.

## Structured logging

Serialize exceptions into a single structured field so stack traces don't break the
one-line-per-record contract, and correlate to traces. General guidance: `../../signals/logs.md`.

- **Logback + `logstash-logback-encoder`.** `LogstashEncoder` emits single-line JSON and
  serializes exceptions passed to `logger.error("msg", ex)` into a `stack_trace` field.
- **Log4j2 + `JsonTemplateLayout`** (e.g. `EcsLayout.json`) — single-line JSON with the stack
  trace in a structured field. **Avoid `PatternLayout`** in production: it prints multi-line
  stack traces that break collectors.

Trace correlation: the agent's MDC instrumentation injects `trace_id`/`span_id` into MDC;
reference them in the encoder pattern, or use the OTel appender/log bridge.

## Auto-instrumentation coverage

For the libraries auto-instrumentation covers and the telemetry they emit, see the
**opentelemetry-auto-instrumentation** skill; enrich those spans rather than duplicating them.

## Testing

The language-neutral test policy is in `../../verification/telemetry-unit-tests.md`. Use
`io.opentelemetry:opentelemetry-sdk-testing` — **no backend, no OTLP, no network.** The
`OpenTelemetryExtension` (JUnit 5) registers an in-memory SDK and resets it between tests:

```java
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.sdk.testing.junit5.OpenTelemetryExtension;
import io.opentelemetry.sdk.trace.data.SpanData;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import static org.assertj.core.api.Assertions.assertThat;

class OrderServiceTest {

    @RegisterExtension
    static final OpenTelemetryExtension otel = OpenTelemetryExtension.create();

    // Construct the service with otel.getOpenTelemetry() so it resolves the TEST tracer,
    // OR (if it uses GlobalOpenTelemetry) ensure the extension is registered first.
    final OrderService service = new OrderService(otel.getOpenTelemetry());

    @Test
    void createsStableSpanOnSuccess() {
        service.processOrder("o-1", 3, "us");
        assertThat(otel.getSpans())
            .singleElement()
            .satisfies(s -> {
                assertThat(s.getName()).isEqualTo("orders.process");   // stable, low-cardinality
                assertThat(s.getAttributes().asMap())
                    .containsEntry(io.opentelemetry.api.common.AttributeKey.stringKey("app.order.region"), "us");
            });
    }

    @Test
    void recordsExceptionAndErrorStatusOnFailure() {
        assertThatThrownBy(() -> service.processOrder("o-2", 1, "us"));  // force failure
        SpanData s = otel.getSpans().get(0);
        assertThat(s.getStatus().getStatusCode()).isEqualTo(StatusCode.ERROR);
        assertThat(s.getEvents()).anyMatch(e -> e.getName().equals("exception"));
    }
}
```

**Pitfall — resolve the test SDK, not the global.** If the code under test caches a tracer
from `GlobalOpenTelemetry` at class-load time before the extension registers, the exporter
sees **0 spans**. Prefer injecting the `OpenTelemetry` instance (constructor param) so the
test passes `otel.getOpenTelemetry()`, or acquire the tracer lazily. The extension resets
captured spans per test automatically.

### Metrics test (only if metrics were added)

`OpenTelemetryExtension` also captures metrics: exercise the code, then
`otel.getMetrics()` and assert instrument name, type, unit, datapoint value/count, and
BOUNDED attributes. (For a hand-built SDK, use `InMemoryMetricReader` +
`reader.collectAllMetrics()`.)

### Logs test (only if logs were added)

The extension captures logs too (`otel.getLogRecords()` in recent `sdk-testing`
versions; otherwise wire an `InMemoryLogRecordExporter`). Emit a log inside an active
span, then assert body/severity, that trace/span IDs match the span, and that sensitive
fields are redacted.

## Framework notes

Setup and general rules live above / in the neutral references; below are only the
framework-specific deltas.

### Spring Boot

Most Spring apps run under the **javaagent** (zero-code) — your job is enriching its spans
(`Span.current()`), plus manual spans for domain operations. For agentless setups, the
`opentelemetry-spring-boot-starter` autoconfigures the SDK from `application.properties`/env.
Instrument **service-layer** operations, not every controller (the agent already covers
HTTP). `@WithSpan` (from `opentelemetry-instrumentation-annotations`) creates a span around a
method when the agent/starter is present — handy, but prefer explicit spans where you need
attributes/status control.

### Executors, `@Async`, reactive

Context is thread-local; work dispatched to a pool loses it. Wrap executors
(`Context.taskWrapping(...)`) or capture/re-activate the context (see **Context
propagation**). Reactor/WebFlux needs its own context-propagation wiring.

### Workers / background jobs

Producer→consumer **context propagation across the broker** is the core concern (Kafka/JMS/
RabbitMQ instrumentation injects/extracts); a worker is its **own** entrypoint with its own
shutdown; batch consumers use **links**. See `../../signals/context-propagation.md`.

## Bootstrap file (copy-ready, applications only)

For a **standalone** (no-agent) app. It hardcodes **no** endpoint/token/header — delivery is
via standard `OTEL_*` env vars. A **library** must not use this, and if the javaagent is
attached, do **not** run this (double init).

**Preferred — autoconfigure** (reads `OTEL_*` for you; add
`opentelemetry-sdk-extension-autoconfigure`):

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;

// Once, first thing in main(): builds providers + OTLP exporters + resource from OTEL_* env,
// sets GlobalOpenTelemetry, and registers a JVM shutdown hook that flushes on exit.
OpenTelemetry openTelemetry =
    AutoConfiguredOpenTelemetrySdk.initialize().getOpenTelemetrySdk();
```

**Manual builder** (full control; you own shutdown):

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;

Resource resource = Resource.getDefault();  // merge env/code attrs on top — see ../../discovery/resolve-values.md
SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(
        OtlpGrpcSpanExporter.builder().build()).build())   // reads OTEL_EXPORTER_OTLP_* if unset
    .setResource(resource)
    .build();
OpenTelemetrySdk sdk = OpenTelemetrySdk.builder()
    .setTracerProvider(tracerProvider)
    // .setMeterProvider(...) with SdkMeterProvider + PeriodicMetricReader + OtlpGrpcMetricExporter
    .buildAndRegisterGlobal();

// Flush + release on exit, or the final batch is lost.
Runtime.getRuntime().addShutdownHook(new Thread(sdk::close));
```

Use `OtlpHttpSpanExporter` (and set `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`, port 4318) if
the receiver is HTTP. Logs use `SdkLoggerProvider` + a log-record processor + OTLP log
exporter — see `../../signals/logs.md`.
