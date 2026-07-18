# Prometheus Autodiscovery Reference

How to configure Prometheus metric scraping from application pods.

## Pod Annotations for Autodiscovery

Expose metrics from a pod using one of these methods:

### Method 1: Port annotation

```yaml
annotations:
  prometheus.io/port: "9999"
```

Only surfaces one HTTP endpoint.

### Method 2: Named port (preferred)

```yaml
ports:
  - containerPort: 9999
    name: metrics
  - containerPort: 12345
    name: sidecar-metrics
```

Any port with a name ending in `metrics` is discovered.

### Additional annotations

| Annotation              | Default    | Description                      |
| ----------------------- | ---------- | -------------------------------- |
| `prometheus.io/scheme`  | `http`     | Set to `https` for TLS endpoints |
| `prometheus.io/path`    | `/metrics` | Custom metrics path              |
| `prometheus.io/scrape`  | `true`     | Set `false` to skip this pod     |
| `observeinc_com_scrape` | `true`     | Observe-specific scrape toggle   |

## Values Configuration

```yaml
application:
  prometheusScrape:
    enabled: true
    interval: 60s
    namespaceDropRegex: (.*istio.*|.*ingress.*|kube-system)
    namespaceKeepRegex: (.*)
    portKeepRegex: .*metrics
    metricDropRegex: ""
    metricKeepRegex: (.*)
```

Key fields:

- `namespaceDropRegex` is applied first — matching namespaces are excluded
- `namespaceKeepRegex` is applied second — only remaining matches are kept
- `portKeepRegex` filters which named ports to scrape
- `metricDropRegex` / `metricKeepRegex` filter individual metric names

## Independent Deployment

For high-volume metric scraping, use a dedicated deployment:

```yaml
application:
  prometheusScrape:
    enabled: true
    independentDeployment: true
```

## Applying Changes

```bash
helm upgrade --reuse-values observe-agent observe/agent -n observe \
  --values prometheus-scrape.yaml

kubectl rollout restart deployment -n observe
kubectl rollout restart daemonset -n observe
kubectl get pods -o wide -n observe
```
