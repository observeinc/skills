# OPAL Aggregation Strategy — Match User Intent

## Summary (default) — one row per group

Use when the question asks for comparisons, rankings, totals, or KPIs. This is the DEFAULT.

- Keywords: "by service", "compare", "which service", "top N", "error rate", "how many", "what is the p99", "summary", "breakdown"
- Spans/logs: `statsby ... group_by(dimension)`
- Metrics: `align options(bins: 1), val:sum(m("x"))` then `aggregate total:sum(val), group_by(dimension)`

## Per-timestamp — aggregation without time bucketing

Use when you want to aggregate at each point along the timeline without merging rows into fixed-width bins. Preserves the input's temporal kind (Event, Interval, or Resource).

- Keywords: "at each timestamp", "per-event", "point-by-point", "without bucketing", "per observation"
- Spans/logs: `timestats ProcessCount:count(), group_by(service_name)`
- When `group_by` is omitted, default grouping is the input's primary key (often per-entity per-timestamp)

`timestats` vs `timechart`: `timechart` merges rows into fixed bins (e.g., 5m buckets); `timestats` keeps each original timestamp and computes aggregates at that instant. Use `timestats` when you need the native event cadence without artificial binning.

## Time-series — multiple rows over time

Use ONLY when the question explicitly asks for trends, charts, or time-bucketed data.

- Keywords: "over time", "trend", "time series", "chart", "hourly", "daily", "weekly", "per hour", "per minute", "per day", "per week", "progression", "pattern over time"
- Spans/logs: `timechart 5m, count:count(), group_by(dimension)`
- Metrics: `align 5m, val:sum(m("x"))` then `aggregate total:sum(val), group_by(dimension)`

## Rolling window / moving aggregate — smoothed time-series

Use when the question asks for a rolling, sliding, or moving calculation over a time window. A rolling window produces a time-series where each point is computed from a lookback window (e.g., "7-day rolling average"). This is NOT a summary — it is a time-series with smoothing. Also load [opal-metrics](opal-metrics.md) if the source is a metric dataset.

- Keywords: "rolling", "sliding", "moving average", "N-day average", "N-day rate", "7-day", "30-day", "rolling window"
- Spans/logs: `timechart <interval>, frame(back:<window>), agg:func(), group_by(dim)` — `frame()` is a direct positional argument to `timechart`
- Metrics: `align <interval>, ...` then `aggregate ..., group_by(dim)` then `make_col smoothed:window(avg(col), group_by(dim), frame(back:<window>))`

### Spans/logs — native `timechart frame()`

`timechart` accepts `frame()` as a direct positional argument for sliding window aggregation. This is the simplest way to compute rolling aggregates on span or log datasets:

    timechart 1h, frame(back:7d), error_count:sum(is_error), total:count(), group_by(service_name)

IMPORTANT: `frame()` is a **separate argument** to `timechart`, NOT an option inside `options()`. `options()` only accepts `bins`, `min_bin`, `empty_bins`, and `offset`.

    CORRECT: timechart 1h, frame(back:7d), total:count(), group_by(svc)
    WRONG:   timechart 1h, options(frame: 7d), total:count(), group_by(svc)

When you need to **derive a new column from multiple rolling aggregates** (e.g., dividing rolling errors by rolling requests), use the two-step `window()` approach instead, since `timechart` cannot compute cross-column ratios in a single verb:

    timechart 1h, errors:sum(is_error), total:count(), group_by(service_name)
    make_col rolling_errors:window(sum(errors), group_by(service_name), frame(back:7d)), rolling_total:window(sum(total), group_by(service_name), frame(back:7d))
    make_col failure_rate:if(rolling_total > 0, 100.0 * float64(rolling_errors) / float64(rolling_total), 0.0)

### Metrics — `align` + `aggregate` + `window(frame())`

Metric datasets must use `align` (not `timechart`), so rolling windows require the `window()` function in a `make_col` step after aggregation. See "Rolling window on metrics" in [opal-metrics](opal-metrics.md).

The same `frame()` rule applies to `align`: `frame()` is a **separate positional argument**, never inside `options()`:

    CORRECT: align 1m, frame(back:10m), avg_val:avg(m("metric"))
    WRONG:   align 1m, options(frame: 10m), avg_val:avg(m("metric"))

Example — 7-day rolling failure rate from metrics:

    align 1h, errors:sum(m("error_count")), requests:sum(m("request_count"))
    aggregate total_errors:sum(errors), total_requests:sum(requests), group_by(svc:string(tags."service.name"))
    make_col rolling_errors:window(sum(total_errors), group_by(svc), frame(back:7d)), rolling_requests:window(sum(total_requests), group_by(svc), frame(back:7d))
    make_col rolling_failure_rate:100.0 * float64(rolling_errors) / float64(rolling_requests)

## Histogram — approximate distribution bins

Use when the question asks for a histogram or distribution of a numeric/duration column.

    histogram options(bins:20), col:latency_ms, group_by(service_name)

`histogram` aggregates a numeric, duration, or tdigest column into approximate histogram bins for the query window. The output is a table of bin boundaries and counts.

## Session aggregation — make_session

Use when grouping rows that are close in time into sessions (e.g., user sessions, network connections).

    make_session options(expiry:duration_min(30)), session_count:count(), group_by(user_id)

`make_session` groups rows close in time into sessions and evaluates aggregates once per session. Output is an Interval dataset whose intervals span each session. The `expiry` option sets the inactivity timeout for session boundaries.

**When in doubt, default to summary.**

---

## Entity Count vs Event Count — Match the Noun

When the question asks "how many [entities] [verb]ed", the answer is a **count of distinct entities**, not a sum of events.

| User asks…                      | The noun is…        | Aggregation                                      |
| :------------------------------ | :------------------ | :----------------------------------------------- |
| "How many errors happened?"     | events              | `count()` or `sum(delta)`                        |
| "How many pods restarted?"      | entities (pods)     | Group by pod → filter > 0 → `count()`            |
| "How many services had errors?" | entities (services) | Group by service → filter errors > 0 → `count()` |
| "How many hosts went down?"     | entities (hosts)    | Group by host → filter status = down → `count()` |

For event datasets, use `count_distinct(entity_column)`. For counter metrics, use the two-phase aggregation pattern in [opal-metrics](opal-metrics.md) (group by entity, filter > 0, then count).

---

## Cardinality Awareness — Scope Dimensions for Time-series

When generating a time-series or rolling window query grouped by a dimension:

- **High-cardinality dimensions** (pod name, host, container, trace ID) can produce hundreds of series, making charts unreadable and queries slow. Ask the user to scope to specific entities, or pre-filter to the most active using a two-phase approach: first aggregate to find top-N entities, then query the time-series for those entities.
- **Low-cardinality dimensions** (service name, namespace, cluster, region, status code) are usually safe to group by directly.
- When unsure about cardinality, add a brief note in the query card summary suggesting the user narrow the filter if the chart is too busy.

---

## Entity Targeting — Match group_by to the Question's Subject

When the user asks "which [entities]", the `group_by` dimension MUST correspond to that entity level. Do not group by a higher-level dimension (e.g., cluster) when the question targets a lower-level entity (e.g., node). After aggregation, use `topk` to rank and limit results so the answer directly lists the top entities.

| User asks "which…" | group_by should use | NOT                |
| :----------------- | :------------------ | :----------------- |
| nodes              | node name           | cluster, namespace |
| services           | service name        | namespace, host    |
| pods               | pod name            | node, deployment   |
| namespaces         | namespace           | cluster            |
| hosts              | host name           | datacenter, region |
| containers         | container name      | pod, node          |

**When summarizing results for "which" questions**, always list the top entities by name with their specific metric values. Do not report only averages across all entities or summarize at a higher aggregation level than the question asks for.

---

## Data Source Selection — Counters vs Events (CRITICAL)

When a question involves counting operational occurrences, you MUST determine whether each concept is a counter metric or a discrete event BEFORE writing the pipeline.

**Decision tree — for each concept in the question:**

1. Does a `_total` or `_count` counter metric exist for this concept?
   - **YES** → Use the counter metric with `delta_monotonic`. Examples: restarts (`_restarts_total`), errors (`_errors_total`), requests (`_requests_total`), OOM kills (`_oom_kills_total`), bytes transferred.
   - **NO** → Continue to step 2.
2. Is the concept a discrete, categorical event with a reason/type field?
   - **YES** → Use the event dataset filtered by reason/type. Examples: scheduled, evicted, deployed, pulled.
   - **NO** → Search for alternative data sources or ask the user.

**Do NOT use event reason strings as proxies for counter-based operations.** Event reasons are categorical labels that don't map 1:1 to operational counts. For example, an Event with reason "Killing" fires during normal rollouts — not just crashes — and "BackOff" only covers CrashLoopBackOff, not all restart scenarios. If a counter metric exists for the concept (e.g., `_restarts_total`), always prefer it.

**Worked example: "How many pods were scheduled, restarted, or evicted?"**

| Concept   | Data source type                                            | Why                                                                                |
| :-------- | :---------------------------------------------------------- | :--------------------------------------------------------------------------------- |
| Scheduled | Event dataset (reason="Scheduled")                          | Discrete event — no counter metric for scheduling                                  |
| Restarted | Counter metric (`kube_pod_container_status_restarts_total`) | Cumulative counter — event reasons like "BackOff"/"Killing" are unreliable proxies |
| Evicted   | Event dataset (reason="Evicted")                            | Discrete event — no counter metric for eviction                                    |

When concepts span both event datasets and counter metrics, combine results with `union` (load [opal-join-patterns](opal-join-patterns.md)). For the restart count, use the two-phase aggregation pattern from [opal-metrics](opal-metrics.md) (delta per pod → filter > 0 → count).
