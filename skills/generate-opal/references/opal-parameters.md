# Referencing parameters in OPAL

A parameter is a caller-controlled value exposed to a query's OPAL as `$paramId`. This reference covers how to **write the OPAL that consumes one**. Declaring a parameter is handled outside the pipeline; here we only cover the OPAL.

## The filter line

For a `tag` or `correlation-tag` parameter, scope a query by adding exactly:

```
filter array_contains($paramId, #tagName) or is_null($paramId)
```

- `$paramId` — the parameter, referenced with a leading `$`. It must already be declared before the OPAL is dry-run, or the reference fails to compile.
- `#tagName` — the correlation tag to scope by. It **must be a correlation tag mapped on this card's own input dataset**, not just any tag that exists somewhere in the Knowledge Graph. A `#tag` that is not mapped on the input dataset still compiles, but the filter silently matches nothing and the preview comes back with a `tag "tagName" not found` warning. Pull from KG context; do NOT invent.
- `or is_null($paramId)` — required. Without it, an unset (empty) parameter blanks the result.

Place the filter line **immediately after the dataset-entry verb(s)** and BEFORE any `align` / `aggregate` / `timechart` — scope before you aggregate.

During the correctness dry-run every declared parameter is bound to an **empty** value, ignoring any `defaultValue`, so `$paramId` references compile without you supplying a value. This means an unguarded `$paramId` still **compiles** — it does not error — but with the parameter empty it matches nothing, so the card returns no data and renders blank. A `defaultValue` does not change this: it only sets the picker's initial selection on the saved dashboard (and the preview image), never the value used at validation, and a reader who clears the picker binds empty regardless. That is why `or is_null($paramId)` is mandatory.

Agents that support more parameter view types beyond `tag` / `correlation-tag` (e.g. `text` / `numeric` / `bool` / `resource-instance`) have agent-specific references for those per-view-type filter patterns. This reference covers only the common multi-select tag case.

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

- **Forgetting `or is_null($paramId)`.** The card still compiles, but an empty parameter matches nothing → the card returns no data and renders blank. A default does NOT rescue it (validation binds empty and ignores defaults; a cleared picker binds empty too).
- **`#paramId` instead of `$paramId`.** Parameters use `$`; tags use `#`.
- **Referencing a `$paramId` that was never declared.** Dry-run fails with `reference to parameter <id>: no value supplied for it`. This is a _different_ problem from a missing guard: the fix is to declare (or reuse) the parameter — adding `or is_null` does NOT fix it, because it still references an unbound parameter.
- **Scoping by a `#tagName` not mapped on the input dataset.** The query compiles but the filter does nothing; the preview returns a `tag "tagName" not found` warning. Confirm the tag is in the dataset's correlation-tag mappings, or pick a real column on the dataset instead.
- **Putting the filter AFTER `align` / `aggregate` / `timechart`.** Scope belongs before aggregation.
