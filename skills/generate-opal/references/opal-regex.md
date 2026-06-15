# OPAL Regex & Pattern Matching — Complete Reference

## POSIX ERE — NOT PCRE (CRITICAL)

**All OPAL regex uses POSIX ERE.** This applies everywhere: `~ /pattern/`, `match_regex()`, `replace_regex()`, `get_regex()`, `extract_regex`, and `regex()`. PCRE shorthands do not exist — use character classes instead:

| PCRE (WRONG) | POSIX ERE (CORRECT)   |
| :----------- | :-------------------- |
| `\d`         | `[0-9]`               |
| `\w`         | `[a-zA-Z0-9_]`        |
| `\s`         | `[[:space:]]`         |
| `\b`         | `[[:<:]]` / `[[:>:]]` |

Non-greedy quantifiers (`*?`, `+?`) are NOT supported.

Regex flags: `(?i)` for case-insensitive, `(?m)` for multiline.

---

## Operator & Function Reference

### String matching overview

-   **Substring match** — `contains(col, "text")`
-   **Glob search** — `col ~ "pattern*"` or `col ~ <pattern*>`
-   **Regex match** — `match_regex(col, regex("pattern"))`
-   **Multi-term search** — `search(col, "term")`

### match_regex — boolean regex test

Use `match_regex()` with the `regex()` constructor for reusable patterns and flags:

    filter match_regex(string(body), regex("error.*timeout", "i"))   # case-insensitive
    filter match_regex(string(body), regex("status=[45][0-9]{2}"))   # status 4xx/5xx

The `"i"` flag makes the match case-insensitive. Without it, matching is case-sensitive.

### extract_regex — structured text parsing (VERB, not a function)

`extract_regex` is a **verb** (standalone pipeline stage), NOT a function. It CANNOT be used inside `make_col` or any expression. Named capture groups become columns directly on the dataset.

    WRONG:   make_col parsed:extract_regex(string(body), /.../)   <- "unknown function" error
    CORRECT: extract_regex string(body), /(?P<status>[0-9]+)/
             // creates column "status" from the capture group

Multi-field extraction:

    extract_regex string(body), /(?P<method>[A-Z]+) (?P<path>[^ ]+) HTTP\/(?P<version>[0-9.]+)/

Typed capture groups (avoid separate casting):

    extract_regex string(body), /took (?P<duration_ms::float64>[0-9.]+)ms, rows=(?P<row_count::int64>[0-9]+)/

NEVER name a capture group after a temporal column (check `validFromField`/`validToField` in the schema) — use alternative names like `log_ts`, `event_time`.

### get_regex / get_regex_all — extract matched strings

-   `get_regex(col, /pattern/)` — single matched string
-   `get_regex_all(col, /pat/)` — array of all matched strings

### replace_regex — regex substitution

    make_col cleaned:replace_regex(string(body), /[0-9]{4}-[0-9]{4}/, "XXXX-XXXX")

Replaces all matches of the pattern with the replacement string.

---

## Token-Indexed vs Non-Indexed Fields

The `~` operator on a body field leverages a **token index** for fast searches — but ONLY if the field has a token index configured. Using `~` on a non-indexed field forces a full scan, which is slow and may time out.

**Default to `search()` for substring search and `match_regex()` for regex.** Both work whether or not the field is indexed (`search()` uses the token index when present). Only switch to `~` when you have positive evidence the field is token-indexed.

### How to check for token indexes

The dataset's `## Fields` section in the knowledge graph context will explicitly mark indexed fields with a tag in square brackets (e.g. `[TokenIndex]`, `[SubstringIndex]`, `[EqualityIndex]`). Treat the absence of an explicit tag as "no index."

    ## Fields
    body (varchar) [TokenIndex]: log message body  ← use ~ for fast search
    timestamp (timestamp): event timestamp           ← no tag, do NOT use ~ on this field

If no `[TokenIndex]` tag is visible in the schema, assume the field is not indexed.

### Operator choice by index availability

| Body field index status                          | Substring search               | Regex/alternation search                                          |
| :----------------------------------------------- | :----------------------------- | :---------------------------------------------------------------- |
| **Default (no `[TokenIndex]` tag visible)**      | `search(string(body), "text")` | `match_regex(string(body), regex("error\|fail\|exception", "i"))` |
| **Explicit `[TokenIndex]` tag visible on field** | `string(body) ~ "text"`        | `string(body) ~ /pattern/i`                                       |

The `~` operator has two forms (use only when `[TokenIndex]` is visible on the field):

-   **Plain string** (`~ "text"`) — **case-INSENSITIVE** substring match. Use `!~` for negation.
-   **Regex** (`~ /pattern/`) — case-sensitive by default. Add `i` flag for case-insensitive.

---

## Wide-Net Error Regex Patterns

Combine multiple signals to catch errors regardless of log structure. Keyword matching alone misses HTTP error status codes (401, 403, 404, 500, 502, 503, etc.).

**CRITICAL: Default to `match_regex()` for the keyword alternation, NOT `~`.** Only use `~ /.../i` when the body field is explicitly marked `[TokenIndex]` in the schema's `## Fields` section.

DEFAULT (no `[TokenIndex]` marker visible) — WITH `severity_number` in schema:

    filter match_regex(string(body), regex("error|exception|fail|fatal|panic|critical", "i")) or severity_number >= 17 or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

DEFAULT (no `[TokenIndex]` marker visible) — WITHOUT `severity_number` in schema:

    filter match_regex(string(body), regex("error|exception|fail|fatal|panic|critical", "i")) or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

OPTIMIZED (body field IS explicitly marked `[TokenIndex]`) — WITH `severity_number` in schema:

    filter string(body) ~ /error|exception|fail|fatal|panic|critical/i or severity_number >= 17 or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

OPTIMIZED (body field IS explicitly marked `[TokenIndex]`) — WITHOUT `severity_number` in schema:

    filter string(body) ~ /error|exception|fail|fatal|panic|critical/i or match_regex(string(body), regex("[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]"))

**CRITICAL: Check the dataset schema BEFORE using `severity_number`.** Many log datasets do NOT have this field.

| Signal                                                           | What it catches                                        | Requires schema field?      |
| :--------------------------------------------------------------- | :----------------------------------------------------- | :-------------------------- |
| Keyword regex (`error\|exception\|fail\|fatal\|panic\|critical`) | Application error messages, stack traces, failure logs | No — works on any body      |
| `severity_number >= 17`                                          | OTel ERROR (17-20) and FATAL (21-24) severity levels   | **Yes** — `severity_number` |
| HTTP status regex (`4[0-9]{2}\|5[0-9]{2}`)                       | 4xx client errors and 5xx server errors in access logs | No — works on any body      |
