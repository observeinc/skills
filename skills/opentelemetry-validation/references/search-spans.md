# Search Spans Query

Use this when troubleshooting or when you need representative spans for manual audit review.

Pre-requisites:

- Observe CLI

Data source:

- `Tracing/Span`

Commands:

1. `observe dataset list --filter 'label == "Tracing/Span"' --json`
2. `observe dataset view <tracing-span-id> --json`
3. `observe query --input <tracing-span-id> --pipeline '<opal>' --json`

Example query:

```bash
observe query --input <tracing-span-id> --interval 4h --limit 5 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col dur_ms:float64(duration)/1000000
pick_col start_time, end_time, trace_id, span_id, service_name, service_namespace, environment, span_name, span_type, dur_ms, response_status, status_code, error, instrumentation_library
sort desc(start_time)'
```

If `service.namespace` is known, add `filter service_namespace = "<service-namespace>"`.
