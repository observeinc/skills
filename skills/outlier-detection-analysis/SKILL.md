---
name: outlier-detection-analysis
description: "Phi-coefficient of field values vs a bad/good cohort for why X is slow/failing or which attributes they share/have in common, not a plain top-N ranking."
---

**MANDATORY:** Run `observe skill view outlier-detection-analysis --content 2>/dev/null` before proceeding to load the latest version of this skill. When the skill points you to a supporting file, load it with `observe skill view outlier-detection-analysis --path <relative-path> 2>/dev/null`. Ignore errors and fall back to the file contents.

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

1. **Standalone.** The user directly asks "what attributes correlate with slow / failed / unusual rows", "why is X bad", "what's different about the bad cohort", or similar. The agent loads this skill and runs through Step 1 — Step 5 from scratch. A plain ranking ("top 10 memory-hot hosts", "which hosts use the most memory compared with the fleet") is not this mode — do not load the skill.
2. **As a companion to `alert-investigation`.** When the broader alert-investigation methodology reaches the point of asking "which attributes / services / hosts / regions correlate with the bad behavior?", the alert-investigation skill should defer the phi-coefficient sub-problem to this skill rather than reinventing it. In that mode, some Step 1 inputs (dataset, time range, scoping filter, named service) are typically already resolved by the outer investigation and you can carry them forward — but you still need to determine `thresholdField`, `thresholdValue` / `thresholdOperator`, and `fieldsToAnalyze`, and you still need to run the narrow span-perspective caveat (Step 1) before composing any trace filter.

The phi algorithm itself is the same in both modes. This skill is the canonical place that owns it; do not duplicate the pipeline elsewhere.

## Prerequisites — load these skills first

You MUST use `generate-opal` skill and relevant references first before writing OPAL. The pipeline templates in this skill rely on `flatten_leaves`, `statsby`, `align`, `case`, `window`, and `coalesce` semantics that are documented there.

## Workflow

Choose the mode once, at the start of the run. This paragraph is the source of the rule. `alert-investigation`, `investigate-service-latency`, and `investigate-error-or-exception` keep a short inline copy for when this skill is not mounted (CLI/External); do not invent a third detector.

- **Non-interactive** when the prompt or system context contains any of these product-emitted sentences: `no user available to answer questions`, `Do not stop for clarification`, or `No one will answer questions during this task`. Match the full sentence, not a substring such as `headless`, `auto-investigate`, or `subagent` — those appear in interactive tool lists and ordinary user text (Kubernetes headless services, `transformer-headless-*`). Do not wait for a reply that cannot arrive. State each assumption and continue.
- **Interactive** otherwise, including an alert investigation where a person is in the chat. Prefer `AskUserQuestion` when it is in the tool list; otherwise ask in plain text and wait. Absence of `AskUserQuestion` alone is not a non-interactive signal — external chat clients still have a human who can reply. Do not treat the word "alert" by itself as non-interactive.

Every "ask the user" instruction below applies only in interactive mode.

### Step 1 — Resolve required inputs

Resolve each input from the user message, page / selection context, or dataset-discovery tools. Do not guess values that change the cohort or scope.

**Interactive:** ask for any required input you cannot determine. Do not continue until it is resolved.

**Non-interactive:** apply the fallback after the table, state it, and continue. If discovery returns no dataset at all, say so and stop — that is a failed lookup, not a question.

If the user asked to walk through the inputs, or not to run yet, state the resolved inputs (or the non-interactive fallbacks you would use) and the next step, and do not run the pipeline. In non-interactive mode that walk-through is the answer — do not wait and do not run.

| Input                                      | How to resolve                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Ask (interactive only) when                                                                    |
| :----------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------------- |
| Dataset (and optional metric)              | Use whatever dataset/metric discovery capability the host agent provides; prefer schemas that match the user's intent                                                                                                                                                                                                                                                                                                                                               | More than one plausible match, or no clear match                                               |
| Time range (`timeStart`, `timeEnd`)        | Use the time range stated by the user or implied by page / selection context, **exactly as given** — do not silently widen it. If the user did not specify a range, default to the **last 15 minutes** for the first run. The phi pipeline is expensive on high-volume datasets, so always start with the smallest range that satisfies the user's question and only widen later (Step 5) if the first run produced no meaningful correlations and the user agrees. | The user implied "over the past X" but the value is ambiguous                                  |
| `thresholdField` (the "performance" field) | Match against common names — `duration`, `latency`, `response_time`, `request_duration`, `elapsed_time`, `processing_time`, `execution_time`, `bytes`, `response_size`, `request_size`, `status_code`, `status` — among numeric/duration fields in the schema                                                                                                                                                                                                       | No numeric field matches and the user did not name one                                         |
| `thresholdValue` and `thresholdOperator`   | Use the user's stated threshold; otherwise run the threshold pipeline (Step 3) at P95 with operator `>`                                                                                                                                                                                                                                                                                                                                                             | The percentile or operator is ambiguous (e.g., user says "slow" but field is `status_code`)    |
| `fieldFilter` (optional scoping)           | Only include if the user explicitly scoped the analysis ("for the cart service")                                                                                                                                                                                                                                                                                                                                                                                    | Never invent a filter; ask only when the user implied scoping but did not name the field/value |
| `fieldsToAnalyze` (optional)               | Default to schema-driven selection — see `outlier-detection-field-selection` reference                                                                                                                                                                                                                                                                                                                                                                              | The user requested a specific subset that is unclear                                           |

Non-interactive fallbacks (state each one you use):

- Dataset: pick the best discovery match for the user's intent. If several look equally good, pick the one whose schema has a performance field from the list above.
- Time range: last 15 minutes when unspecified; when "over the past X" is ambiguous, use 15 minutes.
- `thresholdField`: if the user asked about errors / failures / faults / exceptions / broken behavior, pick boolean `error` if present, else the schema's status equivalent (`status`, `response_status`, `status_code`). Do not pick `duration` or another latency field — that builds a slowness cohort. If they asked about slowness / latency / performance, skip `status_code` and `status` — those measure errors, not duration; take the first remaining common name, else the first numeric/duration field that is not `status_code` / `status`. Otherwise first matching common name. If none exists, say so and stop.
- `thresholdValue` / operator: for an error/failure question, operator `=` and value `true` on `error` (or `Error` / `ERROR` on a status field). Skip Step 3. For slowness: Step 3 at P95 with `>`. Do not rewrite a slowness question onto `status_code >= 500`.
- `fieldFilter`: include only an explicit user scope. If they implied a scope but did not name field/value, omit the filter.
- `fieldsToAnalyze`: schema-driven selection.

Do not proceed to Step 2 until every required input above is resolved (a stated non-interactive fallback counts as resolved).

#### Narrow caveat — span perspective on trace-error questions

This caveat is narrow. It applies when **all three** of the following are true; if any one is missing, skip it and proceed normally:

1. The user named a specific service (e.g. `redis-result-cache`, `apiserver`) — not a generic noun like "pods", "the cluster", or "my service".
2. The question is about errors / failures / faults / exceptions / broken behavior on that service. Pure slowness questions do NOT count.
3. The investigation will run against distributed-trace data (`Tracing/Span`, OTEL spans). Metric, log, pod-health, and any other non-span investigations do NOT count.

When all three hold, the user's question is genuinely ambiguous: a request through a distributed system produces span records on both the receiver and the caller, so "errors on `X`" can mean (A) errors X experienced (server-side spans where `service_name = X` and `error = true`), (B) errors X caused for callers (client-side spans from OTHER services whose target was X), or (C) the full-trace view (all spans in traces that contained an error involving X). These three cohorts produce very different phi-correlation results, and silently picking (A) — which is what `service_name = "X"` filtering does — hides upstream root causes.

When all three preconditions hold, the perspective is unresolved until you pick A / B / C. If they have already specified it in their question, no clarification needed. Always include `span_kind` (or `span_type`) in `scalarFields[]` so the inbound/outbound distinction is preserved in the phi output.

- **Interactive:** stop and ask A / B / C before any other tool call: no query execution, no dataset/metric/schema lookups, no exploratory queries. Only after they answer may you compose the filter and continue.
- **Non-interactive:** do not wait. Default to (C) full-trace so you do not hide caller-side or upstream attributes: use the (C) full-trace cohort snippet in [outlier-detection-pipeline](references/outlier-detection-pipeline.md) (traces containing service X with an error, then all spans in those traces). State that assumption. Include `service_name` in `scalarFields[]`. Do not apply `filter service_name = "X"` as the sole scope; that is (A).

For metric, log, or non-trace investigations: this caveat does not apply — proceed normally.

### Step 2 — Pick the candidate fields to analyze

Load the [outlier-detection-field-selection](references/outlier-detection-field-selection.md) reference and apply its rules to the dataset schema. Categorize each selected field as scalar (string/number/boolean) or complex (Object/Array). You will need both lists in Step 4.

### Step 3 — Compute the threshold (only when not provided)

Skip this step when an error/failure fallback already set an equality cohort (`error = true` or the status equivalent). If the user did not give an explicit `thresholdValue`, load the [outlier-detection-threshold](references/outlier-detection-threshold.md) reference and run that OPAL pipeline first. Use the requested percentile (default P95) as the recommended threshold value. Confirm it with `AskUserQuestion` only when the run is interactive, `AskUserQuestion` is in the tool list, and the user has not already said to proceed. Otherwise state the threshold in the answer and run the correlation without waiting; this step never blocks on a plain-text reply.

### Step 4 — Build and run the correlation pipeline

Load the [outlier-detection-pipeline](references/outlier-detection-pipeline.md) reference and assemble the pipeline using the templates verbatim, substituting only the placeholders. Execute it with whatever OPAL execution capability the host agent provides.

The pipeline is non-trivial; do NOT improvise. The phi formula `((a*d) - (b*c)) / sqrt((a+b)*(c+d)*(a+c)*(b+d))` and the cohort/window construction must match the templates verbatim.

Two rules the template enforces — read [outlier-detection-pipeline](references/outlier-detection-pipeline.md) carefully before composing the OPAL:

- **Field categorization.** Use the rules in [outlier-detection-field-selection](references/outlier-detection-field-selection.md) to split the chosen fields into `scalarFields[]` (string/number/bool) and `complexFields[]` (Object/Array). Never put a complex field into the scalar bundle — the `make_object("name":string(field), …)` step would coerce the entire object to a single string. Scalars are packed into `_scalar_fields_obj` and flattened once; complex fields each get their own empty-object guard and their own `flatten_leaves`.
- **Always include a scoping `filter`** when the user's question implies one (e.g. `filter service_name = "checkout"`, `filter error = true`), except when the Step 1 span-perspective caveat applies: follow that caveat's A / B / C choice instead of a lone `service_name` filter. Never invent a filter when the user did not imply one. Interactive: ask. Non-interactive: omit the filter and say so.

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

If the pipeline returned no rows, or every `phi` is below ~0.1 (no meaningful correlation), do not silently re-run with a wider window. Summarize what was tried (range, threshold, fields, filter).

- **Interactive:** ask whether they want to (1) widen the time range (e.g. 15 min → 1 hour), (2) adjust the threshold (e.g. P95 → P90), (3) add or remove fields, or (4) add a scoping filter. Wait for that pick before re-running.
- **Non-interactive:** report the empty/weak result and list those four next-step options as suggestions in the final answer. Do not wait, and do not widen.

## Things to NOT do

- Do not load this skill for a top-N / ranking / "which entities have the most X compared with the fleet" question. That is a per-entity aggregate of the metric (for host memory: `system.memory.utilization` with `state="used"`, grouped by `host.name`, averaged over the window). Load `generate-opal` and query; do not run the phi pipeline.
- Do not invent dataset IDs, field names, percentile values, threshold operators, or filter values. Interactive: ask. Non-interactive: use the Step 1 fallbacks and state them.
- Do not regenerate the phi formula or rebuild the cohort/window logic from scratch — copy the template.
- Do not post-process or rename the query result columns. The pipeline already produces the final shape; read the rows directly.
- Do not include `validFromField` / `validToField` columns in the analysis field list — they are temporal columns, not categorical attributes.
- Do not include hidden, const, metric-aggregation, or `_`-prefixed fields.
- Do not analyze more than 10 fields at once — high cardinality kills query performance.
- Do not skip the empty-object / empty-array guard `make_col` step before `flatten_leaves` for complex fields, otherwise null/empty rows drop out and bias the cohort counts.
- Do not run on a raw spans / logs dataset without a scoping `filter` when the user's question implies one. If no scope is implied: interactive, ask; non-interactive, omit the filter and say so.
- When (and only when) the narrow trace-perspective caveat in Step 1 applies, do not equate "errors on service X" with `filter service_name = "X" and error = true` and do not omit `span_kind` / `span_type` from the analyzed fields — both are easy ways to surface the wrong side of a distributed call. This caveat does not apply to metric, log, or non-trace investigations.

## Output format

Present the results as a ranked markdown table followed by a short root-cause narrative. See `outlier-detection-interpretation` for the exact format and example.
