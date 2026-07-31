# Metrics

Metrics are cheap and continuous, but they are **especially sensitive to
cardinality**: every unique combination of attribute values creates a separate
time series, which costs memory, storage, and money.

## Instrument selection

Choose the instrument by what you are measuring:

- **Counter** — a value that only **increases** (requests served, jobs completed,
  bytes sent). Monotonic; use UpDownCounter for anything that can decrease.
- **UpDownCounter** — a value that goes **up and down** (in-flight requests, queue
  depth, open connections).
- **Histogram** — a **distribution** of values where you care about
  percentiles/buckets (request duration, payload size). Records individual
  measurements; the backend aggregates.
- **ObservableGauge / observable instruments** — a **current value** sampled via a
  callback (pool size, memory in use, temperature). Use an observable counter /
  up-down counter when the source is a cumulative value you read rather than
  increment. Exact instrument names vary by SDK.

Rule of thumb: rates and totals → Counter; levels that move both ways →
UpDownCounter or ObservableGauge; latency/size distributions → Histogram.

## Semantic conventions first — don't duplicate auto-instrumentation

Before creating a custom metric:

1. **Check the semantic conventions** for an existing metric that fits (HTTP server
   latency, DB client duration, runtime memory, etc.). Prefer it.
2. **Check what installed auto-instrumentation already emits** — creating a custom
   metric that duplicates an auto one wastes money and produces conflicting series.
   Common auto metrics include `http.server.request.duration`,
   `http.client.request.duration`, `db.client.operation.duration`,
   `messaging.process.duration`, and `rpc.*.duration`.
3. Only create a **custom** metric for a domain measurement nothing else covers
   (e.g. `app.orders.processed`); name it following the convention structure.

Many RED signals (rate, errors, duration) are derivable from a single latency
histogram, so you rarely need to hand-roll counters the backend can derive.

## Naming principles

- Use a clear, hierarchical, lowercase, dot- or namespace-delimited name following
  the metric conventions (e.g. `http.server.request.duration`,
  `app.orders.processed`).
- Name the **measurement**; put dimensions in attributes and the unit in the unit
  field.
- Keep names **stable** across deployments; renaming breaks dashboards and alerts.
- Prefer an existing semantic-convention metric name when one applies.

## Unit principles

- Always set a **unit** using canonical (UCUM-style) units: `s` for seconds, `By`
  for bytes, `1` for a dimensionless ratio, `{request}`/`{job}` for counts of
  things.
- Record durations in **seconds** unless a convention says otherwise; do not mix
  units for the same concept.

## Description guidance

Give every instrument a short **description** of what it measures and, where useful,
when it changes.

## Safe metric attributes

Attributes (labels) must be **bounded and low-cardinality**:

- HTTP method, normalized status class, route **template** (`/orders/{id}`)
- operation name, outcome enum (`ok` / `error` / `timeout`)
- bounded resource type, region, queue name (if bounded)

Keep the number of attribute keys small. The risky-value list is in
`../review/cardinality.md`.

## Why high-cardinality labels are dangerous

Each distinct attribute set is its own time series. Metrics are stored, indexed,
and billed per series. A single unbounded label (a user ID, a request ID) can
explode one metric into millions of series — driving cost up and query performance
down, and potentially destabilizing the collector/backend.

## Cardinality zones (estimate before shipping)

Active series ≈ the **product** of each attribute's distinct value-count, times the
number of instances. Adding one attribute with 10 values multiplies the total by 10. Estimate the product and keep it in a safe zone:

| Series (per metric) | Zone                                     |
| ------------------- | ---------------------------------------- |
| < 1,000             | Minimal — fine                           |
| 1k – 10k            | Ideal                                    |
| 10k – 50k           | Acceptable — review periodically         |
| 50k – 100k          | Caution — justify each attribute         |
| > 100k              | Danger — remove unbounded attributes now |
| > 1M                | Critical — will destabilize the backend  |

If an attribute is unbounded, it does not belong on a metric (see below).

## Create instruments once

Create instruments once at module load / startup; re-creating them per request (in
handlers, hot loops, or per-call paths) is wasteful and can fragment or duplicate
series.

## When a span attribute is safer than a metric label

If a dimension is **high-cardinality but valuable per-operation** (e.g. an order ID
for debugging one request), put it on a **span attribute**. Spans are stored
per-operation and tolerate higher cardinality; metrics aggregate and do not. Rule of
thumb: _bounded → metric label; unbounded but useful per-operation → span attribute;
raw sensitive → neither._
