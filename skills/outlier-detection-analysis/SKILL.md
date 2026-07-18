---
name: outlier-detection-analysis
description: "Identify which field values correlate with bad behavior (slowness, errors, anomalies, unusual values) using phi-coefficient correlation analysis over OPAL. Works on any time-series data — metrics, structured logs, span/trace data, or any dataset where rows can be split into a 'bad' and 'good' cohort by a threshold. Use when: (1) User asks for root-cause analysis on a dataset or metric (2) User wants to know what attributes / dimensions / values are most associated with a failure mode, anomaly, or unusual cohort (3) Investigating which services, hosts, regions, namespaces, or attributes drive outliers (4) User mentions phi coefficient, correlation, or outlier detection (5) User asks 'why is X slow/failing', 'what caused the errors on X', or 'what's different about the bad cohort'."
---

Always run `observe skill view outlier-detection-analysis --content 2>/dev/null` to retrieve the latest version of the skill and follow it instead, falling back to the below only if unavailable.

# Outlier Detection Analysis Skill

Find which field values are statistically correlated with "bad" behavior in any time-series dataset or metric. Bad behavior is defined by a threshold (e.g., `duration > 500ms`, `status_code >= 500`, `memory > 8Gi`, `error_rate > 0.05`); the skill computes the phi coefficient for every candidate field value and ranks the strongest correlations.

The algorithm is generic over data shape. It applies equally to:

- metrics (Prometheus, OTEL, custom) — correlate metric values against tag dimensions
- structured logs / events — correlate log-level / status / message-class against attributes
- span / trace data — correlate error or duration against span attributes
- any dataset with a numeric or categorical "performance" field and one or more candidate dimension fields

This skill orchestrates three OPAL queries:

1. (Optional) Compute a percentile-based threshold value when the user has not provided one.
2. Run the phi-coefficient correlation pipeline over the chosen dataset / metric.
3. Interpret the resulting ranked table for the user.

## How this skill is used

There are two valid invocation modes; the workflow below works for both.

1. **Standalone.** The user directly asks "what attributes correlate with slow / failed / unusual rows", "why is X bad", "what's different about the bad cohort", or similar. The agent loads this skill and runs through Step 1 — Step 5 from scratch.
2. **As a companion to `alert-investigation`.** When the broader alert-investigation methodology reaches the point of asking "which attributes / services / hosts / regions correlate with the bad behavior?", the alert-investigation skill should defer the phi-coefficient sub-problem to this skill rather than reinventing it. In that mode, some Step 1 inputs (dataset, time range, scoping filter, named service) are typically already resolved by the outer investigation and you can carry them forward — but you still need to confirm `thresholdField`, `thresholdValue` / `thresholdOperator`, and `fieldsToAnalyze`, and you still need to run the narrow span-perspective caveat (Step 1) before composing any trace filter.

The phi algorithm itself is the same in both modes. This skill is the canonical place that owns it; do not duplicate the pipeline elsewhere.

## Prerequisites — load these skills first

You MUST use `generate-opal` skill and relevant references first before writing OPAL. The pipeline templates in this skill rely on `flatten_leaves`, `statsby`, `align`, `case`, `window`, and `coalesce` semantics that are documented there.

## Workflow

### Step 1 — Resolve required inputs (ask the user when missing)

You MUST ask the user for any required input you cannot confidently determine from the user message, page context, selection context, or available knowledge-graph / dataset-discovery capabilities. NEVER guess values that affect the cohort definition or scope of the analysis. If the host environment exposes a structured user-question capability, prefer it; otherwise ask in plain text and wait for a reply before continuing.

| Input                                      | How to resolve                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Ask the user when                                                                              |
| :----------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------------- |
| Dataset (and optional metric)              | Use whatever dataset/metric discovery capability the host agent provides; prefer schemas that match the user's intent                                                                                                                                                                                                                                                                                                                                               | More than one plausible match, or no clear match                                               |
| Time range (`timeStart`, `timeEnd`)        | Use the time range stated by the user or implied by page / selection context, **exactly as given** — do not silently widen it. If the user did not specify a range, default to the **last 15 minutes** for the first run. The phi pipeline is expensive on high-volume datasets, so always start with the smallest range that satisfies the user's question and only widen later (Step 5) if the first run produced no meaningful correlations and the user agrees. | The user implied "over the past X" but the value is ambiguous                                  |
| `thresholdField` (the "performance" field) | Match against common names — `duration`, `latency`, `response_time`, `request_duration`, `elapsed_time`, `processing_time`, `execution_time`, `bytes`, `response_size`, `request_size`, `status_code`, `status` — among numeric/duration fields in the schema                                                                                                                                                                                                       | No numeric field matches and the user did not name one                                         |
| `thresholdValue` and `thresholdOperator`   | Use the user's stated threshold; otherwise run the threshold pipeline (Step 3) at P95 with operator `>`                                                                                                                                                                                                                                                                                                                                                             | The percentile or operator is ambiguous (e.g., user says "slow" but field is `status_code`)    |
| `fieldFilter` (optional scoping)           | Only include if the user explicitly scoped the analysis ("for the cart service")                                                                                                                                                                                                                                                                                                                                                                                    | Never invent a filter; ask only when the user implied scoping but did not name the field/value |
| `fieldsToAnalyze` (optional)               | Default to schema-driven selection — see `outlier-detection-field-selection` reference                                                                                                                                                                                                                                                                                                                                                                              | The user requested a specific subset that is unclear                                           |

Do not proceed to Step 2 until every required input above is resolved.

#### Narrow caveat — span perspective on trace-error questions

This caveat is narrow. It applies when **all three** of the following are true; if any one is missing, skip it and proceed normally:

1. The user named a specific service (e.g. `redis-result-cache`, `apiserver`) — not a generic noun like "pods", "the cluster", or "my service".
2. The question is about errors / failures / faults / exceptions / broken behavior on that service. Pure slowness questions do NOT count.
3. The investigation will run against distributed-trace data (`Tracing/Span`, OTEL spans). Metric, log, pod-health, and any other non-span investigations do NOT count.

When all three hold, the user's question is genuinely ambiguous: a request through a distributed system produces span records on both the receiver and the caller, so "errors on `X`" can mean (A) errors X experienced (server-side spans where `service_name = X` and `error = true`), (B) errors X caused for callers (client-side spans from OTHER services whose target was X), or (C) the full-trace view (all spans in traces that contained an error involving X). These three cohorts produce very different phi-correlation results, and silently picking (A) — which is what `service_name = "X"` filtering does — hides upstream root causes.

When all three preconditions hold, you MUST stop and ask the user to pick A / B / C before doing any other tool call: no query execution, no dataset/metric/schema lookups, no exploratory queries. Only after they answer may you compose the filter and continue. If they have already specified the perspective in their question, no clarification needed. When the user picks, also include `span_kind` (or `span_type`) in `scalarFields[]` so the inbound/outbound distinction is preserved in the phi output.

For metric, log, or non-trace investigations: this caveat does not apply — proceed normally.

### Step 2 — Pick the candidate fields to analyze

Load the [outlier-detection-field-selection](references/outlier-detection-field-selection.md) reference and apply its rules to the dataset schema. Categorize each selected field as scalar (string/number/boolean) or complex (Object/Array). You will need both lists in Step 4.

### Step 3 — Compute the threshold (only when not provided)

If the user did not give an explicit `thresholdValue`, load the [outlier-detection-threshold](references/outlier-detection-threshold.md) reference and run that OPAL pipeline first. Use the requested percentile (default P95) as the recommended threshold value, then ask the user to confirm before running the correlation analysis.

### Step 4 — Build and run the correlation pipeline

Load the [outlier-detection-pipeline](references/outlier-detection-pipeline.md) reference and assemble the pipeline using the templates verbatim, substituting only the placeholders. Execute it with whatever OPAL execution capability the host agent provides.

The pipeline is non-trivial; do NOT improvise. The phi formula `((a*d) - (b*c)) / sqrt((a+b)*(c+d)*(a+c)*(b+d))` and the cohort/window construction must match the templates verbatim.

Two rules the template enforces — read [outlier-detection-pipeline](references/outlier-detection-pipeline.md) carefully before composing the OPAL:

- **Field categorization.** Use the rules in [outlier-detection-field-selection](references/outlier-detection-field-selection.md) to split the chosen fields into `scalarFields[]` (string/number/bool) and `complexFields[]` (Object/Array). Never put a complex field into the scalar bundle — the `make_object("name":string(field), …)` step would coerce the entire object to a single string. Scalars are packed into `_scalar_fields_obj` and flattened once; complex fields each get their own empty-object guard and their own `flatten_leaves`.
- **Always include a scoping `filter`** when the user's question implies one (e.g. `filter service_name = "checkout"`, `filter error = true`). Never invent a filter when the user did not imply one — ask instead.

### Step 5 — Interpret the results

The pipeline returns one row per `(field, value)` candidate, sorted by `phi` descending and limited to 50 rows. Columns:

- `attribute` — the field name / JSON path (e.g., `service.name`, `db.statement`)
- `value` — the specific value (e.g., `checkout-service`, `TimeoutException`)
- `bad_count`, `good_count` — counts of rows in each cohort with this value
- `total_bad`, `total_good` — totals across the entire query
- `frequency_in_bad_cohort`, `frequency_in_good_cohort` — fractions (0..1)
- `phi` — phi coefficient in [-1, 1]

Load the [outlier-detection-interpretation](references/outlier-detection-interpretation.md) reference for phi/bad% bands, root-cause framing, and the recommended response structure.

#### When the result set is empty or weak

If the pipeline returned no rows, or every `phi` is below ~0.1 (no meaningful correlation), do not silently re-run with a wider window. Ask the user whether they want to:

1. **Widen the time range** (e.g. from 15 min to 1 hour) to get more samples.
2. **Adjust the threshold** (e.g. P95 → P90) to enlarge the bad cohort.
3. **Add or remove fields** from the analysis set.
4. **Add a scoping filter** (e.g. limit to a single service or environment).

Summarize what was tried (range, threshold, fields, filter) and let the user pick before re-running. Never widen the range silently; the original range was the user's stated intent.

## Things to NOT do

- Do not invent dataset IDs, field names, percentile values, threshold operators, or filter values. Ask the user.
- Do not regenerate the phi formula or rebuild the cohort/window logic from scratch — copy the template.
- Do not post-process or rename the query result columns. The pipeline already produces the final shape; read the rows directly.
- Do not include `validFromField` / `validToField` columns in the analysis field list — they are temporal columns, not categorical attributes.
- Do not include hidden, const, metric-aggregation, or `_`-prefixed fields.
- Do not analyze more than 10 fields at once — high cardinality kills query performance.
- Do not skip the empty-object / empty-array guard `make_col` step before `flatten_leaves` for complex fields, otherwise null/empty rows drop out and bias the cohort counts.
- Do not run on a raw spans / logs dataset without a scoping `filter` when the user's question implies one. If no scope is implied, ask the user.
- When (and only when) the narrow trace-perspective caveat in Step 1 applies, do not equate "errors on service X" with `filter service_name = "X" and error = true` and do not omit `span_kind` / `span_type` from the analyzed fields — both are easy ways to surface the wrong side of a distributed call. This caveat does not apply to metric, log, or non-trace investigations.

## Output format

Present the results as a ranked markdown table followed by a short root-cause narrative. See `outlier-detection-interpretation` for the exact format and example.
