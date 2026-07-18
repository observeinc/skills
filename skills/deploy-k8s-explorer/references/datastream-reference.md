# Datastream Reference for K8s Explorer

## Datastreams Created by the K8s Connection Flow

| Datastream Name                          | directWrite Flag    | Source Datasets Created             | When Created       | Dataset ID Used For                                                                      |
| ---------------------------------------- | ------------------- | ----------------------------------- | ------------------ | ---------------------------------------------------------------------------------------- |
| `Kubernetes Explorer/OpenTelemetry Logs` | `otelLogs: true`    | 1 (OtelLogs)                        | Always             | `otelLogsDatasetId` in `updateKubernetesContent`                                         |
| `Kubernetes Explorer/Prometheus`         | `prometheus: true`  | 2 (Prometheus + PrometheusMetadata) | Always             | `prometheusDatasetId` in `updateKubernetesContent`                                       |
| `Kubernetes Explorer/Kubernetes Entity`  | `k8sEntity: true`   | 1 (K8sEntity)                       | Always             | `entityDatasetId` in `updateKubernetesContent`                                           |
| `Tracing/Span`                           | `otelTrace: true`   | 3 (Span, SpanEvent, SpanLink)       | If traces selected | `spanRawDatasetId`, `spanEventDatasetId`, `spanLinkDatasetId` in `installTracingContent` |
| `Metrics/OpenTelemetry`                  | `otelMetrics: true` | 1 (OtelMetrics)                     | If traces selected | `otelMetricsDatasetId` in `installTracingContent`                                        |
| `Observe Agent/Events`                   | `k8sEntity: true`   | 1 (K8sEntity)                       | Always             | Not used in content install                                                              |

## How to Extract Dataset IDs from createDatastream Responses

Each `createDatastream` mutation returns dataset IDs nested under `directWrite`. The exact path depends on the directWrite type:

| directWrite Type | Response Path to Dataset ID                                                 |
| ---------------- | --------------------------------------------------------------------------- |
| `otelLogs`       | `data.createDatastream.directWrite.otelLogs.datasetId`                      |
| `prometheus`     | `data.createDatastream.directWrite.prometheus.datasetId` (metrics)          |
| `prometheus`     | `data.createDatastream.directWrite.prometheus.metadataDatasetId` (metadata) |
| `k8sEntity`      | `data.createDatastream.directWrite.k8sEntity.datasetId`                     |
| `otelTrace`      | `data.createDatastream.directWrite.otelTrace.spanDatasetId`                 |
| `otelTrace`      | `data.createDatastream.directWrite.otelTrace.spanEventDatasetId`            |
| `otelTrace`      | `data.createDatastream.directWrite.otelTrace.spanLinkDatasetId`             |
| `otelMetrics`    | `data.createDatastream.directWrite.otelMetrics.datasetId`                   |

## Content Installation Mapping

### updateKubernetesContent

| Input Field           | Source Datastream                        | Source Path                        |
| --------------------- | ---------------------------------------- | ---------------------------------- |
| `otelLogsDatasetId`   | `Kubernetes Explorer/OpenTelemetry Logs` | `directWrite.otelLogs.datasetId`   |
| `prometheusDatasetId` | `Kubernetes Explorer/Prometheus`         | `directWrite.prometheus.datasetId` |
| `entityDatasetId`     | `Kubernetes Explorer/Kubernetes Entity`  | `directWrite.k8sEntity.datasetId`  |

### installTracingContent

| Input Field            | Source Datastream       | Source Path                                |
| ---------------------- | ----------------------- | ------------------------------------------ |
| `spanRawDatasetId`     | `Tracing/Span`          | `directWrite.otelTrace.spanDatasetId`      |
| `spanEventDatasetId`   | `Tracing/Span`          | `directWrite.otelTrace.spanEventDatasetId` |
| `spanLinkDatasetId`    | `Tracing/Span`          | `directWrite.otelTrace.spanLinkDatasetId`  |
| `otelMetricsDatasetId` | `Metrics/OpenTelemetry` | `directWrite.otelMetrics.datasetId`        |

## Legacy Datastream Detection

If a datastream named exactly `"Kubernetes Explorer"` (no slash) exists, this is the **legacy monolithic datastream**. The UI handles this by:

1. Reusing the legacy datastream's dataset IDs for content binding
2. Creating datastream tokens on it instead of ingest tokens

The skill should **warn the user** about this and ask whether to migrate to the new multi-stream setup.

## Execution Order

```
1. createDatastream × 4 (core K8s) or × 6 (with traces)
2. updateKubernetesContent (using dataset IDs from step 1)
3. installTracingContent (if traces, using dataset IDs from step 1)
4. createIngestToken (for helm chart secret)
```

Each step depends on the previous. If any step fails, stop and report.
