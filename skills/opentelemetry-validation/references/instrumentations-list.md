# Instrumentations List Query

Returns a list of instrumentation libraries for a given service.

Note that this is a temporal query which returns only instrumentations seen in a query window:

- This is not an accurate representation of all instrumentation libraries injected for a given app
- Instrumentations that produce telemetry in sparse or irregular intervals may get missed

Pre-requisites:

- Observe CLI

## Tracing Instrumentation Libraries

Data source:

- `Tracing/Span`

Commands:

1. `observe dataset list --filter 'label == "Tracing/Span"' --json`
2. `observe dataset view <tracing-span-id> --json`
3. `observe query --input <tracing-span-id> --pipeline '<opal>' --json`

Example query:

```bash
observe query --input <tracing-span-id> --interval 24h --limit 10 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col lib_name:string(instrumentation_library.name), lib_version:string(instrumentation_library.version)
filter lib_name != "" and not is_null(lib_name)
statsby span_count:count(), group_by(service_name, environment, service_namespace, lib_name, lib_version)
sort desc(span_count)'
```

If `service.namespace` is known, add `filter service_namespace = "<service-namespace>"`.

## Metrics Instrumentation Libraries

Data source:

- `Metrics/OpenTelemetry`

Commands:

1. `observe dataset list --filter 'label == "Metrics/OpenTelemetry"' --json`
2. `observe dataset view <metrics-opentelemetry-id> --json`
3. `observe query --input <metrics-opentelemetry-id> --pipeline '<opal>' --json`

Example query:

```bash
observe query --input <metrics-opentelemetry-id> --interval 24h --limit 10 --json --pipeline '
filter resource_attributes."deployment.environment.name" = "<environment>" or resource_attributes."deployment.environment" = "<environment>"
filter resource_attributes."service.name" = "<service-name>"
make_col service_name:string(resource_attributes."service.name"), environment:string(coalesce(resource_attributes."deployment.environment.name", resource_attributes."deployment.environment")), service_namespace:string(resource_attributes."service.namespace")
make_col lib_name:string(instrumentation_scope.name), lib_version:string(instrumentation_scope.version)
filter lib_name != "" and not is_null(lib_name)
statsby points:count(), group_by(service_name, environment, service_namespace, lib_name, lib_version)
sort desc(points)'
```

If `service.namespace` is known, add `filter resource_attributes."service.namespace" = "<service-namespace>"`.

## Logging Instrumentation Libraries

Data sources:

- `Host Explorer/OpenTelemetry Logs`
- `Kubernetes Explorer/OpenTelemetry Logs`

Commands:

1. `observe dataset list --filter 'label == "<dataset>"' --json`
2. `observe dataset view <dataset-id> --json`
3. `observe query --input <dataset-id> --pipeline '<opal>' --json`

Example query (common across data sources):

```bash
observe query --input <dataset-id> --interval 4h --limit 10 --json --pipeline '
filter resource_attributes."deployment.environment.name" = "<environment>" or resource_attributes."deployment.environment" = "<environment>"
filter resource_attributes."service.name" = "<service-name>"
filter string(instrumentation_scope.name) != "spanmetricsconnector"
make_col service_name:string(resource_attributes."service.name"), environment:string(coalesce(resource_attributes."deployment.environment.name", resource_attributes."deployment.environment")), service_namespace:string(resource_attributes."service.namespace")
make_col lib_name:string(instrumentation_scope.name), lib_version:string(instrumentation_scope.version)
filter lib_name != "" and not is_null(lib_name)
statsby logs:count(), group_by(service_name, environment, service_namespace, lib_name, lib_version)
sort desc(logs)'
```

If `service.namespace` is known, add `filter resource_attributes."service.namespace" = "<service-namespace>"`.
