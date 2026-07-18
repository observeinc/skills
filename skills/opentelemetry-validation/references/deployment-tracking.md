# Deployment Tracking Query

Returns a list of versions of an instrumented service.

Prerequisites:

- Observe CLI
- `generate-opal` skill for producing metric OPAL
- `service.version` resource attribute set

Skip this check if `service.version` was not set for the application.

## Query 1: Deployment Record in Tracing/Deployment

Returns deployment records materialized for various service versions for the service.

Data source:

- `Tracing/Deployment`

Commands:

1. `observe dataset list --filter 'label == "Tracing/Deployment"' --json`
2. `observe dataset view <tracing-deployment-id> --json`
3. `observe query --input <tracing-deployment-id> --pipeline '<opal>' --json`

Relevant fields:

- `service_version` — the `service.version` resource attribute value
- `deployment_time` — timestamp of the deployment record

Example query:

```bash
observe query --input <tracing-deployment-id> --interval 4h --limit 20 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
pick_col deployment_time, @."Valid To", environment, service_name, service_namespace, service_version
sort desc(deployment_time)'
```

`pick_col` must keep `deployment_time` and `@."Valid To"` — these are the interval bounds for `Tracing/Deployment`. Do not substitute `@."Valid From"`; it does not satisfy the schema and the query fails with `Query returned no schema`.

## Query 2: Version dimension in span metrics

Returns service call volume grouped by service version, confirming `service.version` is recorded on emitted telemetry.

Dataset label:

- `Metrics/OpenTelemetry`

Metric families:

- `traces.span.metrics.calls`

Relevant attributes:

- `resource_attributes."service.name"` = `<service-name>`
- `resource_attributes."deployment.environment.name"` = `<environment>`
- `resource_attributes."service.version"` must be present and non-null
- add `resource_attributes."service.namespace"` when known

Example query:

```bash
observe query --input <metrics-opentelemetry-id> --interval 4h --json --pipeline '
filter string(resource_attributes."service.name") = "<service-name>"
filter string(resource_attributes."deployment.environment.name") = "<environment>"
filter not is_null(resource_attributes."service.version")
align options(bins: 1, empty_bins: true), call_count:sum(m("traces.span.metrics.calls"))
aggregate total:sum(coalesce(call_count, 0)), group_by(ver:string(resource_attributes."service.version"))
sort desc(total)'
```
