# Advanced join patterns: leftjoin, fulljoin, join, exists, not_exists, follow, surrounding, union, lookup, semi-join, anti-join, temporal join, cross-dataset correlation

## Workflow

1. Identify the datasets to combine (confirm the dataset IDs and schemas).
2. Determine what you need from the join:
   - Only filtering rows (no extra columns) → use `exists` / `not_exists`
   - Enriching events with Resource/Table metadata → use `lookup`
   - Combining columns from two Event datasets → use `join` / `leftjoin`
   - All rows from both sides → use `fulljoin`
   - Temporal event sequencing → use `follow` / `follow_not`
3. Always pick the cheapest join type that works (see cost hierarchy below).
4. Use `on(local_col = @"input_name".remote_col)` syntax for all join predicates. For multi-key joins, use `and`: `on(col1 = @"input".key1 and col2 = @"input".key2)`. Use `@"input_name".column` for column bindings.
5. **Use input names, not dataset IDs.** `@"input_name"` must match the `inputName` from your GetQueryCard `inputs` parameter — never use the raw numeric dataset ID. For example, if your inputs include `{inputName: "customer", datasetId: "41007202"}`, write `@"customer".field`, not `@"41007202".field`. Always quote the reference: `@"input_name"` — bare `@id` fails when the ID is numeric or contains special characters.
6. **Use exact field names from the joined dataset's schema** — never guess field names on the remote side.

---

## Join cost hierarchy — choose the cheapest that works

| Cheapest → Expensive         | Verb                                 | Use when                                          | Unmatched left rows                                  | PK on target? |
| :--------------------------- | :----------------------------------- | :------------------------------------------------ | :--------------------------------------------------- | :------------ |
| 1. Semi-join filter          | `exists` / `not_exists`              | Only need to filter rows, no extra columns needed | `exists`: dropped; `not_exists`: kept                | No            |
| 1b. Reverse semi-join        | `follow` / `follow_not`              | Need rows from the OTHER dataset, not the default | `follow`: returns right; `follow_not`: returns right | No            |
| 2. Resource/Table enrichment | `lookup`                             | Enrich events with Resource/Table metadata        | Kept (nulls)                                         | **Yes**       |
| 2b. IP geolocation           | `lookup_ip_info`                     | Enrich rows with IP geo data from built-in tables | Kept (nulls)                                         | N/A           |
| 3. Simple relational         | `join` / `leftjoin` (Event↔Event)    | Both sides are Event datasets, need columns       | `join`: dropped; `leftjoin`: kept (nulls)            | No            |
| 4. Point-in-interval         | `join` / `leftjoin` (Event↔Resource) | Enrich events with state valid at event time      | `join`: dropped; `leftjoin`: kept (nulls)            | No            |
| 5. Full outer                | `fulljoin`                           | Need all rows from both sides (Event↔Event only)  | Kept (both sides)                                    | No            |
| 6. Resource enrichment       | `update_resource`                    | Add Event-derived columns to a Resource dataset   | Kept (nulls)                                         | **Yes**       |

**Rule of thumb:** If you only need to filter (not bring columns), use `exists`/`not_exists`. If you need columns from the other side, use `join` or `leftjoin`. When joining a Resource/Table on a column that is NOT its primary key, use `leftjoin` instead of `lookup`.

## lookup — Resource/Table enrichment (simplest)

    lookup on(LOCAL_KEY = @"INPUT_NAME".REMOTE_KEY), NEW_COL:@"INPUT_NAME".REMOTE_COL

The `on(...)` predicate is **required** — `lookup` without `on(...)` is a syntax error. The lookup automatically matches the resource state valid at each event's timestamp.

Full pattern:

    filter service_name = "payments"
    lookup on(client_ip = @"ip_geo".ip), country:@"ip_geo".country, city:@"ip_geo".city
    statsby count:count(), group_by(country)

Rules:

- **`lookup` requires the join key to be a primary or candidate key on the target dataset.** If the column you're joining on is NOT in the target dataset's `primaryKey` array, `lookup` will fail with `"target columns [...] must constitute a candidate/primary key"`. In that case, use `leftjoin` instead — it has no primary key constraint.
- lookup returns null for unmatched rows (left join behavior)
- Multiple lookup columns: `lookup on(src = @"ref".key), col1:@"ref".a, col2:@"ref".b`
- Only use lookup with Resource or Table datasets. For Event↔Event, use `join`/`leftjoin`.

## exists / not_exists — semi-join and anti-join

Filter rows based on whether a match exists in another dataset. No columns are added. **These verbs use bare boolean predicates — do NOT wrap in `on(...)`** (unlike `join`/`leftjoin`/`fulljoin`/`lookup` which require `on(...)`).

    exists service_name = @"error_dataset".service_name
    not_exists service_name = @"error_dataset".service_name

Multi-key predicates use `AND` or pass multiple arguments (implicitly ANDed):

    exists service_name = @"error_dataset".service_name, env = @"error_dataset".env
    exists service_name = @"error_dataset".service_name AND env = @"error_dataset".env

With `frame()` for temporal matching:

    exists frame(back:1h, ahead:1h), host = @"deployments".host AND env = @"deployments".env
    not_exists frame(back:2h, ahead:15m), host = @"deployments".host AND env = @"deployments".env

### Pattern: Find services with no errors

    not_exists service_name = @"error_logs".service
    statsby healthy_count:count(), group_by(service_name)

## join — inner join

Returns only rows that match on both sides. Adds columns from the right side.

    join on(service_name = @"other_dataset".service_name), remote_col:@"other_dataset".value

- For Event↔Event joins: simple equality join
- For Event↔Resource/Interval joins: temporal point-in-interval join

## leftjoin — preserve all left-side rows

Like `join`, but keeps all left rows even without a right match. Unmatched right-side columns are null.

    leftjoin on(local_key = @"resource_dataset".key), enriched_col:@"resource_dataset".value

## fulljoin — full outer join

Keeps all rows from both sides. Use sparingly — produces the most rows.

    fulljoin on(local_key = @"other_dataset".key), other_val:@"other_dataset".value

**CRITICAL constraint:** `fulljoin` is **not allowed** when exactly one side is an Event and the other is not. Event paired with Event works; mixing Event with Resource, Interval, or Table on the opposite side is rejected at compilation. For those cases, use `leftjoin` or `join` instead.

## follow / follow_not — temporal event sequencing

`follow` returns rows from the **joined** dataset that match the default dataset within the time window. `follow_not` returns unmatched right rows. Like `exists`/`not_exists`, these verbs use **bare boolean predicates — do NOT wrap in `on(...)`**.

    follow user_id = @"audit".user_id
    follow frame(back:1h, ahead:1h), host = @"metrics".host AND env = @"metrics".env
    follow_not user_id = @"audit".user_id
    follow_not frame(back:30m, ahead:30m), trace_id = @"errors".trace_id

| Verb         | Returns rows from      | Match required                           |
| :----------- | :--------------------- | :--------------------------------------- |
| `exists`     | Default (left) dataset | Yes — keeps left rows that match         |
| `not_exists` | Default (left) dataset | No — keeps left rows that don't match    |
| `follow`     | Joined (right) dataset | Yes — returns right rows that match      |
| `follow_not` | Joined (right) dataset | No — returns right rows that don't match |

**Syntax distinction:** `exists`/`not_exists`/`follow`/`follow_not` take bare predicates. `join`/`leftjoin`/`fulljoin`/`lookup` require `on(...)` wrapper.

## surrounding — enrich events with nearby events from another dataset

Unions rows from a second **Event** dataset whose timestamps fall within the given `frame()` around at least one row on the left, then aligns schemas like `union`. Both sides must be Event datasets.

**Syntax:** `surrounding frame(...), @dataset_ref, [const_binding]*`

- First argument: `frame(back: ..., ahead: ...)` — the temporal window
- Second argument: bare dataset reference (e.g., `@"measurements"`) — NOT an `on(...)` predicate
- Optional trailing arguments: `name: constant` bindings that add const columns to the **left** input only (right-side rows get `null` for these)

Column names and types are merged across both inputs (like `union`). No column projection from the right side — all columns come through automatically.

    surrounding frame(back: 2m, ahead: 2m), @"measurements", flag: true

This keeps the left-side rows and adds nearby `measurements` events within two minutes, tagging only the left-side rows with `flag: true`.

## lookup_ip_info — IP geolocation enrichment

Enriches rows with geographic data from Observe's built-in IP geolocation tables. Uses a left-outer join so unmatched IPs get `null`.

**Syntax:** `lookup_ip_info ip_expression, column_bindings...`

    lookup_ip_info src_ip, country:@ip_info.geo.country, city:@ip_info.geo.city
    lookup_ip_info ipv4(string(client_addr)), timezone:@ip_info.geo.timezone, geo:@ip_info.geo

Rules:

- First argument must be an `ipv4` expression (use `ipv4(string(col))` if the column is not already ipv4 typed)
- Column bindings must reference `@ip_info` with qualified paths (e.g., `@ip_info.geo.city`) — bare `@ip_info` is not allowed

## update_resource — enrich Resource with Event data

Builds new columns on a Resource dataset by matching each resource row to Event rows using primary-key or candidate-key equality predicates, then projecting values from those events.

**Syntax:** `update_resource [options(expiry: duration)]?, key_predicate+, column_binding+`

- Default input must be a Resource; the additional dataset must be Events
- Key predicates use `eq(local_col, @"events".remote_col)` syntax and must cover a full primary or candidate key
- Column bindings add new columns: `new_col:@"events".expr` — cannot overwrite existing resource columns

Example:

    update_resource station_id = @"telemetry".station_id, rpm:@"telemetry".rpm
    update_resource options(expiry:duration_hr(12)), station_id = @"events".station_id, note:@"events".message

The `expiry` option controls how long event-sourced values remain (default `24h`).

## union — combine datasets vertically

    union @"other_dataset"

Or with subquery labels:

    @errors <- @ { filter status >= 500 }
    @warnings <- @ { filter status >= 400 and status < 500 }
    union @errors, @warnings

Columns that don't exist on one side are null. Use `any_not_null()` after union to collapse sparse columns.

---

## Cross-Dataset Correlation

Correlation tags map the same concept (e.g., "service.name") to the correct OPAL field path in each dataset, even when field names differ. Each dataset's catalog entry shows `CorrelationTags: tag→opalField`.

### Example: spans → logs correlation via service.name

    # Query 1: Find error services in spans (opalField: service_name)
    filter error = true
    statsby error_count:count(), group_by(service_name)

    # Query 2: Get logs for that service (opalField: resource_attributes."service.name")
    filter string(resource_attributes."service.name") = "checkout-service"
    pick_col timestamp, severity_text, body

### How to use correlation tags

1. Find which datasets share a correlation tag (same tag name in both catalog entries).
2. Use each dataset's opalField — do NOT assume field names are the same across datasets.
3. Filter each dataset by the same tag value to correlate results.

---

## Entity Name Resolution via Join

When the user asks for entity names (e.g., "which customers", "give me their names", "who owns this"), do NOT infer names from internal identifiers like hostnames, cluster names, or account IDs. Internal identifiers can mislabel environments (e.g., "prod", "eu-1", "staging") as distinct entities and miss the authoritative mapping.

Instead, join the primary dataset to a reference dataset (Resource or Table kind) that provides the authoritative identifier-to-name mapping.

### Pattern

1. Start from the primary dataset (e.g., error logs, spans).
2. **Join first** — use `lookup` (for Resource/Table on primary key) or `leftjoin` to resolve identifiers to authoritative human-readable names from the reference dataset. Joining must happen BEFORE aggregation (see "Join ordering" above).
3. Filter out unresolved rows (`filter not is_null(resolved_name)`).
4. **Then aggregate** — group by the resolved name, not the internal identifier.

### Example: resolve internal identifiers to customer names

    filter severity_number >= 17
    lookup on(cluster_name = @"customers".cluster_id), customer_name:@"customers".customerName
    filter not is_null(customer_name)
    statsby issue_count:count(), group_by(customer_name)
    sort desc(issue_count)

### Key rules

- **Never guess names from identifiers.** A hostname like `acme-prod-1` does not reliably map to customer "Acme" — the reference dataset is the source of truth.
- Filter out nulls after the join (`filter not is_null(resolved_name)`) if you only want entities that exist in the reference dataset.

---

## Join ordering — joins BEFORE aggregation (CRITICAL)

After `statsby`, temporal columns are consumed and the pipeline result becomes **non-temporal** (a Table dataset). `leftjoin` and `fulljoin` between a non-temporal result and a temporal (Resource/Interval) dataset are unsupported, producing:

    unsupported join type left_outer between a non-temporal and interval dataset

Note: `aggregate` (for metrics) preserves temporal structure, so this restriction does not apply after `aggregate`. It applies after `statsby`, which always produces non-temporal output.

**Always perform joins BEFORE `statsby`** when the joined dataset is temporal (Resource or Interval kind).

### Wrong — join after aggregation (fails)

    statsby error_count:count(), group_by(customerId)
    leftjoin on(customerId = @"customer".customerId), customer_name:@"customer".customerName
    filter not is_null(customer_name)
    sort desc(error_count)

### Correct — join before aggregation

    leftjoin on(customerId = @"customer".customerId), customer_name:@"customer".customerName
    filter not is_null(customer_name)
    statsby error_count:count(), group_by(customer_name)
    sort desc(error_count)

This also applies to `lookup` — perform it before `statsby` so the enriched columns are available during aggregation.

The only exception is joining two non-temporal results (e.g., two subquery outputs that have both been through `statsby`), which uses simple relational joins and does not involve temporal matching.

---

## Common pitfalls

- **Using `on(...)` with `exists`/`not_exists`/`follow`/`follow_not`** — These verbs take bare boolean predicates, NOT `on(...)`. Use `on(...)` only with `join`/`leftjoin`/`fulljoin`/`lookup`.
- **Using `join` when `exists` suffices** — Use `exists`/`not_exists` if you only need to filter.
- **Using `lookup` for Event↔Event** — Use `join`/`leftjoin`; `lookup` is for Resource/Table.
- **Using `lookup` on a non-primary-key column** — `lookup` requires the target join column to be a primary key. Check the target dataset's `primaryKey` array — if the column isn't there, use `leftjoin` instead.
- **Forgetting temporal semantics** — Event↔Resource join matches by event time within resource validity.
- **Wrong direction: exists vs follow** — `exists` keeps left rows; `follow` returns right rows.
- **Joining after `statsby` to a temporal dataset** — Move the join BEFORE `statsby` — after `statsby` the result is non-temporal and cannot `leftjoin`/`fulljoin` with Resource/Interval datasets.
- **Using `fulljoin` with Event↔non-Event** — `fulljoin` is rejected when exactly one side is Event and the other is not. Use `leftjoin` or `join` instead.
- **Missing `frame()` on semi-joins** — `exists`/`not_exists`/`follow`/`follow_not` without `frame()` treat the right side as non-temporal within the query window. Provide `frame()` for temporal band matching.
