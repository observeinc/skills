# Ruby SDK Rule

Ruby specifics for manual OTel instrumentation — apply alongside the neutral rules in
SKILL.md's Rule routing (fundamentals, traces, metrics, semantic-review,
cardinality, …). Ruby's auto-instrumentation is opt-in gems activated in code
(`c.use_all`); this file is about the **manual** API/SDK you write. For zero-code coverage
use **opentelemetry-auto-instrumentation**.

**Signal maturity:** only **traces are stable** in Ruby; the **metrics and logs SDKs are in
development** ([status](https://opentelemetry.io/docs/languages/ruby/)). Don't wire the OTel
metrics SDK or logs SDK/bridge. Instrument with traces (spans, span events,
`record_exception`); emit a metric as a span attribute and a log as structured stdout (see
**Metrics**, **Structured logging**).

## Files to inspect

- `Gemfile` / `Gemfile.lock` (existing `opentelemetry-*` gems, framework, Ruby version)
- `.ruby-version`, `.tool-versions`
- entrypoints (`config.ru`, `bin/*`, a plain `main.rb`) and framework config
- Rails: `config/application.rb`, `config/initializers/` (existing
  `opentelemetry.rb`), `config/environments/*.rb`
- background workers (`config/sidekiq.yml`, `Procfile`, `sidekiq`/`resque` invocations)
- server config (`config/puma.rb` — worker/preload settings), `Dockerfile`
- existing OTel setup (an `OpenTelemetry::SDK.configure` block, a `telemetry.rb`)
- existing vendor instrumentation (`ddtrace`, `newrelic_rpm`, `sentry-ruby`)
- existing logging (`semantic_logger`, `lograge`, `Rails.logger`) and metrics setup

These feed the environment checkpoint (SKILL.md → Workflow step 1): Ruby version, framework,
app server (Puma/Unicorn — forking), background workers, existing OTel gems, start commands.

## Dependency guidance

- **Applications** may use: `opentelemetry-sdk`, `opentelemetry-exporter-otlp`, and — for
  zero-code coverage — `opentelemetry-instrumentation-all` (or individual
  `opentelemetry-instrumentation-*` gems). Attribute-name constants come from
  `opentelemetry-semantic_conventions`.
- **Libraries** should depend **only** on `opentelemetry-api` — never the SDK or an exporter.
  Get a tracer via `OpenTelemetry.tracer_provider.tracer('stable.scope', 'x.y.z')`; the host
  app owns the SDK.
- **Don't add the metrics or logs SDK gems** (`opentelemetry-metrics-api`/`-sdk`,
  `opentelemetry-exporter-otlp-metrics`, any logs SDK/appender bridge); both are in
  development. Applications ship traces only.
- **Infer via Bundler** — add gems with `bundle add`, run under `bundle exec`, and inspect
  `Gemfile.lock` for installed versions before editing.

## Environment variables

| Variable                      | Notes                                                                                                                                              |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_SERVICE_NAME`           | Required; else `unknown_service`. Also settable via `c.service_name=` in the configure block.                                                      |
| `OTEL_TRACES_EXPORTER`        | **Defaults to `otlp` under `opentelemetry-sdk`**, but is often driven by which exporter gem is loaded; set `console` for local, `none` to disable. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Default `http://localhost:4318`.                                                                                                                   |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Default `http/protobuf` (port 4318); `grpc` is port 4317.                                                                                          |
| `OTEL_EXPORTER_OTLP_HEADERS`  | e.g. auth; comma-separated `k=v`.                                                                                                                  |
| `OTEL_RESOURCE_ATTRIBUTES`    | `deployment.environment.name`, `service.version`, …                                                                                                |

The exporter only sends if the OTLP exporter gem is required and reachable. **Protocol/port:**
match the protocol to the endpoint port (`http/protobuf` ↔ 4318, `grpc` ↔ 4317); a mismatch
fails to connect. Ruby does **not** auto-load `.env`; use the `dotenv` gem or export in the
shell/process manager. See `../../verification/validation-and-debugging.md`.

## App vs library (Ruby)

- **Application/service** → owns the `OpenTelemetry::SDK.configure` block: sets service name,
  enables auto-instrumentation (`c.use_all` / `c.use`), attaches exporters + resource, and
  registers shutdown.
- **Library/gem** → depend on `opentelemetry-api` only; get a tracer via
  `OpenTelemetry.tracer_provider.tracer('stable.scope', 'x.y.z')`; **never** call
  `OpenTelemetry::SDK.configure` or add an exporter.

Full rule: `../../app-vs-library.md`.

## Startup and bootstrap ordering

The SDK is initialized **in code** and **must run before app/framework code**, so
auto-instrumentation can wrap libraries as they load:

- **Rails** → `config/initializers/opentelemetry.rb` (initializers run during boot, before
  request handling).
- **Non-Rails** → at the top of the entrypoint, **before other `require`s** of instrumented
  libraries.

See **Bootstrap file**. For **forking servers** (Puma/Unicorn in cluster mode), see the
worker-fork note under **SDK wiring pitfalls**.

## Manual tracing

```ruby
# Acquire a tracer ONCE with a stable scope name + version.
TRACER = OpenTelemetry.tracer_provider.tracer('orders.service', '1.0.0')

def process_order(order_id, item_count:, region:)
  # Span name = operation class, NOT "process order #{order_id}" (high-cardinality).
  # in_span yields the span and ENDS it automatically on block exit (every path).
  TRACER.in_span('orders.process') do |span|
    # SAFE attributes: bounded, operation-describing, no PII. Business dimensions under a
    # bounded namespace (app.*), documented.
    span.set_attribute('app.order.item_count', item_count)
    span.set_attribute('app.order.region', region)   # bounded enum "us"/"eu"/"apac"
    span.add_event('validation.completed')            # notable moment inside the op, no extra span
    save_order(order_id)
    # On success leave status UNSET; set ok only when logic CONFIRMS success.
  rescue StandardError => e
    span.record_exception(e)
    # ERROR needs a short message: class + reason, NOT the backtrace (log that).
    span.status = OpenTelemetry::Trace::Status.error("#{e.class}: #{e.message}")
    raise                                             # re-raise — do NOT swallow
  end
end
```

The block-level `rescue` (inside `do … end`) is valid Ruby and keeps the handler in the span
scope. Message text may contain user input — see `../../review/cardinality.md`.

## Metrics (avoid; SDK in development)

The OTel Ruby **metrics SDK is in development**
([status](https://opentelemetry.io/docs/languages/ruby/)); don't wire a `meter_provider` or add
the metrics gems. Put the measurement on a **bounded span attribute** or a structured stdout
log, and record native metrics as out of scope in the final report.

## Context propagation

The active span lives in an `OpenTelemetry::Context` (fiber-local). The neutral model is
`../../signals/context-propagation.md`; Ruby specifics:

- **Within a fiber / normal call flow**, `in_span` sets the span current for the block and
  restores it after — children attach automatically.
- **Threads.** A new `Thread` does **not** inherit the context. Capture and re-attach:
    ```ruby
    ctx = OpenTelemetry::Context.current
    Thread.new { OpenTelemetry::Context.with_current(ctx) { work } }
    ```
- **Async / job enqueue.** Work resumed later (a queued job, a callback) must carry the
  context — propagate it explicitly the same way, or rely on the broker instrumentation
  (below) to inject/extract it.
- **Cross-service.** `OpenTelemetry.propagation.inject(carrier)` before an outbound request
  and `OpenTelemetry.propagation.extract(carrier)` on the server. Rack/Faraday/Net::HTTP
  instrumentation does this for you — don't double-propagate.

## Enriching auto-instrumentation (don't duplicate)

When a path is already covered (the Rack/Rails instrumentation owns the HTTP `SERVER` span),
enrich the existing span rather than wrapping it (neutral decision order in
`../../instrumentation-rules.md`):

```ruby
span = OpenTelemetry::Trace.current_span
span.set_attribute('app.tenant.id', tenant_id)   # bounded
```

`OpenTelemetry::Trace.current_span` returns a **non-recording** span when none is active, and
`set_attribute`/`status=` on it are no-ops — so **no guard is needed**. The server span sets
ERROR on 5xx but leaves a 4xx **UNSET** (see `../../signals/traces.md`).

## SDK wiring pitfalls (Ruby)

- **Configure once, before app code.** `OpenTelemetry::SDK.configure` must run before the
  libraries it instruments are used; a second `configure` re-initializes providers. Find the
  existing initializer and extend it.
- **No automatic shutdown hook — you must add one.** The Ruby SDK registers no exit hook.
  Without an `at_exit { …shutdown }` the batch span processor's final buffer is **lost on
  exit**. See **Bootstrap file**.
- **Forking servers (Puma/Unicorn cluster, Sidekiq).** The batch processor's background thread
  does not survive `fork`. Re-initialize (or restart the exporter) in the worker after fork —
  Puma `on_worker_boot`, Unicorn `after_fork`, Sidekiq the auto-instrumentation handles — or
  child processes export nothing.
- **Resource from env.** `OTEL_RESOURCE_ATTRIBUTES` + `OTEL_SERVICE_NAME` are merged; add
  code-set attributes via `c.resource=`. See `../../discovery/resolve-values.md`.

## Database query parameters

The OpenTelemetry Ruby instrumentation gems (`opentelemetry-instrumentation-pg`,
`-mysql2`, `-active_record`) don't capture prepared-statement parameter values and expose no
option to enable it. The `:db_statement` option (`:include` / `:omit` / `:obfuscate`) controls
only `db.statement` sanitization:

```ruby
OpenTelemetry::SDK.configure do |c|
  c.use 'OpenTelemetry::Instrumentation::PG', { db_statement: :obfuscate }  # sanitization only
end
```

The instrumentation creates and **ends its span inside a `Module#prepend` wrapper** around the
driver method, so trying to attach `db.query.parameter.*` from application code after the call
returns is a no-op — the span is already closed. Don't work around this to capture parameter
values; they routinely hold PII/secrets.

## Structured logging (stdout, not the OTel logs SDK)

The OTel Ruby **logs SDK/bridge is in development**
([status](https://opentelemetry.io/docs/languages/ruby/)); don't wire it or an OTLP log
exporter. Emit **structured JSON to stdout** and let a collector forward it (the recommended
path in `../../signals/logs.md`). Serialize exceptions into a single structured field so
backtraces don't break the one-line-per-record contract, and correlate to traces.

- **`semantic_logger`.** `SemanticLogger.add_appender(io: $stdout, formatter: :json)` emits
  single-line JSON and serializes the exception class/message/backtrace into structured
  fields: `logger.error('order.failed', exception: e, order_id: id)`.
- **`lograge` (Rails).** Replaces the multi-line request log with one JSON line
  (`config.lograge.formatter = Lograge::Formatters::Json.new`). Lograge does **not** serialize
  exception backtraces — pair it with semantic_logger or a JSON formatter that folds
  exceptions into a single field.

For trace correlation, attach `trace_id`/`span_id` from `OpenTelemetry::Trace.current_span
.context` (`hex_trace_id` / `hex_span_id`).

## Auto-instrumentation coverage

For the libraries auto-instrumentation covers and the telemetry they emit, see the
**opentelemetry-auto-instrumentation** skill; enrich those spans rather than duplicating them.

## Testing

The language-neutral test policy is in `../../verification/telemetry-unit-tests.md`. Use the
SDK's `InMemorySpanExporter` — **no backend, no OTLP, no network.** (The
`opentelemetry-test-helpers` gem provides conveniences, but the plain SDK is enough.)

```ruby
require 'opentelemetry/sdk'

EXPORTER = OpenTelemetry::SDK::Trace::Export::InMemorySpanExporter.new

# Register ONCE for the test suite, BEFORE the code under test resolves its tracer.
OpenTelemetry::SDK.configure do |c|
  c.add_span_processor(OpenTelemetry::SDK::Trace::Export::SimpleSpanProcessor.new(EXPORTER))
  # SimpleSpanProcessor exports synchronously — spans visible immediately, no flush.
end

# minitest example
class OrderServiceTest < Minitest::Test
  def setup
    EXPORTER.reset          # isolate spans per test
  end

  def test_creates_stable_span_on_success
    process_order('o-1', item_count: 3, region: 'us')
    spans = EXPORTER.finished_spans
    assert_equal 1, spans.length
    assert_equal 'orders.process', spans[0].name          # stable, low-cardinality
    assert_equal 'us', spans[0].attributes['app.order.region']
  end

  def test_records_exception_and_error_status_on_failure
    assert_raises(StandardError) { process_order('o-2', item_count: 1, region: 'us') }
    span = EXPORTER.finished_spans[0]
    assert_equal OpenTelemetry::Trace::Status::ERROR, span.status.code
    assert(span.events.any? { |e| e.name == 'exception' })
  end
end
```

**Pitfall — the "0 spans" trap.** If a tracer was resolved (or `configure` ran) before your
test registers the in-memory processor, spans go to the earlier provider and the exporter sees
**0 spans**. Register the test processor before any code path runs (a `spec_helper`/
`test_helper` require at the top), and `reset` the **exporter** between tests. RSpec is the same
shape — register in a `before(:suite)`, `EXPORTER.reset` in `before(:each)`.

**Metrics and logs.** Both SDKs are in development, so there's no OTel metric/log exporter to
assert against; don't test them. Verify the measurement landed as a span attribute, confirm
structured logs are single-line JSON with trace correlation, and document native metrics/logs
out of scope.

## Framework notes

Setup and general rules live above / in the neutral references; below are only the
framework-specific deltas.

### Rails

Put `OpenTelemetry::SDK.configure` in `config/initializers/opentelemetry.rb` (+ the `at_exit`
shutdown). The Rack/Rails instrumentation **owns the `SERVER` span** and sets ERROR on 5xx —
add manual spans only as **children** for domain operations (service/model layer), and set
status manually only on your **business-logic** spans (a 4xx is UNSET on the server span). Use
`lograge` for single-line request logs.

### Sidekiq & background workers

Producer→consumer **context propagation across Redis** is handled by the Sidekiq
instrumentation (it injects/extracts context so the job continues the caller's trace). A worker
process is its **own** entrypoint — ensure `SDK.configure` runs there too, and that the batch
processor is (re)started after fork. Batch jobs use **links**, not one shared parent. See
`../../signals/context-propagation.md`.

### Rack / Sinatra & forking app servers

Puma/Unicorn in cluster mode **fork** workers; the exporter's background thread doesn't survive
fork — re-init or restart it in `on_worker_boot`/`after_fork`, or workers export nothing.

## Bootstrap file (copy-ready, applications only)

A complete initializer. It hardcodes **no** endpoint/token/header — delivery is via standard
`OTEL_*` env vars read by the OTLP exporter gem. A **library** must not use this.

```ruby
# config/initializers/opentelemetry.rb   (or top of the entrypoint, before other requires)
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'
require 'opentelemetry/instrumentation/all'   # only for zero-code coverage of known libs

OpenTelemetry::SDK.configure do |c|
  c.service_name = ENV.fetch('OTEL_SERVICE_NAME', 'orders-service')
  # OTLP exporter + BatchSpanProcessor are wired from OTEL_EXPORTER_OTLP_* env by default.
  c.use_all                                    # or c.use 'OpenTelemetry::Instrumentation::Rails', etc.
end

# The Ruby SDK registers NO automatic shutdown hook — add one, or the final batch is lost.
at_exit do
  OpenTelemetry.tracer_provider.shutdown if OpenTelemetry.respond_to?(:tracer_provider)
end
```

Set `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` if you point the endpoint at 4317 rather than the
`http/protobuf` (4318) default. `shutdown` blocks until export completes or the timeout expires
(~30s). Don't wire an OTel logs SDK / log-record processor here; the Ruby logs SDK is in
development. Emit structured logs to stdout instead (see **Structured logging**).
