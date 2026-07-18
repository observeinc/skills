# Performance Breakdown Query

Returns the latency breakdown metrics for the target service.

Prerequisites:

- Observe CLI
- `generate-opal` skill for producing metric OPAL

Data source:

- `Tracing/Service Explorer Drilldown Metrics`

Commands:

1. `observe dataset list --filter 'label == "Tracing/Service Explorer Drilldown Metrics"' --json`
2. `observe dataset view <tracing-service-explorer-drilldown-metrics-id> --json`
3. Use the `generate-opal` skill to produce the metric OPAL.
4. `observe query --input <tracing-service-explorer-drilldown-id> --pipeline '<opal>' --json`

Metric families:

- `apm_service_edge_duration_sum_by_endpoint`
- `apm_service_edge_call_count_by_endpoint`
- `apm_service_edge_call_count_by_endpoint_unique`

Example query:

`align options(bins: 1)` collapses the window into a single summary bin so `binSize` equals the whole query window and `rate` is the average call rate over that window. Without it, `align` emits fine-grained time bins and `rate` is computed against a single bin's width while invocations are summed across all bins, producing inflated rates.

```bash
observe query --input <tracing-service-edge-metrics-id> --interval 4h --limit 20 --json --pipeline '
filter string(tags."parent.service.name") = "<service-name>"
filter string(tags."parent.environment") = "<environment>"
align options(bins: 1),
    duration:sum(m("apm_service_edge_duration_sum_by_endpoint")),
    invocations:sum(m("apm_service_edge_call_count_by_endpoint")),
    parent_invocations:sum(m("apm_service_edge_call_count_by_endpoint_unique"))
make_col service_kind:coalesce(string(tags."service.kind"), "INTERNAL")

make_col service_kind:case(
    service_kind = "INTERNAL", "Internal",
    service_kind = "DATABASE", "Database",
    service_kind = "E_HTTP", "HTTP",
    service_kind = "E_OTHER", "External Non-HTTP",
    true, lower(service_kind)
)
make_col binSize:valid_to - valid_from
aggregate
    total_duration:sum(duration),
    invocations:sum(invocations),
    parent_invocations:sum(parent_invocations),
    binSize:any(binSize),
    group_by(
        service_kind,
        downstream_span_name:string(tags."span.name"),
        peer_db_name:string(tags."peer.db.name"),
        service_namespace:string(tags."service.namespace"),
        environment:string(tags."environment")
    )
make_col duration:duration(total_duration),
    rate:invocations/float64(to_seconds(binSize)),
    avg:duration(total_duration/coalesce(parent_invocations, invocations))
drop_col binSize
filter invocations > 0
sort desc(invocations)'
```

If `service.namespace` is known, add `filter string(tags."parent.service.namespace") = "<service-namespace>"` after the service filter.
