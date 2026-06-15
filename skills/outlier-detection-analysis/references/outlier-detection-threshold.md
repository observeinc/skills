# Outlier Threshold Calculation Pipeline

When the user has not specified an explicit threshold, run this pipeline first to derive a data-driven value. Default to P95 with operator `>` unless the user requested otherwise.

## Inputs

-   `datasetId` — required.
-   `metricName` — required when computing from a metric (otherwise the pipeline runs over the dataset directly).
-   `performanceField` — required for non-metric mode. Must be a numeric / duration field. Auto-detect from the schema using the names `duration`, `latency`, `response_time`, `request_duration`, `elapsed_time`, `processing_time`, `execution_time`, `bytes`, `response_size`, `request_size`, `status_code`, `status`. If none match, ask the user.
-   `percentileThreshold` — number in `[50, 99.9]`. Default `95`.
-   `fieldFilter` (optional) — `{field, value}` to scope the pipeline.
-   `timeStart`, `timeEnd` — required.

If any required input is missing or ambiguous, ask the user before running the pipeline. Do not guess.

## Pipeline — non-metric (dataset) mode

Replace `{{filter_clause}}` with `filter <field> = <value>` if the user provided a scoping filter, or with an empty line otherwise. Replace `{{perf_field}}` with the chosen field reference (use `@."Field Name"` quoting per the `generate-opal` rules when the name has spaces).

```
{{filter_clause}}
statsby
    count:count(),
    mean:avg({{perf_field}}),
    median:percentile({{perf_field}}, 0.5),
    stddev:stddev({{perf_field}}),
    min:min({{perf_field}}),
    max:max({{perf_field}}),
    p50:percentile({{perf_field}}, 0.5),
    p75:percentile({{perf_field}}, 0.75),
    p90:percentile({{perf_field}}, 0.9),
    p95:percentile({{perf_field}}, 0.95),
    p99:percentile({{perf_field}}, 0.99)
```

## Pipeline — metric mode

Replace `{{metric}}` with the metric name (string-quoted in `m(...)`) and `{{aligned_col}}` with `A_<metric>` (use `@."A_<metric>"` for the column reference).

```
{{filter_clause}}
align {{aligned_col}}:avg(m("{{metric}}"))
make_col {{perf_field}}:{{aligned_col_ref}}
statsby
    count:count(),
    mean:avg({{perf_field}}),
    median:percentile({{perf_field}}, 0.5),
    stddev:stddev({{perf_field}}),
    min:min({{perf_field}}),
    max:max({{perf_field}}),
    p50:percentile({{perf_field}}, 0.5),
    p75:percentile({{perf_field}}, 0.75),
    p90:percentile({{perf_field}}, 0.9),
    p95:percentile({{perf_field}}, 0.95),
    p99:percentile({{perf_field}}, 0.99)
```

## Picking the threshold value

After execution, read the single result row and pick the column matching the requested percentile (`p50`, `p75`, `p90`, `p95`, or `p99`). Default to `p95`. The recommended threshold is:

```text
field:    <performanceField>
operator: >
value:    <percentile column from result>
```

## Why P95 by default

P95 is the industry default for performance monitoring because:

-   It is robust to a small number of extreme outliers, unlike `mean`.
-   It captures the experience of the vast majority of requests while still flagging the slow tail.
-   It produces a `bad` cohort large enough (~5%) for stable phi-coefficient estimation.

If the user said "extremely slow", lean to P99. If they said "any slowness", lean to P75 / P90.

## Confirming with the user

Before passing the computed value to the correlation pipeline, summarize the distribution and proposed threshold to the user and ask them to confirm. Example:

> Across 124,317 samples of `duration`, P95 = 482ms. I'll use `duration > 482ms` as the "bad" threshold. Confirm?

If the user changes the percentile, operator, or field, re-run this pipeline before continuing.
