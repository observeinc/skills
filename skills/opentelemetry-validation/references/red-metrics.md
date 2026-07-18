# R.E.D Metrics Query

Returns R.E.D (Rate, Errors, Duration) metrics data for an instrumented service.

Prerequisites:

- Observe CLI
- `generate-opal` skill for producing metric OPAL

Use `generate-opal` to build the metric query for the dataset below, then run it with the filters and metric families in this reference.

Dataset label:

- `Metrics/OpenTelemetry`

Required filters:

- `resource_attributes."service.name"` = `<service-name>`
- `resource_attributes."deployment.environment.name"` = `<environment>`
- add `resource_attributes."service.namespace"` when known

Metric families:

- `traces.span.metrics.calls`
- `traces.span.metrics.calls.summary`
- `traces.span.metrics.duration`
- `traces.span.metrics.duration.summary`

## Base class vs summary class

Each family has a high-resolution **base** form and a pre-aggregated **summary** form (`*.summary`):

- **Base** (`traces.span.metrics.calls`, `traces.span.metrics.duration`) bins at fine resolution. Prefer it for recent, short windows and detailed validation.
- **Summary** (`traces.span.metrics.calls.summary`, `traces.span.metrics.duration.summary`) is a coarser rollup that is cheaper over long ranges, but the most recent bin lags and can read as empty.

Notes that apply to both queries below:

- **Errors** have no dedicated family. The calls family carries span status on the `otel.status_code` attribute; isolate errors by passing the predicate as the second argument to `m()`. Metric-attribute predicates belong inside `m()`, not in a top-level `filter`.
- **Duration** average comes from the histogram's `.sum` / `.count` components (`.summary.sum` / `.summary.count` for the summary class).
- To scope to entry-point traffic only, add `filter string(attributes."span.kind") = "SPAN_KIND_SERVER"`.
- Add `filter string(resource_attributes."service.namespace") = "<service-namespace>"` when the namespace is known.
- Both examples use `align options(bins: 1)` to collapse the window into one row for a presence/aggregate check. Drop `bins: 1` from the base-class query to get a fine-resolution time series instead.

### Base class example

```bash
observe query --input <metrics-opentelemetry-id> --interval 4h --json --pipeline '
filter string(resource_attributes."service.name") = "<service-name>"
filter string(resource_attributes."deployment.environment.name") = "<environment>"
align options(bins: 1, empty_bins: true),
  callCount:sum(m("traces.span.metrics.calls")),
  errorCount:sum(m("traces.span.metrics.calls", string(attributes."otel.status_code") = "ERROR")),
  durationSum:sum(m("traces.span.metrics.duration.sum")),
  durationCount:sum(m("traces.span.metrics.duration.count"))
aggregate
  calls:sum(coalesce(callCount, 0)),
  errors:sum(coalesce(errorCount, 0)),
  duration_sum:sum(coalesce(durationSum, 0)),
  duration_count:sum(coalesce(durationCount, 0)),
  group_by()
make_col error_rate:if(calls = 0, float64_null(), errors / calls * 100)
make_col avg_duration:if(duration_count = 0, float64_null(), duration_sum / duration_count)'
```

### Summary class example

```bash
observe query --input <metrics-opentelemetry-id> --interval 4h --json --pipeline '
filter string(resource_attributes."service.name") = "<service-name>"
filter string(resource_attributes."deployment.environment.name") = "<environment>"
align options(bins: 1, empty_bins: true),
  callCount:sum(m("traces.span.metrics.calls.summary")),
  errorCount:sum(m("traces.span.metrics.calls.summary", string(attributes."otel.status_code") = "ERROR")),
  durationSum:sum(m("traces.span.metrics.duration.summary.sum")),
  durationCount:sum(m("traces.span.metrics.duration.summary.count"))
aggregate
  calls:sum(coalesce(callCount, 0)),
  errors:sum(coalesce(errorCount, 0)),
  duration_sum:sum(coalesce(durationSum, 0)),
  duration_count:sum(coalesce(durationCount, 0)),
  group_by()
make_col error_rate:if(calls = 0, float64_null(), errors / calls * 100)
make_col avg_duration:if(duration_count = 0, float64_null(), duration_sum / duration_count)'
```

In both, **R** is `calls`, **E** is `errors` / `error_rate`, and **D** is `avg_duration`.
