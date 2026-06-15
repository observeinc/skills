# Advanced transforms: semistructured parsing, pivot/unpivot, window functions, time operations

## Workflow

1. Identify the transform needed:
    - Parsing JSON/text/KV data → see **Semistructured Data Parsing** (use the decision tree below)
    - Base64, URI encoding/decoding, or string manipulation → see **Encoding, Decoding, and String Utilities**
    - Reshaping rows↔columns → see **Pivot / Unpivot**
    - Per-row computation over related rows (rank, lag, moving avg) → see **Window Functions**
    - Duration/timestamp math → load [opal-duration](opal-duration.md)
2. For parsing: determine the data shape (JSON string, nested object, structured text, key=value) and pick the matching tool from the decision tree.
3. For window functions: always specify `group_by()` for partitioning and `order_by()` for row ordering.

---

## Semistructured Data Parsing

### Decision tree: which tool to use

| Data shape                         | Tool                                                      | Output                                |
| :--------------------------------- | :-------------------------------------------------------- | :------------------------------------ |
| JSON string column                 | `parse_json(string(col))`                                 | Navigable object                      |
| Nested object column               | Dot navigation: `col.field.subfield`                      | Scalar (cast needed)                  |
| Object with unknown/dynamic keys   | `flatten_leaves col`                                      | Rows with `_path` and `_value`        |
| Structured text with known pattern | `extract_regex col, /pattern/` (**verb**, not a function) | Columns from named capture groups     |
| Need one regex match as string     | `get_regex(col, /pattern/)`                               | Single matched string                 |
| Need all regex matches             | `get_regex_all(col, /pattern/)`                           | Array of matched strings              |
| `key=value` format                 | `parse_kvs(col)`                                          | Object with key-value pairs           |
| URL string                         | `parse_url(col)`                                          | Object with scheme, host, path, query |
| CSV/delimited string               | `parse_csv(col)`                                          | Array of values                       |

### parse_json

    make_col parsed:parse_json(string(body))
    make_col status:int64(parsed.status), method:string(parsed.request.method)

Input must be a string — use `string(col)` if variant/object. Always cast leaf values.

### Dot navigation (already-object columns)

    make_col region:string(EXTRA.cloud.region)
    filter string(EXTRA.kubernetes.namespace) = "production"

### extract_regex — structured text parsing

`extract_regex` is a **verb** (standalone pipeline stage), NOT a function. It CANNOT be used inside `make_col` or any expression. Named capture groups become columns directly on the dataset.

    WRONG:   make_col parsed:extract_regex(string(body), /.../)   ← "unknown function" error
    CORRECT: extract_regex string(body), /(?P<status>[0-9]+)/      ← verb on its own line
             // creates column "status" from the capture group; use it downstream:
             filter int64(status) >= 500

**Verb form** (the only form — produces columns directly):

    extract_regex string(body), /(?P<method>[A-Z]+) (?P<path>\S+) HTTP\/(?P<version>[0-9.]+)/

**Typed capture groups** (avoid separate casting):

    extract_regex string(body), /took (?P<duration_ms::float64>[0-9.]+)ms, rows=(?P<row_count::int64>[0-9]+)/

All OPAL regex uses **POSIX ERE** (NOT PCRE). For the full POSIX ERE reference, operator/function details, and token-indexed vs non-indexed guidance, load [opal-regex](opal-regex.md).

### flatten_leaves — expand unknown/dynamic object keys

    flatten_leaves EXTRA
    filter _path = "kubernetes.pod_name"
    make_col pod:string(_value)

Output: one row per leaf. `_path` is the dot-separated key path, `_value` is variant (always cast).

| Variant          | Intermediate nodes | Leaf nodes       | Performance        |
| :--------------- | :----------------- | :--------------- | :----------------- |
| `flatten_leaves` | Excluded           | Included         | Best (fewest rows) |
| `flatten_single` | First level only   | First level only | Good               |
| `flatten`        | Included (null)    | Included         | Most rows          |

### parse_kvs — key=value parsing

    make_col kv:parse_kvs(string(body))
    make_col level:string(kv.level), user:string(kv.user)

### Object manipulation

-   `drop_fields(obj, "field1", "field2")` — remove fields
-   `make_fields(obj, "key", value, ...)` — add fields
-   `make_object("key1", val1, "key2", val2)` — create from scratch
-   `merge_objects(obj1, obj2)` — combine (later wins on conflict)
-   `object_keys(obj)` — array of top-level key names
-   `pick_fields(obj, "field1", "field2")` — keep only named fields
-   `get_field(obj, field_name_col)` — dynamic field access

### Array operations

-   `split(str, ",")` — string to array
-   `array_length(arr)` — count elements
-   `array_to_string(arr, ", ")` — join to string
-   `array_contains(arr, "value")` — membership test
-   `array_distinct(arr)` — deduplicate
-   `concat_arrays(a, b)` — merge arrays
-   `make_array(val1, val2, ...)` — construct from values
-   `get_item(arr, 0)` — zero-based index (null if out of bounds)

---

## Encoding, Decoding, and String Utilities

OPAL function names do NOT follow other languages. Use `decode_base64` not `base64_decode`, `encode_base64` not `base64_encode`, `concat_strings` not `concat` or `strcat`.

### Encoding / Decoding

-   `decode_base64(str [, urlSafe [, ignorePadding]])` — Decode base64 string. Set urlSafe=true for URL-safe variant, ignorePadding=true to tolerate missing padding.
-   `encode_base64(str [, urlSafe])` — Encode string to base64. Set urlSafe=true for URL-safe variant.
-   `decode_uri(str)` — Decode percent-encoded URI (preserves #$&+,/:;=?@).
-   `decode_uri_component(str)` — Decode all percent-encoded sequences.
-   `encode_uri(str)` — Percent-encode a URI string.
-   `encode_uri_component(str)` — Percent-encode all special characters.
-   `parse_hex(hexstr)` — Parse hex string to int64.

    make_col decoded:decode_base64(string(encoded_field))
    make_col decoded_url_safe:decode_base64(string(encoded_field), true)
    make_col clean_path:decode_uri_component(string(url_path))

### String utilities

-   `concat_strings(str, ...)` — Concatenate strings (variadic).
-   `strlen(str)` — String length.
-   `upper(str)` / `lower(str)` — Case conversion.
-   `trim(str)` / `ltrim(str)` / `rtrim(str)` — Remove whitespace (leading, trailing, or both).
-   `left(str, n)` / `right(str, n)` — Leftmost/rightmost n characters.
-   `substring(str, start [, length])` — Extract substring (1-based start index).
-   `position(haystack, needle)` — Find first occurrence index (0 if not found).
-   `replace(str, old, new)` — Replace all occurrences of substring.
-   `replace_regex(str, pattern, replacement)` — Replace all regex matches.
-   `split_part(str, delimiter, part)` — Split and return Nth part (1-based).
-   `starts_with(str, prefix)` / `ends_with(str, suffix)` — Prefix/suffix test (returns bool).
-   `hash(str, ...)` — Signed 64-bit hash of one or more values.

    make_col name_lower:lower(string(name))
    make_col first_segment:split_part(string(path), "/", 1)
    make_col display:concat_strings(string(first_name), " ", string(last_name))

---

## Column Operations — rename_col, drop_col

### rename_col — rename without full reprojection

`rename_col` uses `newName:@.oldName` bindings. Supports simultaneous swaps and chained renames in one step.

    rename_col city_name:@.city, elev_m:@.elevation_ft, station_key:@.station_id

Rules:

-   Each source column may be renamed at most once
-   Cannot rename valid-from/valid-to columns — use `set_valid_from`/`set_valid_to` first
-   Cannot overwrite primary-key columns on Resources
-   Overwriting an existing column name removes the old column

### drop_col — remove specific columns

    drop_col debug_info, status_code, internal_id

Rules:

-   Cannot drop valid-from/valid-to columns
-   On Resources, cannot drop primary key columns until they are removed from the key
-   Pipeline must retain at least one column
-   For keeping a subset instead, use `pick_col`

---

## Pivot / Unpivot — Data Reshaping

### pivot — rows to columns

    statsby avg_latency:avg(latency_ms), group_by(service_name)
    pivot service_name, avg_latency

Time-aligned comparison:

    timechart 5m, error_rate:sum(case(status >= 400, 1, true, 0)) / count(), group_by(service_name)
    pivot service_name, error_rate

With explicit group_by:

    statsby total:sum(bytes), group_by(region, method)
    pivot method, total, group_by(region)

Rules: Too many distinct name values → sparse output. Always aggregate before pivot.

### unpivot — columns to rows

**Syntax:** `unpivot [name_column]?, [value_column]?, column_list...`

The optional name/value output column names are **positional string constants that come FIRST**, before the column list. If omitted, defaults are `name` and `value`.

    unpivot temp_c, humidity_pct
    // → output columns: name (holds "temp_c"/"humidity_pct"), value (holds cell values)

    unpivot "metric", "reading", temp_c, humidity_pct
    // → output columns: metric, reading

Cast the value column: `float64(value)` since it holds variant values.

Preserving context:

    pick_col valid_from, host, cpu_usage, mem_usage, disk_usage
    unpivot "metric", "val", cpu_usage, mem_usage, disk_usage
    statsby avg_val:avg(float64(val)), group_by(host, metric)

---

## Window Functions

Window functions compute per-row values based on related rows without collapsing them.

### Syntax

    make_col new_col:window(func(), group_by(partition_col), order_by(sort_col), frame(...))

### Ranking

-   `row_number()` — Sequential index within group.
-   `rank()` — Rank with gaps for ties.
-   `dense_rank()` — Rank without gaps.

Top-N slowest spans per service:

    make_col dur_ms:float64(duration) / 1000000
    make_col rank:window(row_number(), group_by(service_name), order_by(desc(dur_ms)))
    filter rank <= 10

### Navigation (lag/lead)

-   `lag(col, N)` — Value from N rows before.
-   `lead(col, N)` — Value from N rows after.
-   `first(col)` / `last(col)` — First/last value in ordered group.

Delta from previous row:

    make_col dur_ms:float64(duration) / 1000000
    make_col delta:dur_ms - window(lag(dur_ms, 1), group_by(service_name), order_by(start_time))

### Moving average with frame()

    make_col dur_ms:float64(duration) / 1000000
    make_col avg_5m:window(avg(dur_ms), group_by(service_name), order_by(start_time), frame(back:5m))

### Frame functions

-   `frame(back:duration)` — Sliding window looking back.
-   `frame_exact(back:duration)` — Exact sliding window (more expensive).
-   `frame_preceding()` — All rows from start to current (cumulative).
-   `frame_following()` — Current through end of window.

### Rolling windows on datasets

For **span/log datasets**, `timechart` accepts `frame()` as a direct positional argument for sliding window aggregation — no `window()` function needed. See "Rolling window" in [opal-aggregation](opal-aggregation.md):

    timechart 1h, frame(back:7d), error_count:sum(is_error), group_by(service_name)

For **metric datasets**, first use `align` + `aggregate` from [opal-metrics](opal-metrics.md) to produce a time-series, then apply `window(frame())` on the aggregated result. See "Rolling window on metrics" in [opal-metrics](opal-metrics.md).

### Limitations

-   **No row-based frames.** OPAL only supports time-based frames.
-   **No true cumulative sums.** Use `frame_preceding()` for approximate running sums.
-   Any aggregate can be a window function: `avg`, `sum`, `count`, `min`, `max`, `stddev`, `median`, `percentile`.

---

## Time and Duration Operations

For duration constructors, literals, extractors, age/staleness calculations, and timestamp formatting/parsing, load [opal-duration](opal-duration.md).
