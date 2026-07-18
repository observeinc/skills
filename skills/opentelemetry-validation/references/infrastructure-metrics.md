# Infrastructure Metrics Query

Choose the query source based on the deployment target.

Prerequisites:

- Observe CLI
- `generate-opal` skill for producing metric OPAL

Use `generate-opal` to build the metric query for the dataset below, then run it with the filters and metric families in this reference.

Data sources:

- `Host Explorer/Prometheus Metrics` (hosts or VMs)
- `Kubernetes Explorer/Prometheus Metrics` (Kubernetes)

Shared label filters:

- `labels."service_name"` = `<service-name>`
- `labels."deployment_environment"` = `<environment>`
- add `labels."service_namespace"` when known

## Hosts or VMs

- Dataset: `Host Explorer/Prometheus Metrics`
- Metrics: `system_cpu_utilization_ratio`, `system_memory_utilization_ratio`

## Kubernetes

- Dataset: `Kubernetes Explorer/Prometheus Metrics`
- Metrics: `k8s_pod_cpu_usage`, `k8s_pod_memory_limit_utilization_ratio`
