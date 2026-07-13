# Outlier Detection — OPAL Pipeline Templates

This is the core of the skill. The pipeline classifies every input row as `bad` or `good` based on a threshold, flattens scalar / nested fields into `(path, value)` pairs, and then computes a phi coefficient for every distinct `(path, value)` pair using a 2x2 contingency table:

|                     | bad cohort | good cohort |
| :------------------ | :--------- | :---------- |
| has this value      | a          | c           |
| does NOT have value | b          | d           |

`phi = (a*d - b*c) / sqrt((a+b)*(c+d)*(a+c)*(b+d))`

You MUST load `generate-opal` first. The templates below assume its rules: `@."Column Name"` quoting, `case()` paired arguments, `sort desc(col)` syntax, no `count_if`, no SQL `IN`, etc.

## Inputs (resolved by Steps 1-2 of the skill)

- `thresholdField` — the field that defines bad behavior (e.g., `duration`, `latency`, the aligned metric column).
- `thresholdOperator` — one of `>`, `>=`, `<`, `<=`, `=`, `!=`. Default `>`.
- `thresholdValue` — numeric / string value to compare against.
- `scalarFields[]` — string/number/bool fields to analyze.
- `complexFields[]` — Object / Array fields to flatten and analyze.
- `fieldFilter` (optional) — `{field, value}` for an upstream `filter` clause.
- `metricName` (optional) — when present, run the metric-mode prefix.

## Universal helpers

### Filter clause

If `fieldFilter` is present:

```
filter <field-opal> = <value-opal>
```

Otherwise omit the line entirely. Use `@."Field Name"` quoting for non-identifier names (see `generate-opal`).

### Sanitized column-name suffix

`flatten_leaves <col>` produces `_c_<col>_path` and `_c_<col>_value` in the output. When `<col>` contains characters outside `[a-zA-Z0-9_]`, OPAL substitutes `_` for each disallowed character. When you reference these output columns in `group_by` / `coalesce`, mirror that sanitization. For example, `flatten_leaves @."My Field"` produces `_c__My_Field__path` and `_c__My_Field__value` (one underscore per disallowed character, including the surrounding quotes). When in doubt, use plain identifier-safe names for the bundle column (`_scalar_fields_obj`).

## Template — dataset mode

Use this when the data source is the dataset directly (no metric alignment).

```
{{filter_clause}}
make_col threshold_field:{{threshold_field_opal}}
make_col cohort:if(threshold_field {{threshold_operator}} {{threshold_value_opal}}, "bad", "good")
make_col total_bad:window(count(case(cohort="bad", true)), group_by())
make_col total_good:window(count(case(cohort="good", true)), group_by())

{{scalar_obj_step}}
{{empty_fix_steps}}
{{scalar_flatten_step}}
{{complex_flatten_steps}}

statsby
    a:count(case(cohort="bad", true)),
    b:first_not_null(total_bad) - count(case(cohort="bad", true)),
    c:count(case(cohort="good", true)),
    d:first_not_null(total_good) - count(case(cohort="good", true)),
    group_by({{group_by_columns}})
filter a >= 1 or c >= 1
pick_col
    attribute:coalesce({{coalesce_attribute}}),
    value:coalesce({{coalesce_value}}),
    bad_count:a,
    good_count:c,
    total_bad:a + b,
    total_good:c + d,
    frequency_in_bad_cohort:a / (a + b),
    frequency_in_good_cohort:c / (c + d),
    phi:((a * d) - (b * c)) / sqrt((a + b) * (c + d) * (a + c) * (b + d))
sort desc(phi)
dedup
limit 50
```

### Building each substitution

`{{scalar_obj_step}}` — only when `scalarFields[]` is non-empty. Pack scalars into a single object whose keys are the field names (string-literals) and values are the scalar columns cast to `string()`:

```
make_col _scalar_fields_obj:make_object("svc":string(svc), "env":string(env), "status":string(status))
```

Always cast each value with `string(...)`. Always use double-quoted field names as keys.

`{{scalar_flatten_step}}` — only when `scalarFields[]` is non-empty:

```
flatten_leaves _scalar_fields_obj
```

`{{empty_fix_steps}}` — one block per complex field. Required so empty objects / empty arrays still produce a row in the flattened output (otherwise null/empty rows silently drop and bias the cohort counts).

For Object-typed fields:

```
make_col attributes: if(
    is_null(attributes) or attributes = make_object(),
    make_object(__empty__: string_null()),
    attributes
)
```

For Array-typed fields:

```
make_col tags: if(
    is_null(tags) or tags = make_array(),
    make_array(make_object(__empty__: string_null())),
    tags
)
```

`{{complex_flatten_steps}}` — one `flatten_leaves` per complex field, in the same order they appear in `complexFields[]`:

```
flatten_leaves attributes
flatten_leaves resource_attributes
```

`{{group_by_columns}}` — for every flattened "field handle" (in order: `_scalar_fields_obj` first when scalars exist, then each complex field), include both `_c_<sanitized>_path` and `_c_<sanitized>_value`:

```
_c__scalar_fields_obj_path, _c__scalar_fields_obj_value, _c_attributes_path, _c_attributes_value, _c_resource_attributes_path, _c_resource_attributes_value
```

`{{coalesce_attribute}}` — list of `_c_<sanitized>_path` columns in the same order:

```
_c__scalar_fields_obj_path, _c_attributes_path, _c_resource_attributes_path
```

`{{coalesce_value}}` — list of `_c_<sanitized>_value` columns wrapped in `string(...)`, same order:

```
string(_c__scalar_fields_obj_value), string(_c_attributes_value), string(_c_resource_attributes_value)
```

### Edge cases

- **Scalar-only input** — omit `{{empty_fix_steps}}` and `{{complex_flatten_steps}}`; only `_scalar_fields_obj` participates in the flatten / group_by / coalesce.
- **Complex-only input** — omit `{{scalar_obj_step}}` and `{{scalar_flatten_step}}`; group_by / coalesce reference only the complex fields.
- **Empty input** — abort and ask the user to specify `fieldsToAnalyze`. Never run the pipeline with no fields.

## Template — metric mode

Use this when `metricName` is provided. The metric prefix aligns the metric value as the threshold field, then defers to the same `statsby` / phi shape used in dataset mode with `complexFields[] = []` (metric tags are scalar by construction).

CRITICAL OPAL rule: `align` does NOT take a `group_by(...)` argument. The metric's tag columns are preserved automatically as implicit grouping; if you pass an explicit `group_by` to `align`, OPAL rejects it with `"align" should always group by all metric tag columns, and does that implicitly`. After `align`, reference each tag column by name (for Prometheus-style metrics this is usually `labels.<tag>`; for OTEL metrics it is the unprefixed tag name). Build `_scalar_fields_obj` from those tag references.

Replace `{{metric}}` with the metric name. `{{scalar_obj_args}}` is the comma-separated list of `"name":string(<tag-reference>)` pairs for the metric tags you're analyzing.

```
{{filter_clause}}
align A_{{metric}}:sum(m("{{metric}}"))
make_col threshold_field:A_{{metric}}
make_col cohort:if(threshold_field {{threshold_operator}} {{threshold_value}}, "bad", "good")
make_col total_bad:window(count(case(cohort="bad", true)), group_by())
make_col total_good:window(count(case(cohort="good", true)), group_by())

make_col _scalar_fields_obj:make_object({{scalar_obj_args}})
flatten_leaves _scalar_fields_obj

statsby
    a:count(case(cohort="bad", true)),
    b:first_not_null(total_bad) - count(case(cohort="bad", true)),
    c:count(case(cohort="good", true)),
    d:first_not_null(total_good) - count(case(cohort="good", true)),
    group_by(_c__scalar_fields_obj_path, _c__scalar_fields_obj_value)
filter a >= 1 or c >= 1
pick_col
    attribute:coalesce(_c__scalar_fields_obj_path),
    value:coalesce(string(_c__scalar_fields_obj_value)),
    bad_count:a,
    good_count:c,
    total_bad:a + b,
    total_good:c + d,
    frequency_in_bad_cohort:a / (a + b),
    frequency_in_good_cohort:c / (c + d),
    phi:((a * d) - (b * c)) / sqrt((a + b) * (c + d) * (a + c) * (b + d))
sort desc(phi)
dedup
limit 50
```

Notes:

- Do NOT pass `group_by(...)` to `align`. Reference metric tag columns directly after `align` (e.g. `string(labels.k8s_namespace_name)`, `string(labels.image)` for Prometheus, or just `string(k8s_namespace_name)` for OTEL).
- Use `sum(m(...))` for counters, `avg(m(...))` for gauges, etc. — pick the aggregation that matches the metric's semantics.
- For native histogram metrics (`m_histogram`), only use `histogram_combine` + `histogram_quantile` when the dataset's metric interface defines `objectValue` for the histogram column. If the engine returns `OB-145: Selecting metric "m_object" requires the "metric" interface to have a objectValue column`, the metric is a Prometheus-style sibling-counter histogram (`_bucket`/`_sum`/`_count`) and there is no native histogram interface to use here — pick a different metric or fall back to dataset mode against the metric's source dataset.
- The threshold value is numeric and NOT string-quoted (the aligned metric column is numeric).

## Result columns (verbatim)

After the pipeline runs you will get exactly these columns, sorted by `phi` desc, deduped, limited to 50 rows:

| Column                     | Type   | Meaning                                              |
| :------------------------- | :----- | :--------------------------------------------------- |
| `attribute`                | string | The flattened path / field name                      |
| `value`                    | string | The specific value at that path                      |
| `bad_count`                | int    | Rows in the bad cohort with this value               |
| `good_count`               | int    | Rows in the good cohort with this value              |
| `total_bad`                | int    | Total rows in the bad cohort across the whole query  |
| `total_good`               | int    | Total rows in the good cohort across the whole query |
| `frequency_in_bad_cohort`  | float  | `bad_count / total_bad`                              |
| `frequency_in_good_cohort` | float  | `good_count / total_good`                            |
| `phi`                      | float  | Phi coefficient in `[-1, 1]`                         |

Do not rename or post-process these columns. Read them as-is when interpreting results.

## Validation checklist before submitting the pipeline

- `generate-opal` is loaded and the OPAL syntax conforms to its rules.
- Every column reference in `group_by` / `coalesce` matches a `flatten_leaves` output, with sanitized name.
- For each complex field there is exactly one `make_col` empty-fix step and exactly one `flatten_leaves`.
- Scalar fields are bundled into `_scalar_fields_obj` with `string(...)` casts before `flatten_leaves`.
- The `case()` calls in `statsby` use paired `(condition, result)` arguments and the bare value `true` (no SQL `1`).
- The phi formula is byte-identical to the template.
- `sort desc(phi)` is followed by `dedup` and `limit 50`.
