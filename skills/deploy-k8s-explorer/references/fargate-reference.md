# EKS Fargate Reference

Detailed setup for deploying Observe Agent on AWS EKS Fargate.

## Why this is different from Standard

EKS Fargate does not support DaemonSets. The agent must use Deployments and rely on the OpenTelemetry Operator to inject sidecars into monitored application pods. Logs require an `emptyDir` shared volume since hostPath-style log access isn't possible.

## Confirm prerequisites with the user

Before generating values, ask the user to confirm:

1. A **Fargate profile** exists for the `observe` namespace (commands below if not).
2. The **OpenTelemetry Operator** is installed in the `observe` namespace (commands below if not).

## Collect service account info

Ask which namespaces and service accounts to monitor — these are needed for the `nodeless.serviceAccounts` map in the values file:

```
AskQuestion:
  id: fargate-sa
  prompt: "List the namespaces and service accounts for pods you want to monitor (e.g. 'default: [my-app-sa, other-sa]')"
```

## Prerequisites Setup

### 1. Create Fargate Profile

The `observe` namespace must have a Fargate profile:

```bash
eksctl create fargateprofile \
  --cluster <CLUSTER_NAME> \
  --name observe-profile \
  --namespace observe \
  --region <AWS_REGION>
```

### 2. Install OpenTelemetry Operator

Required for sidecar injection into monitored pods:

```bash
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --set "manager.collectorImage.repository=ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector-k8s" \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true \
  --namespace observe
```

Wait for operator pod: `kubectl get pods -n observe`

## Values File Template

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
  enabled: false
  forwarder:
    enabled: true

forwarder:
  mode: deployment
  replicaCount: 2

nodeless:
  enabled: true
  hostingPlatform: fargate
  metrics:
    enabled: true
  logs:
    enabled: true
    containerNameFromFile: false
    retryOnFailure:
      enabled: true
      initialInterval: 1s
      maxInterval: 30s
      maxElapsedTime: 5m
    include: '["/applogs/**/*.log", "/applogs/**/*.log.*"]'
    exclude: '["**/*.gz", "**/*.tmp"]'
    lookbackPeriod: 24h
    startAt: end
    autoMultilineDetection: false
  serviceAccounts:
    # Map of namespace -> list of service accounts
    # Example:
    # default: ["my-app-sa"]
    # production: ["prod-app-sa", "worker-sa"]

application:
  prometheusScrape:
    enabled: false

agent:
  selfMonitor:
    enabled: true # always on per skill defaults
  config:
    global:
      fleet:
        enabled: true # populates the "Observe Agent/Events" datastream with 10m heartbeats
```

## Post-Install: Annotate Pods

Annotate all deployments in namespaces you want to monitor:

```bash
TARGET_NAMESPACE=<your-namespace>
for d in $(kubectl get deployments -n $TARGET_NAMESPACE -o name); do
  kubectl patch $d -n $TARGET_NAMESPACE --type='merge' \
    -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.opentelemetry.io/inject":"observe/fargate-collector"}}}}}'
done
```

Force rolling restart:

```bash
kubectl -n <namespace> rollout restart deploy
```

## Pod Log Collection Setup

For Fargate log collection, each monitored pod needs:

### Volume in pod spec

```yaml
volumes:
  - name: log-storage
    emptyDir:
      sizeLimit: 50Mi
```

### Volume mount in container spec

```yaml
volumeMounts:
  - name: log-storage
    mountPath: /applogs
    readOnly: false
```

### Application must write to /applogs

Example Dockerfile modification:

```dockerfile
CMD ["sh", "-c", "your-app > >(tee -a /applogs/container.log >&1) 2> >(tee -a /applogs/container.log >&2)"]
```

Logs are automatically rotated by the `logging-sidecar` injected by the operator.

## Manual RBAC (if not using `serviceAccounts` in values)

If you don't populate `nodeless.serviceAccounts`, manually create RBAC:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-sidecar-role
rules:
  - apiGroups: [""]
    resources: [nodes, nodes/proxy, namespaces, pods]
    verbs: [get, list, watch]
  - apiGroups: ["apps"]
    resources: [replicasets]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-sidecar-role-binding
subjects:
  - kind: ServiceAccount
    name: <SERVICE_ACCOUNT>
    namespace: <NAMESPACE>
roleRef:
  kind: ClusterRole
  name: otel-sidecar-role
  apiGroup: rbac.authorization.k8s.io
```

Apply: `kubectl apply -f cluster-role.yaml`

## Cross-platform additions

After the helm install, run the shared **Cross-platform additions** section in the parent [`../SKILL.md`](../SKILL.md#cross-platform-additions-all-flows) to wire the deployment-environment attribution key and apply any add-ons the user selected in `deploy-k8s-explorer` Step 1f.
