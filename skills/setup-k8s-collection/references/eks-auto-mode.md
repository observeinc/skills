# EKS Auto Mode Reference

Detailed setup for deploying Observe Agent on AWS EKS Auto Mode (Karpenter-managed nodes).

## Why this is different from Standard

EKS Auto Mode uses Karpenter to provision nodes on demand. Two specifics affect the agent:

1. **Karpenter doesn't provision nodes for DaemonSets alone.** Without a PriorityClass, DaemonSet pods stay Pending because no node will be created just for them.
2. **Auto Mode uses instance IDs as node names** instead of hostnames. The kubeletstats receiver fails silently unless it is told to address kubelets by node IP.

## Step 1: Apply a PriorityClass

Apply this before `helm install` so DaemonSets can preempt and trigger Karpenter:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: observe-agent-priority
globalDefault: false
value: 10000
preemptionPolicy: "PreemptLowerPriority"
```

```bash
kubectl apply -f observe-agent-priority.yaml
```

## Step 2: Values file overrides

Start from the Standard Flow values, then add:

```yaml
node:
  kubeletstats:
    useNodeIp: true

node-logs-metrics:
  priorityClassName: "observe-agent-priority"
```

Why these matter:

- `node.kubeletstats.useNodeIp: true` — addresses kubelets by IP rather than hostname (required for Auto Mode).
- `node-logs-metrics.priorityClassName` — lets the DaemonSet trigger node provisioning via Karpenter.

## Install, verification, and post-install

Same as the Standard Flow.

## Cross-platform additions

After the helm install, run the shared **Cross-platform additions** section in the parent [`../SKILL.md`](../SKILL.md#cross-platform-additions-all-flows) to wire the deployment-environment attribution key and apply any add-ons the user selected in `deploy-k8s-explorer` Step 1f.

## External references

- Observe docs: https://docs.observeinc.com/docs/deploy-to-an-aws-auto-mode-cluster
- EKS Auto Mode setup: https://docs.aws.amazon.com/eks/latest/userguide/automode-get-started-cli.html
