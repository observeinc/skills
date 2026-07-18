# Search Span Events Query

Use this to check exception events and other event-level troubleshooting evidence.

Pre-requisites:

- Observe CLI
- Trace ID and / or Span ID to query events for (can be determined by searching spans or endpoint spans)

Data source:

- `Tracing/Span Event`

Commands:

1. `observe dataset list --filter 'label == "Tracing/Span Event"' --json`
2. `observe dataset view <tracing-span-event-id> --json`
3. `observe query --input <tracing-span-event-id> --pipeline '<opal>' --json`

Example query:

```bash
observe query --input <tracing-span-event-id> --interval 4h --limit 5 --json --pipeline '
filter resource_attributes."deployment.environment.name" = "<environment>"
filter trace_id = "<trace-id>"
make_col env:string(resource_attributes."deployment.environment.name"), svc:string(resource_attributes."service.name"), svc_ns:string(resource_attributes."service.namespace")
pick_col timestamp, trace_id, span_id, event_name, env, svc, svc_ns, attributes
sort asc(timestamp)'
```

When validating exception coverage, add `filter event_name = "exception"` and use a `trace_id` from a representative failed span.
