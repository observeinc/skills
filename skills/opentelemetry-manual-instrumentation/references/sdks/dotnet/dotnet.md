# .NET SDK Rule

.NET specifics for manual OTel instrumentation — apply alongside the neutral rules in
SKILL.md's Rule routing (fundamentals, traces, metrics, semantic-review,
cardinality, …). The tracing/metrics API is the BCL:
`System.Diagnostics.ActivitySource`/`Activity` (spans) and `System.Diagnostics.Metrics.Meter`
(metrics). The OpenTelemetry NuGet packages are the **SDK/exporters** that listen to those
built-in types. For zero-code coverage use **opentelemetry-auto-instrumentation**.

## Files to inspect

- project/solution files: `*.csproj`, `*.fsproj`, `*.sln`, `Directory.Packages.props`
  (central package management), `Directory.Build.props`
- target framework(s) (`<TargetFramework>net8.0</TargetFramework>`)
- entrypoints (`Program.cs` — minimal-hosting `WebApplication`/`Host` builder, or a
  console `Main`)
- run config (`Properties/launchSettings.json`, `appsettings*.json`, `Dockerfile`)
- existing OTel setup (`builder.Services.AddOpenTelemetry()`, `Sdk.CreateTracerProviderBuilder`,
  a `Telemetry`/`Diagnostics` static class defining an `ActivitySource`/`Meter`)
- existing vendor instrumentation (Datadog `Datadog.Trace`, New Relic, `Sentry`)
- existing logging (`Serilog`, `Microsoft.Extensions.Logging`) and metrics setup

These feed the environment checkpoint (SKILL.md → Workflow step 1): target framework, host
model (ASP.NET Core / worker / console), existing OTel deps, run commands.

## Dependency guidance

- **Applications** may use: `OpenTelemetry`, `OpenTelemetry.Extensions.Hosting` (the
  `AddOpenTelemetry()` DI entrypoint), `OpenTelemetry.Exporter.OpenTelemetryProtocol`
  (OTLP), the instrumentation packages
  (`OpenTelemetry.Instrumentation.AspNetCore`/`.Http`/`.SqlClient`/`.EntityFrameworkCore`),
  and `OpenTelemetry.SemanticConventions` for attribute-name constants.
- **The trace/metric API needs no package** — `ActivitySource` and `Meter` are in the BCL
  (`System.Diagnostics.DiagnosticSource`, referenced by the SDK). **A library** should emit
  via its own `ActivitySource`/`Meter` and take **no** dependency on the OpenTelemetry SDK
  packages — the host app registers a provider that listens to the library's source.
- **Inspect installed versions** before editing — the exporter/instrumentation packages and
  semconv constants shift across releases; use central package management if the repo has it.

## Environment variables

Read by `AddOtlpExporter()` / the exporter defaults:

| Variable                      | Notes                                                                                           |
| ----------------------------- | ----------------------------------------------------------------------------------------------- |
| `OTEL_SERVICE_NAME`           | Sets `service.name`; else `unknown_service`. Often set via `ConfigureResource` in code instead. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Default `http://localhost:4318`.                                                                |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Default `http/protobuf` (port 4318); `grpc` is port 4317.                                       |
| `OTEL_EXPORTER_OTLP_HEADERS`  | e.g. auth; comma-separated `k=v`.                                                               |
| `OTEL_RESOURCE_ATTRIBUTES`    | `deployment.environment.name`, `service.version`, …                                             |

With the **manual SDK** (NuGet packages), what gets exported is decided by which pipelines you
build (`WithTracing`/`WithMetrics`) and `AddOtlpExporter()`. (The separate **zero-code**
auto-instrumentation defaults `OTEL_TRACES_EXPORTER=none` and must be set to `otlp` — a
different code path; see **opentelemetry-auto-instrumentation**.)
**Protocol/port:** match the protocol to the endpoint port (`http/protobuf` ↔ 4318,
`grpc` ↔ 4317); a mismatch fails to connect. See `../../verification/validation-and-debugging.md`.

## App vs library (.NET)

- **Application/service** → owns the provider: `AddOpenTelemetry().WithTracing(...)
.WithMetrics(...)` (or `Sdk.CreateTracerProviderBuilder()`), registers every
  `ActivitySource`/`Meter` name it wants to collect, attaches OTLP + resource, owns shutdown.
- **Library/package** → declare a `static readonly ActivitySource`/`Meter` with a stable name
  and version and emit through it; **never** build a provider or add an exporter. Document the
  source name so hosts can `AddSource(...)`/`AddMeter(...)` it.

Full rule: `../../app-vs-library.md`.

## Startup and bootstrap ordering

- **Hosted apps (ASP.NET Core / worker)** — configure in `Program.cs` on the host builder
  (`builder.Services.AddOpenTelemetry()...`). The host starts/stops the provider with the
  app; no manual ordering needed. See **Bootstrap file**.
- **Non-host apps (console/CLI)** — build the provider explicitly with `Sdk.Create*Builder()`
  early in `Main`, and **dispose it before exit** to flush.

The provider must be built before the instrumented `ActivitySource`/`Meter` produce data you
care about; sources created before the provider still work (the provider attaches a listener),
but data produced before there is any listener is dropped.

## Manual tracing

```csharp
using System.Diagnostics;

public class OrderService
{
    // One ActivitySource per library/component, stable name + version. Register this
    // name via AddSource(...) in the provider or StartActivity returns null (see pitfall).
    private static readonly ActivitySource Source = new("Orders.Service", "1.0.0");
    private readonly ILogger<OrderService> _logger;
    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public async Task<Order> ProcessOrderAsync(string orderId, int itemCount, string region)
    {
        // Activity name = operation class, NOT $"process order {orderId}" (high-cardinality).
        // StartActivity returns NULL if no listener samples this source — always use ?.
        using var activity = Source.StartActivity("orders.process");
        try
        {
            // SAFE tags: bounded, operation-describing, no PII. Business dimensions under a
            // bounded namespace (app.*), documented.
            activity?.SetTag("app.order.item_count", itemCount);
            activity?.SetTag("app.order.region", region);        // bounded enum us/eu/apac
            activity?.AddEvent(new ActivityEvent("validation.completed"));
            var result = await SaveOrderAsync(orderId);
            return result;
            // On success leave status UNSET; set Ok only when logic CONFIRMS success.
        }
        catch (Exception ex)
        {
            activity?.AddException(ex);   // .NET 9+; else use RecordException from OpenTelemetry.Trace
            // ERROR needs a short message: type + reason, NOT the stack trace (log that).
            activity?.SetStatus(ActivityStatusCode.Error, $"{ex.GetType().Name}: {ex.Message}");
            _logger.LogError(ex, "order.process.failed");        // stack trace goes to the log
            throw;                                               // re-throw — do NOT swallow
        }
        // `using` disposes (ends) the activity on every path.
    }
}
```

**The null-Activity pitfall:** `StartActivity` returns `null` whenever no listener is sampling
the source (commonly a missing `AddSource` — see **SDK wiring pitfalls**), so every call must
be null-conditional (`activity?.`). Tag values may contain user input — see
`../../review/cardinality.md`.

## Manual metrics

```csharp
using System.Diagnostics.Metrics;

// One Meter per component, stable name + version. Register via AddMeter(...) in the provider.
private static readonly Meter Meter = new("Orders.Service", "1.0.0");

private static readonly Counter<long> OrdersProcessed =
    Meter.CreateCounter<long>("app.orders.processed", unit: "{order}", description: "Total orders processed");
private static readonly UpDownCounter<long> OrdersInFlight =
    Meter.CreateUpDownCounter<long>("app.orders.in_flight", unit: "{order}", description: "Orders being processed");
private static readonly Histogram<double> OrderDuration =
    Meter.CreateHistogram<double>("app.orders.duration", unit: "s", description: "Order processing duration");

// ObservableGauge: a current value sampled via a callback.
private static readonly ObservableGauge<long> QueueDepth =
    Meter.CreateObservableGauge<long>("app.orders.queue_depth",
        () => new Measurement<long>(GetQueueDepth(), new KeyValuePair<string, object?>("app.queue.name", "orders")),
        unit: "{order}");

public async Task RecordOrderAsync(string region, Func<Task> run)
{
    var baseTag = new KeyValuePair<string, object?>("app.order.region", region); // bounded
    OrdersInFlight.Add(1, baseTag);
    var sw = System.Diagnostics.Stopwatch.StartNew();
    var outcome = "ok";
    try { await run(); }
    catch { outcome = "error"; throw; }
    finally
    {
        // outcome is a bounded enum — safe as a tag.
        var tags = new TagList { baseTag, new("app.order.outcome", outcome) };
        OrdersProcessed.Add(1, tags);
        OrderDuration.Record(sw.Elapsed.TotalSeconds, tags);
        OrdersInFlight.Add(-1, baseTag);
        // ❌ NEVER add order_id / user_id / url as tags — cardinality blowup.
    }
}
```

## Context propagation

The active span is `Activity.Current`, stored in an `AsyncLocal<Activity>` — so it **flows
automatically across `await`** and into continuations on the default scheduler. The neutral
model is `../../signals/context-propagation.md`; .NET specifics:

- **`async`/`await`** preserves `Activity.Current` — no manual work in normal async code.
- **Detached work** — `Task.Run` / manual threads / `ThreadPool.QueueUserWorkItem` capture
  `Activity.Current` at the point of scheduling, but long-lived background work started
  outside the operation won't be a child. Capture the `Activity`/`ActivityContext` and start
  the child from it (`Source.StartActivity(name, ActivityKind.Internal, parentContext)`).
- **Cross-service.** Propagation uses W3C `traceparent`. ASP.NET Core and `HttpClient`
  instrumentation inject/extract for you; for raw transports use
  `DistributedContextPropagator` / OpenTelemetry's `Propagators.DefaultTextMapPropagator`.
  Don't double-propagate a path instrumentation already covers.

## Enriching auto-instrumentation (don't duplicate)

When a path is already covered (ASP.NET Core owns the HTTP `SERVER` activity, `HttpClient`
the client one), enrich the existing activity rather than wrapping it (neutral decision order
in `../../instrumentation-rules.md`):

```csharp
Activity.Current?.SetTag("app.tenant.id", tenantId);   // bounded
```

Always null-conditional — `Activity.Current` is `null` when nothing is active. The server
activity sets ERROR on 5xx but leaves a 4xx **UNSET** (see `../../signals/traces.md`).
`AddAspNetCoreInstrumentation`/`AddHttpClientInstrumentation` expose enrich callbacks
(`EnrichWithHttpRequest`, etc.) — configure them where you build the pipeline, not in
business code.

## SDK wiring pitfalls (.NET)

- **`AddSource` / `AddMeter` are mandatory.** The provider only collects `ActivitySource`/
  `Meter` names you explicitly register. A missing `AddSource("Orders.Service")` means
  `StartActivity` returns `null` and **nothing is emitted** — the single most common "no
  spans" cause. Names must match exactly.
- **Build the provider once.** In hosted apps call `AddOpenTelemetry()` once; don't also
  `Sdk.Create*Builder()` (double provider).
- **Metric flush for short-lived runs.** Console apps exit before the periodic export — call
  `meterProvider.ForceFlush()` / dispose before exit, or metrics are lost.
- **Resource from env + code.** `ConfigureResource(r => r.AddService("orders", serviceVersion: "1.0.0"))`
  merges with `OTEL_RESOURCE_ATTRIBUTES`; see `../../discovery/resolve-values.md`.

## Structured logging

Serialize exceptions into a single structured field so stack traces don't break the
one-line-per-record contract, and correlate to traces. General guidance: `../../signals/logs.md`.

- **`Microsoft.Extensions.Logging` + JSON console.** `builder.Logging.AddJsonConsole()`
  emits single-line JSON and serializes the exception into a structured field. Add
  `builder.Logging.AddOpenTelemetry(...)` to route log records through the OTLP pipeline with
  trace correlation.
- **Serilog + `Serilog.Formatting.Compact`.** `CompactJsonFormatter` serializes the exception
  (incl. stack) into a single `"x"` field. Pass the exception as the first arg
  (`Log.Error(ex, "order.failed {OrderId}", id)`), not string-concatenated.

## Auto-instrumentation coverage

For the libraries auto-instrumentation covers and the telemetry they emit, see the
**opentelemetry-auto-instrumentation** skill; enrich those spans rather than duplicating them.

## Testing

The language-neutral test policy is in `../../verification/telemetry-unit-tests.md`. Use
`OpenTelemetry.Exporter.InMemory` — **no backend, no OTLP, no network.** Register the source
under test and export activities into a list:

```csharp
using System.Diagnostics;
using OpenTelemetry;
using OpenTelemetry.Trace;
using Xunit;

public class OrderServiceTelemetryTests
{
    [Fact]
    public async Task CreatesStableSpanOnSuccess()
    {
        var exported = new List<Activity>();
        using var provider = Sdk.CreateTracerProviderBuilder()
            .AddSource("Orders.Service")           // MUST match the ActivitySource name
            .AddInMemoryExporter(exported)
            .Build();

        var service = new OrderService(NullLogger<OrderService>.Instance);
        await service.ProcessOrderAsync("o-1", 3, "us");

        provider.ForceFlush();                     // SimpleExportProcessor is used, but flush to be safe
        var span = Assert.Single(exported);
        Assert.Equal("orders.process", span.DisplayName);   // stable, low-cardinality
        Assert.Equal("us", span.GetTagItem("app.order.region"));
    }

    [Fact]
    public async Task RecordsExceptionAndErrorStatusOnFailure()
    {
        var exported = new List<Activity>();
        using var provider = Sdk.CreateTracerProviderBuilder()
            .AddSource("Orders.Service").AddInMemoryExporter(exported).Build();

        await Assert.ThrowsAnyAsync<Exception>(() =>
            new OrderService(NullLogger<OrderService>.Instance).ProcessOrderAsync("o-2", 1, "us"));

        provider.ForceFlush();
        var span = exported[0];
        Assert.Equal(ActivityStatusCode.Error, span.Status);
        Assert.Contains(span.Events, e => e.Name == "exception");
    }
}
```

**Pitfall — no `AddSource`, no spans.** The most common .NET test failure is a missing or
mismatched `AddSource("...")` (see **SDK wiring pitfalls**). Register the exact source name,
and build the provider **before** exercising the code.

### Metrics test (only if metrics were added)

```csharp
var exportedMetrics = new List<Metric>();
using var meterProvider = Sdk.CreateMeterProviderBuilder()
    .AddMeter("Orders.Service")                    // MUST match the Meter name
    .AddInMemoryExporter(exportedMetrics)
    .Build();
// exercise code, then:
meterProvider.ForceFlush();
// assert instrument name, type, unit, datapoint value/count, and BOUNDED tags.
```

### Logs test (only if logs were added)

```csharp
var exportedLogs = new List<LogRecord>();
using var loggerFactory = LoggerFactory.Create(b => b.AddOpenTelemetry(o =>
    o.AddInMemoryExporter(exportedLogs)));
// log inside an active Activity, then assert body/severity, TraceId/SpanId
// correlation, and that sensitive fields are redacted.
```

## Framework notes

Setup and general rules live above / in the neutral references; below are only the
framework-specific deltas.

### ASP.NET Core (minimal API / MVC)

`AddAspNetCoreInstrumentation()` **owns the `SERVER` activity** and sets ERROR on 5xx — add
manual activities only as **children** for domain operations, and set status manually only on
your **business-logic** activities (a 4xx is UNSET on the server activity). The host disposes
the provider on shutdown; no manual flush needed. Instrument service-layer operations, not
every endpoint (HTTP is already covered).

### Worker services / `BackgroundService`

Same DI registration as ASP.NET Core; the generic host manages the provider lifecycle. For
message consumers, producer→consumer **context propagation across the broker** is the concern;
batch consumers use **links**. See `../../signals/context-propagation.md`.

### Console / non-host apps

No host to manage the provider — build it with `Sdk.Create*Builder()`, keep it in a `using`
(or dispose explicitly), and dispose **before** exit so buffered telemetry flushes. Short-lived
tools should `ForceFlush()` before returning.

## Bootstrap file (copy-ready, applications only)

Hardcodes **no** endpoint/token/header — delivery is via standard `OTEL_*` env vars read by
`AddOtlpExporter()`. A **library** must not use this.

**Hosted app (ASP.NET Core / worker), `Program.cs`:**

```csharp
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(               // merges with OTEL_RESOURCE_ATTRIBUTES
        serviceName: "orders-service", serviceVersion: "1.0.0"))
    .WithTracing(t => t
        .AddSource("Orders.Service")                    // your ActivitySource name(s)
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())                             // reads OTEL_EXPORTER_OTLP_* from env
    .WithMetrics(m => m
        .AddMeter("Orders.Service")                     // your Meter name(s)
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter());

// Route ILogger records through OTLP with trace correlation:
builder.Logging.AddOpenTelemetry(o => { o.IncludeScopes = true; o.AddOtlpExporter(); });

var app = builder.Build();
// The host builds + disposes the providers with the app — no manual shutdown needed.
```

**Non-host (console) app:**

```csharp
using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .ConfigureResource(r => r.AddService("orders-cli", serviceVersion: "1.0.0"))
    .AddSource("Orders.Service")
    .AddOtlpExporter()
    .Build();
// ... run work ...
tracerProvider.ForceFlush();   // flush before exit; `using` disposes (also flushes)
```

Set `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` if you point the endpoint at 4317 rather than the
`http/protobuf` (4318) default.
