# Optional Add-ons — Option Manifest

Single source of truth for the optional agent add-ons surfaced in `deploy-linux-host-explorer` Step 4d. Each option block below describes its user-facing menu metadata: what to call it, what to ask, what to ask next, and where its YAML configuration lives in the Observe Agent docs.

The skill is purely a UI for navigating these options. The actual YAML configuration content lives at <https://docs.observeinc.com/docs/configure-the-observe-agent-on-linux-windows-and-macos> — each option here points to a section on that page via `docs_anchor`. At apply time, the agent reads the docs section, applies the user's captured choices to the YAML it finds there, and follows the docs' merge semantics.

## Block schema

Each option block uses these fields:

id — machine-readable identifier
label — user-facing menu text
tier — `primary` (named in the Step 4d multi-select)
or `other` (shown only if the user picks
"Show me other available options")
docs_anchor — fragment URL on the agent configuration page at
https://docs.observeinc.com/docs/configure-the-observe-agent-on-linux-windows-and-macos
(omit at the option level if the option has `choices`
with their own anchors)
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
(e.g. EKS Fargate disables resourcedetection)
unsupported_platforms — platforms where this option is unavailable. Today no
Linux platforms appear here; the field exists for
parity with the K8s manifest

An option with no `prompt`, `choices`, or `followups` is informational — selecting it just routes the user to the docs anchor; the skill captures no further values.

---

## Primary options

These are named explicitly in the Step 4d multi-select. Each is shown by `label`; selecting one runs the prompts / follow-ups in the option's block.

### option: mask

```yaml
id: mask
label: "Mask sensitive data (passwords, credit cards, SSNs, etc.)"
tier: primary
docs_anchor: "#mask-sensitive-data"
followups:
  - id: patterns
    prompt: "Which sensitive-data patterns should the agent redact from logs? Multi-select from: passwords, credit cards, SSNs, bearer tokens, emails, phone numbers, IPv4 addresses, names."
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
    docs_anchor: "#automatic-detection"
    excludes: [multiline.custom]
  custom:
    label: "Custom regex (for non-standard log formats)"
    docs_anchor: "#custom-pattern-with-the-recombine-operator"
    excludes: [multiline.auto]
    followups:
      - id: regex
        prompt: "Provide an is_first_entry regex — a pattern that matches only the FIRST line of each log record (e.g. ^\\d{4}-\\d{2}-\\d{2} for ISO-8601 timestamps). Lines that don't match are treated as continuations of the previous record."
```

### option: ec2_tags

```yaml
id: ec2_tags
label: "Fetch EC2 instance tags as resource attributes"
tier: primary
docs_anchor: "#fetch-ec2-instance-tags"
uses_processors: [resourcedetection]
prompt: "EC2 tag retrieval requires the host's IAM role to include `ec2:DescribeTags`. Has that permission been granted? (If not, the agent still runs — the resourcedetection processor will log permission-denied errors and skip tag retrieval.)"
followups:
  - id: scope
    prompt: "Which EC2 tag keys should be included as resource attributes? Either `all`, or a comma-separated list of specific key names."
```

### option: tail_log

```yaml
id: tail_log
label: "Tail a custom log file (paths outside /var/log)"
tier: primary
docs_anchor: "#tail-a-log-file"
followups:
  - id: paths
    prompt: "Which file paths should the agent tail? List as glob patterns, one per line (e.g. /var/log/myapp/*.log, /opt/myapp/logs/*.log)."
  - id: attributes
    optional: true
    prompt: "Should logs from these files carry any per-source resource attributes (e.g. service.name=myapp, log_source=app)? Comma-separated key=value list, or `none`."
```

---

## "Other" options

Shown only if the user picks "Show me other available options" in the Step 4d multi-select. Each routes the user to the docs anchor; no follow-up prompts are scripted in the skill — the agent walks the user through the docs section at apply time.

### option: process_metrics

```yaml
id: process_metrics
label: "Process-level metrics (per-process CPU, memory, disk, threads)"
tier: other
docs_anchor: "#enable-or-disable-connections"
```

### option: filter_metrics

```yaml
id: filter_metrics
label: "Filter metrics (regex include/exclude on metric names — cost control)"
tier: other
docs_anchor: "#filter-metrics"
```

### option: splunk_forwarder

```yaml
id: splunk_forwarder
label: "Receive data from a Splunk forwarder"
tier: other
docs_anchor: "#receive-data-from-a-splunk-forwarder"
```

### option: mongodb_metrics

```yaml
id: mongodb_metrics
label: "Collect metrics from a MongoDB instance"
tier: other
docs_anchor: "#collect-metrics-from-a-mongodb-instance"
```

### option: mongodb_logs

```yaml
id: mongodb_logs
label: "Collect logs from a MongoDB instance"
tier: other
docs_anchor: "#collect-logs-from-a-mongodb-instance"
```

### option: sample_data

```yaml
id: sample_data
label: "Sample spans or log records (probabilistic sampling)"
tier: other
docs_anchor: "#sample-spans-or-log-records"
```

### option: host_id

```yaml
id: host_id
label: "Detect a host.id (system-level resource attributes)"
tier: other
docs_anchor: "#detect-a-hostid"
uses_processors: [resourcedetection]
```

### option: extra_attributes

```yaml
id: extra_attributes
label: "Add custom attributes or tags to all data"
tier: other
docs_anchor: "#add-attributes-or-tags"
```

### option: custom_otel

```yaml
id: custom_otel
label: "Fully override OTel collector config (omit_base_components)"
tier: other
docs_anchor: "#override-existing-otel-collector-configuration"
```
