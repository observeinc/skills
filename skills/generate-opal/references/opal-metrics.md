# Querying metric datasets: align/aggregate, distribution metric double-combine, tags OBJECT, timechart, combining metrics, fill patterns

## Workflow

1. Confirm the dataset has metric interface (check the dataset schema).
2. Identify metric names from the schema or catalog — never guess metric names.
3. **Map each metric to its `m_*` function before writing OPAL.** Read the metric's `type` field (e.g., `"gauge"`, `"tdigest"`, `"cumulativeCounter"`). Using the wrong function is a fatal validation error.
   - `type` = `"gauge"` / `"cumulativeCounter"` / `"delta"` → `m("metric_name")`
   - `type` = `"tdigest"` → `m_tdigest("metric_name")` with `histogram_combine()` (DOUBLE COMBINE pattern)
   - `type` = `"histogram"` → `m_histogram("metric_name")` with `histogram_combine()` (DOUBLE COMBINE pattern)
   - `type` = `"exponentialHistogram"` → `m_exponential_histogram("metric_name")` with `histogram_combine()` (DOUBLE COMBINE pattern)
4. **Verify dimension/tag field names from the dataset schema.** The top-level field that holds metric dimensions varies by dataset — it may be `tags`, `resource_attributes`, `labels`, or something else. NEVER assume `resource_attributes` exists. Check the schema's field list and use only fields that actually exist on the target dataset (see **Dimension Field Names Vary by Dataset** below).
5. Choose output shape:
   - Summary (default) → use `align options(bins: 1)` pattern
   - Time-series (only if user asks for trend/chart) → use `align` with duration pattern
6. For distribution metrics (tdigest, histogram, or exponential histogram), use the DOUBLE COMBINE pattern — `histogram_combine` in both `align` and `aggregate`.
7. For cumulative counters (`type` = `"cumulativeCounter"`), never use `sum()` — use `rate()` or `delta_monotonic()`.

---

## MANDATORY: Read the `type` Field Before Writing align

Before writing ANY `align` expression, you MUST check the metric's `type` field and choose the correct `m_*` function and aggregation. This is the #1 source of query errors.

**NEVER infer metric type from the metric name.** Metric names like "utilization", "ratio", or "percent" do NOT reliably indicate storage type. A metric named `system.cpu.utilization_percent` could be a gauge OR a tdigest — only the `type` field tells you.

**Step-by-step:**

1. Read the metric's `type` field (e.g., `"gauge"`, `"tdigest"`, `"cumulativeCounter"`).
2. Select the `m_*` function based on `type` (see Workflow step 3 above).
3. Select the aggregation function based on `type` (see **Metric type determines align aggregation function** table below).

**Semantic pitfall — "rate" questions need two metrics:**

- When the user asks for a "rate" (e.g., "error rate", "throttle rate"), you need BOTH a numerator metric AND a denominator metric. A single `rate()` gives per-second throughput, not a percentage rate.

**Common mistakes from production evals:**

- `rate(m("k8s_container_cpu_node_utilization_ratio"))` — WRONG: metric `type` is `"gauge"`. Use `avg()`.
- `rate(m("k8s.pod.phase"))` — WRONG: metric `type` is `"gauge"`. Use `last_not_null()`.
- `avg(m("system.cpu.utilization_percent"))` — WRONG: metric `type` is `"tdigest"`. Must use `histogram_combine(m_tdigest(...))` (DOUBLE COMBINE pattern).
- `sum(m("kube_job_status_failed"))` with `bins: 1` — WRONG: metric `type` is `"gauge"`. Produces inflated counts (1440/day = one per scrape). Use `last_not_null()` or `max()`.
- User asks "highest error rates" but agent returns `delta_monotonic(m("Errors"))` — WRONG: that gives counts, not rates. Must divide by `Invocations` to get a rate.

---

## Pattern: Metrics — MUST use align verb

Metric datasets (interface=metric) store pre-aggregated measurements. They MUST use align before aggregation.

### NEVER filter before align on metric datasets

Do NOT use `filter` before `align` — metric datasets require `align` as the first verb. To select a specific metric, use `m("metric_name")` inside `align`. To filter by tag values, put `filter` AFTER `aggregate`.

WRONG — filter before align:

```opal
filter service_name = "api-gateway"
align ...
```

CORRECT — select metric via `m()`:

```opal
align ..., val:sum(m("x"))
aggregate ...
```

CORRECT — include the dimension in group_by, then filter on the alias:

```opal
align 5m, cpu:avg(m("system.cpu.utilization"))
aggregate avg_cpu:avg(cpu), group_by(svc:string(tags."service.name"), ns:string(tags."k8s.namespace.name"))
filter svc = "scheduler"
filter ns = "o2"
```

WRONG — filtering on original field paths after aggregate (fields no longer exist — `"tags" does not exist after aggregate`):

```opal
align 5m, cpu:avg(m("system.cpu.utilization"))
aggregate avg_cpu:avg(cpu), group_by(svc:string(tags."service.name"))
filter string(tags."k8s.namespace.name") = "o2"
```

After `aggregate`, the ONLY columns that exist are the `group_by` aliases and aggregate results. To filter on a dimension post-aggregate, that dimension MUST appear in `group_by()` — then filter on the alias name, not the original field path.

### Scope qualifiers — filtering metrics by user-specified subset

When the user asks for a specific _type_ of request, operation, or category within a metric (e.g., "login latency", "write throughput", "payment errors"), the qualifying term MUST become a filter in the pipeline. Check the metric's `heuristics.tags` for a dimension that expresses the qualifier (e.g., `http.route`, `http.target`, `url.path`, `rpc.method`, `verb`), include it in `group_by`, and filter post-aggregate.

Example — user asks for "login latency of the apiserver by namespace last hour" against metric `apiserver.request.duration` (exponentialHistogram). "login" is a scope qualifier → filter on an HTTP path/route dimension:

```opal
align options(bins: 1), combined:histogram_combine(m_exponential_histogram("apiserver.request.duration"))
aggregate combined:histogram_combine(combined), group_by(ns:string(resource_attributes."k8s.namespace.name"), path:string(resource_attributes."http.target"))
filter contains(path, "login")
make_col p50:histogram_quantile(combined, 0.50),
         p95:histogram_quantile(combined, 0.95),
         p99:histogram_quantile(combined, 0.99)
make_col p50_ms:round(p50 * 1000, 2), p95_ms:round(p95 * 1000, 2), p99_ms:round(p99 * 1000, 2)
pick_col valid_from, valid_to, ns, path, p50_ms, p95_ms, p99_ms
sort desc(p99_ms)
```

Example — user asks for "write throughput by host" against metric `disk.io.ops` (cumulativeCounter). "write" is a scope qualifier → filter on an operation direction dimension:

```opal
align options(bins: 1), ops_rate:rate(m("disk.io.ops"))
aggregate total_rate:sum(ops_rate), group_by(host:string(tags."host.name"), direction:string(tags."direction"))
filter direction = "write"
pick_col valid_from, valid_to, host, total_rate
sort desc(total_rate)
```

**Exception: Prometheus-style metrics** stored in datasets like **Kubernetes Explorer/Prometheus Metrics** — filter on `labels.*` fields BEFORE `align`. Do NOT filter on the `metric` column — `m("metric_name")` inside `align` already selects the metric by name.

```opal
filter labels.status ~ <5*>
filter labels.k8s_cluster_name = 'k8s.eu-1.observeinc.com'
```

Note: Prometheus datasets use `labels.*` for metric labels, while OpenTelemetry metric datasets use `tags.*` or `resource_attributes.*`. Check the dataset schema. Dimensions are typically nested inside an OBJECT column — **always use the dataset schema to find the correct access path** rather than assuming bare field names exist at the top level.

**Complete Prometheus-style query examples** (filter labels → align → aggregate):

Gauge metric — last known value grouped by dimensions:

```opal
filter string(labels.service_name) = "transformer"
filter string(labels.deployment_environment) = "prod"
align options(bins: 1), startup_dur:last_not_null(m("transformer_startup_duration_seconds"))
aggregate max_dur:max(startup_dur), group_by(version:string(labels.service_version), pod:string(labels.k8s_pod_name))
sort desc(max_dur)
```

Counter metric — request rate grouped by dimensions:

```opal
filter string(labels.deployment_environment) = "prod"
align options(bins: 1), req_rate:rate(m("nginx_ingress_controller_requests_total"))
aggregate total_rate:sum(req_rate), group_by(status:string(labels.status), pod:string(labels.k8s_pod_name))
```

**Default to summary output.** Unless the user explicitly asks for a time-series, trend, or chart, use `align options(bins: 1)` to collapse the entire time window into a single bin.

### Metric type determines align aggregation function

The metric type also determines which aggregation function to wrap around the `m_*` accessor inside `align`. **Selection rule** — for each metric:

1. If the metric definition provides a rollup (`rollup`, `defaultRollup`, or `heuristics.rollup`), use it as the `align` aggregation function.
2. If no rollup is provided, fall back based on the metric's `type` field per the table below.

| Metric `type`                                    | Fallback `align` function (no rollup provided) | Other reasonable choices            | Typically misleading                                                                |
| :----------------------------------------------- | :--------------------------------------------- | :---------------------------------- | :---------------------------------------------------------------------------------- |
| `gauge`                                          | `avg()`                                        | `max()`, `min()`, `last_not_null()` | `rate()`, `delta_monotonic()` — treat decreases as counter resets, wrong for gauges |
| `cumulativeCounter`                              | `rate()`                                       | `delta_monotonic()`                 | `sum()` — sums raw ever-increasing values, producing meaningless totals             |
| `delta`                                          | `sum()`                                        | `rate()`                            | `last_not_null()` — discards all but one sample per bin, losing delta contributions |
| `tdigest` / `histogram` / `exponentialHistogram` | `histogram_combine()` (DOUBLE COMBINE)         | —                                   | `avg()`, `sum()` — type error; distributions require `histogram_combine`            |
| unknown / missing `type`                         | `avg()`                                        | —                                   | —                                                                                   |

- `gauge` default is `avg`. Use `last_not_null` for snapshot/state queries (e.g., "is the host up?"), `max`/`min` for peak/trough analysis.
- `cumulativeCounter` default is `rate`. Use `delta_monotonic` when you need absolute change instead of per-second rate. Never `sum` — it sums raw counter values across a bin, not deltas.
- `delta` default is `sum`. Use `rate` when the user asks for per-second throughput rather than total change per bin.
- `rate()` on a gauge will silently treat any value decrease as a counter reset. The query will run without error, but the result will be wrong for metrics that naturally fluctuate (CPU, memory, temperature).

### align expressions MUST use metric accessors — dimensions go in aggregate group_by

Every expression in `align` must wrap a metric accessor (`m()`, `m_tdigest()`, `m_histogram()`, `m_exponential_histogram()`). You CANNOT extract tag/dimension values inside `align` — dimensions are extracted in `aggregate`'s `group_by()`.

WRONG — extracting a dimension in align causes "Please select a metric" error:

```opal
align 5m, val:sum(m("cpu")), env:last_not_null(string(resource_attributes."deployment.environment"))
```

CORRECT — dimensions go in aggregate's group_by:

```opal
align 5m, val:sum(m("cpu"))
aggregate total:sum(val), group_by(env:string(resource_attributes."deployment.environment"))
```

### Summary (one row per group): align options(bins: 1) — THE DEFAULT

```opal
align options(bins: 1), rate:sum(m("my_counter_metric"))
aggregate total:sum(rate), group_by(some_dimension)
fill total:0
```

CRITICAL: `aggregate` without `group_by()` produces a time series, NOT a single scalar. For single-stat panels, use `group_by()` with no arguments.

CRITICAL: `align` and `aggregate` go on separate lines.
The `fill` verb goes on the line directly AFTER aggregate, then make_col on the next line:

```opal
align options(bins: 1), val:sum(m("x"))
aggregate total:sum(val), group_by(dim)
fill total:0
make_col rate:float64(total)/100.0
```

### Time-series: align with duration — ONLY when user asks for trends/charts

```opal
align 5m, rate:sum(m("my_counter_metric"))
aggregate rate_per_5min:sum(rate), group_by(service_name)
```

### Rolling window on metrics — smoothed time-series

Use when the user asks for a rolling, sliding, or moving calculation (e.g., "7-day rolling failure rate", "moving average of CPU"). This produces a time-series where each point is computed from a lookback window. Load [opal-transforms](opal-transforms.md) for full `window`/`frame` syntax reference.

**This pattern is for metric datasets**, which must use `align` (not `timechart`). For span/log datasets, `timechart` supports `frame()` as a direct argument — see "Rolling window" in [opal-aggregation](opal-aggregation.md).

**Pattern:** `align` with a fine interval → `aggregate` by dimension → `window(func(), group_by(dim), frame(back:N))`.

IMPORTANT: Do NOT put `frame()` inside `options()` on `align`. `frame()` is either a separate positional argument to `align` (for simple sliding window aggregation) or used inside `window()` after `aggregate` (for cross-column rolling calculations). `options()` only accepts `bins`, `min_bin`, `empty_bins`.

Rolling rate example (7-day rolling failure rate by service):

```opal
align 1h, errors:sum(m("error_count")), requests:sum(m("request_count"))
aggregate total_errors:sum(errors), total_requests:sum(requests), group_by(svc:string(tags."service.name"))
make_col rolling_errors:window(sum(total_errors), group_by(svc), frame(back:7d)), rolling_requests:window(sum(total_requests), group_by(svc), frame(back:7d))
make_col rolling_failure_rate:100.0 * float64(rolling_errors) / float64(rolling_requests)
```

Rolling average example (24-hour moving average of a gauge):

```opal
align 1h, val:avg(m("cpu_usage_percent"))
aggregate avg_cpu:avg(val), group_by(host:string(tags."host.name"))
make_col rolling_avg:window(avg(avg_cpu), group_by(host), frame(back:24h))
```

**Rules:**

- Choose an `align` interval that is finer than the rolling window (e.g., `1h` align for a `7d` window)
- Use `frame(back:<duration>)` for the lookback — see [opal-transforms](opal-transforms.md) for `frame_exact` and other variants
- The result is a time-series, not a summary
- For high-cardinality dimensions, pre-filter to top-N entities or ask the user to scope

### Distribution metrics — DOUBLE COMBINE pattern (CRITICAL)

Applies to ALL distribution metric types: tdigest, histogram, and exponential histogram.
All three use the same `histogram_combine` / `histogram_quantile` functions — only the `m_*` accessor differs.

`histogram_combine` must appear TWICE — once in `align`, once in `aggregate`. Quantiles are extracted in `make_col`:

```opal
align options(bins: 1), combined:histogram_combine(m_tdigest("my_duration_metric"))
aggregate combined:histogram_combine(combined), group_by(some_dimension)
make_col p50:histogram_quantile(combined, 0.50),
         p95:histogram_quantile(combined, 0.95),
         p99:histogram_quantile(combined, 0.99)
make_col p50_ms:p50/1000000, p95_ms:p95/1000000, p99_ms:p99/1000000
```

For histogram metrics, use `m_histogram()` instead of `m_tdigest()`:

```opal
align options(bins: 1), combined:histogram_combine(m_histogram("apiserver.request.duration"))
aggregate combined:histogram_combine(combined), group_by(namespace:string(resource_attributes."k8s.namespace.name"))
make_col p50:histogram_quantile(combined, 0.50),
         p95:histogram_quantile(combined, 0.95),
         p99:histogram_quantile(combined, 0.99)
```

For exponential histogram metrics, use `m_exponential_histogram()`:

```opal
align options(bins: 1), combined:histogram_combine(m_exponential_histogram("my_exp_hist_metric"))
aggregate combined:histogram_combine(combined), group_by(some_dimension)
make_col p50:histogram_quantile(combined, 0.50),
         p99:histogram_quantile(combined, 0.99)
```

WHY double-combine: align combines digests/buckets within each time bucket; aggregate combines across all buckets. Omitting either produces wrong percentiles.

CRITICAL: Using `m()` on a metric whose `type` is `"tdigest"`, `"histogram"`, or `"exponentialHistogram"` causes a fatal validation error. Always check the metric's `type` and use the corresponding `m_*` function.

### Combining multiple metrics (e.g., error rate)

```opal
align options(bins: 1),
      requests:sum(m("my_request_count_metric")),
      errors:sum(m("my_error_count_metric"))
aggregate total_requests:sum(requests),
          total_errors:sum(errors),
          group_by(some_dimension)
fill total_errors:0
make_col error_rate:100.0 * float64(total_errors) / float64(total_requests)
sort desc(error_rate)
```

### Dimension Field Names Vary by Dataset (CRITICAL)

The field that holds metric dimensions/tags is NOT the same across all metric datasets. You MUST check the dataset schema to determine the correct top-level field name. Common variants:

| Dataset type        | Dimension field       | Example access                       |
| :------------------ | :-------------------- | :----------------------------------- |
| OTel metrics (some) | `tags`                | `tags."service.name"`                |
| OTel metrics (some) | `resource_attributes` | `resource_attributes."service.name"` |
| Prometheus metrics  | `labels`              | `labels."k8s_pod_name"`              |
| APM/Tracing metrics | `tags`                | `tags."service.name"`                |
| AWS CloudWatch      | `FIELDS`              | `FIELDS."namespace"`                 |

**Using the wrong field name is a fatal validation error** — e.g., referencing `resource_attributes."service.name"` when the dataset only has `tags` produces: `the field "resource_attributes" does not exist`. Always confirm the field exists in the schema before using it in `group_by`, `filter`, or `make_col`.

### Tags OBJECT grouping & column naming

Metric datasets group by a compound dimension OBJECT (often `tags`, `resource_attributes`, or `labels` — check the schema):
`group_by(svc:string(tags."service.name"))` → output column: `svc`
Use the alias name (not the expression) in subsequent `make_col`/`sort`/`filter`.

Null tags: metric tags can be null. Add `filter not is_null(svc)` after `aggregate` to exclude.

### Timechart for time-series visualization

```opal
timechart 5m, val:sum(m("metric_name")), group_by(service_name)
```

Output columns: `_c_valid_from`, `_c_valid_to` for time bounds; `_c_bucket` for bin label.

## Snapshot Queries — `options(bins: 1)` + `statsby`

Use for **"current state"** queries — e.g., "which hosts are down?", "what processes use the most CPU?":

```opal
align options(bins: 1), host_status:last_not_null(m("up"))
fill host_status:0
statsby host_status:last_not_null(host_status), group_by(node_name:resource_attributes."k8s.node.name")
filter host_status = 0
```

## Scoped Ranking — Answering "X on my top-N Y" Questions

Use when the question asks about one dimension scoped to the top-N of another dimension within the same metric dataset — e.g., "processes on my busiest hosts", "containers in my most loaded nodes", "queries from the services with highest error rates".

### Single-query approach

Group by BOTH the detail dimension and the scoping dimension. `topk` naturally surfaces the highest combinations:

```opal
align options(bins: 1), avg_cpu:avg(m("process.cpu.utilization"))
aggregate avg_cpu:avg(avg_cpu), group_by(host_name:resource_attributes."host.name", process_name:resource_attributes."service.name")
topk 50, max(avg_cpu)
```

### Multi-step investigation approach (more precise)

For precise scoping, use two query cards in sequence:

**Step 1** — Identify the scoping entities (e.g., busiest hosts by total resource usage):

```opal
align options(bins: 1), cpu:avg(m("process.cpu.utilization"))
aggregate total_cpu:sum(cpu), group_by(host_name:resource_attributes."host.name")
topk 10, max(total_cpu)
```

**Step 2** — After examining results, generate a follow-up query filtered to those entities:

```opal
align options(bins: 1), cpu:avg(m("process.cpu.utilization"))
aggregate avg_cpu:avg(cpu), group_by(host_name:resource_attributes."host.name", process_name:resource_attributes."service.name")
filter host_name = "host-1" or host_name = "host-2" or host_name = "host-3"
topk 20, max(avg_cpu)
```

**When summarizing results**, always tie findings back to the constraint: explicitly state which hosts were identified as "busiest" and what processes dominate on those specific hosts. Do not draw generic conclusions across all entities.

---

## Complementary Metric Categories

A single user concept often maps to multiple metric categories. When you have multiple candidate metrics spanning different categories — usage/utilization metrics, boolean condition/status metrics, ratio metrics, counter metrics — generate separate query cards for each category rather than picking only one. They provide complementary views of the same concept.

Don't discard metrics just because you already found one that seems sufficient. Usage metrics show resource consumption levels; condition/status metrics show whether thresholds have been breached; ratio metrics normalize across different-sized entities. Each adds information the others lack.

Common complementary pairs:

- Resource usage gauge + condition/status boolean (e.g., memory bytes used vs memory-pressure flag)
- Absolute counter + utilization ratio (e.g., CPU seconds vs CPU utilization percentage)
- Request rate + error rate + latency (RED methodology)

When multiple categories are available, create a query card for each and synthesize across them in the analysis. A user asking about "pressure" or "saturation" benefits from seeing both the raw utilization and whether the system has flagged it as problematic.

---

## Count Entities from Counters — Two-Phase Aggregation

When the question asks "how many [entities] had [something]" and the data comes from a counter metric, you need a **two-phase aggregation**: first compute per-entity deltas, then count entities with non-zero deltas.

This applies to questions like: "how many pods restarted?", "how many services had errors?", "how many hosts had OOM kills?"

WRONG — returns total restart count, not pod count:

```opal
align options(bins: 1), restarts:delta_monotonic(m("kube_pod_container_status_restarts_total"))
aggregate total_restarts:sum(restarts), group_by()
```

CORRECT — returns count of pods that had at least one restart:

```opal
align options(bins: 1), restart_delta:delta_monotonic(m("kube_pod_container_status_restarts_total"))
aggregate restart_delta:sum(restart_delta), group_by(pod:string(labels."k8s_pod_name"))
filter restart_delta > 0
statsby restarted_pods:count(), group_by()
```

Steps:

1. `align` + `delta_monotonic` to compute the change per time series
2. `aggregate` grouped by entity to get per-entity totals
3. `filter > 0` to keep only entities that experienced the event
4. `statsby count()` to count the matching entities

**Distinguish the question type:**

| User asks…                       | Pattern                                    | Example                                                                               |
| :------------------------------- | :----------------------------------------- | :------------------------------------------------------------------------------------ |
| "How many restarts happened?"    | Sum the deltas                             | `aggregate total:sum(restart_delta), group_by()`                                      |
| "How many pods restarted?"       | Two-phase: group by pod, filter > 0, count | See above                                                                             |
| "Which pods restarted the most?" | Group by pod, sort                         | `aggregate restarts:sum(restart_delta), group_by(pod:...)` then `sort desc(restarts)` |

---

## Cumulative Counters — Critical Rules

Metrics with `type` = `"cumulativeCounter"` are monotonically increasing (common in Prometheus-style metrics with `_total` or `_count` suffixes).

### Never use `sum()` directly on a cumulative counter

- **Per-second rate:** `align rate(m("http_requests_total"))`
- **Change over window:** `align delta_monotonic(m("http_requests_total"))`
- **Change over full range:** `align options(bins: 1), change:delta_monotonic(m("http_requests_total"))`

### Quick reference: counter metric functions

- **Per-second rate** — `rate(m("..."))`
- **Change per window** — `delta_monotonic(m("..."))`
- **Change over full range** — `delta_monotonic(m("..."))` with `options(bins: 1)`
- **WRONG** — `sum(m("..."))` on a counter

## Fill Patterns for Metrics

### fill syntax placement (CRITICAL)

`fill` goes on the line directly AFTER aggregate/timechart. Then the next verb on the following line.

CORRECT:

```opal
timechart 5m, total:count(), group_by(service_name)
fill total:0
make_col rate:total / 5.0
```

WRONG — blank line or extra verbs between aggregate and fill:

```opal
timechart 5m, total:count(), group_by(service_name)
make_col something:1
fill total:0
```

### fill with frame() — bounded forward fill

`fill` without `frame()` materializes **every** empty bucket in the query window — suitable for ad-hoc queries.

`fill` with `frame(ahead: <duration>)` only fills a bounded forward horizon beyond observed data — required for materialized dataset definitions.

Fill all empty buckets in the query window:

```opal
fill total:0
```

Fill only a bounded forward horizon:

```opal
fill frame(ahead:15m), total:0
```

Rules for `frame(ahead: ...)`:

- `ahead` must be strictly between zero and one week
- `ahead` must be at least as large as the alignment step (e.g., `frame(ahead:15m)` for 5m bins)
- `back` is not used for fill

### Fill strategies

`fill` values must be compile-time constants (`0`, `float64_null()`, `"string"`, etc.). The verb replaces NULLs in empty time buckets with the specified constant.

| Strategy       | Syntax                    | Use for                                                                           |
| :------------- | :------------------------ | :-------------------------------------------------------------------------------- |
| Zero           | `fill col:0`              | Counters, counts, sums — absence means zero                                       |
| Typed null     | `fill col:float64_null()` | Averages, gauges — absence means no data (use `int64_null()` for integer columns) |
| Specific value | `fill col:42`             | Custom default                                                                    |

For gauges where absence means "unchanged", use `last_not_null()` in `align` to carry forward the last observation, then `fill` with `0` or a typed null for any remaining gaps:

```opal
align 5m, val:last_not_null(m("gauge_metric"))
aggregate latest:last_not_null(val), group_by(dimension)
fill latest:0
```

### Multiple columns with different fill values

```opal
timechart 5m, requests:count(), avg_latency:avg(duration_ms), group_by(service_name)
fill requests:0, avg_latency:float64_null()
```

## Unit inference for duration/latency distribution metrics

When computing percentiles from distribution metrics (tdigest, histogram, exponential histogram), infer the unit from the metric name and value magnitude:

- Metric names with `duration`, `latency`, `response_time` and values >1,000,000 → nanoseconds. Divide by 1,000,000.
- Metric names with `_ms` or `_milliseconds` → already ms.
- After computing percentiles, add: `make_col p50_ms:round(p50/1000000, 2), p99_ms:round(p99/1000000, 2)`.
- `duration_ms(x)` tells Observe to render a number as a duration — it does NOT convert units.

## Period-over-Period Comparison — timeshift / timewrap

### timeshift — shift data forward or backward in time

Adds a compile-time duration to all temporal columns. Useful for clock correction, backtesting, or aligning two copies of the same dataset at different offsets.

```opal
timeshift 1h
timeshift -30m
```

Shift forward by `1h` or backward by `30m`. Only works on temporal datasets (not Tables). Metric alignment metadata is updated consistently.

### timewrap — overlay multiple time periods

Replicates the input N times, shifting each copy by multiples of a fixed duration, and adds a label column for comparison. Use for period-over-period charts ("today vs yesterday vs last week").

Produces 4 copies (`"now"`, `"1d ago"`, `"2d ago"`, `"3d ago"`). The new string column `period` identifies each copy:

```opal
timewrap 1d, 4, "period"
```

Also emits an int64 index column alongside the label:

```opal
timewrap 1h, 3, "window_label", "window_index"
```

The label column is added to the primary key and grouping key. Temporal alignment is cleared. Only works on temporal datasets.

**Example — compare this week vs last week metrics:**

```opal
align 1h, cpu:avg(m("system.cpu.utilization"))
aggregate avg_cpu:avg(cpu), group_by(host:string(tags."host.name"))
timewrap 7d, 2, "week"
```

---

## Pitfalls

- **Always name `align` output columns.** Use `name:func(m("..."))`. Auto-generated names are unpredictable.
- **Use the same name through the pipeline.** Keep the column name consistent across `align`, `aggregate`, and `fill`.
- **Don't use `timechart` or `count()` for native metric datasets.** Use `align` + `aggregate`. Exception: event-formatted metric datasets (AWS CloudWatch).
- **`m()` and other metric accessors can ONLY appear inside `align`.** `make_col x:m(...)` is invalid — metric values are extracted exclusively through `align`.
- **`align` does NOT accept `group_by`.** Dimensions are extracted in `aggregate`'s `group_by()`, never in `align`.
- **`frame()` is a SEPARATE argument to `align`, NOT an option inside `options()`.** `options()` only accepts `bins`, `min_bin`, and `empty_bins`. To use a sliding window with `align`, pass `frame()` as a positional argument:

  ```opal
  align 1m, frame(back:10m), avg_mem:avg(m("memory_used"))
  ```

  These are WRONG:

  ```opal
  align 1m, options(frame: 10m), avg_mem:avg(m("memory_used"))
  align options(frame: 7d), val:sum(m("x"))
  ```

- **`make_col` before `align` is discarded.** `align` resets the column set. If you need a derived column in `aggregate group_by()`, create it AFTER `align`.
- **Don't use `_c_valid_from` for metric visualizations.** The time column is the one named by `validFromField` in the schema; reference it generically with `row_start_time()` (returns the `validFromField` value) rather than guessing its name.
- **`group_by()` is required in `aggregate`.** Even for a global aggregate, include `group_by()` with no arguments.
- **NEVER include temporal columns in any `group_by()`.** The time axis is implicit — each verb handles time bucketing automatically. To get time-series output grouped by a dimension, use `group_by(dimension)` only.
- **Filter on labels before `align` only for Prometheus-style datasets.** For OTel metrics, filter after `aggregate`.
- **Use `topk`/`bottomk` after `aggregate` — not `sort` + `limit`.** Always use an aggregate scoring function: `topk 20, max(col)` or `bottomk 10, min(col)`. Never pass a bare column to `topk` — `topk 20, col` is invalid.
- **Don't `sort` time-series output.** After `align` + `aggregate` (without `options(bins: 1)`), the output is a time series ordered by `valid_from`. Adding `sort desc(value)` scrambles the time axis, breaking chart rendering. Use `sort` only on summary output (`options(bins: 1)`) where each row is a group, not a time bin. For ranking time-series groups, use `topk` instead.
- **Don't reuse existing column names in `align` or `group_by`.** Check the dataset schema before choosing names. `align` aliases that collide with existing columns cause `"cannot create column X more than once"`. `group_by` aliases that collide cause `"attempting to overwrite existing column"`. For `group_by`, use the bare column name when no casting is needed, or pick a new alias — `group_by(host, datacenter)` is fine, but `group_by(host:string(host))` is rejected because `host` already exists. For `align`, use a distinct alias — `cpu_avg:avg(m("cpu_utilization"))` instead of `cpu:avg(...)` when a `cpu` column already exists.
- **Don't assume dimension field names.** The top-level field for metric dimensions varies: `tags`, `resource_attributes`, `labels`, `FIELDS`, etc. Using the wrong one (e.g., `resource_attributes` when the dataset has `tags`) causes `"field does not exist"` errors. Always verify from the dataset schema.
