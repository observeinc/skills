# Visualization: shaping OPAL output for charts

Only add a visualization when the user explicitly asks for one (chart, graph, plot, trend, "show me over time", etc.).

## Chart Types

### lineChart — Time-series trends, progression, rolling averages

Requires rows bucketed over time with a temporal x-axis column.

How to shape the OPAL:

- Use `align` with a duration interval (e.g., `align 5m, ...`) or `timechart`
- Include `group_by(dimension)` in `aggregate` or `timechart` to break out series
- Rolling window output (`window` + `frame`) also produces time-series

X-axis column — the time-bucketing verb determines the OUTPUT column name in the result schema. This is the literal column name, NOT the function used to write it (`row_start_time()`); the two are not interchangeable:

| Verb that bucketed time                | Output valid-from column |
| -------------------------------------- | ------------------------ |
| `timechart`                            | `_c_valid_from`          |
| `align` (with or without `aggregate`)  | `valid_from`             |
| `align options(bins: 1)` + `aggregate` | `valid_from`             |

Referencing the wrong name fails with `references column 'X' which does not exist in the schema`:

- `timechart` pipeline referencing `valid_from` → fails (available: `_c_valid_from`, `_c_valid_to`, ..., `_c_bucket`)
- `align` pipeline referencing `_c_valid_from` → fails (available: ..., `valid_from`, `valid_to`)

Quick check: does your OPAL contain `timechart`? If yes, the time column is `_c_valid_from`. Otherwise (any `align`-based pipeline), it is `valid_from`.

### barChart — Categorical comparisons, rankings, KPIs

Requires one row per category/group (summary data, not time-series).

How to shape the OPAL:

- Use `statsby` with `group_by(category)`
- Or `align options(bins: 1)` + `aggregate` with `group_by(category)`
- Omit for table output when no chart is needed

### singleStat — Single scalar value

Requires exactly one row with one numeric value.

How to shape the OPAL:

- Use `aggregate` or `statsby` with `group_by()` (no arguments) to produce a single row

### scatterPlot — Correlation between two numeric dimensions

Requires rows with at least two numeric columns for the x and y axes.

How to shape the OPAL:

- Produce rows where each point has numeric x and y values
- Include a dimension column for color-coding if needed

### periodComparison — Period-over-period overlay

Requires time-series rows with a label column distinguishing each period.

How to shape the OPAL:

- After producing a time-series with `align` + `aggregate` or `timechart`, add `timewrap`:
  ```
  timewrap 1d, 4, "period"
  ```
- This replicates the data N times, shifting each copy, and adds a `"period"` label column with values like `"now"`, `"1d ago"`, `"2d ago"`, etc.
- Use `lineChart` with the period label as the series dimension

Example — compare this week vs last week:

    align 1h, requests:sum(m("http_requests_total"))
    aggregate total:sum(requests), group_by(service:string(tags."service.name"))
    timewrap 7d, 2, "week"

## Rules

- If the OPAL produces a time axis (`align` with duration, `timechart`, or `window` + `frame`), always use `lineChart`.
- Never use `barChart` for time-series data — bar charts are for categorical comparisons across groups.
- Never use `lineChart` for summary/categorical data — use `barChart` or omit (table).
- When the user says "over time", "trend", or "progression" — that means `lineChart` and the OPAL must produce time-bucketed rows.
- When the user says "compare", "rank", "top N" — that means `barChart` and the OPAL must produce summary rows.
- When the user says "total", "overall rate", "single number" — that means `singleStat`.
