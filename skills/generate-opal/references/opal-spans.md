# Querying Interval/span datasets: duration conversion, latency percentiles, error rates, APM, RED methodology, dependency tracking

## Workflow

1. Confirm the dataset kind is Interval with otel_span interface (check the dataset schema).
2. Choose the analysis pattern:
    - Latency analysis → use the latency percentiles pattern below
    - Error rate → use the error rate pattern below
    - Dependency mapping → use the dependency tracking patterns below
3. Duration is in nanoseconds — always convert before aggregation.
4. ALWAYS include `group_by()` in `statsby` — omitting it on Interval datasets explodes into per-span rows.
5. If the question is better answered by pre-aggregated metrics, consider using [opal-metrics](opal-metrics.md) instead.

---

## Pattern: Intervals (Spans) — filter → make_col (duration) → statsby

Span datasets (kind=Interval, interface=otel_span) have start/end time and duration.

### Duration conversion

Duration is in nanoseconds. ALWAYS convert and suffix with the unit (`_ms`, `_sec`) — never overwrite the original column. When unsure about the source unit, leave values unconverted.

    make_col dur_ms:float64(duration)/1000000
    make_col dur_sec:float64(duration)/1000000000

The span `duration` column is int64 nanoseconds, so raw arithmetic is correct here. For OPAL Duration types (from timestamp subtraction like `valid_to - valid_from`), use `to_milliseconds()`, `to_seconds()`, etc. For duration thresholds, age/staleness calculations, or timestamp formatting, see [opal-duration](opal-duration.md).

### Latency percentiles by service

    make_col dur_ms:float64(duration)/1000000
    statsby p50:percentile(dur_ms, 0.50),
              p95:percentile(dur_ms, 0.95),
              p99:percentile(dur_ms, 0.99),
              group_by(service_name)
    sort desc(p99)

### Error rate from spans

    make_col is_error:if(error = true, 1, 0)
    statsby total:count(), errors:sum(is_error), group_by(service_name)
    make_col error_rate:round(100.0 * float64(errors) / float64(total), 2)
    sort desc(error_rate)

### CRITICAL: group_by on Interval datasets

statsby without group_by() on Interval datasets implicitly groups by the primary key (trace_id, span_id), producing millions of per-row results:

    statsby total:count()               ← WRONG — explodes into per-span rows
    statsby total:count(), group_by()   ← CORRECT — single summary row
    statsby total:count(), group_by(service_name)  ← CORRECT — per service

### Duration bucketing

    make_col dur_ms:float64(duration)/1000000
    make_col latency_tier:case(dur_ms < 100, "fast", dur_ms < 1000, "medium", dur_ms < 5000, "slow", true, "very_slow")
    statsby count:count(), group_by(service_name, latency_tier)

---

## APM: When to use metrics vs spans

| Question type                                 | Use                         | Why                                       |
| :-------------------------------------------- | :-------------------------- | :---------------------------------------- |
| Service-level rates, throughput, error counts | Metrics                     | Pre-aggregated, fast, covers long windows |
| Per-request latency percentiles at scale      | Metrics (tdigest/histogram) | Avoids scanning billions of spans         |
| Specific error messages, stack traces         | Spans + logs                | Need raw event detail                     |
| Per-endpoint breakdown, trace drill-down      | Spans                       | Need span_name, attributes                |
| Dependency mapping, downstream failures       | Spans                       | Need span kind + attributes               |

When investigating, start with metrics for the overview, then drill into spans for detail.

## RED metrics from metric datasets

When the dataset has pre-aggregated APM metrics (interface=metric), match actual metric names from the catalog.

### Error rate from metrics

    align options(bins: 1),
          requests:sum(m("apm_service_call_count")),
          errors:sum(m("apm_service_error_count"))
    aggregate total_requests:sum(requests),
              total_errors:sum(errors),
              group_by(svc:string(tags."service.name"))
    fill total_errors:0
    make_col error_rate:round(100.0 * float64(total_errors) / float64(total_requests), 2)
    sort desc(error_rate)

### Latency percentiles from distribution metrics

Use the correct `m_*` function based on metric type: `m_tdigest()` for tdigest, `m_histogram()` for histogram, `m_exponential_histogram()` for exponential histogram. All three use `histogram_combine` / `histogram_quantile`.

    align options(bins: 1), combined:histogram_combine(m_tdigest("apm_service_call_duration"))
    aggregate combined:histogram_combine(combined), group_by(svc:string(tags."service.name"))
    make_col p50:histogram_quantile(combined, 0.50),
             p95:histogram_quantile(combined, 0.95),
             p99:histogram_quantile(combined, 0.99)
    make_col p50_ms:round(p50/1000000, 2), p95_ms:round(p95/1000000, 2), p99_ms:round(p99/1000000, 2)
    sort desc(p99_ms)

## Dependency tracking patterns

Find downstream dependencies (what does service X call?):
filter service_name = "checkout-service" and kind = "CLIENT"
statsby call_count:count(), error_count:sum(if(error = true, 1, 0)), group_by(peer_svc:string(attributes."peer.service"))

Find database dependencies:
filter not is_null(string(attributes."db.system"))
statsby call_count:count(), group_by(service_name, db_system:string(attributes."db.system"))

Find upstream callers (what calls service X?):
filter string(attributes."peer.service") = "payments-service" and kind = "CLIENT"
statsby call_count:count(), group_by(service_name)

---

## Tracing Investigation Workflow

When users ask about traces, spans, and especially **span events**, follow this multi-step workflow:

1. **Understand the tracing data model.** Tracing data is typically split across separate datasets:

    - **Span dataset** (e.g., "Tracing/Span") — span attributes, duration, status, service name
    - **Span Event dataset** (e.g., "Tracing/Span Event") — in-span events like exceptions, logs, annotations
    - **Trace dataset** (e.g., "Tracing/Trace") — trace-level summaries

2. **If the user mentions "span events"**, you **must** query the Span Event dataset. Span events are NOT part of the Span dataset.

3. **Typical flow:** Find spans matching criteria → identify traces → query span events using `trace_id` or `span_id`.

## Data Volume & Service Ranking Queries

When users ask "which services are sending the most data?":

-   **Prefer Tracing/Span data** as the default. Counting spans by `service_name` is straightforward.
-   Byte-level metrics require `rate()` or `delta_monotonic()` — harder to get right.
-   If the user specifically asks about bytes/bandwidth, use byte metrics with proper counter aggregation (see [opal-metrics](opal-metrics.md)).
