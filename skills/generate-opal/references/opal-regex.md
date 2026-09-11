# OPAL Regex & Pattern Matching — Complete Reference

## POSIX ERE plus Perl shorthands — NOT full PCRE

**OPAL regex is Snowflake's POSIX ERE plus the Perl backslash shorthands.** This applies everywhere: `~ /pattern/`, `match_regex()`, `replace_regex()`, `get_regex()`, `extract_regex`, and `regex()`.

`\d`, `\w`, `\s`, and `\b` all work. So do the POSIX classes — use whichever reads better:

| Shorthand | POSIX equivalent |
| :-------- | :--------------- |
| `\d`      | `[0-9]`          |
| `\w`      | `[a-zA-Z0-9_]`   |
| `\s`      | `[[:space:]]`    |
| `\b`      | none — use `\b`  |

**NEVER use `[[:<:]]` / `[[:>:]]` for word boundaries.** That is MySQL syntax. Snowflake rejects it outright with `invalid character class range: [:<:]`, failing the whole query. Use `\b`.

What genuinely does not exist: **lookahead `(?=`, lookbehind `(?<=`, and backreferences `\1`**. OPAL validates every pattern with Go's `regexp` (RE2), which does not support them, so such a pattern is rejected at compile time and never reaches the warehouse. Contrast this with the inline flags `(?i)` and non-capturing groups `(?:...)` below: those _pass_ RE2 validation and DO reach the warehouse, then fail at Snowflake execution — a more dangerous class, because there is no compile-time safety net.

Non-greedy quantifiers (`*?`, `+?`) DO work in queries — `<.*?>` against `<a><b>` matches just `<a>`.

**NEVER use inline `(?...)` flag groups (`(?i)`, `(?m)`, `(?s)`) or non-capturing groups `(?:...)`.** Both are PCRE syntax that OPAL's RE2 validator _accepts_ — so they compile, and can even work under `REGEXP_SUBSTR` — but Snowflake's `REGEXP_LIKE`/`RLIKE` rejects both with `no argument for repetition operator: ?` and fails the query. This makes them unreliable in the hardest way to debug: `match_regex`/`~` normally compile to `REGEXP_SUBSTR`, but on a substring-indexed body column they silently switch to the `RLIKE` path, so the exact same pattern that worked on one dataset hard-errors on another. This is confirmed in production. Rewrite instead — pass an inline flag as a separate argument or a slash suffix, and replace `(?:group)` with a plain group `(group)`:

    filter match_regex(eventName, /getpolicy/i)                              # slash suffix — case-insensitive
    filter match_regex(log, /^debug/, 'i')                                   # flags argument
    filter match_regex(string(body), regex("error.*timeout", "i"))           # regex() constructor with flags

Supported flags: `c` (case-sensitive, default), `i` (case-insensitive), `m` (multiline), `s` (dot matches newline). The same optional flags argument applies to `extract_regex`, `get_regex`, and `get_regex_all`.

Note: `(?P<name>)` **named capture groups** and plain `(...)` groups are valid OPAL syntax and remain supported — only the `(?i)`/`(?m)`/`(?s)` **flag** form and the `(?:...)` **non-capturing** form are rejected. Rewriting `(?:group)` to `(group)` is safe for `match_regex`/`~`, where the capture is never read.

WRONG/CORRECT:

    WRONG:   filter match_regex(body, /(?i)order[_-]?id.*/)
    CORRECT: filter match_regex(body, /order[_-]?id.*/i)
    CORRECT: filter match_regex(body, /order[_-]?id.*/, 'i')
    CORRECT: filter match_regex(body, regex("order[_-]?id.*", "i"))

    WRONG:   filter match_regex(body, /(aws-waf-logs-prod(?:-[0-9])?)-[0-9]{4}/)
    CORRECT: filter match_regex(body, /(aws-waf-logs-prod(-[0-9])?)-[0-9]{4}/)

---

## Operator & Function Reference

### String matching overview

- **Substring match** — `contains(col, "text")`
- **Glob search** — `col ~ "pattern*"` or `col ~ <pattern*>`
- **Regex match** — `match_regex(col, regex("pattern"))`
- **Multi-term search** — `search(col, "term")`

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

- `get_regex(col, /pattern/)` — single matched string
- `get_regex_all(col, /pat/)` — array of all matched strings

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

- **Plain string** (`~ "text"`) — **case-INSENSITIVE** substring match. Use `!~` for negation.
- **Regex** (`~ /pattern/`) — case-sensitive by default. Add `i` flag for case-insensitive.

---

## Wide-Net Error Regex Patterns

Combine multiple signals to catch errors regardless of log structure.

**NEVER regex the body for HTTP status codes.** A pattern like `[^0-9](4[0-9]{2}|5[0-9]{2})[^0-9]` matches any three-digit number starting with 4 or 5 anywhere in the line, which on real log data means 43% of all rows at roughly 5% precision — `AppleWebKit/537.36` reads as a 5xx, as does a latency of `0.412` or `replicas 500`. It buries the keyword and severity signals it is OR'd with.

To catch HTTP errors, filter the **status field** the dataset already has (`status_code`, `status`, `response_code` — check the schema's `## Fields` section):

    filter int64(status_code) >= 400

If the dataset has no status field, the keyword and severity signals below are the whole answer. A body scan for bare numbers is not a substitute.

**CRITICAL: Default to `match_regex()` for the keyword alternation, NOT `~`.** Only use `~ /.../i` when the body field is explicitly marked `[TokenIndex]` in the schema's `## Fields` section.

DEFAULT (no `[TokenIndex]` marker visible) — WITH `severity_number` in schema:

    filter match_regex(string(body), regex("error|exception|fail|fatal|panic|critical", "i")) or severity_number >= 17

DEFAULT (no `[TokenIndex]` marker visible) — WITHOUT `severity_number` in schema:

    filter match_regex(string(body), regex("error|exception|fail|fatal|panic|critical", "i"))

OPTIMIZED (body field IS explicitly marked `[TokenIndex]`) — WITH `severity_number` in schema:

    filter string(body) ~ /error|exception|fail|fatal|panic|critical/i or severity_number >= 17

OPTIMIZED (body field IS explicitly marked `[TokenIndex]`) — WITHOUT `severity_number` in schema:

    filter string(body) ~ /error|exception|fail|fatal|panic|critical/i

**CRITICAL: Check the dataset schema BEFORE using `severity_number`.** Many log datasets do NOT have this field.

| Signal                                                           | What it catches                                        | Requires schema field?      |
| :--------------------------------------------------------------- | :----------------------------------------------------- | :-------------------------- |
| Keyword regex (`error\|exception\|fail\|fatal\|panic\|critical`) | Application error messages, stack traces, failure logs | No — works on any body      |
| `severity_number >= 17`                                          | OTel ERROR (17-20) and FATAL (21-24) severity levels   | **Yes** — `severity_number` |
| `int64(status_code) >= 400`                                      | 4xx client errors and 5xx server errors in access logs | **Yes** — a status field    |

**Do not add a fourth signal that regexes the body for status codes.** See the warning at the top of this section.
