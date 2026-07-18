# Optional Add-ons — Option Manifest

Single source of truth for the optional agent add-ons surfaced in `deploy-k8s-explorer` Step 1f. Each option block below describes its user-facing menu metadata: what to call it, what to ask, what to ask next, and where its YAML configuration lives in the Observe docs.

The skill is purely a UI for navigating these options. The actual YAML configuration content lives on <https://docs.observeinc.com> — each option here points to a docs page via `docs_anchor`. At apply time (in `setup-k8s-collection`), the agent reads the docs page, applies the user's captured choices to the YAML it finds there, and follows the docs' merge semantics.

> Unlike the host manifest, K8s docs are organized as **one page per topic** rather than one big page with fragments. `docs_anchor` here carries a full path like `/docs/<slug>` on `docs.observeinc.com`, not a `#fragment`.

## Block schema

Each option block uses these fields:

id — machine-readable identifier
label — user-facing menu text
tier — `primary` (named in the Step 1f multi-select)
or `other` (shown only if the user picks
"Show me other available options")
docs_anchor — relative path on docs.observeinc.com (e.g.
`/docs/mask-sensitive-data`). Full URL is
`https://docs.observeinc.com<docs_anchor>`.
Omit at the option level if the option has
`choices` with their own anchors.
prompt — first question asked when the option is selected
(omit if the option has no top-level question — e.g.
it has only `choices`, only `followups`, or is
informational)
choices — used when an option has mutually exclusive sub-choices;
each choice has its own label, docs_anchor, excludes,
and (optionally) followups
followups — list of additional prompts to capture user values;
each has { id, prompt, optional?: bool }
excludes — other option-or-choice ids that cannot be selected at
the same time. Declared symmetrically — if A excludes
B, B's manifest must exclude A
uses_processors — OTel processor names this option depends on. Consumed
by platform-level rules to determine availability
(e.g. EKS Fargate has different processor availability
than Standard)
unsupported_platforms — Kubernetes infrastructure types where this option is
unavailable. Valid values: `standard`, `eks-fargate`,
`gke-autopilot`, `eks-auto-mode` (matching Step 1a)

An option with no `prompt`, `choices`, or `followups` is informational — selecting it just routes the user to the docs anchor; the skill captures no further values.

---

## Primary options

These are named explicitly in the Step 1f multi-select. Each is shown by `label`; selecting one runs the prompts / follow-ups in the option's block.

### option: mask

```yaml
id: mask
label: "Mask sensitive data (passwords, credit cards, SSNs, tokens, etc.)"
tier: primary
docs_anchor: "/docs/mask-sensitive-data"
followups:
  - id: patterns
    prompt: "Which sensitive-data patterns should the agent redact from pod logs? Multi-select from: passwords, credit cards, SSNs, bearer tokens, emails, phone numbers, IPv4 addresses."
```

### option: multiline

```yaml
id: multiline
label: "Handle multiline log records (Java/Python stack traces, etc.)"
tier: primary
prompt: "How should the agent handle multiline log records?"
choices:
  auto:
    label: "Automatic (handles common timestamp-prefixed formats)"
    docs_anchor: "/docs/handle-multiline-log-records"
    excludes: [multiline.custom]
  custom:
    label: "Custom regex (for non-standard log formats)"
    docs_anchor: "/docs/handle-multiline-log-records"
    excludes: [multiline.auto]
    followups:
      - id: regex
        prompt: "Provide an is_first_entry regex — a pattern that matches only the FIRST line of each log record (e.g. ^\\d{4}-\\d{2}-\\d{2} for ISO-8601 timestamps). Lines that don't match are treated as continuations of the previous record."
```

### option: annotations

```yaml
id: annotations
label: "Map pod annotations/labels to OTel resource attributes (incl. team.name)"
tier: primary
docs_anchor: "/docs/collect-annotations-and-labels"
uses_processors: [k8sattributes]
prompt: "Which pod annotations or labels should the agent surface as resource attributes on the telemetry it sends? A few common ones — leave blank to skip any."
followups:
  - id: team_annotation
    optional: true
    prompt: "Annotation key for team attribution (becomes `team.name`). Default if you want it: `observeinc.com/team`. Skip if you don't want a per-workload team attribute."
  - id: extra_annotations
    optional: true
    prompt: "Any other pod annotations to forward as resource attributes? Comma-separated list of `<annotation-key>=<attribute-name>` pairs, or `none`."
  - id: extra_labels
    optional: true
    prompt: "Any pod labels to forward as resource attributes? Comma-separated list of `<label-key>=<attribute-name>` pairs, or `none`."
```

### option: filter

```yaml
id: filter
label: "Filter logs / metrics (cost control via include/exclude rules)"
tier: primary
docs_anchor: "/docs/filter-logs-and-metrics"
prompt: "What do you want to filter? Pick one or both: drop logs by namespace/pod/glob, or drop metrics by name regex."
followups:
  - id: log_rules
    optional: true
    prompt: "Log filter rules. List as `include:<pattern>` or `exclude:<pattern>` per line (patterns are glob expressions over the log body or pod namespace). `none` to skip."
  - id: metric_rules
    optional: true
    prompt: "Metric filter rules. List as `include:<regex>` or `exclude:<regex>` per line (regex matches the metric name). `none` to skip."
```

### option: prom_scrape

```yaml
id: prom_scrape
label: "Constrain Prometheus autodiscovery (which annotated pods get scraped)"
tier: primary
docs_anchor: "/docs/prometheus-autodiscovery"
unsupported_platforms: [eks-fargate]
prompt: 'By default, every pod annotated with `prometheus.io/scrape: "true"` is scraped. Narrow this?'
followups:
  - id: namespaces
    optional: true
    prompt: "Limit scraping to specific namespaces? Comma-separated list, or `all`."
  - id: extra_pod_annotations
    optional: true
    prompt: "Use a different annotation key than `prometheus.io/scrape` (e.g. `observeinc_com_scrape`)? Specify the annotation key, or `default` to keep `prometheus.io/scrape`."
```

---

## "Other" options

Shown only if the user picks "Show me other available options" in the Step 1f multi-select. Each routes the user to the docs anchor; no follow-up prompts are scripted in the skill — the agent walks the user through the docs section at apply time.

### option: gateway_sampling

```yaml
id: gateway_sampling
label: "Tail-based trace sampling (gateway deployment)"
tier: other
docs_anchor: "/docs/trace-tail-sampling"
```

### option: add_del_attributes

```yaml
id: add_del_attributes
label: "Add / delete resource attributes on telemetry"
tier: other
docs_anchor: "/docs/add-and-delete-attributes"
```

### option: statsd

```yaml
id: statsd
label: "Receive StatsD-format metrics from apps"
tier: other
docs_anchor: "/docs/collect-statsd-metrics"
```

### option: custom_otel

```yaml
id: custom_otel
label: "Configure your own OTel collector (escape hatch for arbitrary config)"
tier: other
docs_anchor: "/docs/configure-your-own-otel-collector"
```

### option: node_affinity

```yaml
id: node_affinity
label: "Customize node affinity, taints, and tolerations"
tier: other
docs_anchor: "/docs/node-affinity-taints-and-tolerations"
```
