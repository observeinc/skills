# GKE Autopilot Reference

Detailed setup for deploying Observe Agent on GKE Autopilot.

## Constraints to be aware of

GKE Autopilot imposes restrictions that affect the deployment:

1. Only `/var/log` is allowed as a `hostPath` for volumes — the `hostmetrics` receiver needs mounts outside this, so **node-level host metrics are disabled**.
2. The `observeinc.com/unschedulable` affinity selector is not allowed — it must be replaced with OS-only node affinity for **all** components, otherwise pods will stay Pending.

## Values File Template

Toggle `<PLACEHOLDER>` values based on user selections (same selections as Standard Flow).

```yaml
observe:
  collectionEndpoint:
    value: "<COLLECTION_ENDPOINT>"
  token:
    create: false

cluster:
  name: "<CLUSTER_NAME>"
  events:
    enabled: true
  metrics:
    enabled: true

node:
  enabled: true
  metrics:
    enabled: false
  containers:
    logs:
      enabled: <LOGS_SELECTED>
  forwarder:
    enabled: true
    mode: deployment
    replicaCount: 2
    metrics:
      outputFormat: prometheus

application:
  prometheusScrape:
    enabled: false # off by default per skill defaults; flip to true if user opts in
  REDMetrics:
    enabled: true # always on per skill defaults; harmless if traces are off

# Override affinity for ALL components to remove observeinc.com/unschedulable
cluster-events:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: NotIn
                values: [windows]

cluster-metrics:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: NotIn
                values: [windows]

prometheus-scraper:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: NotIn
                values: [windows]

node-logs-metrics:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: NotIn
                values: [windows]
  extraVolumes:
    - name: "observe-agent-deployment-config"
      configMap:
        name: "observe-agent"
        items:
          - key: "relay"
            path: "observe-agent.yaml"
        defaultMode: 420
    - name: varlogpods
      hostPath:
        path: /var/log/pods
  extraVolumeMounts:
    - name: observe-agent-deployment-config
      mountPath: /observe-agent-conf
    - name: varlogpods
      mountPath: /var/log/pods
      readOnly: true

monitor:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: NotIn
                values: [windows]

forwarder:
  mode: deployment
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: NotIn
                values: [windows]

gateway:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: NotIn
                values: [windows]

agent:
  selfMonitor:
    enabled: true # always on per skill defaults
  config:
    global:
      fleet:
        enabled: true # populates the "Observe Agent/Events" datastream with 10m heartbeats
```

## Install commands

Use the same install commands as the Standard Flow (helm repo add, namespace, secret, helm install).

## Post-install notes

- Same post-install steps as the Standard Flow (traces endpoint, prometheus annotations, etc.).
- **Node-level host metrics are not available** on Autopilot — `node.metrics.enabled` must remain `false`.

## Cross-platform additions

After the helm install, run the shared **Cross-platform additions** section in the parent [`../SKILL.md`](../SKILL.md#cross-platform-additions-all-flows) to wire the deployment-environment attribution key and apply any add-ons the user selected in `deploy-k8s-explorer` Step 1f.

## External references

- Observe docs: https://docs.observeinc.com/docs/deploy-to-a-serverless-gke-autopilot-cluster
- Example values: https://github.com/observeinc/helm-charts/blob/main/examples/agent/gke-autopilot/gke-autopilot-values.yaml
