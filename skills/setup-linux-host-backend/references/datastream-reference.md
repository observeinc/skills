# Datastream Reference for Host Explorer

## Datastreams Created by the Host Explorer Flow

| Datastream Name                    | directWrite Flag   | Source Datasets Created             | When Created | Dataset ID Used For                                         |
| ---------------------------------- | ------------------ | ----------------------------------- | ------------ | ----------------------------------------------------------- |
| `Host Explorer/OpenTelemetry Logs` | `otelLogs: true`   | 1 (OtelLogs)                        | Always       | `otelLogsDatasetId` in `installHostContent`                 |
| `Host Explorer/Prometheus`         | `prometheus: true` | 2 (Prometheus + PrometheusMetadata) | Always       | `prometheusDatasetId` in `installHostContent`               |
| `Observe Agent/Events`             | `k8sEntity: true`  | 1 (K8sEntity)                       | Always       | Receives fleet heartbeats; not used by `installHostContent` |

## How to Extract Dataset IDs from createDatastream Responses

Each `createDatastream` response returns dataset IDs nested under `directWrite`. The exact path depends on the directWrite type:

| directWrite Type | Response Path to Dataset ID                           |
| ---------------- | ----------------------------------------------------- |
| `otelLogs`       | `directWrite.otelLogs.datasetId`                      |
| `prometheus`     | `directWrite.prometheus.datasetId` (metrics)          |
| `prometheus`     | `directWrite.prometheus.metadataDatasetId` (metadata) |
| `k8sEntity`      | `directWrite.k8sEntity.datasetId`                     |

## Content Installation Mapping

### installHostContent (observe content host install)

| Input Flag                | Source Datastream                  | Source Path                        |
| ------------------------- | ---------------------------------- | ---------------------------------- |
| `--otel-logs-dataset-id`  | `Host Explorer/OpenTelemetry Logs` | `directWrite.otelLogs.datasetId`   |
| `--prometheus-dataset-id` | `Host Explorer/Prometheus`         | `directWrite.prometheus.datasetId` |

## Data Routing

The ingest token does not need explicit datastream associations. The Observe ingestion layer matches incoming data to datastreams via target-package prefix matching:

- OpenTelemetry log payloads from observe-agent are routed to `Host Explorer/OpenTelemetry Logs`
- Prometheus metric payloads from observe-agent are routed to `Host Explorer/Prometheus`
- Fleet heartbeats (sent with header `X-Observe-Target-Package: Observe Agent`) are routed to `Observe Agent/Events`

This routing is automatic as long as the datastream names match the expected prefixes (`Host Explorer/` and `Observe Agent/`).

## Execution Order

```
1. createDatastream: Host Explorer/OpenTelemetry Logs
2. createDatastream: Host Explorer/Prometheus
3. createDatastream: Observe Agent/Events
4. observe content host install (using dataset IDs from steps 1 and 2 — Observe Agent/Events is not passed to content install)
5. createIngestToken (for observe-agent config)
```

Each step depends on the previous. If any step fails, stop and report.

## Observe Agent Data Flow

The observe-agent on the host sends data to Observe using the following pipelines:

```
observe-agent
  -- host_monitoring.logs (OTel logs receiver)
       /var/log/**  ->  OTLP export  ->  Host Explorer/OpenTelemetry Logs datastream
  -- host_monitoring.metrics.host (hostmetrics receiver)
       CPU, disk, memory, network, processes  ->  Prometheus export  ->  Host Explorer/Prometheus datastream
  -- self_monitoring.fleet (heartbeat receiver)
       Agent identity + version + config heartbeat  ->  OTLP export
       (header X-Observe-Target-Package: Observe Agent)  ->  Observe Agent/Events datastream
```

The agent authenticates with the ingest token set in `observe-agent init-config --token`.
