# Referencing parameters in OPAL

A parameter is a caller-controlled value exposed to a query's OPAL as `$paramId`. This reference covers how to **write the OPAL that consumes one**. Declaring a parameter is handled outside the pipeline; here we only cover the OPAL.

## The filter line

For a `tag` or `correlation-tag` parameter, scope a query by adding exactly:

```
filter array_contains($paramId, #tagName) or is_null($paramId)
```

-   `$paramId` — the parameter, referenced with a leading `$`. It must already be declared before the OPAL is dry-run, or the reference fails to compile.
-   `#tagName` — the metric or correlation tag to scope by. Pull from KG context; do NOT invent.
-   `or is_null($paramId)` — required. Without it, an unset (empty) parameter blanks the result.

Place the filter line **immediately after the dataset-entry verb(s)** and BEFORE any `align` / `aggregate` / `timechart` — scope before you aggregate.

A declared parameter is automatically bound (to its `defaultValue`, or an empty value) when the OPAL is dry-run, so `$paramId` references compile without you supplying a value.

## Reusing a parameter across queries

Reference the same `$paramId` in any query's OPAL. Use already-declared parameter ids verbatim, and do NOT redeclare one.

## Multiple axes on one query

Chain one filter line per axis, before aggregation:

```
filter array_contains($cluster, #k8s.cluster.name) or is_null($cluster)
| filter array_contains($customer, #customer.id) or is_null($customer)
| timechart 1m, qps:rate(count())
```

## Common mistakes

-   **Forgetting `or is_null($paramId)`.** An unset parameter → blanks the result.
-   **`#paramId` instead of `$paramId`.** Parameters use `$`; tags use `#`.
-   **Referencing a `$paramId` that was never declared.** Dry-run fails with an undefined-parameter error.
-   **Putting the filter AFTER `align` / `aggregate` / `timechart`.** Scope belongs before aggregation.
