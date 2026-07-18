# Service List Query

Returns a row if an instrumented service was successfully catalogued by Observe APM.

Pre-requisites:

- Observe CLI

Data source:

- `Tracing/Service`

Commands:

1. `observe dataset list --filter 'label == "Tracing/Service"' --json`
2. `observe dataset view <tracing-service-id> --json`
3. `observe query --input <tracing-service-id> --pipeline '<opal>' --json`

Example query:

```bash
observe query --input <tracing-service-id> --interval 4h --limit 20 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
pick_col @."Valid From", @."Valid To", service_name, environment, service_namespace, language, host_type, service_type, database
sort asc(service_name)'
```

If `service.namespace` is known, add `filter service_namespace = "<service-namespace>"`.
