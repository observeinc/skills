# C++ SDK Rule

C++ specifics for manual OTel instrumentation — apply alongside the neutral rules in
SKILL.md's Rule routing (fundamentals, traces, metrics, semantic-review,
cardinality, …). `opentelemetry-cpp` requires you to construct providers and
exporters in code, keep them alive for the process lifetime, and manage
context/shutdown explicitly. The API deliberately uses `nostd::` ABI-stable types so
API and SDK can be built/linked separately.

## Files to inspect

- build system: `CMakeLists.txt` (`find_package(opentelemetry-cpp …)`, target links) or
  Bazel (`BUILD`, `WORKSPACE`/`MODULE.bazel`, `@io_opentelemetry_cpp//...`)
- package manager manifests: `vcpkg.json`, `conanfile.txt`/`.py`
- which OTLP transport is compiled in (`WITH_OTLP_GRPC` / `WITH_OTLP_HTTP` cache vars)
- entrypoints (`int main`) and any daemon/service init
- C++ standard in use (`CMAKE_CXX_STANDARD`; opentelemetry-cpp requires **C++14+**,
  and some features/exporters require C++17)
- existing OTel setup (files calling `Provider::SetTracerProvider` / `…Factory::Create`)
- existing logging (`spdlog`, `glog`, custom) and metrics/stats
- threading model (thread pools, event loops) — drives context-propagation choices
- vendored vs system opentelemetry-cpp, and its **version** (API differs across majors)

These feed the environment checkpoint (SKILL.md → Workflow step 1): build system,
compiled-in exporters, C++ standard, existing OTel/logging, threading model.

## Dependency & build guidance

- **Applications** link the API **and** SDK **and** an exporter:
  `opentelemetry-cpp::api`, `::trace` / `::metrics` / `::logs` (SDK),
  `::otlp_grpc_exporter` or `::otlp_http_exporter`, `::resources`. In CMake, link the
  targets exposed by `find_package(opentelemetry-cpp CONFIG REQUIRED)`.
- **Libraries** link **only** `opentelemetry-cpp::api`. With no SDK installed the API
  is a no-op, so a library emits nothing until an application installs an SDK provider.
  Never call `…Factory::Create` or `SetTracerProvider` from a library.
- **Exporters are compile-time.** OTLP gRPC/HTTP exist only if opentelemetry-cpp was
  **built** with `-DWITH_OTLP_GRPC=ON` / `-DWITH_OTLP_HTTP=ON`. Verify how the
  dependency was built (vcpkg feature flags / conan options / your CMake cache) before
  writing exporter code — a missing flag surfaces as a link error at build time.
- **ABI note.** Because the API crosses the API/SDK boundary with `nostd::string_view`,
  `nostd::shared_ptr`, `nostd::span`, build the API, SDK, and your code with a
  **matching C++ standard and compiler/stdlib** to avoid ABI mismatches.

## Environment variables

The OTLP **exporters** read these when constructed with default options
(`OtlpGrpcExporterFactory::Create()` / `OtlpHttpExporterFactory::Create()`):

| Variable                                                           | Notes                                                                       |
| ------------------------------------------------------------------ | --------------------------------------------------------------------------- |
| `OTEL_SERVICE_NAME`                                                | Picked up by the SDK resource (`OTELResourceDetector`) as `service.name`.   |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                      | gRPC default `http://localhost:4317`; HTTP default `http://localhost:4318`. |
| `OTEL_EXPORTER_OTLP_HEADERS`                                       | Comma-separated `k=v` → gRPC metadata / HTTP headers (auth).                |
| `OTEL_EXPORTER_OTLP_TIMEOUT`                                       | Export deadline (e.g. `10s`).                                               |
| `OTEL_EXPORTER_OTLP_SSL_ENABLE` / `OTEL_EXPORTER_OTLP_CERTIFICATE` | TLS for the gRPC exporter.                                                  |
| `OTEL_RESOURCE_ATTRIBUTES`                                         | Merged into the resource by the SDK's env detector.                         |

**C++-specific pitfall — the exporter, processor, and provider are all constructed in
code.** You pick the exporter by calling its factory; `OTEL_TRACES_EXPORTER` /
`OTEL_METRICS_EXPORTER` are not consulted. Env vars only tune the
already-chosen OTLP exporter's endpoint/headers/TLS. Choose gRPC (4317) vs HTTP (4318)
by which factory you call — match `OTEL_EXPORTER_OTLP_ENDPOINT`'s port. See
`../../verification/validation-and-debugging.md`.

## App vs library (C++)

- **Application/service** → constructs `TracerProvider`/`MeterProvider`/
  `LoggerProvider` (via the `…Factory::Create` helpers), registers each with
  `trace::Provider::SetTracerProvider(...)` etc., and owns flush + teardown.
- **Library/component** → acquires a tracer lazily via
  `trace::Provider::GetTracerProvider()->GetTracer("scope", "version")`; links only
  `::api`; **never** creates providers/exporters or calls `Set*Provider`.

Full rule: `../../app-vs-library.md`.

## Startup and shutdown

Initialize once, early in `main`, and keep providers alive for the whole process — the
active provider is a global singleton. The hazards are provider lifetime and flush on exit:

- **Keep the provider alive.** `Set*Provider` stores a `nostd::shared_ptr`; if your
  local goes out of scope and the last reference drops, spans silently stop exporting.
- **Flush and tear down cleanly on exit.** Before returning from `main`, `ForceFlush`
  and shut down each provider so batched telemetry is sent, then reset the global
  provider to a no-op so no span is created during static destruction. The
  `TracerProviderFactory` returns a `unique_ptr`; wrap it in `shared_ptr` for
  `SetTracerProvider`, and cast to the SDK type to call `ForceFlush`/`Shutdown` (see
  the **Bootstrap file**).
- **Static-destruction order.** Don't create spans/log records in destructors of
  static/global objects — the provider may already be gone. End every span before exit.

## Manual tracing

```cpp
#include <opentelemetry/trace/provider.h>
#include <opentelemetry/trace/scope.h>
#include <opentelemetry/trace/span_startoptions.h>
#include <string>

namespace trace_api = opentelemetry::trace;

// Acquire a tracer with a stable instrumentation-scope name + version.
// GetTracer is cheap to call, but cache it (e.g. a function-local static).
opentelemetry::nostd::shared_ptr<trace_api::Tracer> GetTracer() {
  auto provider = trace_api::Provider::GetTracerProvider();
  return provider->GetTracer("acme.shop.orders", "1.0.0");
}

struct ProcessOrderInput {
  std::string order_id;  // application logic only; NEVER a span name or metric label
  int item_count;
  std::string region;    // bounded enum ("us"/"eu"/"apac") — safe as an attribute
};

bool ProcessOrder(const ProcessOrderInput &in) {
  auto tracer = GetTracer();

  trace_api::StartSpanOptions opts;
  opts.kind = trace_api::SpanKind::kInternal; // or kServer/kClient/kProducer/kConsumer

  // Span name = operation class, NOT ("process order " + in.order_id).
  auto span = tracer->StartSpan("orders.process", opts);
  auto scope = tracer->WithActiveSpan(span); // makes this span the parent of nested spans

  // SAFE attributes: bounded, operation-describing, no PII. Custom business
  // dimensions live under a bounded namespace (app.*).
  span->SetAttribute("app.order.item_count", in.item_count);
  span->SetAttribute("app.order.region", in.region);
  span->AddEvent("validation.completed"); // a notable moment inside the op (no extra span)

  if (!ReserveInventory(in)) {
    // Set ERROR status with a short message (type + reason). Messages may contain
    // user input — see ../../review/cardinality.md.
    span->SetStatus(trace_api::StatusCode::kError, "reserve inventory failed");
    span->End(); // end on EVERY path (no exceptions/RAII span-ender here)
    return false;
  }

  // On success leave status Unset; set kOk only when logic CONFIRMS success.
  span->End();
  return true;
}
```

Notes: the `Scope`/RAII object manages only active-span context; call `span->End()`
yourself on every path. If the function can throw,
wrap the body so `span->End()` (and status) still run (a small RAII ender or a
try/catch that records the exception via `span->AddEvent("exception", {...})` and
`SetStatus(kError, …)` before rethrowing).

## Manual metrics

```cpp
#include <opentelemetry/metrics/provider.h>
#include <opentelemetry/common/key_value_iterable_view.h>
#include <map>

namespace metrics_api = opentelemetry::metrics;

opentelemetry::nostd::shared_ptr<metrics_api::Meter> GetMeter() {
  auto provider = metrics_api::Provider::GetMeterProvider();
  return provider->GetMeter("acme.shop.orders", "1.0.0");
}

void RecordOrder(const std::string &region, const std::string &outcome, double seconds) {
  auto meter = GetMeter();

  // Create instruments ONCE and reuse (cache as statics/members) — construction is
  // not free and repeated creation fragments aggregation. Name/description/unit:
  static auto orders_processed =
      meter->CreateUInt64Counter("app.orders.processed", "Total orders processed", "{order}");
  static auto orders_in_flight =
      meter->CreateInt64UpDownCounter("app.orders.in_flight", "Orders in progress", "{order}");
  static auto order_duration =
      meter->CreateDoubleHistogram("app.orders.duration", "Order processing duration", "s");

  // BOUNDED labels only — region + outcome are enums. NEVER order_id/user_id/url.
  std::map<std::string, std::string> labels = {{"app.order.region", region},
                                               {"app.order.outcome", outcome}};
  auto labelkv = opentelemetry::common::KeyValueIterableView<decltype(labels)>{labels};

  orders_processed->Add(1, labelkv);
  order_duration->Record(seconds, labelkv, opentelemetry::context::Context{});
}
```

**Observable (async) gauge** — sample a current value via a callback; keep the
returned instrument alive for the collection lifetime, and register/unregister the
callback:

```cpp
static auto queue_depth = meter->CreateInt64ObservableGauge(
    "app.orders.queue_depth", "Current order-queue depth", "{order}");
queue_depth->AddCallback(
    [](metrics_api::ObserverResult result, void * /*state*/) {
      auto obs = opentelemetry::nostd::get<
          opentelemetry::nostd::shared_ptr<metrics_api::ObserverResultT<int64_t>>>(result);
      obs->Observe(CurrentQueueDepth(), {{"app.queue.name", "orders"}});
    },
    /*state=*/nullptr);
```

Histogram bucket layout and non-default aggregations are configured with **Views** on
the `MeterProvider` (`AddView(InstrumentSelector, MeterSelector, View)`); the default
view maps each instrument to its natural aggregation, so add Views only to customize.

## Context propagation

There is no ambient span unless you make one active. The neutral model is in
`../../signals/context-propagation.md`; C++ specifics:

- **In-process active span.** `tracer->WithActiveSpan(span)` (or `trace::Scope{span}`)
  pushes the span onto the thread-local `RuntimeContext`; nested `StartSpan` calls
  parent to it until the `Scope` object is destroyed. Retrieve the active span with
  `trace::GetSpan(context::RuntimeContext::GetCurrent())`.
- **Threads / thread pools.** `RuntimeContext` is **thread-local** — it does **not**
  follow work onto another thread. Capture the context on the submitting thread
  (`auto ctx = context::RuntimeContext::GetCurrent();`) and re-attach it on the worker
  with `context::RuntimeContext::Attach(ctx)` (keep the returned token to `Detach`), or
  start the worker's span with the captured span as an explicit parent
  (`opts.parent = span->GetContext();`). For fire-and-forget background work that
  outlives the request, prefer a **link** to the originating span rather than a parent.
- **Cross-service.** Register a global propagator once
  (`context::propagation::GlobalTextMapPropagator::SetGlobalPropagator(...)` with
  `trace::propagation::HttpTraceContext`), then `Inject` the current context into an
  outgoing carrier and `Extract` an incoming carrier into a context on the server. A
  carrier is any type implementing `TextMapCarrier` (get/set over your header map):

```cpp
auto propagator = context::propagation::GlobalTextMapPropagator::GetGlobalPropagator();
// client: inject
MyHeaderCarrier carrier{outgoing_headers};
propagator->Inject(carrier, context::RuntimeContext::GetCurrent());
// server: extract, then start the server span under the remote parent
auto ctx = propagator->Extract(incoming_carrier, context::RuntimeContext::GetCurrent());
opts.parent = trace::GetSpan(ctx)->GetContext();
```

## Enriching existing spans (don't duplicate)

opentelemetry-cpp has no auto-instrumentation agent, so most spans are ones you create.
Where a library provides its **own** built-in instrumentation (e.g. a gRPC interceptor,
an HTTP server module from contrib), don't wrap it in a redundant manual span — enrich
the active span it created via
`trace::GetSpan(context::RuntimeContext::GetCurrent())->SetAttribute(...)` /
`SetStatus(...)`. `GetSpan` returns a no-op span when none is active, so the calls are
safe without a null check. Neutral decision order (enrich vs child span) is in
`../../instrumentation-rules.md`.

## SDK wiring pitfalls (C++)

- **Nothing exports until you build it in code** — exporter + processor + provider,
  then `SetTracerProvider`. No env var wires OTLP. #1 "no telemetry, no error" cause.
  Keep the provider alive for the process (see **Startup and shutdown**).
- **Batch vs simple processor.** `BatchSpanProcessor` in production;
  `SimpleSpanProcessor` (synchronous) for tests/local. Short-lived programs must
  `ForceFlush` before exit or the last batch is lost.
- **Fast metric export for short runs.** `PeriodicExportingMetricReader`'s default
  interval is far longer than a CLI/job lives — set `PeriodicExportingMetricReaderOptions
.export_interval_millis` low (and `export_timeout_millis` ≤ it), or `ForceFlush` the
  meter provider before exit.
- **Resource merge.** Build the resource so `OTEL_RESOURCE_ATTRIBUTES`/`OTEL_SERVICE_NAME`
  (via the env detector) survive; add code-set attributes on top, don't clobber. See
  `../../discovery/resolve-values.md`.

## Database spans

opentelemetry-cpp ships no database auto-instrumentation, so query spans are ones you
create around the call. **Do not** put raw SQL with inlined literals in a span attribute.
Prefer the parameterized statement text as `db.statement` (the attribute Observe APM
reads; OTel's newer semconv renames it `db.query.text`) and keep parameter values out of
telemetry — they routinely hold PII/secrets. Follow the semantic conventions in
`../../review/semantic-conventions.md` for `db.system.name`, `db.namespace`,
`db.operation.name`, etc.

## Structured logging

Use the OTel **logs** signal (logs SDK + OTLP log exporter) so records carry trace
context and go through the same pipeline. General guidance: `../../signals/logs.md`.

- Acquire a logger from the global `LoggerProvider`
  (`logs::Provider::GetLoggerProvider()->GetLogger("scope", "1.0.0")`) and emit with a
  severity (`kInfo`, `kError`, …).
- **Trace correlation** is automatic when a log record is emitted **inside an active
  span scope** (`trace::Scope{span}`) — the record picks up `trace_id`/`span_id`/flags;
  or pass them explicitly (`span->GetContext()`).
- If the app already uses `spdlog`/`glog`, bridge it (a sink/handler that forwards to
  the OTel logger) instead of replacing it, and keep one structured record per line so
  a stack trace doesn't break the log parser (serialize the exception into a single
  `exception.stacktrace` field).

## Instrumentation libraries

Reusable instrumentations live in
[`opentelemetry-cpp-contrib`](https://github.com/open-telemetry/opentelemetry-cpp-contrib)
(e.g. Apache httpd / nginx modules, some client libraries) and are built/linked
individually; otherwise you instrument manually with the tracer/meter above. gRPC is
commonly instrumented via a custom interceptor around `opentelemetry-cpp`. Runtime/host
metrics come from separate collectors, not the SDK. Install/link only what you use.

## Testing

The language-neutral test policy (required gate, what to assert, forbidden practices) is
in `../../verification/telemetry-unit-tests.md`. Below is the C++ harness using the
SDK's `InMemorySpanExporter` with a `SimpleSpanProcessor` (synchronous — spans are
queryable immediately) — **no backend, no OTLP, no network.** Example uses GoogleTest.

```cpp
#include <gtest/gtest.h>
#include <opentelemetry/exporters/memory/in_memory_span_exporter_factory.h>
#include <opentelemetry/exporters/memory/in_memory_span_data.h>
#include <opentelemetry/sdk/trace/simple_processor_factory.h>
#include <opentelemetry/sdk/trace/tracer_provider_factory.h>
#include <opentelemetry/trace/provider.h>

namespace sdktrace = opentelemetry::sdk::trace;
namespace trace_api = opentelemetry::trace;
namespace memory = opentelemetry::exporter::memory;

class OrdersTelemetryTest : public ::testing::Test {
 protected:
  std::shared_ptr<memory::InMemorySpanData> span_data_;

  void SetUp() override {
    // Hold the in-memory data handle so we can read spans after they end.
    auto exporter = memory::InMemorySpanExporterFactory::Create(&span_data_);
    auto processor = sdktrace::SimpleSpanProcessorFactory::Create(std::move(exporter));
    std::shared_ptr<trace_api::TracerProvider> provider =
        sdktrace::TracerProviderFactory::Create(std::move(processor));
    // Register BEFORE the code under test resolves its tracer (Pitfall 1).
    trace_api::Provider::SetTracerProvider(provider);
  }

  void TearDown() override {
    // Reset to a no-op provider so tests don't leak state into each other (Pitfall 2).
    trace_api::Provider::SetTracerProvider(
        opentelemetry::nostd::shared_ptr<trace_api::TracerProvider>(new trace_api::NoopTracerProvider()));
  }
};

TEST_F(OrdersTelemetryTest, EmitsStableLowCardinalitySpanOnSuccess) {
  ProcessOrder({/*order_id=*/"o-1", /*item_count=*/3, /*region=*/"us"});
  auto spans = span_data_->GetSpans(); // vector<unique_ptr<SpanData>>
  ASSERT_EQ(spans.size(), 1u);
  EXPECT_EQ(spans[0]->GetName(), "orders.process"); // no embedded IDs
  // assert a bounded attribute: spans[0]->GetAttributes().at("app.order.region") == "us".
}

TEST_F(OrdersTelemetryTest, SetsErrorStatusOnFailure) {
  ProcessOrder({/*order_id=*/"o-2", /*item_count=*/1, /*region=*/"us" /* force failure */});
  auto spans = span_data_->GetSpans();
  ASSERT_EQ(spans.size(), 1u);
  EXPECT_EQ(spans[0]->GetStatus(), trace_api::StatusCode::kError);
}
```

### Two pitfalls to decide BEFORE writing the test

- **Pitfall 1 — register the provider first (the "no spans" trap).** `GetTracer` binds
  to whatever provider is installed **at call time**. If the code under test caches its
  tracer at first use before the test installs the in-memory provider, spans go to the
  no-op provider and `GetSpans()` returns empty. Fixes: register the test provider in
  `SetUp` before exercising the code, and acquire the tracer lazily (as in the tracing
  example) rather than caching a global tracer at static-init.
- **Pitfall 2 — reset global state between tests.** The provider is a process global;
  without resetting it in `TearDown`, one test's provider (and its captured spans) leak
  into the next. Reset to a `NoopTracerProvider` (or a fresh in-memory one per test).

### Metrics test (only if metrics were added)

Use an **in-memory metric reader** (or a periodic reader with a very short interval and
an in-memory exporter), exercise the code, force a collection, and assert instrument
name/type/unit, datapoint value/count, and **bounded** attributes. Tear down the meter
provider afterward.

### Logs test (only if logs were added)

Register a `SimpleLogRecordProcessor` feeding an **in-memory log record exporter**, emit
a log inside an active span, and assert body/severity, that trace/span IDs match the
enclosing span, and that sensitive fields are redacted. Reset the logger provider
between tests. See `../../signals/logs.md`.

## Framework / runtime notes

Setup and general rules are above / in the neutral references; below are only the deltas.

- **gRPC.** Instrument via a client/server interceptor that starts a `kClient`/`kServer`
  span and injects/extracts W3C context through gRPC metadata (the carrier). Don't also
  manually propagate on top of an interceptor that already does.
- **HTTP servers.** Wrap the outermost request handling so one `kServer` span covers the
  request and (for parity with other languages) emit request-duration as a metric, not
  only a span. Set the ERROR status message on that server span from application code on
  5xx — the transport layer only knows the status code.
- **Threaded / async servers & event loops.** The big pitfall is `RuntimeContext` being
  thread-local: propagate context across thread-pool/executor boundaries explicitly (see
  **Context propagation → Threads**) or spans orphan into separate traces.
- **Embedded / long-running daemons.** Initialize once at startup, keep providers alive
  for the process, size the batch processor/queue for the throughput, and install a
  signal handler that `ForceFlush`es and shuts down providers before exit.

## Bootstrap file (copy-ready, applications only)

A complete application init + cleanup. It hardcodes **no** endpoint/token/header —
delivery is via standard `OTEL_*` env vars read by the OTLP exporter. Requires
opentelemetry-cpp built with `WITH_OTLP_GRPC=ON`. Inspect the installed version's
headers before copying (factory signatures shift across majors). A **library** must not
use this.

```cpp
#include <opentelemetry/exporters/otlp/otlp_grpc_exporter_factory.h>
#include <opentelemetry/exporters/otlp/otlp_grpc_metric_exporter_factory.h>
#include <opentelemetry/sdk/metrics/meter_provider_factory.h>
#include <opentelemetry/sdk/metrics/export/periodic_exporting_metric_reader_factory.h>
#include <opentelemetry/sdk/resource/resource.h>
#include <opentelemetry/sdk/trace/batch_span_processor_factory.h>
#include <opentelemetry/sdk/trace/tracer_provider_factory.h>
#include <opentelemetry/metrics/provider.h>
#include <opentelemetry/trace/provider.h>
#include <opentelemetry/context/propagation/global_propagator.h>
#include <opentelemetry/trace/propagation/http_trace_context.h>

namespace otlp = opentelemetry::exporter::otlp;
namespace sdktrace = opentelemetry::sdk::trace;
namespace sdkmetrics = opentelemetry::sdk::metrics;
namespace resource = opentelemetry::sdk::resource;
namespace trace_api = opentelemetry::trace;
namespace metrics_api = opentelemetry::metrics;
namespace ctx_prop = opentelemetry::context::propagation;

void InitTelemetry() {
  // Resource: the env detector merges OTEL_SERVICE_NAME / OTEL_RESOURCE_ATTRIBUTES;
  // code-set attributes are layered on top. See ../../discovery/resolve-values.md.
  auto res = resource::Resource::Create({{"service.name", "orders"}});

  // --- Traces: OTLP gRPC exporter (reads OTEL_EXPORTER_OTLP_* from env) -> batch. ---
  otlp::OtlpGrpcExporterOptions trace_opts;   // default-constructed => env-driven
  auto trace_exporter  = otlp::OtlpGrpcExporterFactory::Create(trace_opts);
  auto trace_processor = sdktrace::BatchSpanProcessorFactory::Create(std::move(trace_exporter), {});
  std::shared_ptr<trace_api::TracerProvider> tp =
      sdktrace::TracerProviderFactory::Create(std::move(trace_processor), res);
  trace_api::Provider::SetTracerProvider(tp);

  // --- Metrics: OTLP gRPC exporter -> periodic reader (short interval for short runs). ---
  otlp::OtlpGrpcMetricExporterOptions metric_opts;
  auto metric_exporter = otlp::OtlpGrpcMetricExporterFactory::Create(metric_opts);
  sdkmetrics::PeriodicExportingMetricReaderOptions reader_opts;
  reader_opts.export_interval_millis = std::chrono::milliseconds(5000);
  reader_opts.export_timeout_millis  = std::chrono::milliseconds(2000);
  auto reader = sdkmetrics::PeriodicExportingMetricReaderFactory::Create(std::move(metric_exporter), reader_opts);
  auto mp = sdkmetrics::MeterProviderFactory::Create(
      std::make_unique<sdkmetrics::ViewRegistry>(), res);
  static_cast<sdkmetrics::MeterProvider *>(mp.get())->AddMetricReader(std::move(reader));
  std::shared_ptr<metrics_api::MeterProvider> mp_shared(std::move(mp));
  metrics_api::Provider::SetMeterProvider(mp_shared);

  // Cross-service W3C trace context.
  ctx_prop::GlobalTextMapPropagator::SetGlobalPropagator(
      opentelemetry::nostd::shared_ptr<ctx_prop::TextMapPropagator>(
          new trace_api::propagation::HttpTraceContext()));
}

// Flush and tear down BEFORE main returns; then reset globals to no-ops so nothing is
// created during static destruction. os.Exit-style aborts still lose the last batch.
void CleanupTelemetry() {
  if (auto *tp = static_cast<sdktrace::TracerProvider *>(
          trace_api::Provider::GetTracerProvider().get())) {
    tp->ForceFlush(std::chrono::microseconds(2'000'000));
    tp->Shutdown();
  }
  trace_api::Provider::SetTracerProvider(
      opentelemetry::nostd::shared_ptr<trace_api::TracerProvider>(new trace_api::NoopTracerProvider()));

  std::shared_ptr<metrics_api::MeterProvider> none;
  metrics_api::Provider::SetMeterProvider(none);
}

int main() {
  InitTelemetry();
  // ... application work; end every span before returning ...
  CleanupTelemetry();
  return 0;
}
```

Cast the provider to the SDK type to reach `ForceFlush`/`Shutdown` (the API type
doesn't expose them). Exact factory names/signatures vary by version — verify against
the installed opentelemetry-cpp headers before building.
