# Outlier Detection — Field Selection Rules

You are choosing the categorical fields whose values will be tested for correlation with the "bad" cohort. Quality of results depends on choosing fields with semantically meaningful values, not high-cardinality unique IDs.

## Per-dataset-kind heuristics

Use these defaults when the user has not named specific fields. Always intersect them with the actual schema field list.

- **OTEL spans / traces**: `service_name`, `span_name`, `status_code`, `error_type`, `attributes`, `resource_attributes`. When the schema exposes `span_kind` (or `span_type`), include it in scalarFields[] for trace investigations that involve a single named service or a call-graph question — it distinguishes inbound (`server`) from outbound (`client`) spans, which is high-signal for root-cause analysis in distributed traces. For non-trace data this field does not apply.
- **Metrics**: `labels`, `tags`, dimension columns referenced after `align` (metric tag columns are preserved implicitly; reference each one by name when packing into `_scalar_fields_obj`).
- **Logs / events**: `service_name`, `host`, `environment`, log-level fields, error-type fields, structured `attributes` / `body` JSON.

Prefer including BOTH scalar fields (high-level categorization) AND complex fields (Object / Array — for nested attribute analysis) when the dataset has them.

## Required exclusions

For every candidate field, skip it if any of the following are true:

- The field name equals the dataset's `validFromField` or `validToField` (temporal columns).
- The field name equals the threshold field used to define the bad cohort (the cohort would be tautologically perfectly correlated with itself).
- The field name starts with `_` (internal / system field).
- The field's schema has `isHidden: true`, `isConst: true`, or `isMetric: true`.
- The field is not present in the dataset schema's `fieldList`.

## Scalar vs complex classification

After exclusions, classify each remaining selected field by its type tag:

- **Complex** — `Object` or `Array` types (the OPAL pipeline will run `flatten_leaves` on these and may need an empty-fix `make_col` first).
- **Scalar** — everything else (strings, numbers, booleans, durations, timestamps used as categorical). The OPAL pipeline will pack these into a `make_object(...)` and then `flatten_leaves` that bundle.

If both lists are empty after exclusions, abort: there is nothing to analyze; ask the user to specify `fieldsToAnalyze` directly.

## Sizing

Keep the selected list to 2–10 fields total. More than 10 leads to combinatorial cardinality explosion in the `statsby ... group_by(...)` step. If the schema has many candidates, prefer the most semantically meaningful (error / status / service / region) over generic high-cardinality identifiers (request_id, trace_id, span_id, user_id).

## Output of this step

Produce two ordered lists for the pipeline assembly step:

```text
scalarFields:  ["service_name", "status_code", "environment"]
complexFields: ["attributes", "resource_attributes"]
```

Plus a list of skipped fields and reasons (useful when explaining the analysis to the user).
