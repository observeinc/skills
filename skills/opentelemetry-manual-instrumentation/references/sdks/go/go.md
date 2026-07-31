# Go SDK Rule

Go specifics for manual OTel instrumentation — apply alongside the neutral rules in
SKILL.md's Rule routing (fundamentals, traces, metrics, semantic-review,
cardinality, …). Go has **no single auto-instrumentation agent**: you wire
the SDK and each instrumentation library explicitly, and telemetry flows through an
explicit `context.Context`.

## Files to inspect

- `go.mod` / `go.sum` (module path, Go version, existing `go.opentelemetry.io/*` deps)
- `vendor/modules.txt` if the repo vendors dependencies
- entrypoints (`func main`), and any `cmd/*/main.go` in a multi-binary layout
- workspace files (`go.work`) for multi-module repos
- start/build commands (`Makefile`, `Taskfile.yml`, `Dockerfile`, `go run`/`go build` targets)
- existing OTel setup (files defining `NewTracerProvider` / `otel.SetTracerProvider`,
  or an `otel`/`telemetry`/`observability` package)
- existing vendor instrumentation (`dd-trace-go`, `newrelic`, `sentry-go`)
- existing logging (`log/slog`, `zerolog`, `zap`, `logrus`) and metrics (`prometheus`)
- HTTP/gRPC framework in use (net/http, gin, echo, chi, fiber, `google.golang.org/grpc`)

These feed the environment checkpoint (SKILL.md → Workflow step 1): Go version,
module layout, framework, existing OTel deps, build/run commands.

## Dependency guidance

- **Applications** may use: `go.opentelemetry.io/otel` (API), `go.opentelemetry.io/otel/sdk`
  (SDK: trace/metric/log/resource), the OTLP exporters
  (`.../exporters/otlp/otlptrace/otlptracegrpc` or `…otlptracehttp`, and the
  `otlpmetric*` / `otlplog*` equivalents), a pinned `semconv` package, and the
  per-library **contrib** instrumentations from `go.opentelemetry.io/contrib/...`.
- **Libraries** should depend **only** on `go.opentelemetry.io/otel` (API) — never
  import `.../sdk` or an exporter. Acquire tracers/meters from the global providers
  the host app installs.
- **Pin the `semconv` version** to a specific module path — it is versioned in the
  import path itself, e.g. `go.opentelemetry.io/otel/semconv/v1.41.0` (shipped with
  otel `v1.44.0`). Different majors expose different constant names; **inspect `go.mod`
  for the installed version** and import the matching path — do not guess.
- The API (`otel`), the SDK, and the exporters version **independently** (SDK/metric
  and log modules carry `v0.x` / `v1.x` lines). Read `go.sum`; keep them compatible.

## Environment variables

The OTLP **exporters** read these when constructed with defaults (via
`otlptracegrpc.New(ctx)` etc.):

| Variable                                          | Notes                                                                                                    |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `OTEL_SERVICE_NAME`                               | Sets `service.name`; else `unknown_service:<binary>`. Also settable via `resource.WithAttributes`.       |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                     | Default `http://localhost:4317` (grpc) / `:4318` (http).                                                 |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                     | `grpc` or `http/protobuf` — must match the exporter package you import.                                  |
| `OTEL_EXPORTER_OTLP_HEADERS`                      | e.g. auth headers; comma-separated `k=v`.                                                                |
| `OTEL_RESOURCE_ATTRIBUTES`                        | Merged when the resource is built with `resource.WithFromEnv()`.                                         |
| `OTEL_TRACES_SAMPLER` / `OTEL_TRACES_SAMPLER_ARG` | Honored only if you build the provider's sampler from env (or leave it default `parentbased_always_on`). |

**Two Go-specific exporter pitfalls:**

- **The exporter exists only if you construct it in code.** Go has no autoconfigure
  agent, so `OTEL_TRACES_EXPORTER=otlp` alone exports nothing — build the exporter and
  pass it to the provider.
- **Protocol = the imported package, not just the env var.** Importing
  `otlptracegrpc` fixes gRPC (4317); `otlptracehttp` fixes HTTP (4318). Set
  `OTEL_EXPORTER_OTLP_PROTOCOL` to match, and point the endpoint at the right port —
  a grpc exporter against a 4318 receiver fails to connect. See `../../verification/validation-and-debugging.md`.

## App vs library (Go)

- **Application/service** → owns provider construction: build `TracerProvider` /
  `MeterProvider` / `LoggerProvider`, register them with
  `otel.SetTracerProvider(...)` / `otel.SetMeterProvider(...)` /
  `global.SetLoggerProvider(...)`, and own graceful shutdown.
- **Library/package** → import `go.opentelemetry.io/otel` only; get a tracer lazily
  via `otel.Tracer("stable/scope/name")` (or accept a `TracerProvider`); **never**
  construct SDK providers or exporters, and never call `otel.SetTracerProvider`.

Full rule: `../../app-vs-library.md`.

## Startup and shutdown

You initialize explicitly at the top of `main`, so there's no import-ordering hazard.
The hazard is **losing buffered telemetry on exit**: the batch processors flush on
`Shutdown`, but `os.Exit`, `log.Fatal`, and unhandled signals **bypass `defer`**.

- Call the returned `shutdown(ctx)` **explicitly in the signal handler**, before the
  process exits — `defer shutdown()` alone is not enough for a server.
- For short-lived programs (CLIs, jobs) that return from `main` normally, `defer
shutdown()` is sufficient — but you must still call it, or the final batch never flushes.
- Give `Shutdown` a context with a deadline; it blocks until export completes or the
  deadline expires. See the **Bootstrap file** below for both shapes.

## Manual tracing

```go
package orders

import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"
	// Prefer constants from the pinned semconv module over string literals, e.g.
	// semconv "go.opentelemetry.io/otel/semconv/v1.41.0".
)

// Acquire a tracer ONCE with a stable scope name (the instrumentation scope,
// conventionally the package/module import path) + version.
var tracer = otel.Tracer("github.com/acme/shop/orders")

type ProcessOrderInput struct {
	OrderID   string // application logic only; NEVER a span name or metric label
	ItemCount int
	Region    string // bounded enum ("us"/"eu"/"apac") — safe as an attribute
}

// ctx is threaded in and out: the returned ctx carries the new span as parent.
func ProcessOrder(ctx context.Context, in ProcessOrderInput) error {
	// Span name = operation class, NOT fmt.Sprintf("process order %s", in.OrderID).
	ctx, span := tracer.Start(ctx, "orders.process")
	defer span.End() // ALWAYS end the span, on every return path

	// SAFE attributes: bounded, operation-describing, no PII. Custom business
	// dimensions live under a bounded namespace (app.*) and should be documented.
	span.SetAttributes(
		attribute.Int("app.order.item_count", in.ItemCount),
		attribute.String("app.order.region", in.Region),
	)
	span.AddEvent("validation.completed") // a notable moment inside the op (no extra span)

	if err := reserveInventory(ctx, in); err != nil {
		// Record the exception, set ERROR status with a short message (type + reason),
		// then return — do NOT swallow. Messages may contain user input; see
		// ../../review/cardinality.md.
		span.RecordError(err)
		span.SetStatus(codes.Error, fmt.Sprintf("reserve inventory: %s", err))
		return err
	}
	// On success leave status Unset; set codes.Ok only when logic CONFIRMS success.
	return nil
}
```

Threading rule: every function on the traced path takes `ctx context.Context` as its
**first parameter** and forwards it. A function that drops `ctx` breaks the trace —
child spans become orphaned roots. See **Context propagation** below.

## Manual metrics

```go
package orders

import (
	"context"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

// Acquire a meter ONCE, and create each instrument ONCE (package scope), then reuse.
var (
	meter = otel.Meter("github.com/acme/shop/orders")

	// Counter: monotonically increasing total.
	ordersProcessed, _ = meter.Int64Counter("app.orders.processed",
		metric.WithDescription("Total number of orders processed"),
		metric.WithUnit("{order}"))

	// UpDownCounter: a level that rises and falls.
	ordersInFlight, _ = meter.Int64UpDownCounter("app.orders.in_flight",
		metric.WithDescription("Orders currently being processed"),
		metric.WithUnit("{order}"))

	// Histogram: a distribution (durations/sizes), recorded in seconds.
	orderDuration, _ = meter.Float64Histogram("app.orders.duration",
		metric.WithDescription("Order processing duration"),
		metric.WithUnit("s"))
)

// ObservableGauge: a current value sampled via a callback (registered once).
func registerQueueDepthGauge(getDepth func() int64) error {
	_, err := meter.Int64ObservableGauge("app.orders.queue_depth",
		metric.WithDescription("Current depth of the order processing queue"),
		metric.WithUnit("{order}"),
		metric.WithInt64Callback(func(_ context.Context, o metric.Int64Observer) error {
			o.Observe(getDepth(), metric.WithAttributes(attribute.String("app.queue.name", "orders")))
			return nil
		}))
	return err
}

func RecordOrder(ctx context.Context, region string, run func() error) (err error) {
	base := metric.WithAttributes(attribute.String("app.order.region", region)) // bounded
	ordersInFlight.Add(ctx, 1, base)
	start := time.Now()
	outcome := "ok"
	defer func() {
		if err != nil {
			outcome = "error"
		}
		// outcome is a bounded enum — safe as a label.
		attrs := metric.WithAttributes(
			attribute.String("app.order.region", region),
			attribute.String("app.order.outcome", outcome),
		)
		ordersProcessed.Add(ctx, 1, attrs)
		orderDuration.Record(ctx, time.Since(start).Seconds(), attrs)
		ordersInFlight.Add(ctx, -1, base)
		// ❌ NEVER: attribute.String("order_id", id) / user_id / url — cardinality blowup.
	}()
	return run()
}
```

## Context propagation

Go has **no ambient current span** — the active span lives inside `context.Context`.
The neutral model is in `../../signals/context-propagation.md`; Go specifics:

- **Function signatures.** Any function on the request path must accept `ctx
context.Context` (first param, Go convention) and forward it to downstream calls
  and to `tracer.Start`. Retrofitting tracing = auditing every hop for a dropped `ctx`.
- **Goroutines.** Pass the context **explicitly** — do not rely on closure capture of
  a `ctx` the caller may cancel: `go func(ctx context.Context){ … }(ctx)`. For work
  that must outlive the request, start from `context.Background()` and **link** the
  originating span: `tracer.Start(context.Background(), "async.process",
trace.WithLinks(trace.LinkFromContext(ctx)))`.
- **Channels.** The producer must send the context with the payload — pair them in a
  struct (`type work struct{ ctx context.Context; item Item }`) and start the consumer
  span from `w.ctx`. Do not stash a `ctx` in a long-lived struct except for a
  bounded callback (below).
- **Callbacks / interfaces without a `ctx` param.** Prefer a context-aware variant if
  the library offers one; otherwise capture the context on the handler struct and
  start the span from it inside the callback.
- **Cross-service.** Set a global propagator once
  (`otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
propagation.TraceContext{}, propagation.Baggage{}))`) and use
  `otel.GetTextMapPropagator().Inject/Extract` with a carrier
  (`propagation.HeaderCarrier` for `http.Header`). The `otelhttp`/`otelgrpc`
  instrumentations do inject/extract for you — don't double-wrap.

## Enriching auto-instrumentation (don't duplicate)

When a path is already covered (e.g. `otelhttp` owns the HTTP `SERVER` span), enrich
the existing span instead of wrapping it in a redundant manual span — neutral decision
order (hook → processor → child span) is in `../../instrumentation-rules.md`. Go mechanisms:

- **`trace.SpanFromContext(ctx)`** — the primary enrichment tool. It returns the
  active (auto-created) span, or a non-recording no-op span if none — so
  `SetAttributes`/`SetStatus` are always safe with **no nil check**. Use it in an
  HTTP handler to stamp `tenant.id`, or to set the ERROR status message on the
  `otelhttp` server span (which sets ERROR on 5xx but **cannot** fill the message —
  the app must): `trace.SpanFromContext(r.Context()).SetStatus(codes.Error, err.Error())`.
- **A custom `SpanProcessor`** (implements `OnStart`/`OnEnd`) registered on the
  provider — to normalize names or stamp attributes uniformly across many spans.
- **Library options** — many contrib instrumentations expose hooks/filters (e.g.
  `otelhttp.WithSpanNameFormatter`, `otelgrpc` filters) configured where you build
  the handler/interceptor, not in business code.

## SDK wiring pitfalls (Go)

- **The exporter only exists if you build it.** No env var wires OTLP — construct
  `otlptracegrpc.New(ctx)` (or `…http`) and pass it to the provider. This is the #1
  "no telemetry, no error" cause in Go.
- **One global provider per signal.** Find existing `otel.SetTracerProvider` setup and
  **extend** it; never register twice.
- **Fast metric export for short-lived runs.** The `PeriodicReader` default interval
  (~60s) means CLIs/jobs/evals exit before the first flush → false "no metrics".
  Shorten it: `sdkmetric.NewPeriodicReader(exp, sdkmetric.WithInterval(5*time.Second))`,
  and/or call `MeterProvider.ForceFlush(ctx)` before shutdown.
- **Resource merge — don't clobber env/operator values.** Build the resource with
  `resource.WithFromEnv()` (and optionally `resource.WithHost()`/`WithProcess()`) so
  `OTEL_RESOURCE_ATTRIBUTES` and operator-injected values survive; add code-set
  attributes on top. The merge rule + test are in `../../discovery/resolve-values.md`.
- **`WithBatcher` vs `WithSyncer`.** Use `WithBatcher` (batch processor) in
  production; `WithSyncer`/`SimpleSpanProcessor` only for tests/local (it exports
  synchronously and is slow under load).
- **Pin `semconv` by import path** and use its constants (`semconv.ServiceName(...)`,
  `semconv.HTTPResponseStatusCode(...)`) rather than raw strings.

## Structured logging

Configure the logging framework to serialize errors into a single structured field so
stack traces don't break the one-line-per-record contract, and correlate to traces.
General guidance: `../../signals/logs.md`.

- **`log/slog` (standard library).** `slog.NewJSONHandler(os.Stdout, nil)` emits
  single-line JSON; use the `…Context` methods (`logger.ErrorContext(ctx, …)`) so the
  handler can pick up trace context. To attach `trace_id`/`span_id` and/or route logs
  through the OTLP pipeline, bridge slog to OTel logs with
  `go.opentelemetry.io/contrib/bridges/otelslog` (`otelslog.NewLogger(name)`), backed
  by the `LoggerProvider` from the bootstrap. Go errors carry no stack by default;
  if a lib adds one (`pkg/errors`, `cockroachdb/errors`), format with `%+v` and log it
  as a **single string field** to stay one line.
- **`zerolog`.** Single-line JSON by default; `log.Error().Err(err).Str("order_id",
id).Msg("order.failed")` serializes the error into one `"error"` field.

## Supported instrumentation libraries (common)

Installed individually from `go.opentelemetry.io/contrib/instrumentation/...` (there is
no meta-package): HTTP (`net/http/otelhttp`; gin/echo/fiber/chi via their contrib
wrappers); gRPC (`google.golang.org/grpc/otelgrpc`); databases (`database/sql/otelsql`,
`XSAM/otelsql`, `exaring/otelpgx`, `mongo-driver/otelmongo`); messaging
(`Shopify/sarama` Kafka, `amqp091`); AWS (`aws-sdk-go-v2/otelaws`); runtime metrics
(`go.opentelemetry.io/contrib/instrumentation/runtime`). Install only what the project
uses. Browse the [registry](https://opentelemetry.io/ecosystem/registry/?language=go).

## Testing

The language-neutral test policy (required gate, what to assert, forbidden practices)
is in `../../verification/telemetry-unit-tests.md`. Below is the Go harness using the
SDK's `tracetest` in-memory exporter — **no backend, no OTLP, no network.**

```go
package orders

import (
	"context"
	"testing"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/codes"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/sdk/trace/tracetest"
)

func newTestTracing(t *testing.T) *tracetest.InMemoryExporter {
	t.Helper()
	exp := tracetest.NewInMemoryExporter()
	// SimpleSpanProcessor exports synchronously — spans are visible immediately,
	// no flush needed (the opposite of the batch processor; see Pitfall 2).
	tp := sdktrace.NewTracerProvider(sdktrace.WithSpanProcessor(sdktrace.NewSimpleSpanProcessor(exp)))
	otel.SetTracerProvider(tp) // register BEFORE the code under test resolves its tracer (Pitfall 1)
	t.Cleanup(func() { _ = tp.Shutdown(context.Background()) })
	return exp
}

func TestProcessOrder_Success(t *testing.T) {
	exp := newTestTracing(t)
	if err := ProcessOrder(context.Background(), ProcessOrderInput{OrderID: "o-1", ItemCount: 3, Region: "us"}); err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	spans := exp.GetSpans()
	if len(spans) != 1 {
		t.Fatalf("want 1 span, got %d", len(spans))
	}
	s := spans[0]
	if s.Name != "orders.process" { // stable, low-cardinality — no embedded IDs
		t.Errorf("span name = %q", s.Name)
	}
	// assert a bounded attribute is present (iterate s.Attributes for app.order.region == "us").
}

func TestProcessOrder_Error(t *testing.T) {
	exp := newTestTracing(t)
	_ = ProcessOrder(context.Background(), ProcessOrderInput{Region: "us" /* force reserveInventory to fail */})
	s := exp.GetSpans()[0]
	if s.Status.Code != codes.Error {
		t.Errorf("status = %v, want Error", s.Status.Code)
	}
	// assert an exception event was recorded (s.Events contains name "exception").
}
```

### Two pitfalls to decide BEFORE writing the test

- **Pitfall 1 — register the provider first (the "0 spans" trap).** A package-level
  `var tracer = otel.Tracer(...)` resolves against whatever global provider exists at
  **package-init time**. If that's the default no-op provider (test hadn't registered
  yet), the exporter sees **0 spans**. Fixes: register the test provider in the test's
  setup _before_ the code path runs (works because `otel.Tracer` returns a delegating
  tracer that re-resolves), **or** acquire the tracer lazily inside the function.
- **Pitfall 2 — batch processor needs a flush.** If you use `WithBatcher` in a test,
  spans sit in the batch queue — call `tp.ForceFlush(ctx)` before asserting, or use
  `NewSimpleSpanProcessor` (above) which exports on `span.End()`.

### Metrics test (only if metrics were added)

Use a **`ManualReader`** and collect on demand — no timing races:

```go
import (
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/metric/metricdata"
)

reader := sdkmetric.NewManualReader()
otel.SetMeterProvider(sdkmetric.NewMeterProvider(sdkmetric.WithReader(reader)))
// exercise code, then:
var rm metricdata.ResourceMetrics
_ = reader.Collect(context.Background(), &rm)
// assert instrument name, type, unit, datapoint value/count, and BOUNDED attributes.
```

### Logs test (only if logs were added)

Use the log SDK's in-memory exporter (`go.opentelemetry.io/otel/sdk/log` with an
in-memory exporter), emit a log inside an active span, then assert body/severity,
that `trace_id`/`span_id` match the enclosing span, and that sensitive fields are
redacted. See `../../signals/logs.md`.

## Framework notes

Setup and general rules live above / in the neutral references; below are only the
framework-specific deltas.

### net/http (and gin / echo / chi / fiber)

`otelhttp.NewHandler(mux, "server")` owns the **`SERVER` span** and emits the
`http.server.request.duration` metric; `otelhttp.NewTransport(...)` covers the client
side. Add manual spans only as **children** for domain operations, and set status
manually only on your **business-logic** spans (a 4xx is **UNSET** on a server span —
see `../../signals/traces.md`). Frameworks use their own contrib wrapper
(gin `otelgin`, echo `otelecho`, chi `otelchi`, fiber `otelfiber`) — the enrichment
pattern (`trace.SpanFromContext`) is identical.

### gRPC

Add `otelgrpc` stats handlers:
`grpc.NewServer(grpc.StatsHandler(otelgrpc.NewServerHandler()))` and
`grpc.NewClient(target, grpc.WithStatsHandler(otelgrpc.NewClientHandler()))`. This
handles context inject/extract across the wire — don't also manually propagate.

### Queue workers & background jobs

Producer→consumer **context propagation across the broker** is the core concern:
inject the active context into message headers before publishing (`PRODUCER` span);
extract it and run the handler with it active (`CONSUMER` span as child/link). Batch
consumers use **links**, not one shared parent. A worker binary is its **own**
entrypoint and needs its own bootstrap + shutdown. See `../../signals/context-propagation.md`.

### CLIs / short-lived jobs

The exit-flush hazard dominates: use a **short** metric export interval (or a
`ManualReader`+explicit collect), and **must** call `shutdown(ctx)` (or
`ForceFlush`) before returning — otherwise the final batch never leaves the process.

## Bootstrap file (copy-ready, applications only)

A complete application bootstrap. It hardcodes **no** endpoint/token/header — delivery
is via standard `OTEL_*` env vars, read by the exporters. Inspect installed module
versions before copying (`go.mod`); pin `semconv` to the matching path. A **library**
must not use this.

```go
package telemetry

import (
	"context"
	"errors"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/propagation"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

// Setup wires trace + metric providers and returns a shutdown func that flushes and
// stops them. Call shutdown explicitly on exit (signal handler) — os.Exit bypasses defer.
func Setup(ctx context.Context) (shutdown func(context.Context) error, err error) {
	var shutdowns []func(context.Context) error
	shutdown = func(ctx context.Context) error {
		var e error
		for _, fn := range shutdowns {
			e = errors.Join(e, fn(ctx))
		}
		return e
	}

	// Resource: WithFromEnv keeps OTEL_RESOURCE_ATTRIBUTES / operator-injected values;
	// code-set attributes (service.name, version) are merged on top. See ../../discovery/resolve-values.md.
	res, err := resource.New(ctx, resource.WithFromEnv(), resource.WithHost(), resource.WithProcess())
	if err != nil {
		return nil, err
	}

	traceExp, err := otlptracegrpc.New(ctx) // reads OTEL_EXPORTER_OTLP_* from env
	if err != nil {
		return nil, err
	}
	tp := sdktrace.NewTracerProvider(sdktrace.WithBatcher(traceExp), sdktrace.WithResource(res))
	otel.SetTracerProvider(tp)
	shutdowns = append(shutdowns, tp.Shutdown)

	metricExp, err := otlpmetricgrpc.New(ctx)
	if err != nil {
		return shutdown, err
	}
	mp := sdkmetric.NewMeterProvider(
		sdkmetric.WithResource(res),
		// Short interval so short-lived processes flush before exit.
		sdkmetric.WithReader(sdkmetric.NewPeriodicReader(metricExp, sdkmetric.WithInterval(5*time.Second))),
	)
	otel.SetMeterProvider(mp)
	shutdowns = append(shutdowns, mp.Shutdown)

	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, propagation.Baggage{}))
	return shutdown, nil
}
```

Server `main` (signal-safe flush):

```go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer stop()

	shutdown, err := telemetry.Setup(ctx)
	if err != nil {
		log.Fatalf("otel setup: %v", err)
	}

	srv := &http.Server{Addr: ":8080", Handler: otelhttp.NewHandler(mux, "server")}
	go func() { _ = srv.ListenAndServe() }()

	<-ctx.Done()
	sctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	_ = srv.Shutdown(sctx)
	_ = shutdown(sctx) // flush telemetry AFTER the server stops accepting work
}
```

For a short-lived CLI/job, `defer shutdown(ctx)` in `main` after `Setup` is enough —
but it must run, so avoid `os.Exit`/`log.Fatal` after work begins.
