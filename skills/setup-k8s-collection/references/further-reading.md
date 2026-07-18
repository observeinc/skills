# Further Reading

Post-installation configuration topics from the Observe docs.

## Data Collection

| Topic                          | When to use                              | Docs                                                            |
| ------------------------------ | ---------------------------------------- | --------------------------------------------------------------- |
| Collect annotations and labels | Want K8s metadata as attributes          | https://docs.observeinc.com/docs/collect-annotations-and-labels |
| Add and delete attributes      | Need to enrich or strip telemetry fields | https://docs.observeinc.com/docs/add-and-delete-attributes      |
| Prometheus autodiscovery       | Scraping app metrics from pods           | https://docs.observeinc.com/docs/prometheus-autodiscovery       |
| Application RED metrics        | Generate rate/error/duration from traces | https://docs.observeinc.com/docs/application-red-metrics        |
| Filter logs and metrics        | Exclude noisy data sources               | https://docs.observeinc.com/docs/filter-logs-and-metrics        |
| Handle multiline logs          | Java stack traces, etc.                  | https://docs.observeinc.com/docs/handle-multiline-log-records   |
| Mask sensitive data            | Redact PII from logs                     | https://docs.observeinc.com/docs/mask-sensitive-data            |
| Collect StatsD metrics         | StatsD protocol ingestion                | https://docs.observeinc.com/docs/collect-statsd-metrics         |

## APM / Traces

| Topic                        | When to use                        | Docs                                                 |
| ---------------------------- | ---------------------------------- | ---------------------------------------------------- |
| APM instrumentation          | Instrument apps with OpenTelemetry | https://docs.observeinc.com/docs/apm-instrumentation |
| Trace tail sampling          | Reduce trace volume via gateway    | https://docs.observeinc.com/docs/trace-tail-sampling |
| Configure own OTel collector | Already have an OTel collector     | https://docs.observeinc.com/docs/oss-opentelemetry   |

## Deployment Variants

| Topic                    | When to use                        | Docs                                                                          |
| ------------------------ | ---------------------------------- | ----------------------------------------------------------------------------- |
| Serverless K8s (Fargate) | EKS Fargate environments           | https://docs.observeinc.com/docs/deploy-to-a-serverless-kubernetes-cluster    |
| GKE Autopilot            | GKE Autopilot clusters             | https://docs.observeinc.com/docs/deploy-to-a-serverless-gke-autopilot-cluster |
| Custom namespace         | Deploy outside `observe` namespace | https://docs.observeinc.com/docs/deploy-in-a-custom-namespace                 |
| Rancher multi-cluster    | Deploying across many clusters     | https://docs.observeinc.com/docs/deploy-to-multiple-clusters-using-rancher    |
| Node affinity and taints | Scheduling constraints             | https://docs.observeinc.com/docs/node-affinity-taints-and-tolerations         |
| Tune resources           | Adjust CPU/memory requests/limits  | https://docs.observeinc.com/docs/tune-service-resource-requests-and-limits    |

## Helm Chart Internals

| Resource                   | Path                                                                                                       |
| -------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Full values.yaml reference | https://github.com/observeinc/helm-charts/blob/main/charts/agent/values.yaml                               |
| Chart components docs      | https://github.com/observeinc/helm-charts/blob/main/charts/agent/README.md.gotmpl                          |
| GKE Autopilot example      | https://github.com/observeinc/helm-charts/blob/main/examples/agent/gke-autopilot/gke-autopilot-values.yaml |
| Prometheus scrape examples | https://github.com/observeinc/helm-charts/tree/main/examples/agent/pod_metrics                             |
| Log collection examples    | https://github.com/observeinc/helm-charts/tree/main/examples/agent/logs                                    |
| CI test values             | https://github.com/observeinc/helm-charts/blob/main/charts/agent/ci/test-values.yaml                       |
| Recommended config example | https://github.com/observeinc/helm-charts/blob/main/charts/agent/examples/recommended-config/values.yaml   |

## Supported OTLP Endpoints (for app instrumentation)

After deploying with forwarder enabled, apps send telemetry to:

- **gRPC**: `http://observe-agent-forwarder.observe.svc.cluster.local:4317`
- **HTTP**: `http://observe-agent-forwarder.observe.svc.cluster.local:4318`

Set via `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable in application pods.

## Helm Chart Components

| Component          | Type                    | Purpose                             |
| ------------------ | ----------------------- | ----------------------------------- |
| node-logs-metrics  | DaemonSet               | Per-node log and metric collection  |
| cluster-events     | Deployment (single)     | K8s events and resource state       |
| cluster-metrics    | Deployment (single)     | K8s API server metrics              |
| prometheus-scraper | Deployment (single)     | Pod Prometheus metric scraping      |
| forwarder          | DaemonSet or Deployment | OTLP receiver, forwards to Observe  |
| monitor            | Deployment (single)     | Self-monitoring of agent components |
| gateway            | Deployment              | Trace sampling before export        |
