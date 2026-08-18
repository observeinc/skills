# Node.js / JavaScript / TypeScript SDK Rule

Node/TS specifics for manual OTel instrumentation — apply alongside the neutral
rules in SKILL.md's Rule routing (fundamentals, traces, metrics, semantic-review,
cardinality, …).

## Files to inspect

- `package.json` (dependencies, `type`, `scripts`)
- lockfiles: `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`
- start scripts (`scripts.start`, `scripts.dev`)
- `"type": "module"` (ESM) vs absence (CommonJS)
- `tsconfig.json` (`module`, `target`, `outDir`) — TypeScript usage and output
- entrypoint files (e.g. `index.js`, `server.ts`, `src/main.ts`)
- `Dockerfile`, `Procfile`, PM2 config (`ecosystem.config.js`), serverless config
  (`serverless.yml`, `template.yaml`, function handlers)
- framework configuration
- existing OpenTelemetry files (e.g. `instrumentation.*`, `tracing.*`, `otel.*`)
- existing logging / metrics / tracing setup

These feed the environment checkpoint (SKILL.md → Workflow step 1): module system,
package manager, TypeScript usage, framework, existing OTel deps, start scripts.

## Dependency guidance

- **Applications** may use: `@opentelemetry/api`, `@opentelemetry/sdk-node`, OTLP
  exporters (`@opentelemetry/exporter-trace-otlp-*`,
  `@opentelemetry/exporter-metrics-otlp-*`), `@opentelemetry/resources`,
  `@opentelemetry/semantic-conventions`, and optionally
  `@opentelemetry/auto-instrumentations-node`.
- **Libraries** depend **only** on `@opentelemetry/api`.
- **Infer the package manager** from the lockfile (npm / pnpm / yarn / bun).
- **Inspect installed versions** before editing code — the SDK and
  semantic-convention package APIs change between major versions.

## Environment variables

| Variable                                                                | Notes                                                                                                                                              |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_SERVICE_NAME`                                                     | Required; else `unknown_service:node`.                                                                                                             |
| `OTEL_TRACES_EXPORTER` / `OTEL_METRICS_EXPORTER` / `OTEL_LOGS_EXPORTER` | Set to `otlp` explicitly. In manual setups these can default to `none` → the signal is silently dropped. `console` prints to stdout for local dev. |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                           | Default `http://localhost:4317`.                                                                                                                   |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                           | `http/protobuf` (port 4318) or `grpc` (port 4317).                                                                                                 |
| `OTEL_RESOURCE_ATTRIBUTES`                                              | `deployment.environment.name`, `service.version`, …                                                                                                |

**Protocol/port:** the `@opentelemetry/auto-instrumentations-node` package defaults the
protocol to `http/protobuf`. Match it to the endpoint port (`http/protobuf` ↔ 4318,
`grpc` ↔ 4317); a mismatch gives `Parse Error: Expected HTTP/` or silent loss. See
`../../verification/validation-and-debugging.md`.

## App vs library (Node.js)

- Application/service → may create a bootstrap file that initializes `NodeSDK`,
  configures OTLP exporters, sets resource attributes, and registers shutdown.
- Library/package → import from `@opentelemetry/api` only; acquire tracers/meters
  with stable scope names; **never** create a `NodeSDK` or configure exporters.

## Startup and bootstrap ordering

**Telemetry bootstrap must execute before application modules are imported.**
Instrumentation hooks must be installed before the instrumented modules are
imported, or they no-op.

### CommonJS startup

```bash
node --require ./instrumentation.cjs app.js
# or
NODE_OPTIONS="--require ./instrumentation.cjs" node app.js
```

### ESM startup

```bash
node --import ./instrumentation.mjs app.mjs
# or
NODE_OPTIONS="--import ./instrumentation.mjs" node app.mjs
```

`--import` is the ESM-friendly mechanism; it ensures the bootstrap (and its
loader hooks) runs before application ESM is evaluated. Avoid relying on a plain
top-level `import './instrumentation.mjs'` placed _after_ other imports — import
hoisting can load instrumented modules first.

### Loading `.env`

Node does **not** auto-load `.env`. Use `node --env-file=.env.local app.js`
(requires **Node 20.6+**), or a loader the project already uses (dotenv). Add the
flag to the existing `start`/`dev` script, not a separate observability-only path.

### TypeScript startup

The actual command depends on the project's runtime:

- **Compiled JavaScript:** build first, then `--require`/`--import` the compiled
  bootstrap (e.g. `dist/instrumentation.cjs` / `dist/instrumentation.mjs`).
- **tsx:** `tsx` can load the bootstrap before app code (e.g. via
  `--import`/`NODE_OPTIONS`); confirm the project's tsx version behavior.
- **ts-node:** ensure the bootstrap is registered before app modules; mind
  CJS/ESM mode from `tsconfig.json`.
- **NestJS:** see **Framework notes → NestJS** below — bootstrap must run before the
  Nest application module is imported.
- **custom runners:** ensure whatever launches the app loads the bootstrap first.

## Manual tracing

```ts
import {trace, SpanStatusCode, type Attributes} from "@opentelemetry/api";
// Prefer constants from the installed semantic-conventions package over string
// literals, e.g. ATTR_HTTP_RESPONSE_STATUS_CODE from '@opentelemetry/semantic-conventions'.

// Acquire a tracer ONCE with a stable scope name + version.
const tracer = trace.getTracer("orders.service", "1.0.0");

interface ProcessOrderInput {
    orderId: string; // application logic only; NEVER a span name or metric label
    itemCount: number;
    region: "us" | "eu" | "apac"; // bounded enum — safe as an attribute
}

export async function processOrder(input: ProcessOrderInput): Promise<void> {
    // Span name = operation class, NOT `process order ${input.orderId}` (high-cardinality).
    return tracer.startActiveSpan("orders.process", async (span) => {
        try {
            // SAFE attributes: bounded, operation-describing, no PII. Custom business
            // dimensions live under a bounded namespace (app.*) and should be documented.
            const attrs: Attributes = {
                "app.order.item_count": input.itemCount,
                "app.order.region": input.region,
            };
            span.setAttributes(attrs);

            // A span EVENT marks a notable moment inside the operation (no extra span).
            span.addEvent("validation.completed");

            await reserveInventory(input);
            await chargePayment(input);
            // On success the status stays Unset (or set OK if your policy requires it).
        } catch (err) {
            // Messages may contain user input — see `../../review/cardinality.md`.
            if (err instanceof Error) span.recordException(err);
            span.setStatus({code: SpanStatusCode.ERROR});
            throw err; // rethrow — do NOT swallow after recording
        } finally {
            span.end(); // ALWAYS end the span, on every path
        }
    });
}
```

## Manual metrics

```ts
import {metrics} from "@opentelemetry/api";

// Acquire a meter ONCE with a stable scope name + version.
const meter = metrics.getMeter("orders.service", "1.0.0");

// --- Create instruments ONCE (module scope), then reuse them. -----------------

// Counter: monotonically increasing total.
const ordersProcessed = meter.createCounter("app.orders.processed", {
    description: "Total number of orders processed",
    unit: "{order}",
});

// UpDownCounter: a level that goes up and down.
const ordersInFlight = meter.createUpDownCounter("app.orders.in_flight", {
    description: "Number of orders currently being processed",
    unit: "{order}",
});

// Histogram: a distribution (durations/sizes), recorded in seconds.
const orderDuration = meter.createHistogram("app.orders.duration", {
    description: "Order processing duration",
    unit: "s",
});

// ObservableGauge: a current value sampled via a callback.
const queueDepthGauge = meter.createObservableGauge("app.orders.queue_depth", {
    description: "Current depth of the order processing queue",
    unit: "{order}",
});
queueDepthGauge.addCallback((result) => {
    result.observe(getCurrentQueueDepth(), {"app.queue.name": "orders"});
});

// --- Recording measurements with BOUNDED attributes only. ---------------------

type Region = "us" | "eu" | "apac";
type Outcome = "ok" | "error" | "timeout";

export async function recordOrder(region: Region, run: () => Promise<void>): Promise<void> {
    const baseLabels = {"app.order.region": region}; // bounded — safe for metrics
    ordersInFlight.add(1, baseLabels);
    const startSeconds = process.hrtime.bigint();

    let outcome: Outcome = "ok";
    try {
        await run();
    } catch (err) {
        outcome = "error";
        throw err;
    } finally {
        const elapsedSeconds = Number(process.hrtime.bigint() - startSeconds) / 1e9;
        // outcome is a bounded enum — safe as a label.
        ordersProcessed.add(1, {...baseLabels, "app.order.outcome": outcome});
        orderDuration.record(elapsedSeconds, {
            ...baseLabels,
            "app.order.outcome": outcome,
        });
        ordersInFlight.add(-1, baseLabels);

        // ❌ NEVER — unbounded labels cause a cardinality explosion:
        //   ordersProcessed.add(1, { order_id, user_id, url });
    }
}
```

## Context propagation

The active span flows through async work via Node's async-context machinery
(conceptually **`AsyncLocalStorage`**, with **`AsyncResource`** for binding a
context to callbacks that would otherwise lose it). Most `await`/promise chains
preserve the active context when you use `startActiveSpan`. You may need to
explicitly bind/propagate context for:

- custom `EventEmitter` flows
- callback queues
- worker pools
- message brokers
- custom protocols

When a callback is scheduled outside the current async context (e.g. stored and
invoked later), bind it to the current context (conceptually via `AsyncResource`)
or capture and re-activate the context with the API's context utilities so child
spans attach to the right parent. See `../../signals/context-propagation.md` for the
language-neutral model and **Framework notes → Queue workers** below for worker/queue
specifics.

## Enriching auto-instrumentation (hooks & span processors)

When a path is **already** covered by auto-instrumentation (HTTP/undici, graphql,
express/fastify), do not wrap it in a redundant manual span — enrich the existing
span (neutral rule + the hook → processor → child-span decision order live in
`../../instrumentation-rules.md`). Node mechanisms:

- **Instrumentation hooks** — most instrumentations expose `requestHook` /
  `responseHook` / `startSpanHook` (and `ignoreRequestHook`). Use them to add
  attributes or call `span.updateName(...)` / `span.setAttribute(...)` on the
  auto-created span (e.g. add `url.template` to undici client spans). Configure in
  the instrumentation's options where the SDK is initialized — not in business code.
- **A custom `SpanProcessor`** — implement `onStart(span)` to normalize names or
  stamp attributes uniformly across many spans (e.g. rename client spans to
  `${method} ${server.address}`).

## SDK wiring pitfalls (Node)

- **Metric reader interval/timeout.** For short-lived runs (CLIs, jobs, evals),
  construct the reader with a fast interval — env vars alone aren't read when you
  build the reader manually, and the export **timeout must be ≤ the interval** (the
  SDK rejects interval 1000 with the default 30000 timeout):
    ```ts
    new PeriodicExportingMetricReader({
        exporter: new OTLPMetricExporter(),
        exportIntervalMillis: Number(process.env.OTEL_METRIC_EXPORT_INTERVAL || 1000),
        exportTimeoutMillis: Number(process.env.OTEL_METRIC_EXPORT_TIMEOUT || 500),
    });
    ```
- **Use `metricReader` (singular)** in `NodeSDK` options; older SDKs silently ignore
  a mistyped `metricReaders` and fall back to env-based setup.
- **Registration order:** `@opentelemetry/instrumentation-http` must be registered
  **before** framework instrumentations (Express/Fastify/Koa) — the `instrumentations`
  array is order-sensitive; framework spans depend on the HTTP span existing first.
- **Prefer named instrumentation packages** over the `auto-instrumentations-node`
  meta-package when the stack is known — smaller dependency surface, fewer surprises.

## Structured logging

- **pino:** pass the error as `{ err }` (pino serializes type/message/stack to a
  single line) — `logger.error({ err }, 'order.charge.failed')`. Don't log
  `err.stack` as the message (multi-line).
- **winston:** enable `format.errors({ stack: true })`, or the stack is silently
  dropped. Emit structured JSON (`format.json()`).

Attach trace correlation via the OTel logs bridge / a helper that reads the active
span, and write single-line JSON to stdout (see `../../signals/logs.md`).

## Auto-instrumentation coverage

For the libraries auto-instrumentation covers and the telemetry they emit, see the
**opentelemetry-auto-instrumentation** skill; enrich those spans rather than duplicating them.

## Testing

The language-neutral test policy (required gate, what to assert, forbidden
practices) is in `../../verification/telemetry-unit-tests.md`. Below is the Node/TS
harness.

### Pattern: Vitest / Jest (TypeScript)

```ts
import {beforeAll, beforeEach, afterAll, describe, it, expect} from "vitest"; // or @jest/globals
import {trace, SpanStatusCode, context} from "@opentelemetry/api";
import {
    BasicTracerProvider,
    InMemorySpanExporter,
    SimpleSpanProcessor,
} from "@opentelemetry/sdk-trace-base";
import {processOrder} from "../src/orders";

let exporter: InMemorySpanExporter;
let provider: BasicTracerProvider;

beforeAll(() => {
    exporter = new InMemorySpanExporter();
    // 1.x supports the spanProcessors constructor option (see Pitfall 1).
    provider = new BasicTracerProvider({
        spanProcessors: [new SimpleSpanProcessor(exporter)],
    });
    // Register ONCE, before the code under test resolves its tracer (Pitfall 2).
    trace.setGlobalTracerProvider(provider);
});

beforeEach(() => {
    exporter.reset(); // isolate spans per test without rebinding the tracer
});

afterAll(async () => {
    exporter.reset();
    await provider.shutdown(); // clean global state — no cross-test contamination
    trace.disable();
    context.disable();
});

describe("processOrder telemetry", () => {
    it("creates a stable, low-cardinality span on success", async () => {
        await processOrder({region: "us"});
        const spans = exporter.getFinishedSpans();
        expect(spans).toHaveLength(1);
        expect(spans[0].name).toBe("orders.process");
        expect(spans[0].name).not.toMatch(/\d{3,}/); // no embedded IDs
        expect(spans[0].attributes["app.order.region"]).toBe("us");
    });

    it("records the exception and sets ERROR status on failure", async () => {
        await expect(processOrder({region: "us", fail: true})).rejects.toThrow();
        const span = exporter.getFinishedSpans()[0];
        expect(span.status.code).toBe(SpanStatusCode.ERROR);
        expect(span.events.some((e) => e.name === "exception")).toBe(true);
    });
});
```

### Two pitfalls to decide BEFORE writing the test

These bite almost every first attempt (0 spans captured, or a `TypeError`
constructing the provider).

**Pitfall 1 — provider construction differs by SDK major.** Read the installed
`@opentelemetry/sdk-trace-base` version (inspect
`node_modules/@opentelemetry/sdk-trace-base/package.json`); do not guess.

| Installed version                       | How to attach the processor                                                                                                                                         |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `>= 1.24` and all `1.x` (e.g. **1.30**) | constructor option: `new BasicTracerProvider({ spanProcessors: [new SimpleSpanProcessor(exporter)] })` (the `addSpanProcessor(...)` method also still works in 1.x) |
| `< 1.24`                                | method only: `const p = new BasicTracerProvider(); p.addSpanProcessor(new SimpleSpanProcessor(exporter));`                                                          |
| `2.x`                                   | constructor option (`spanProcessors:`) only; `addSpanProcessor(...)` was removed                                                                                    |

**Pitfall 2 — the `ProxyTracer` "0 spans" trap.** `trace.getTracer(...)` returns a
`ProxyTracer` that binds on **first use** and caches it. If the module under test
acquires its tracer at **import time**, that binding can resolve to a no-op tracer
_before_ the test registers its provider — so the in-memory exporter sees 0 spans.
Two fixes: (1) register the provider in `beforeAll` before the module is imported
and reset the **exporter** per test (not the provider); or (2) acquire the tracer
**lazily inside the function** — the more robust default for testable code.

### Pattern: metrics (only if metrics were added)

```ts
import {
    MeterProvider,
    InMemoryMetricExporter,
    PeriodicExportingMetricReader,
    AggregationTemporality,
} from "@opentelemetry/sdk-metrics";
import {metrics} from "@opentelemetry/api";

const exporter = new InMemoryMetricExporter(AggregationTemporality.CUMULATIVE);
const reader = new PeriodicExportingMetricReader({exporter});
const provider = new MeterProvider({readers: [reader]});
metrics.setGlobalMeterProvider(provider);
// exercise code, then: await reader.collect(); inspect exporter.getMetrics();
// assert instrument name, type, unit, datapoint value/count, and BOUNDED attributes.
```

### Pattern: logs (only if logs were added)

```ts
import {
    LoggerProvider,
    InMemoryLogRecordExporter,
    SimpleLogRecordProcessor,
} from "@opentelemetry/sdk-logs";
import {logs} from "@opentelemetry/api-logs";

const exporter = new InMemoryLogRecordExporter();
const provider = new LoggerProvider();
provider.addLogRecordProcessor(new SimpleLogRecordProcessor(exporter));
logs.setGlobalLoggerProvider(provider);
// emit a log inside an active span, then: exporter.getFinishedLogRecords();
// assert body/severity, that trace_id/span_id match the enclosing span, and that
// sensitive fields are redacted.
```

### Mocha / Tap / Node test runner

Same shape: build a `BasicTracerProvider` with `InMemorySpanExporter`, register it,
run the code, assert on `getFinishedSpans()`, reset/teardown. Only the test
runner's `describe/it/before/after` equivalents change.

## Framework notes

Setup and the general rules are elsewhere in this file / the neutral references;
below are only the framework-specific deltas.

### Express / Fastify (HTTP servers)

The HTTP auto-instrumentation **owns the `SERVER` span** and sets status **ERROR on
5xx** — so add manual spans only as **children** for domain operations or
uninstrumented calls, and set status manually only on your **business-logic** spans
(a 4xx is **UNSET** on a server span — see `../../signals/traces.md`). The bootstrap
must load before `express`/`fastify`/`http` are imported. Fastify only: a
separately-timed lifecycle hook (`onRequest`/`preHandler`/`onError`) can warrant its
own child span.

### NestJS

Bootstrap-ordering is the main pitfall: `main.ts` imports `AppModule` at the top, so
initializing telemetry inside `main.ts` is **too late**. Launch via
`--require`/`--import` at the compiled bootstrap so instrumentation is in place
before any module loads. Instrument service-layer domain operations, not every
controller.

### Next.js

Next controls the server lifecycle, so `--require`/`--import` often doesn't apply —
use the framework **instrumentation hook** (`register()` in `src/instrumentation.ts`)
and **guard it to the Node runtime** (`process.env.NEXT_RUNTIME === 'nodejs'`; skip
edge). Gotchas: read env vars **inside** `register()` (module-load reads run before
`.env.local` loads); enrich the active span in handlers via
`trace.getActiveSpan()?.setAttribute(...)`; changing `NEXT_PUBLIC_*` requires
clearing the `.next` cache.

### Serverless / FaaS

The process is often frozen/terminated right after the handler returns. Initialize
telemetry at **module load** (outside the handler); **flush before returning** (or
use an immediate exporter / local Collector) within the timeout/memory budget; and
**extract context from the trigger** (HTTP headers, message attributes) so the
invocation continues the caller's trace.

### Queue workers & background jobs

Producer→consumer **context propagation across the broker** is the core concern: the
producer injects the active context into the message before publishing (`PRODUCER`
span); the consumer extracts it and runs the handler with it active (`CONSUMER` span
as child/link). Batch consumers use **links**, not one shared parent. Worker
threads/processes are separate entrypoints and need their **own** bootstrap. See
`../../signals/context-propagation.md`.

## Bootstrap file (copy-ready, applications only)

A complete application bootstrap. It hardcodes **no** endpoints/tokens/headers —
delivery is via standard `OTEL_*` env vars (`OTEL_SERVICE_NAME`,
`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`,
`OTEL_RESOURCE_ATTRIBUTES`). Inspect installed package versions before copying. A
**library** must not use this.

### ESM (`instrumentation.mjs`)

Start with `node --import ./instrumentation.mjs app.mjs` (or `NODE_OPTIONS`).
`--import` ensures the bootstrap runs before application ESM is resolved; don't rely
on a top-level `import` placed after other imports (hoisting loads instrumented
modules first → no-op).

```js
import {NodeSDK} from "@opentelemetry/sdk-node";
import {OTLPTraceExporter} from "@opentelemetry/exporter-trace-otlp-proto";
import {OTLPMetricExporter} from "@opentelemetry/exporter-metrics-otlp-proto";
import {PeriodicExportingMetricReader} from "@opentelemetry/sdk-metrics";
import {getNodeAutoInstrumentations} from "@opentelemetry/auto-instrumentations-node";

const sdk = new NodeSDK({
    traceExporter: new OTLPTraceExporter(), // exporters read OTEL_EXPORTER_OTLP_* from env
    metricReader: new PeriodicExportingMetricReader({
        exporter: new OTLPMetricExporter(),
    }),
    instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();

// Graceful shutdown so buffered telemetry flushes before exit.
const shutdown = (signal) => () => {
    sdk.shutdown()
        .then(() => process.exit(0))
        .catch((err) => {
            console.error(`Error shutting down OpenTelemetry SDK on ${signal}:`, err);
            process.exit(1);
        });
};
process.on("SIGTERM", shutdown("SIGTERM"));
process.on("SIGINT", shutdown("SIGINT"));
```

**CommonJS / TypeScript deltas** — same body, only module syntax / launch differ:

- **CommonJS** (`instrumentation.cjs`): swap imports for
  `const { NodeSDK } = require('@opentelemetry/sdk-node')` etc.; launch with
  `--require ./instrumentation.cjs`.
- **TypeScript** (`instrumentation.ts`): identical to the ESM body (add `: string`
  to the `shutdown` signal param); run the compiled output via `--require`/`--import`,
  or load via `tsx`/`ts-node` (NestJS → Framework notes). Must still execute before
  any application module is imported.
