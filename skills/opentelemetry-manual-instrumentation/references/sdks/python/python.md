# Python SDK Rule

Python specifics for manual OTel instrumentation — apply alongside the neutral rules
in SKILL.md's Rule routing (fundamentals, traces, metrics, semantic-review,
cardinality, …). Python has an auto-instrumentation launcher
(`opentelemetry-instrument`); this file is about the **manual** API/SDK you write in
code — for zero-code coverage use **opentelemetry-auto-instrumentation**.

## Files to inspect

- dependency & lock files: `pyproject.toml`, `requirements*.txt`, `poetry.lock`,
  `uv.lock`, `Pipfile`/`Pipfile.lock`, `setup.py`/`setup.cfg`
- the virtualenv / interpreter in use (`.python-version`, `.venv/`, tox/nox configs)
- entrypoints (`main.py`, `manage.py`, `app.py`, `wsgi.py`, `asgi.py`, `__main__.py`)
- start commands (`Procfile`, `Dockerfile`, `docker-compose.yml`, Makefile targets,
  `gunicorn`/`uvicorn`/`celery` invocations)
- framework config (Flask/Django/FastAPI app factory, Django `settings.py`)
- existing OTel files (e.g. `telemetry.py`, `otel.py`, `tracing.py`) and any
  `opentelemetry-instrument` usage in scripts
- existing vendor instrumentation (`ddtrace`, `newrelic`, `sentry_sdk`)
- existing logging/metrics setup (`logging.config`, `structlog`, `prometheus_client`)

These feed the environment checkpoint (SKILL.md → Workflow step 1): interpreter version,
package manager, framework, async model (WSGI vs ASGI), existing OTel deps, start commands.

## Dependency guidance

- **Applications** may use: `opentelemetry-api`, `opentelemetry-sdk`, an OTLP exporter
  (`opentelemetry-exporter-otlp` — pulls gRPC + HTTP; or `-otlp-proto-http` /
  `-otlp-proto-grpc` for one), `opentelemetry-semantic-conventions`, and — for zero-code
  coverage — `opentelemetry-distro` + the per-library `opentelemetry-instrumentation-*`
  packages (installed via `opentelemetry-bootstrap -a install`).
- **Libraries** should depend **only** on `opentelemetry-api` — never the SDK or an
  exporter. Acquire tracers/meters with a stable scope name; the host app owns providers.
- **Infer the package manager** from the lock/manifest (pip / poetry / uv / pipenv) and
  install into the **same virtualenv** the app runs in.
- **Inspect installed versions** before editing — the semantic-conventions constants and
  the metrics/logs SDK APIs shift across releases.

## Environment variables

| Variable                      | Notes                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------- |
| `OTEL_SERVICE_NAME`           | Required; else `unknown_service`.                                               |
| `OTEL_TRACES_EXPORTER`        | Defaults to `otlp`. `console` for local; `none` disables.                       |
| `OTEL_METRICS_EXPORTER`       | **Defaults to `none`** — set `otlp` explicitly or metrics are silently dropped. |
| `OTEL_LOGS_EXPORTER`          | Defaults to `otlp`.                                                             |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Default `http://localhost:4317`.                                                |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Default `grpc` (port 4317); `http/protobuf` is port 4318.                       |
| `OTEL_RESOURCE_ATTRIBUTES`    | `deployment.environment.name`, `service.version`, …                             |

**Metrics-default pitfall:** traces and logs export by default, but **metrics do not** —
if you added metrics and see none, confirm `OTEL_METRICS_EXPORTER=otlp`. **Protocol/port:**
a `grpc` exporter against a `4318` receiver (or vice-versa) fails to connect — match the
protocol to the port. See `../../verification/validation-and-debugging.md`.

## App vs library (Python)

- **Application/service** → may build providers (`TracerProvider`/`MeterProvider`),
  attach OTLP exporters + resource, register them once via
  `trace.set_tracer_provider(...)` etc., and own shutdown.
- **Library/package** → import from `opentelemetry` (the API) only; get a tracer via
  `trace.get_tracer("stable.scope.name", "x.y.z")`; **never** create a provider or exporter.

Full rule: `../../app-vs-library.md`.

## Startup and bootstrap ordering

Two ways to activate telemetry — pick one, never both (double init = duplicate spans):

- **Launcher (`opentelemetry-instrument python main.py`)** — wires the SDK from `OTEL_*`
  env vars and installs auto-instrumentation before your code imports. Manual spans/metrics
  you add still work; you do **not** also build a provider in code.
- **Programmatic** — call your `configure_telemetry()` (see **Bootstrap file**) **once, at
  the very top of the entrypoint, before importing framework/DB modules** so
  auto-instrumentors can patch them. For Gunicorn/Uvicorn with multiple workers, initialize
  **per worker** (e.g. Gunicorn `post_fork` hook), not just in the master.

Python does **not** auto-load `.env`. Use `python-dotenv` (the project's existing loader),
or export vars in the shell / process manager — don't add an observability-only launch path.

## Manual tracing

```python
from opentelemetry import trace
from opentelemetry.trace import StatusCode
# Prefer constants from opentelemetry.semconv.* over string literals where one exists.

# Acquire a tracer ONCE with a stable scope name + version.
tracer = trace.get_tracer("orders.service", "1.0.0")

def process_order(order_id: str, item_count: int, region: str) -> None:
    # Span name = operation class, NOT f"process order {order_id}" (high-cardinality).
    with tracer.start_as_current_span("orders.process") as span:
        # SAFE attributes: bounded, operation-describing, no PII. Custom business
        # dimensions live under a bounded namespace (app.*) and should be documented.
        span.set_attribute("app.order.item_count", item_count)
        span.set_attribute("app.order.region", region)  # bounded enum "us"/"eu"/"apac"
        span.add_event("validation.completed")  # a notable moment inside the op (no extra span)
        try:
            reserve_inventory(order_id)
            charge_payment(order_id)
            # On success leave status UNSET; set OK only when logic CONFIRMS success.
        except Exception as err:
            span.record_exception(err)
            # ERROR needs a short message: type + reason, NOT the traceback (log that).
            span.set_status(StatusCode.ERROR, f"{type(err).__name__}: {err}")
            raise  # re-raise — do NOT swallow after recording
```

`start_as_current_span` ends the span on block exit automatically. Use `start_span` +
manual `span.end()` only when the span must outlive the current scope (and end it on every
path). Message text may contain user input — see `../../review/cardinality.md`.

## Manual metrics

```python
from opentelemetry import metrics

# Acquire a meter ONCE; create each instrument ONCE (module scope), then reuse.
meter = metrics.get_meter("orders.service", "1.0.0")

orders_processed = meter.create_counter(         # monotonically increasing total
    "app.orders.processed", unit="{order}", description="Total orders processed")
orders_in_flight = meter.create_up_down_counter(  # a level that rises and falls
    "app.orders.in_flight", unit="{order}", description="Orders being processed")
order_duration = meter.create_histogram(          # a distribution, recorded in seconds
    "app.orders.duration", unit="s", description="Order processing duration")

def _observe_queue_depth(options):                # ObservableGauge: sampled via callback
    yield metrics.Observation(get_queue_depth(), {"app.queue.name": "orders"})
meter.create_observable_gauge(
    "app.orders.queue_depth", callbacks=[_observe_queue_depth],
    unit="{order}", description="Current order queue depth")

def record_order(region: str, run) -> None:
    base = {"app.order.region": region}           # bounded — safe as a label
    orders_in_flight.add(1, base)
    import time
    start = time.monotonic()
    outcome = "ok"
    try:
        run()
    except Exception:
        outcome = "error"
        raise
    finally:
        labels = {**base, "app.order.outcome": outcome}  # bounded enum
        orders_processed.add(1, labels)
        order_duration.record(time.monotonic() - start, labels)
        orders_in_flight.add(-1, base)
        # ❌ NEVER: {"order_id": ..., "user_id": ..., "url": ...} — cardinality blowup.
```

## Context propagation

The active span is stored in a **`contextvars.ContextVar`**, so it flows automatically
through `async`/`await` and within a task. The neutral model is
`../../signals/context-propagation.md`; Python specifics:

- **asyncio.** `await`ed calls inherit the active span. But `asyncio.create_task(...)`
  copies the context at creation — a task started _outside_ the span won't be a child;
  start the span before creating the task, or pass context explicitly.
- **Threads / executors.** A new `threading.Thread` or `ThreadPoolExecutor` worker does
  **not** inherit the context. Capture and re-attach it:
    ```python
    from opentelemetry import context as otel_context
    token = otel_context.attach(otel_context.get_current())  # in the worker, from captured ctx
    try:
        ...
    finally:
        otel_context.detach(token)
    ```
- **Cross-service.** Inject on the client, extract on the server:
  `from opentelemetry.propagate import inject, extract` — `inject(headers)` before an
  outbound request; `extract(incoming_headers)` to get a context to start the server span
  under. Auto-instrumentation for `requests`/`aiohttp`/WSGI/ASGI does this for you — don't
  double-propagate.

## Enriching auto-instrumentation (don't duplicate)

When a path is already covered (e.g. the Flask/Django/FastAPI auto-instrumentation owns the
`SERVER` span), enrich the existing span rather than wrapping it in a redundant manual span
(neutral decision order in `../../instrumentation-rules.md`):

```python
span = trace.get_current_span()
span.set_attribute("app.tenant.id", tenant_id)   # bounded
```

`trace.get_current_span()` returns a **non-recording** span when none is active, and
`set_attribute`/`set_status` on it are no-ops — so **no guard is needed**. The server span
owns HTTP status handling; see `../../signals/traces.md`.

## SDK wiring pitfalls (Python)

- **Set each provider once.** `set_tracer_provider` / `set_meter_provider` warn and keep
  the first provider on a second call — find existing setup and extend it, don't re-init
  (and don't mix the launcher with a programmatic provider).
- **Metric reader interval for short-lived runs.** `PeriodicExportingMetricReader` defaults
  to ~60s; a CLI/job/eval exits before the first flush → false "no metrics". Shorten it
  (`PeriodicExportingMetricReader(exporter, export_interval_millis=5000)`) and/or call
  `meter_provider.force_flush()` before exit.
- **`BatchSpanProcessor` needs a flush on exit** — see **Bootstrap file** / shutdown.
- **Resource from env.** Build the resource so `OTEL_RESOURCE_ATTRIBUTES` and
  operator-injected values survive; add code-set attributes on top. See `../../discovery/resolve-values.md`.

## Structured logging (stdout, not the OTel logs SDK)

The Python **logs SDK is in development**
([status](https://opentelemetry.io/docs/languages/python/)); don't wire it or an OTLP log
exporter. Emit **structured JSON to stdout** and let a collector forward it. Serialize
exceptions into a single structured field so stack traces don't break the
one-line-per-record contract; correlate to traces. General guidance: `../../signals/logs.md`.

- **`python-json-logger`.** The stdlib `logging` module prints multi-line tracebacks by
  default. A JSON formatter keeps each record on one line and serializes the traceback into
  a structured field; `logger.exception("order.failed", extra={"app.order_id": oid})`
  captures the stack automatically.
- **`structlog`.** Single-line JSON when configured with `JSONRenderer`; the
  `format_exc_info` processor folds the traceback into one string field.

For trace correlation, the logging auto-instrumentation injects `trace_id`/`span_id` into your
stdout records; enable it rather than formatting IDs by hand.

## Auto-instrumentation coverage

For the libraries auto-instrumentation covers and the telemetry they emit, see the
**opentelemetry-auto-instrumentation** skill; enrich those spans rather than duplicating them.

## Testing

The language-neutral test policy (required gate, what to assert, forbidden practices) is in
`../../verification/telemetry-unit-tests.md`. Below is the Python harness — **no backend,
no OTLP, no network.**

```python
import pytest
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from opentelemetry.sdk.trace.export.in_memory_span_exporter import InMemorySpanExporter
from opentelemetry.trace import StatusCode

from myapp.orders import process_order

@pytest.fixture
def exporter():
    exp = InMemorySpanExporter()
    provider = TracerProvider()
    # SimpleSpanProcessor exports synchronously — spans visible immediately, no flush.
    provider.add_span_processor(SimpleSpanProcessor(exp))
    trace.set_tracer_provider(provider)   # register BEFORE code under test resolves its tracer
    yield exp
    exp.clear()

def test_process_order_success(exporter):
    process_order("o-1", item_count=3, region="us")
    spans = exporter.get_finished_spans()
    assert len(spans) == 1
    assert spans[0].name == "orders.process"          # stable, low-cardinality
    assert spans[0].attributes["app.order.region"] == "us"

def test_process_order_error(exporter):
    with pytest.raises(Exception):
        process_order("o-2", item_count=1, region="us")  # force a downstream failure
    span = exporter.get_finished_spans()[0]
    assert span.status.status_code == StatusCode.ERROR
    assert any(e.name == "exception" for e in span.events)
```

**Pitfall — the "0 spans" trap.** `trace.set_tracer_provider(...)` is honored **once per
process**; a second call keeps the first provider (a warning, not an error). If an earlier
test (or an imported module at import time) already set a provider, your in-memory provider
never registers and the exporter sees **0 spans**. Fixes: register the test provider before
any code resolves its tracer (fixture at session scope, or acquire the tracer **lazily inside
the function** under test), and `clear()` the exporter (not the provider) between tests.

### Metrics test (only if metrics were added)

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import InMemoryMetricReader

reader = InMemoryMetricReader()
metrics.set_meter_provider(MeterProvider(metric_readers=[reader]))
# exercise code, then:
data = reader.get_metrics_data()
# assert instrument name, type, unit, datapoint value/count, and BOUNDED attributes.
```

### Logs

The Python logs SDK is in development, so there's no OTel log exporter to assert against.
Verify structured logs are single-line JSON carrying trace correlation, and document logs
export as out of scope.

## Framework notes

Setup and general rules live above / in the neutral references; below are only the
framework-specific deltas.

### Flask / Django / FastAPI (HTTP servers)

The framework auto-instrumentation **owns the `SERVER` span** and sets ERROR on 5xx — add
manual spans only as **children** for domain operations, and set status manually only on
your **business-logic** spans (a 4xx is UNSET on a server span). FastAPI/Starlette is ASGI
(async) — see async notes above; Flask/Django are typically WSGI (sync). For programmatic
setup, initialize telemetry before `create_app()` / before Django loads apps.

### WSGI / ASGI servers (gunicorn / uvicorn)

Workers are **forked processes** — initialize telemetry **per worker** (Gunicorn `post_fork`,
or rely on the launcher which handles this), not only in the master, or forked workers export
nothing. Batch processors must flush on worker shutdown.

### Celery & background workers

Producer→consumer **context propagation across the broker** is the core concern — the Celery
auto-instrumentation injects/extracts context so the task continues the caller's trace. A
worker process is its **own** entrypoint and needs its own telemetry init + shutdown. Batch
consumers use **links**, not one shared parent. See `../../signals/context-propagation.md`.

### Serverless / FaaS

Initialize at **module load** (outside the handler); **flush before returning**
(`force_flush()` on the processors/providers) within the timeout, since the runtime freezes
after the handler; and **extract context from the trigger** so the invocation continues the
caller's trace.

## Bootstrap file (copy-ready, applications only)

A complete **programmatic** bootstrap for when you are not using the
`opentelemetry-instrument` launcher. It hardcodes **no** endpoint/token/header — delivery is
via standard `OTEL_*` env vars read by the exporters. Call `configure_telemetry()` once at
the top of the entrypoint, before importing framework/DB modules. A **library** must not use
this.

```python
# telemetry.py
import atexit
from opentelemetry import trace, metrics
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

def configure_telemetry() -> None:
    # Resource.create() merges OTEL_RESOURCE_ATTRIBUTES + OTEL_SERVICE_NAME from env;
    # add code-set attributes on top. See ../../discovery/resolve-values.md.
    resource = Resource.create()

    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
    trace.set_tracer_provider(tracer_provider)

    meter_provider = MeterProvider(
        resource=resource,
        # Short interval so short-lived processes flush before exit.
        metric_readers=[PeriodicExportingMetricReader(OTLPMetricExporter(), export_interval_millis=5000)],
    )
    metrics.set_meter_provider(meter_provider)

    # BatchSpanProcessor / PeriodicReader buffer — flush and shut down on exit,
    # or the final batch is lost.
    atexit.register(tracer_provider.shutdown)
    atexit.register(meter_provider.shutdown)
```

Swap `.proto.grpc.` for `.proto.http.` (and set `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`,
port 4318) if the collector receiver is HTTP. Don't wire an OTel logs SDK (`LoggerProvider` +
log exporter) — the Python logs SDK is in development; emit structured logs to stdout instead
(see **Structured logging**).
