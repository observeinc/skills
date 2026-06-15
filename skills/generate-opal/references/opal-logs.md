# Querying log/Event datasets: body filtering, severity levels, structured log parsing, JSON extraction

## Workflow

1. Confirm the dataset kind is Event with log interface (check the dataset schema).
2. Determine the body field type (STRING vs OBJECT) from schema — default to wrapping with `string(body)`.
3. **Check for token indexes** on the body field — see Token Index Awareness below. This determines your operator choice and may influence dataset selection.
4. Choose the filtering approach based on token index availability:
    - **Default (no explicit index info shown)** → use `search(string(body), "text")` — index-aware: uses the token index when one exists and falls back to a normal scan when it doesn't. This is the safe choice when the schema doesn't explicitly mark the field with `[TokenIndex]`.
    - **Body explicitly marked `[TokenIndex]` in the schema** → use `~` operator (fast indexed search). Only use `~` when you can SEE `[TokenIndex]` next to the field in the dataset's `## Fields` section.
    - **Pattern with alternation or flags AND body is `[TokenIndex]`** → use `~ /regex/i` (token-indexed regex literal).
    - **Pattern requires regex semantics that `~` does not support** (e.g. anchors, complex character classes) → use `match_regex(string(body), regex("pattern", "i"))`.
5. If aggregation is needed, use `statsby` with appropriate `group_by`.
6. If the question asks about structured/JSON logs, use `parse_json(string(body))` to extract fields before filtering or aggregating.
7. If the question asks for top-N results, use `topk` after aggregation.

---

## Token Index Awareness (CRITICAL)

The `~` operator on a body field leverages a **token index** for fast searches — but ONLY if the field has a token index configured. Using `~` on a non-indexed field forces a full scan, which is slow and may time out on large datasets.

**Default to `search()` when in doubt.** `search(field, "text")` is index-aware: it uses the token index when one exists and falls back to a scan when it doesn't. Only switch to `~` when you have positive evidence that the field is token-indexed.

### How to check for token indexes

The dataset's `## Fields` section in the knowledge graph context will explicitly mark indexed fields with a tag in square brackets (e.g. `[TokenIndex]`, `[SubstringIndex]`, `[EqualityIndex]`). Treat the absence of an explicit tag as "no index — use `search()`."

    ## Fields
    body (varchar) [TokenIndex]: log message body  ← use ~ for fast search
    timestamp (timestamp): event timestamp           ← no tag, do NOT use ~ on this field

If the schema you see does NOT include any bracketed index tags on string fields, that means the rendering layer did not have index information available for this dataset — assume no token index and use `search()`.

### Operator choice by index availability

| Body field index status                          | Substring search               | Regex/alternation search                                          |
| :----------------------------------------------- | :----------------------------- | :---------------------------------------------------------------- |
| **Default (no `[TokenIndex]` tag visible)**      | `search(string(body), "text")` | `match_regex(string(body), regex("error\|fail\|exception", "i"))` |
| **Explicit `[TokenIndex]` tag visible on field** | `string(body) ~ "text"`        | `string(body) ~ /pattern/i`                                       |

### Dataset selection for text search

When multiple log datasets cover the same data (e.g., "Kubernetes Logs" vs "OpenTelemetry Logs"), **prefer the dataset whose body field is explicitly marked `[TokenIndex]`** for queries involving text search. A token-indexed dataset enables fast `~` searches that avoid full-scan timeouts.

---

## Pattern: Events (Logs) — filter → make_col → statsby

Log datasets (kind=Event, interface=log) store point-in-time log entries.

### Body field handling (CRITICAL)

The body field can be STRING or OBJECT depending on the dataset:

-   If OBJECT type: use string(body) before pattern matching, or access nested fields: string(body.message), string(body.level)
-   If STRING type: filter body ~ /pattern/i works directly
-   ALWAYS use string(body) to be safe — it works for both types

### Filtering logs

**IMPORTANT:** The `~` operator is fast ONLY on token-indexed fields, and is unsafe to use otherwise. When in doubt, default to `search()` (index-aware) for substring matching and `match_regex()` for true regex patterns. Only switch to `~` when the body field is explicitly marked `[TokenIndex]` in the schema. See Token Index Awareness above. For full regex syntax (POSIX ERE rules, `match_regex`, `get_regex`, `replace_regex`, flags), load [opal-regex](opal-regex.md).

Quick reference for log filtering:

```
    // Default substring search — works whether or not body is indexed:
    filter search(string(body), "exception")             # uses token index when present, else scans
    filter match_regex(string(body), regex("error|fail|exception", "i"))   # alternation/regex form

    // Body explicitly marked [TokenIndex] in schema — `~` is fastest:
    filter string(body) ~ "exception"                    # case-insensitive: matches Exception, EXCEPTION, etc.
    filter string(body) ~ /error|fail|exception/i        # case-insensitive regex alternation

    // These work regardless of index:
    filter SERVICE !~ "debug"                            # negated case-insensitive substring
    filter severity_number >= 17                         # OTel severity: ERROR=17-20, FATAL=21-24
```

### Aggregating logs

    filter match_regex(string(body), regex("error", "i"))
    statsby error_count:count(), group_by(some_field)
    sort desc(error_count)
    limit 20

(If body is explicitly marked `[TokenIndex]`, swap the first line for `filter string(body) ~ /error/i` — faster but only safe when the index tag is visible in the schema.)

### Structured log exploration (survey first)

Before filtering on fields, check what exists:
statsby count:count(), group_by(level:string(body.level))
sort desc(count)

### Extracting from unstructured logs

`extract_regex` is a **verb** (pipeline stage), NOT a function. It cannot be used inside `make_col`. Named capture groups become columns directly.

    extract_regex string(body), /status=(?P<status>[0-9]+)/
    make_col status_code:int64(status)

With typed capture groups (avoids separate casting):

    extract_regex string(body), /status=(?P<status_code::int64>[0-9]+)/

### Parsing JSON log bodies

    make_col obj:parse_json(string(body))
    make_col level:string(obj.level), msg:string(obj.message)

### Top-N with topk (after aggregation)

After statsby, use topk for ranked results (cleaner than sort+limit). `topk` requires an aggregate scoring function — never pass a bare column:

    statsby error_count:count(), group_by(service_name)
    topk 10, max(error_count)

### Error detection strategy — prefer structured signals

When filtering for errors or issues in logs, prefer structured signals over substring matching. Broad keyword filters can match non-error contexts (e.g., a field named "error*count" in an INFO log, stack traces logged at DEBUG level, or log messages about error \_recovery*).

**Priority order:**

1. **Severity fields** — `severity_number >= 17` (OTel ERROR and FATAL). Use when the dataset has `severity_number` or `severity_text` in its schema. This is the most reliable signal.
2. **Status codes or error flag columns** — `status_code >= 400`, `error = true`, or similar explicit error indicators. Check the dataset schema for these columns.
3. **Keyword/regex matching on body** — `string(body) ~ /error|exception|fail/i`. Use as a fallback when the dataset lacks severity fields and status columns, or when you need to match specific error messages beyond what severity alone captures.

Combine signals when appropriate — for example, use severity as the primary filter and add keyword matching to catch errors logged at incorrect severity levels. But do not rely on keyword matching alone when structured signals are available.

### Wide-net error filtering

Combine multiple signals to catch errors regardless of log structure. Keyword matching alone misses HTTP error status codes (401, 403, 404, 500, 502, 503, etc.) that commonly appear in access logs without the word "error".

**CRITICAL: Check the dataset schema BEFORE using `severity_number` or `severity_text`.** Many log datasets (especially K8s container logs collected via Fluentd/Fluent Bit) do NOT have these fields. Only include severity conditions if the field exists in the target dataset's schema field list. Using a non-existent field causes a fatal validation error.

**CRITICAL: Default to `match_regex()` for the keyword alternation, NOT `~`.** Only use `~ /.../i` when the body field is explicitly marked `[TokenIndex]` in the schema's `## Fields` section. Without that explicit marker, treat the field as non-indexed.

DEFAULT (no `[TokenIndex]` marker visible) — WITH `severity_number` in schema:

    filter match_regex(string(body), regex("error|exception|fail|fatal|panic|critical", "i")) or severity_number >= 17 or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

DEFAULT (no `[TokenIndex]` marker visible) — WITHOUT `severity_number` in schema:

    filter match_regex(string(body), regex("error|exception|fail|fatal|panic|critical", "i")) or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

OPTIMIZED (body field IS explicitly marked `[TokenIndex]`) — WITH `severity_number` in schema:

    filter string(body) ~ /error|exception|fail|fatal|panic|critical/i or severity_number >= 17 or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

OPTIMIZED (body field IS explicitly marked `[TokenIndex]`) — WITHOUT `severity_number` in schema:

    filter string(body) ~ /error|exception|fail|fatal|panic|critical/i or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

| Signal                                                           | What it catches                                                                        | Requires schema field?       |
| :--------------------------------------------------------------- | :------------------------------------------------------------------------------------- | :--------------------------- |
| Keyword regex (`error\|exception\|fail\|fatal\|panic\|critical`) | Application error messages, stack traces, failure logs                                 | No — works on any body field |
| `severity_number >= 17`                                          | OTel ERROR (17-20) and FATAL (21-24) severity levels                                   | **Yes** — `severity_number`  |
| HTTP status regex (`4[0-9]{2}\|5[0-9]{2}`)                       | 4xx client errors (401, 403, 404) and 5xx server errors (500, 502, 503) in access logs | No — works on any body field |

### Avoid redundant filters

Each filter condition should contribute uniquely to narrowing the result set. Do not combine overlapping conditions that serve the same purpose:

    // WRONG — redundant: keyword filter already narrows to errors, stream="stderr" adds nothing useful
    filter stream = "stderr" or string(body) ~ /error|exception|fail/i
    filter string(body) ~ /error|exception|fail/i    // second filter negates the stderr OR

    // CORRECT — single comprehensive error filter (only if severity_number exists in schema)
    filter match_regex(string(body), regex("error|exception|fail|fatal|panic|critical", "i")) or severity_number >= 17

Common redundancy pitfalls:

-   Filtering `stream = "stderr"` then re-filtering by error keywords (stderr contains non-error output too, and the keyword filter is the real discriminator)
-   Using both `severity_text = "ERROR"` and `severity_number >= 17` (they test the same thing)
-   Combining substring match and regex for the same pattern
-   Using `severity_number` without checking the dataset schema — many K8s log datasets lack this field

### Aggregation with sample messages

Use any() to grab a representative log alongside counts:
statsby error_count:count(), sample_msg:any(string(body)), group_by(service_name)
sort desc(error_count)

### Common field names by dataset type

| Dataset type                  | Body/message field      | Level/severity field                   |
| ----------------------------- | ----------------------- | -------------------------------------- |
| OTel log records              | body                    | severity_text, severity_number         |
| K8s container logs (OTel)     | body                    | severity_text, severity_number         |
| K8s container logs (non-OTel) | body                    | **None** — check schema, do NOT assume |
| CloudWatch logs               | message or body.message | body.level                             |
| Span events                   | event_name + attributes | (use parent span status)               |

**IMPORTANT**: K8s container log datasets vary. OTel-collected logs typically include `severity_number`/`severity_text`, but Fluentd/Fluent Bit-collected logs often have only `body`, `stream`, `cluster`, `namespace`, `container`, `pod`, `node`, and `attributes` — no severity fields. Always check the target dataset's schema field list before referencing severity fields.

### Log severity levels (OpenTelemetry)

TRACE=1-4, DEBUG=5-8, INFO=9-12, WARN=13-16, ERROR=17-20, FATAL=21-24
These fields exist ONLY on OTel log datasets. Always verify `severity_number` is in the schema before using it.
