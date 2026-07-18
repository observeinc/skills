# Service Instances Query

Returns a list of running instances of a service such as processes on a host / VM, or containers.

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
observe query --input <tracing-span-id> --interval 4h --limit 10 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col host_name:string(resource_attributes."host.name"), process_pid:int64(resource_attributes."process.pid"), pod_name:string(resource_attributes."k8s.pod.name"), namespace_name:string(resource_attributes."k8s.namespace.name"), cluster_uid:string(resource_attributes."k8s.cluster.uid"), container_name:string(resource_attributes."container.name"), sdk_lang:string(resource_attributes."telemetry.sdk.language"), sdk_version:string(resource_attributes."telemetry.sdk.version")
make_col kind:if(pod_name != "" and not is_null(pod_name), "kubernetes", "host")
statsby span_count:count(), group_by(kind, host_name, process_pid, pod_name, namespace_name, cluster_uid, container_name, sdk_lang, sdk_version, service_name, service_namespace, service_version, environment)'
```

If `service.namespace` is known, add `filter service_namespace = "<service-namespace>"`.
