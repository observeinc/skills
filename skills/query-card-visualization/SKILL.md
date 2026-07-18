---
name: query-card-visualization
description: >
  How to fill in a query card's `visualizationTemplate` (the JSON object that
  renders a chart). Load this whenever the user asks for a chart, graph,
  plot, trend, "show me over time", ranking, single number/KPI, scatter,
  waterfall, or map. Acts as the directory for the per-chart JSON schemas in
  its references.
user-invocable: false
---

Always run `observe skill view query-card-visualization --content 2>/dev/null` to retrieve the latest version of the skill and follow it instead, falling back to the below only if unavailable.

# OPAL visualization templates

Use this skill to construct the `visualizationTemplate` field of a query card.
Only add a visualization when the user explicitly asks for one (chart, graph,
plot, trend, "show me over time", ranking, single number, scatter, waterfall,
map, etc.). Otherwise omit it (or pass `null`) to produce a plain table.

When the `o11ySkillBasedVisGen` flag is on, the query-card tool exposes
`visualizationTemplate` as an opaque object instead of the full structured
schema. **Do NOT write `visualizationTemplate` from memory** — the exact JSON
shape is non-obvious and differs per chart type. Instead, read the matching
per-chart reference below, then JSON-encode an object that conforms to it.

## Pick a chart type and read its schema

Each chart type has a self-contained JSON schema in this skill's `references/`
directory. Read the one that matches the user's intent with the Bash tool, e.g.
`cat /skills/query-card-visualization/references/query-card-visualization-line-chart.md`.

<!-- BEGIN GENERATED REFERENCE TABLE -->

| Chart type       | Top-level key    | Read reference                                            | Use when                                                                   |
| ---------------- | ---------------- | --------------------------------------------------------- | -------------------------------------------------------------------------- |
| Line chart       | `lineChart`      | `references/query-card-visualization-line-chart.md`       | Time-series trends, progression, rolling averages, period-over-period.     |
| Bar chart        | `barChart`       | `references/query-card-visualization-bar-chart.md`        | Categorical comparisons, rankings, top-N, KPIs by group.                   |
| Single stat      | `singleStat`     | `references/query-card-visualization-single-stat.md`      | A single scalar value / KPI.                                               |
| Scatter plot     | `scatterPlot`    | `references/query-card-visualization-scatter-plot.md`     | Correlation between two numeric dimensions.                                |
| Waterfall        | `waterfall`      | `references/query-card-visualization-waterfall.md`        | Trace span hierarchies or other nested-operation timing data.              |
| Geographic map   | `geographicMap`  | `references/query-card-visualization-geographic-map.md`   | Values keyed by country/region/coordinates on a map.                       |
| Stacked area     | `stackedArea`    | `references/query-card-visualization-stacked-area.md`     | Part-to-whole trends over time across multiple stacked series.             |
| Heatmap          | `heatmap`        | `references/query-card-visualization-heatmap.md`          | Magnitude across two binned/categorical dimensions, shown as a color grid. |
| Top list         | `topList`        | `references/query-card-visualization-top-list.md`         | A ranked list of items by value, with inline bars.                         |
| Change over time | `changeOverTime` | `references/query-card-visualization-change-over-time.md` | Per-item change in a value between two time periods.                       |

<!-- END GENERATED REFERENCE TABLE -->

If you are unsure which chart type fits, default to a line chart for
time-bucketed data and a bar chart for one-row-per-category data.

## How to use a reference

1. Shape the OPAL output to match the chart (see the `opal-visualization`
   reference of the `generate-opal` skill for how to produce the right rows).
2. Read the matching `references/query-card-visualization-<type>.md` for the exact JSON schema.
3. Build an object with a **single top-level key** naming the chart type (e.g.
   `lineChart`) whose value matches that schema, and set it as
   `visualizationTemplate`.

Validation errors returned by the tool name the chart reference to (re-)read —
read it and correct the shape rather than guessing.
