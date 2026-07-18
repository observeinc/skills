# Database Service Detection Query

Verify that downstream database dependencies were detected by Observe APM. Run both checks — they validate different aspects of database detection.

Prerequisites:

- Observe CLI
- `generate-opal` skill for producing metric OPAL

Use this only when the service is expected to make database calls.

## Check 1: Database Service in Service Catalog

Verifies that the APM pipeline detected and cataloged a database service in the target environment.

Data source:

- `Tracing/Service`

Commands:

1. `observe dataset list --filter 'label == "Tracing/Service"' --json`
2. `observe dataset view <tracing-service-id> --json`
3. `observe query --input <tracing-service-id> --pipeline '<opal>' --json`

Relevant fields:

- `database` (Bool) — `true` when Observe identifies the service as a database
- `service_type` — `"Database"` for database services
- `service_name` — matches the database peer name from edge metrics (e.g. `postgresql`, `mysql`)

Example query:

```bash
observe query --input <tracing-service-id> --interval 4h --limit 20 --json --pipeline '
filter environment = "<environment>"
filter database = true
pick_col @."Valid From", @."Valid To", service_name, environment, service_namespace, service_type, database
sort desc(@."Valid From")'
```

## Check 2: R.E.D Metrics for Database Peers

Verifies that R.E.D metrics exist for database calls in `Metrics/OpenTelemetry`. The spanmetrics connector generates these from spans. Database calls appear as `SPAN_KIND_CLIENT` rows identified by `attributes."peer.db.name"` rather than a service name.

Use `generate-opal` to build the metric query for the dataset below, then run it with the filters and metric families in this reference.

Dataset label:

- `Metrics/OpenTelemetry`

Metric families:

- `traces.span.metrics.calls`
- `traces.span.metrics.duration` (use the `.sum` / `.count` components for average duration)

Required filters and attributes:

- `attributes."peer.db.name"` — the database peer name (e.g. `postgresql`, `mysql`)
- `attributes."span.kind"` = `"SPAN_KIND_CLIENT"` for database (and other outbound) calls
- `attributes."span.name"` — individual database operations (e.g. `SELECT`, `sql:query`)
- `resource_attributes."deployment.environment.name"` = `<environment>`
- `resource_attributes."service.namespace"` — when known
- errors: isolate with the `string(attributes."otel.status_code") = "ERROR"` predicate inside `m()`

Example query:

```bash
observe query --input <metrics-opentelemetry-id> --interval 4h --json --pipeline '
filter string(resource_attributes."deployment.environment.name") = "<environment>"
filter string(attributes."span.kind") = "SPAN_KIND_CLIENT"
filter not is_null(attributes."peer.db.name")
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
  group_by(peer_db_name:string(attributes."peer.db.name"))
make_col error_rate:if(calls = 0, float64_null(), errors / calls * 100)
make_col avg_duration:if(duration_count = 0, float64_null(), duration_sum / duration_count)
sort desc(calls)'
```

Add `filter string(resource_attributes."service.namespace") = "<service-namespace>"` when the namespace is known. To scope to a single peer, add `filter string(attributes."peer.db.name") = "<peer-db-name>"`.

The `service_name` from Check 1 corresponds to `attributes."peer.db.name"` from Check 2 (e.g. both return `postgresql`).
