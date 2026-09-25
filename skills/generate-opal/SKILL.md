---
name: generate-opal
description: >-
    Contains dataset kind selection, column selection rules, core syntax, and the skill index. NEVER
    generate OPAL, and NEVER invoke an opal-* reference, without first loading
    this skill.
user-invocable: false
---

**MANDATORY:** Run `observe skill view generate-opal --content 2>/dev/null` before proceeding to load the latest version of this skill. When the skill points you to a supporting file, load it with `observe skill view generate-opal --path <relative-path> 2>/dev/null`. Ignore errors and fall back to the file contents.

# Core OPAL: pipeline generation, dataset kinds, column selection, routing

This skill provides core OPAL guidance for generating valid Observe OPAL pipelines. It does NOT contain domain-specific patterns for logs, spans, metrics, joins, durations, or other specialized areas — those live in `opal-*` references that MUST be loaded before writing any OPAL.

## Workflow

1. Determine the dataset kind and interface for each data source (see Dataset Kinds below).
2. **MANDATORY — STOP and load references NOW.** Consult the Skill Index below. You MUST read EVERY `opal-*` reference file that matches the dataset kind or query intent BEFORE writing any OPAL. Read files from `/skills/generate-opal/references/` (e.g., `cat /skills/generate-opal/references/opal-logs.md`). If multiple references match, read ALL of them. Do NOT proceed to step 3 until all matching references are loaded. Skipping this step leads to incorrect queries — references contain critical dataset-specific patterns, multi-dataset workflows, and syntax that this core skill does not cover.
3. Write the OPAL pipeline using the syntax rules in this skill combined with the loaded domain skills.

## References

**MANDATORY: You MUST load references before writing OPAL.** Check this routing table and read every reference file that applies (e.g., `cat /skills/generate-opal/references/opal-logs.md`). If multiple references are listed, read ALL of them — do not skip any. Do NOT write any OPAL pipeline until all matching references are loaded.

OPAL join/lookup/set-operation syntax is non-standard — do NOT rely on SQL knowledge. If the answer requires data from more than one dataset, you MUST read [opal-join-patterns](references/opal-join-patterns.md) before writing any OPAL.
If the query builds objects or arrays (`make_object`, `make_array`, `array_agg` of objects, flatten), you MUST read [opal-transforms](references/opal-transforms.md) before writing any OPAL.

- [opal-logs](references/opal-logs.md) — Event/log. Log body filtering, severity levels, structured log parsing, JSON extraction.
- [opal-metrics](references/opal-metrics.md) — Event/metric. align, aggregate, error rates, throughput, RED metrics, tdigest, histogram, counters, gauges, Prometheus, fill patterns, rolling windows.
- [opal-spans](references/opal-spans.md) — Interval/otel_span. Latency percentiles, per-span error classification, tracing workflows, dependency tracking.
- [opal-join-patterns](references/opal-join-patterns.md) — Combining datasets via leftjoin, fulljoin, exists, not_exists, follow, surrounding, union, lookup, lookup_ip_info, update_resource, semi/anti-joins, temporal joins.
- [opal-transforms](references/opal-transforms.md) — Nested JSON, flatten_leaves, extract_regex, pivot/unpivot, rename_col, drop_col, window functions, `make_object` / `make_array` / `array_agg` of objects. Load whenever the query constructs objects or arrays.
- [opal-aggregation](references/opal-aggregation.md) — Choosing statsby vs timestats vs timechart vs aggregate, histogram, make_session, entity targeting, entity-vs-event counts, rolling windows, cardinality.
- [opal-resource-datasets](references/opal-resource-datasets.md) — Resource-kind datasets (pods, nodes, deployments). Duration calculation, state filtering, complementary condition datasets.
- [opal-duration](references/opal-duration.md) — Duration thresholds, staleness/age calculations, elapsed time, timestamp formatting/parsing.
- [opal-regex](references/opal-regex.md) — Regex & pattern matching: POSIX ERE rules, match_regex, extract_regex, get_regex, replace_regex, token-indexed vs non-indexed operators.
- [opal-visualization](references/opal-visualization.md) — User asks for a chart, graph, plot, or trend. Maps chart types to required OPAL output shape.
- [opal-parameters](references/opal-parameters.md) — Writing OPAL that references a caller-controlled parameter (`$paramId`): the `filter array_contains($paramId, #tag) or is_null($paramId)` line for `tag` / `correlation-tag` parameters, its placement, and reuse across queries. Load whenever a query filters by a value the caller controls (e.g. a dashboard picker, a worksheet input, a monitor variable) rather than a literal baked into the pipeline.

When a query touches multiple domains, load ALL applicable references from the index. The ONLY case where no reference is needed is a trivial single-dataset query using only `filter` + `pick_col` with no aggregation, no duration logic, no joins, and no domain-specific patterns. When in doubt, load the reference — it is always safer to load an extra reference than to miss one.

---

## Dataset Kinds — How to Choose

| Kind     | Interface | What it stores                                      | Aggregation verb                                                               |
| :------- | :-------- | :-------------------------------------------------- | :----------------------------------------------------------------------------- |
| Event    | log       | Point-in-time events, often logs                    | statsby, timestats                                                             |
| Event    | metric    | Pre-aggregated measurements                         | align + aggregate                                                              |
| Interval | otel_span | Spans with start/end time + duration                | statsby, timestats                                                             |
| Resource | —         | Mutable state tracked over time (pods, deployments) | Usually joined, not queried alone. **`limit` is invalid** — use `topk` instead |
| Table    | —         | Static reference/lookup data                        | Usually joined, not queried alone. `limit` is supported.                       |

---

## Column Selection — Keep Output Compact

Results are consumed by an LLM with limited context. Always use pick_col to return only the columns needed.

| User asks…                     | Output shape          | Example                                               |
| :----------------------------- | :-------------------- | :---------------------------------------------------- |
| "Give me their names"          | Distinct name list    | `statsby group_by(customer_name)`                     |
| "Which services had errors?"   | Distinct service list | `statsby group_by(service_name)`                      |
| "How many errors per service?" | Name + count          | `statsby error_count:count(), group_by(service_name)` |

- **Aggregation queries**: Output columns are already minimal — no pick_col needed.
- **Row-level queries**: ALWAYS use pick_col before `sort`. Include `limit 100` for non-aggregation queries.
- **Prefer aggregation over raw rows** when the question can be answered with counts/averages/percentiles.
- **`pick_col` is destructive** — only listed columns survive. Downstream verbs can ONLY reference columns in `pick_col`.
- **pick_col only accepts top-level column references.** Nested fields cause errors — extract with `make_col` first.
- **Column names with spaces** need `@."Column Name"` quoting (NOT `"Column Name"` which is a string literal).
- **Primary key columns** from the schema's `primaryKey` array are MANDATORY in `pick_col`.
- **Temporal columns** are MANDATORY in `pick_col` whenever `pick_col` is used.

    - **Valid-from**: use the `row_start_time()` function with a required output alias in `pick_col` (e.g. `pick_col valid_from:row_start_time(), ...`). A bare `pick_col row_start_time()` is invalid. The function always resolves to the dataset's valid-from column without you needing to know its name, and satisfies the requirement on any temporal dataset (Event, Interval, Resource; returns null on Tables). Prefer this over hardcoding a guessed name like `timestamp`, `BUNDLE_TIMESTAMP`, or `@."Valid From"`.
    - **Valid-to**: if the dataset has a valid-to column (`validToField` is set — Resource and Interval kinds), you MUST also include it. Use the `row_end_time()` function with a required output alias in `pick_col` (e.g. `pick_col valid_to:row_end_time(), ...`), symmetric with `row_start_time()` for valid-from. A bare `pick_col row_end_time()` is invalid. Like `row_start_time()`, it resolves to the dataset's valid-to column without you needing to know its name. Prefer this over hardcoding a guessed name like `end_time` or `@."Valid To"`.
    - **`statsby` is non-temporal** — it consumes the input's `valid_from`/`valid_to` and produces a Table. Do NOT add `valid_from:row_start_time()` or `valid_to:row_end_time()` to a `pick_col` that follows `statsby`; those columns no longer exist and the query will fail with `the field "<name>" does not exist among fields [...]`. After `statsby`, `pick_col` may include only the group-by and aggregate output columns.
    - **After `align`/`aggregate`** (the metric verb), temporal columns DO persist — if you add `pick_col` after aggregation you MUST include them with aliases (e.g. `valid_from:row_start_time()`, `valid_to:row_end_time()`). If you omit `pick_col` entirely, temporal columns are retained automatically.

        WRONG: pick_col row_start_time(), span_name, dur_ms, status_message, trace_id
        WRONG: pick_col span_name, dur_ms, status_message, trace_id
        CORRECT: pick_col valid_from:row_start_time(), valid_to:row_end_time(), span_name, dur_ms, status_message, trace_id

        Post-`statsby` (non-temporal) — temporal columns are GONE; do not add them:

          WRONG:   ... | statsby ct:count(), group_by(svc) | pick_col valid_from:row_start_time(), valid_to:row_end_time(), svc, ct
                  ↑ FAILS with: the field "valid_from" does not exist among fields [svc, ct]
          CORRECT: ... | statsby ct:count(), group_by(svc) | pick_col svc, ct

- **Resource kind datasets** do NOT support `limit` — use `topk` instead. Table, Event, and Interval datasets DO support `limit`.
- **`topk`/`bottomk`** require aggregate scoring: `topk 20, max(col)`. Never pass a bare column.
- **ALWAYS use field names from the dataset schema — NEVER assume field names.**

---

## Essential Syntax Rules

### Comments

OPAL supports `//` single-line and `/* ... */` multi-line comments.

### Time filtering — NEVER filter on temporal columns in OPAL

Time filtering is handled by the query's time range, not by OPAL. Never filter on `validFromField`/`validToField` columns — set the query's time range instead. In particular, NEVER write `filter is_null(row_end_time())` or `filter is_null(@."Valid To")` to get "current" resource state — it almost always returns zero rows. See [opal-resource-datasets](references/opal-resource-datasets.md).

### Referring to temporal columns by function — `row_start_time()` / `row_end_time()`

**ALWAYS use these functions whenever you reference a row's valid-from or valid-to value** — i.e. the dataset's `validFromField` / `validToField` (or any equivalent temporal value). NEVER hardcode or guess the per-dataset column name (`timestamp`, `BUNDLE_TIMESTAMP`, `start_time`, `end_time`, `@."Valid From"`, `@."Valid To"`, etc.). The functions resolve to the correct column on any dataset kind, so you never need to know its schema name.

- `row_start_time()` → the dataset's **valid-from** value (null on Table kind). In `pick_col`, it MUST have an output alias such as `valid_from:row_start_time()`; the alias binds the output's valid-from column and satisfies the mandatory valid-from requirement.
- `row_end_time()` → the dataset's **valid-to** value (null on Event/Table kinds). In `pick_col`, it MUST have an output alias such as `valid_to:row_end_time()`; the alias binds the output's valid-to column and satisfies the mandatory valid-to requirement.

Both functions require `alias:function()` form in `pick_col`; bare forms such as `pick_col row_start_time()` and `pick_col row_end_time()` are invalid. You never need to reference a temporal column by its schema name, even in `pick_col`. The functions work directly in value expressions and sort direction functions when the dataset has the corresponding temporal column: `sort desc(row_start_time())` is valid on temporal datasets, and `sort asc(row_end_time())` is valid on Resource and Interval datasets.

Examples: `make_col age:now() - row_start_time()`, `pick_col valid_from:row_start_time(), valid_to:row_end_time(), ...`, `sort desc(row_start_time())`.

### Cast before operating

Fields like `body` or `attributes` may be VARIANT/OBJECT types. Wrap with `string()`, `int64()`, etc. before comparisons.

### Pipeline formatting — one verb per line

    make_col dur_ms:float64(duration)/1000000
    make_col is_error:if(error = true, 1, 0)

Do NOT split function arguments across lines.

### Named subqueries — keep the `@` sigil

Named subquery declarations and references both require the `@` sigil. A declaration such as `messages <- @ { ... }` is invalid; write `@messages <- @ { ... }`.

When combining local subqueries, define each branch before referencing it. An explicit final result block makes the data flow clearest:

    @messages <- @ {
      statsby messages:array_agg(message), group_by(conversation_id)
    }
    @conversations <- @ {
      statsby span_count:count(), group_by(conversation_id)
    }
    <- @conversations {
      leftjoin on(conversation_id = @messages.conversation_id), messages:@messages.messages
    }

Keep each verb on its own line inside subquery blocks.

### make_col forward-reference

Bindings in `make_col` are processed left to right — later bindings MAY reference columns introduced earlier in the same `make_col`. However, a binding CANNOT reference a column defined to its **right** (later) in the same verb.

    make_col step:month_number, doubled:int64(step * 2), triple:int64(doubled + step)  ← CORRECT
    make_col doubled:int64(step * 2), step:month_number                                ← WRONG (step not yet defined)

### Name columns explicitly in group_by

Every expression in `group_by()` MUST use `name:expression` form. Bare column references are fine as-is.

    WRONG:   statsby count:count(), group_by(string(resource_attributes."service.name"))
    CORRECT: statsby count:count(), group_by(svc:string(resource_attributes."service.name"))

### Column naming — avoid collisions with existing columns

Several verbs create new columns and will error if the chosen name collides with an existing column in the dataset:

- **`align`** output aliases — error: `"align" cannot create column "X" more than once`
- **`group_by`** aliases in `aggregate`/`statsby`/`timechart` — error: `"attempting to overwrite existing column"`
- **Aggregate alias = group_by column** in `statsby`/`aggregate` — error: `"statsby" cannot create column "X" more than once`
- **`extract_regex`** named capture groups — error if the existing column is a non-string type; silently overwrites if string

To avoid collisions: check the dataset schema's field list and use a distinct alias (e.g., `cpu_avg` instead of `cpu` when `cpu` already exists). For `group_by`, use the bare column name when no casting is needed, or pick a new alias — `group_by(host, datacenter)` is fine, but `group_by(host:string(host))` is rejected because `host` already exists. `make_col` is the exception — it intentionally allows overwriting existing columns.

    WRONG:   group_by(functionName:string(functionName), region:string(region), accountId:string(accountId))
              ↑ FAILS with: attempting to overwrite existing column "functionName"
                (one error per such alias — all three would be flagged)
    CORRECT: group_by(functionName, region, accountId)                ← when the columns are already strings
    CORRECT: group_by(fn:string(functionName), region, accountId)     ← when a cast is genuinely needed, rename

**Aggregate aliases must NOT duplicate group_by column names.** Every name in the output must be unique across both aggregate expressions and group_by columns:

    WRONG:   statsby name:any_not_null(name), group_by(name)       ← "name" appears twice
    CORRECT: statsby latest_name:any_not_null(name), group_by(name) ← distinct alias for the aggregate
    CORRECT: statsby ct:count(), group_by(name)                     ← no collision

### After aggregation, only output columns exist

After `statsby`/`aggregate`, only group-by columns and aggregate results remain. Reference output column names, NOT original field paths.

### sort syntax — desc()/asc() functions, NOT SQL keywords

OPAL uses function-call syntax for sort direction — NOT SQL-style trailing keywords.

    WRONG:   sort TIMESTAMP desc
    WRONG:   sort error_count DESC
    CORRECT: sort desc(TIMESTAMP)
    CORRECT: sort desc(error_count)

### timechart — interval is POSITIONAL, not named

    timechart 5m, total:count(), group_by(svc)       ← CORRECT
    timechart interval:5m, total:count()              ← WRONG

### frame() is NEVER inside options() — it's a separate argument

`options()` only accepts `bins`, `min_bin`, `empty_bins`. `frame()` is a separate positional argument to `timechart` or `align`:

    timechart 1h, frame(back:7d), total:count(), group_by(svc)           ← CORRECT
    align 1m, frame(back:10m), avg_mem:avg(m("memory_used"))             ← CORRECT
    timechart 1h, options(frame: 7d), total:count(), group_by(svc)       ← WRONG
    align options(frame: 7d), val:sum(m("x"))                            ← WRONG

### frame() is used in many verb contexts

Beyond `timechart`/`align`, `frame()` is also accepted by: `ever`/`always`/`never`, `exists`/`not_exists`/`follow`/`follow_not`, `fill`, `set_timestamp`/`set_valid_from`/`set_valid_to`, and `window()`. See the respective references for syntax details.

### rename_col — rename columns without full reprojection

    rename_col new_name:@.old_name, city:@.city_name

Renames columns while keeping the full row shape. Supports simultaneous swaps and chained renames. Use when you need to rename a few columns without listing all columns like `pick_col`.

### drop_col — remove specific columns

    drop_col debug_info, status_code

Removes named columns. Lighter than `pick_col` when you only need to remove a few columns. Cannot drop valid-from/valid-to columns or primary key columns on Resources.

### Dataset kind conversion verbs

| Verb            | Converts from             | Converts to | Key behavior                                                          |
| :-------------- | :------------------------ | :---------- | :-------------------------------------------------------------------- |
| `make_event`    | Resource, Interval, Table | Event       | Resource → expands history into point-shaped update rows              |
| `make_interval` | Event, Table, Resource    | Interval    | Event → pass a `valid_to` column; Table → pass both `valid_from`/`to` |
| `make_resource` | Event, Interval           | Resource    | Packs stream into mutable state tracked by primary key                |
| `make_table`    | Any temporal              | Table       | Strips temporal semantics; no arguments                               |

For detailed syntax and examples, load [opal-resource-datasets](references/opal-resource-datasets.md).

### aggregate group_by() for scalar results

`aggregate` without `group_by()` produces a time series. For a single scalar row: `aggregate total:sum(x), group_by()`

### statsby: single value needs an EMPTY group_by()

`statsby` WITHOUT a `group_by` clause it defaults to grouping by the input's primary key → one row PER ENTITY, not a single value. For a singleStat / single scalar, pass an explicit EMPTY `group_by()` to collapse the whole input to one row:

    statsby total:count()                    ← one row PER ENTITY (default primary-key grouping), NOT a single value
    statsby total:count(), group_by()        ← single scalar row (singleStat)
    statsby total:count(), group_by(svc)     ← one row per service

### NEVER include temporal columns in group_by()

Temporal columns (`validFromField`/`validToField`) must NEVER appear in any `group_by()`.

### No count_if() — use conditional sum

    statsby errors:sum(if(status >= 500, 1, 0)), group_by(svc)

### No SQL CASE/WHEN — use case() or if()

`case(cond1, val1, cond2, val2, true, default)` or `if(condition, then, else)`.

`case()` takes strictly **paired** arguments: `(condition, result, condition, result, ...)`. For a default/fallback, add `true, fallback_value` as the final pair — a bare trailing value is invalid.

    WRONG:   case(x = 1, "one", x = 2, "two", "other")
    CORRECT: case(x = 1, "one", x = 2, "two", true, "other")

### dedup / distinct — collapse duplicate rows

`dedup` (alias `distinct`) removes duplicate rows. With no arguments, any two rows matching in every column are merged. With explicit columns, rows grouped by those columns are deduped (on Event/Interval inputs, `valid_from`/`valid_to` are included automatically).

    dedup                          ← remove exact duplicate rows
    dedup service_name, status     ← one row per unique (service_name, status) combination
    distinct service_name          ← alias for dedup

On Resource inputs, only argumentless `dedup` is allowed.

### No `in` operator — use chained `or`

    filter x = "a" or x = "b" or x = "c"

### Filter null semantics — `=` and `!=` drop NULL rows

Comparisons against NULL return NULL, which `filter` treats as false. Both `filter c = "x"` and `filter c != "x"` exclude rows where `c` is null.

To **keep** null rows with `!=`, OR in an explicit null check (this is the form the UI template emits):

    filter is_null(c) or c != "x"        ← keeps rows where c is null

To **exclude** null rows explicitly, add `not is_null(c)` — useful after `leftjoin` or `aggregate` where the column may be sparse:

    filter not is_null(customer_name) and customer_name != "internal"

Negated function predicates (`not contains`, `not match_regex`, `not starts_with`) also drop NULLs. Do not invent an `is_null` OR for those unless the user explicitly asked to keep missing values.

### String matching — function syntax, no infix operators

- **Substring match** — `contains(col, "text")`
- **Glob search** — `col ~ "pattern*"`
- **Regex match** — `match_regex(col, regex("pattern"))`
- **Multi-term search** — `search(col, "term")`

All OPAL regex is **POSIX ERE plus the Perl backslash shorthands** (NOT full PCRE). `\d`, `\w`, `\s`, `\b` and non-greedy quantifiers (`*?`, `+?`) all work; POSIX classes like `[[:space:]]` work too. What does not exist: lookahead `(?=`, lookbehind `(?<=`, backreferences `\1`. Also NEVER use inline flag groups `(?i)`/`(?m)`/`(?s)` or non-capturing groups `(?:...)` — RE2 accepts them so they compile, but Snowflake's `REGEXP_LIKE`/`RLIKE` rejects them at execution (`no argument for repetition operator: ?`), which crashes `match_regex`/`~` on substring-indexed columns; use a slash-suffix/flag argument for case-insensitivity and a plain group `(...)` instead of `(?:...)`. Never use `[[:<:]]`/`[[:>:]]` for word boundaries — that is MySQL syntax and Snowflake rejects it; use `\b`. For full regex reference, load [opal-regex](references/opal-regex.md).

---

## General Rules

- **Filter before aggregating.** Always apply filters before `statsby`, `timechart`, or `aggregate`.
- **Only use fields from the dataset schema.** Never guess field names.
- **Prefer one query card.** Use `union` for multi-category questions (load [opal-join-patterns](references/opal-join-patterns.md)).
- **Prefer one subquery within that card.** Multi-subquery syntax is only needed for joins, unions, and exists.
- **Metrics use `align` and metric functions.** Exception: datasets with `OBSERVATION_KIND` or `FIELDS` — use `timechart`/`statsby`.
- **Dataset field naming varies.** OTel uses `attributes."..."`, Prometheus uses `labels."..."`, AWS uses `FIELDS."..."`. Always check the schema.
- **Reference datasets by input name.** In join verbs, use `@"inputName"` matching `inputs` — never raw dataset IDs. `@"…"` is never a timestamp literal — use `parse_isotime(...)`.
- **Only use documented functions.** OPAL function names may differ from other languages (e.g., `decode_base64` not `base64_decode`, `concat_strings` not `concat`). If unsure whether a function exists, don't guess — use documented alternatives or describe the transform needed.

---

## Using Dataset Context

1. **Correlation tags** — key-value pairs for filtering and grouping. The `related` field on each tag result lists connected metrics and datasets. Use `correlationTagMappings` to verify that the selected input dataset maps the tag, then prefer native `#tag` syntax (e.g., `filter #k8s.cluster.name = "prod"` or `group_by(#"service.name")`). Native tag references resolve and coalesce every mapped backing path, so use them only when that coalesced concept matches the user's requested dimension. If multiple mappings represent distinct semantic roles such as caller versus downstream service, select the role-specific physical mapped path instead. Do not reconstruct a single-role mapped tag with `make_col`. Quote tag names with `#"..."` when needed. A direct tag grouping produces a normalized output column (`#k8s.cluster.name` becomes `k8s_cluster_name`) for downstream verbs. Use a physical path when the dimension is not exposed as a correlation tag or when the query needs one specific mapping.
2. **Dataset schemas** — use `## Fields` / `columnStats` to pick datasets, identify columns, and verify field names. Dimension field names vary by dataset (`tags`, `resource_attributes`, `labels`, `FIELDS`) — always check the schema.
3. **Metrics** — use the `tags` field in `heuristics` to discover available filtering/grouping dimensions. Confirm the metric supports the dimensions the query needs before writing the pipeline.

## ⚠ Frequent Errors — Check These First

These are the most common OPAL generation mistakes. Verify NONE of them apply before submitting.

1. **`pick_col` missing or failing to alias temporal columns.** Every `pick_col` MUST retain the dataset's temporal columns — UNLESS the immediately-preceding verb is `statsby` (which is non-temporal and drops them). Use the required aliased forms `valid_from:row_start_time()` for valid-from and `valid_to:row_end_time()` for valid-to (Resource/Interval). Bare `row_start_time()` or `row_end_time()` entries are invalid. When in doubt, omit `pick_col` entirely (all columns are kept).
2. **`pick_col` with temporal columns after `statsby`.** `statsby` consumes temporal columns and outputs a Table. The post-`statsby` schema has only group-by and aggregate output columns — referencing `valid_from`/`valid_to` here fails with `the field "<name>" does not exist among fields [...]`.
3. **SQL-style sort direction.** `sort col DESC` is invalid — use `sort desc(col)`.
4. **Filtering on valid-to to get current state.** NEVER write `filter is_null(row_end_time())` or `filter is_null(@."Valid To")` — it almost always returns zero rows on Resource datasets. For "currently in state X", use `filter_last <state predicate>` (last-value semantics), not plain `filter`, and compute durations with `coalesce(row_end_time(), now())`. See [opal-resource-datasets](references/opal-resource-datasets.md).
5. **`visualizationTemplate.lineChart.x` mismatch.** This field references the OUTPUT column name in the schema, NOT the OPAL function. `timechart` produces `_c_valid_from`; `align` (with or without `aggregate`) produces `valid_from`. Mixing them up fails with `references column 'X' which does not exist in the schema. Available fields: ...`. NEVER use `row_start_time()` here.
6. **`group_by(name:string(name))` overwrite.** Casting a bare column to itself collides with the existing column. FAILS with `attempting to overwrite existing column "<name>"`. Either rename (`fn:string(functionName)`) or drop the cast when the column is already a string (`group_by(functionName, region)`).
7. **Blindly `make_event`-ing a Resource dataset.** Do NOT add `make_event` (or any `make_*` conversion) as a routine or first step on a Resource. Query the Resource directly — `filter`, `filter_last`/`ever`/`always`/`never`, `make_col`, `statsby`, `timechart`, and `topk` all operate on Resources natively (none of the worked examples convert first). A blind `make_event` explodes each entity's state history into per-update rows, which double-counts entities and breaks current-state and duration logic. Convert only when the question is specifically about state _transitions_ (one row per change), or a downstream verb strictly requires a different kind. See [opal-resource-datasets](references/opal-resource-datasets.md).

---

## Validation Checklist

Before returning the pipeline, verify:

- **Per-dataset field verification**: For EACH expression, confirm every referenced field exists in THAT dataset's schema. Do not reference fields from other datasets — field names vary (`tags` vs `resource_attributes` vs `labels` vs `FIELDS`).
- All column names after aggregation (`statsby`/`aggregate`) reference **output columns**, not original field paths
- All column names after `pick_col` appear in the `pick_col` list
- **CRITICAL**: `pick_col` retains temporal columns whenever `pick_col` is used — including after `align`/`aggregate` — plus ALL `primaryKey` columns, EXCEPT after `statsby`, which is non-temporal and drops them. Use aliased expressions such as `valid_from:row_start_time()` and `valid_to:row_end_time()`; bare function calls in `pick_col` are invalid. If unsure which columns are temporal, omit `pick_col` entirely.
- `float64()` wraps division operands to prevent integer truncation
- Output alias names (from `align`, `group_by`, `extract_regex`, `statsby`) do not collide with existing schema column names or other output names in the same verb
- `case()` uses strictly paired `(condition, result)` arguments — default uses `true, value`, never a bare trailing value
- `frame()` is a separate positional argument to `align`/`timechart`, NEVER inside `options()`
- Every function used exists in OPAL — use only documented functions; for duration functions load [opal-duration](references/opal-duration.md)
- Resource datasets use `topk` (never `limit`) and retain BOTH temporal columns in `pick_col` with required aliases — e.g. `valid_from:row_start_time()` and `valid_to:row_end_time()` — and NEVER filter on valid-to (`filter is_null(row_end_time())` / `filter is_null(@."Valid To")`)
- When filtering on ambiguous values (e.g., "us west", "prod"), never guess exact values — use partial-match functions like `starts_with()`, `contains()`, or `match_regex()`
