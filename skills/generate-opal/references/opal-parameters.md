# Referencing parameters in OPAL

A parameter is a caller-controlled value exposed to a query's OPAL as `$paramId`. This reference covers how to **write the OPAL that consumes one**. Declaring a parameter is handled outside the pipeline; here we only cover the OPAL.

Write a `$paramId` into a query only when your own instructions have you author parameter filters in OPAL. Where your parameter tools bind cards themselves and your instructions forbid hand-writing a `$token`, follow them: this reference does not override them, and the text and number filters below are not an exception.

## The filter line

For a `tag` or `correlation-tag` parameter, scope a query by adding exactly:

```
filter array_contains($paramId, #tagName) or is_null($paramId)
```

- `$paramId` — the parameter, referenced with a leading `$`. It must already be declared before the OPAL is dry-run, or the reference fails to compile.
- `#tagName` — the correlation tag to scope by. It **must be a correlation tag mapped on this card's own input dataset**, not just any tag that exists somewhere in the workspace. A `#tag` that is not mapped on the input dataset still compiles, but the filter silently matches nothing and the preview comes back with a `tag "tagName" not found` warning. Take it from that dataset's `correlationTags`; do NOT invent.
- `or is_null($paramId)` — required. Without it, an unset (empty) parameter blanks the result.

Place the filter line **immediately after the dataset-entry verb(s)** and BEFORE any `align` / `aggregate` / `timechart` — scope before you aggregate.

During the correctness dry-run every declared parameter is bound to an **empty** value, ignoring any `defaultValue`, so `$paramId` references compile without you supplying a value. This means an unguarded `$paramId` still **compiles** — it does not error — but with the parameter empty it matches nothing, so the card returns no data and renders blank. A `defaultValue` does not change this: it only sets the picker's initial selection on the saved dashboard (and the preview image), never the value used at validation, and a reader who clears the picker binds empty regardless. That is why `or is_null($paramId)` is mandatory.

## Text and number parameters

A `text` or `number` parameter holds one value the reader types in. Compare it to a column instead of a tag, with the same empty guard:

```
filter is_null($min_latency) or duration_ms >= $min_latency
filter is_null($user) or $user = "" or user_id = $user
```

- **Number:** a cleared number binds null, so `is_null($paramId)` is the whole guard, exactly as for a tag. Compare it to a numeric column.
- **Text:** a cleared text input may bind the empty string `""` rather than null (a dashboard text input does), so guard both: `is_null($paramId) or $paramId = ""`. With only `is_null`, an empty text input still compiles but matches nothing and blanks the card. For a substring match use `contains(message, $paramId)` in place of `=`.

Place these filter lines where a tag filter line goes: after the dataset-entry verb(s), before aggregation.

## Resource (`resource-input`) parameters

A `resource-input` parameter is a picker over one **instance** of a Resource dataset, such as a specific cluster, pod, or customer. It is not a tag, so it is not referenced with `#` and not tested with `array_contains`. Instead it exposes the selected instance's **primary-key columns** as fields:

```
filter clusterUid = $cluster.uid or is_null($cluster)
```

- `$cluster.uid` — the `uid` primary-key column of the selected resource. The accessor after the dot must be one of the target dataset's primary-key columns; take them from where the parameter is listed.
- `clusterUid` — a column on **this card's own input dataset** that holds the same value (usually the foreign key that links this dataset to that resource). Both sides of the `=` matter: a card whose dataset has no such column can't be scoped by this parameter.
- `or is_null($cluster)` — required, exactly as for tags. An unselected picker binds a null link; without the guard the card renders blank.

When the resource has a **composite primary key**, match every column and `and` them together, keeping one guard around the whole match. A partial match would collapse distinct instances:

```
filter (name = $deployment.name and namespace = $deployment.namespace and clusterUid = $deployment.clusterUid) or is_null($deployment)
```

Place it where a tag filter line goes: before `align` / `aggregate` / `timechart`.

## Select parameters

A `single-select` or `multi-select` parameter is a dropdown over a fixed list or the values of a dataset column. Compare it to the matching column on **this card's own input dataset**:

```
filter is_null($size) or $size = "" or warehouse_size = $size
filter is_null($models) or array_contains($models, model_id)
```

- **Single-select:** it binds one string. A cleared single-select binds null or `""`, depending on how the picker is set up, so guard both, exactly as for text. With only `is_null`, a picker that clears to `""` blanks the card.
- **Multi-select:** it binds an array of strings, so test membership with `array_contains($p, <column>)`, as for a tag. A cleared multi-select binds null, so `is_null($p)` is the whole guard.
- **Multi-select over a non-string column:** the options are always strings, so cast the column: `array_contains($codes, string(status_code))`. Without the cast nothing ever matches, and validation can't catch it (it binds null), so the card saves and goes blank once a reader picks a value.
- **An existing `<column> = $p` on a multi-select:** the dashboard page writes that form for one, and it is correct there. Write `array_contains` for a new line, and leave an existing `= $p` line as it is.
- The values are the option **values**, not the labels the dropdown shows. Compare to the column that holds those values.

Place these filter lines where a tag filter line goes: after the dataset-entry verb(s), before aggregation. A `bool` parameter is not covered here.

## Reusing a parameter across queries

Reference the same `$paramId` in any query's OPAL. Use already-declared parameter ids verbatim, and do NOT redeclare one.

## Multiple axes on one query

Chain one filter line per axis, before aggregation:

```
filter array_contains($cluster, #k8s.cluster.name) or is_null($cluster)
filter array_contains($customer, #customer.id) or is_null($customer)
timechart 1m, qps:rate(count())
```

## Common mistakes

- **Guarding a text parameter with `is_null` alone.** A cleared text input can bind `""`, which `is_null` does not catch; add `or $paramId = ""`.
- **Forgetting `or is_null($paramId)`.** The card still compiles, but an empty parameter matches nothing → the card returns no data and renders blank. A default does NOT rescue it (validation binds empty and ignores defaults; a cleared picker binds empty too).
- **`#paramId` instead of `$paramId`.** Parameters use `$`; tags use `#`.
- **Using `array_contains($p, #tag)` on a `resource-input` parameter, or `$p.column` on a tag one.** A resource picker binds one instance (field access), a tag picker binds an array (membership test). Match the form to the parameter.
- **Writing a new multi-select line with `=`.** Use `array_contains($p, <column>)` (with `string(<column>)` for a non-string column). An `=` line the dashboard page already wrote is fine; leave it.
- **Matching a resource parameter against a column that isn't the corresponding foreign key.** It compiles, but the values never line up, so the card comes back empty for every selection.
- **Referencing a `$paramId` that was never declared.** Dry-run fails with `reference to parameter <id>: no value supplied for it`. This is a _different_ problem from a missing guard: the fix is to declare (or reuse) the parameter — adding `or is_null` does NOT fix it, because it still references an unbound parameter.
- **Scoping by a `#tagName` not mapped on the input dataset.** The query compiles but the filter does nothing; the preview returns a `tag "tagName" not found` warning. Confirm the tag is in the dataset's correlation-tag mappings, or pick a real column on the dataset instead.
- **Putting the filter AFTER `align` / `aggregate` / `timechart`.** Scope belongs before aggregation.
