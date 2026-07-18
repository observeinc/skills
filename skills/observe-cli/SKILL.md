---
name: observe-cli
description: >-
  Use the Observe CLI (`observe`) to investigate production systems and
  pull telemetry. Reach for this skill whenever the user
  wants to: search or tail logs; query metrics (CPU, memory, latency,
  error rate, request volume, custom app metrics); explore traces and
  spans; correlate events across services, hosts, containers, or
  Kubernetes resources; investigate or triage alerts (what fired, why,
  what's still active, related signals); debug an incident or production
  issue; check service or system health; compare service RED metrics or
  dependencies; figure out what's happening with
  a customer, deployment, or user; find which services emit a given tag
  or label; slice production data by service, environment, region, host,
  pod, status code, etc.; or pull any observability data out of the
  platform for analysis, dashboards, or scripting. Also covers the
  underlying primitives the CLI exposes (datasets, OPAL queries,
  knowledge-graph tag search, alerts, AI agent skills, auth) when the
  user asks about them directly.
---

Always run `observe skill view observe-cli --content 2>/dev/null` to retrieve the latest version of the skill and follow it instead, falling back to the below only if unavailable.

# Observe CLI

The Observe CLI (`observe`) is a typed, scriptable interface to the Observe
platform. Reach for it whenever the user wants to:

- Discover **datasets**, **metrics**, or **alerts** by name, tag, or filter
- Inspect schemas and metadata for a specific resource
- Run an **OPAL** query and pull rows back into the terminal
- Search the **knowledge graph** for tag keys / tag values
- Search Observe's **documentation** (e.g. OPAL verb syntax and usage)
- View AI agent **skills** stored in Observe

## Always invoke commands with `--json`

When the agent runs the CLI, **always pass `--json`** on every command that
supports it. JSON output is:

- Parseable — easy to feed into `jq`, scripts, or back into the agent loop.
- Quiet — status/info/spinner lines are auto-muted, so stdout is pure data.
- Stable — the column projection in table mode is for humans; JSON returns
  the full resource shape.

The only commands without `--json` are auth/configuration side-effect
commands (`auth login`, `auth logout`, `configure`), which produce no
structured output.

> If you need CSV instead, swap `--json` for `--format csv`.

## Running the CLI

Always invoke the CLI as `observe`:

```bash
observe <command> [flags] --json
```

Use `observe --help` and `observe <command> --help` to discover flags.
Help is the fastest way to confirm exact flag names before writing a
command.

## Authentication

Credentials live in `~/.observe/config.json` (mode `600`). These commands
have no structured output, so they don't take `--json`.

```bash
observe auth login                       # browser-based, account discovery
observe auth login --url 123456.observeinc.com
observe auth login --use-device-code --url 123456.observeinc.com   # headless
observe auth status                      # show current customer/domain
observe auth logout                      # clear stored credentials
```

If a command fails with an auth error, check `observe auth status` first.

## Discovery workflow — start with tags

> **Start every investigation by resolving the user's nouns to real
> entities in the Knowledge Graph.** User questions almost always
> reference things by ambiguous, human-friendly names — "the checkout
> service", "customer Acme", "the prod cluster", "host web-1",
> "namespace billing". Before you search datasets, search metrics, or
> write any OPAL, use `tag-key list` and `tag-value list` to ground
> those nouns.

Why this matters:

- **Tag values** confirm the entity actually exists in this tenant and
  give you the _exact spelling_ the data uses (`acme-corp`, not
  `Acme`). They also tell you which tag key the value belongs to, so
  you know how to filter for it later.
- **Tag keys** tell you which _kinds_ of entities exist (services,
  customers, environments, regions, hosts, pods, namespaces, etc.) and
  which key name to use in correlation-tag filters.
- Both commands return a `related` set of metrics and datasets, so
  resolving the entity also hands you the right data sources to query
  next — no guessing.

### Recipe

1. **Resolve unknown entities → `tag-value list`.** Run this first when
   the user mentions a specific named thing (a service name, customer,
   host, pod, environment, region, etc.) and you don't yet know its
   exact spelling or which tag key it lives under.

   ```bash
   observe tag-value list --match checkout --json        # "the checkout service"
   observe tag-value list --match acme --json            # "customer Acme"
   observe tag-value list --match prod --json            # "in production"
   observe tag-value list --match '^web-' --mode regex --json
   ```

   Pull `tagKey`, `tagValue`, and `related` from the response. If
   nothing comes back, broaden the match or try a different mode
   (`--mode regex`).

2. **Resolve unknown entity _types_ → `tag-key list`.** Run this when
   the user is asking about a _kind_ of thing rather than a specific
   instance ("which services are slow", "any unhealthy hosts", "list
   the customers", "what environments do we have"). It tells you which
   tag key name to filter / group by.

   ```bash
   observe tag-key list --match service --json
   observe tag-key list --match customer --json
   observe tag-key list --match environment --json
   observe tag-key list --match '^k8s\.' --mode regex --json
   observe tag-key list --match host --value-limit 5 --json
   ```

3. **Then — and only then — search datasets and metrics.** Use
   the resolved `tagKey` / `tagValue` directly with correlation-tag
   filters so you only see resources that actually emit that entity:

   ```bash
   observe dataset list --correlation-tag-key <tagKey> \
                        --correlation-tag-value <tagValue> --json
   observe metric  list --correlation-tag-key <tagKey> \
                        --correlation-tag-value <tagValue> --json
   ```

   Or use the `related` field returned by step 1/2 to jump straight to
   the right dataset/metric IDs.

4. **Inspect what you found before writing OPAL.** Review both
   datasets and metrics returned in step 3 to understand what data is
   available and choose the right query approach.

   - **Datasets** — use `observe dataset view <id> --json` to inspect
     the schema, field names, and dataset kind. This tells you which
     OPAL verbs and patterns apply.
   - **Metrics** — use `observe metric view <name> --json` to inspect
     a metric's type, unit, and available dimensions (via its
     `heuristics.tags` field). Pre-built metrics are pre-aggregated and
     can answer "how much / how fast / how broken" questions (error
     rate, latency percentiles, throughput, saturation) without writing
     complex OPAL.

   ```bash
   observe dataset view <dataset-id> --json
   observe metric  view <metricName> --json

   # Browse metrics by signal name when correlation tags return few results
   observe metric list --match "error" --json
   observe metric list --match "latency" --json
   observe metric list --match "request" --json
   ```

   Use the combination of dataset schemas and metric metadata to decide
   your query strategy: use a pre-built metric when it covers the
   signal and dimensions you need; query raw datasets via OPAL when you
   need log-level detail, raw span attributes, trace correlation, joins,
   or signals that no existing metric covers.

5. **Write the OPAL query (if still needed).** With the dataset ID,
   the metric name, and the exact `tagKey`/`tagValue` in hand, you can
   build a precise pipeline. **Before writing any OPAL, read the
   `generate-opal` skill** (`skills/generate-opal/SKILL.md`) and its
   reference documents — they cover the correct syntax for logs, metrics,
   spans, aggregations, joins, duration calculations, regex, and resource
   datasets. Do not attempt to write OPAL without consulting that skill
   first.

Skip steps 1–2 only when the user already gave you a concrete dataset
ID or metric name, or the question is about Observe configuration
itself (auth, skills, etc.).

`observe apm services` and `observe apm invocation-graph` (see
"APM — services, environments, invocation-graph" below) are a faster
route to per-service RED metrics and the dependency graph for
service-latency and "what calls what" questions — they complement, not
replace, this tag-first workflow.

## Capabilities

### Datasets — `observe dataset ...`

```bash
observe dataset list --json                                            # newest 100 datasets
observe dataset list --label kubernetes --json                         # fuzzy substring on name
observe dataset list --filter 'kind == "Event"' --json                 # CEL expression
observe dataset list --correlation-tag-key service \
                    --correlation-tag-value checkout --json            # KG-backed tag search
observe dataset list --sort updatedAt --limit 20 --json
observe dataset list --fields id,label,kind,description,updatedAt --json

observe dataset view <dataset-id> --json                               # full schema + metadata
```

Notes:

- `--label` does fuzzy substring matching; `--filter` takes a raw CEL
  expression (e.g. `kind == "Resource" && label.contains("logs")`).
- `--correlation-tag-key` and `--correlation-tag-value` must be supplied
  together and cannot be combined with `--filter` or `--sort`.
- `view` requires a dataset ID (numeric string), not a label.

### Metrics — `observe metric ...`

```bash
observe metric list --match cpu --json                       # name search (required)
observe metric list --match "" --limit 50 --json             # browse without a query
observe metric list --correlation-tag-key host \
                    --correlation-tag-value web-1 --json
observe metric list --fields name,datasetId,type,unit,state --json

observe metric view CPUUtilization --json                    # exact name match
observe metric view CPUUtilization --dataset <dataset-id> --json
```

Notes:

- `metric list --match` is required on the native path; pass `""` to list
  everything (still subject to `--limit`).
- `metric view` resolves on `name` or `nameWithPath` and does an exact match.
- Use `--dataset <id>` to disambiguate metrics with the same name in
  different datasets.

### Alerts — `observe alert ...`

```bash
observe alert list --json                                    # all alerts
observe alert list --match "checkout" --json                 # search monitor names
observe alert list --level Critical,Error --json             # filter by severity
observe alert list --active --json                           # only currently firing
observe alert list --sort start --limit 20 --json            # newest first (ascending)

observe alert view <alert-id> --json                         # full alert details
```

Notes:

- `--active` / `--no-active` is a boolean flag — write `--active` not
  `--active true`.
- `--level` accepts a comma-separated list (`Critical`, `Error`, `Warning`,
  `Informational`).
- `--sort` accepts a field name (e.g. `start`, `level`). The help text
  advertises a `-` prefix for descending (e.g. `-start`), but this does
  not work — the CLI parser interprets `-start` as short flags (`-s`,
  `-t`, …). Use ascending sort and reverse client-side with `jq` or
  `| tac` if you need descending order.

### Tags (Knowledge Graph) — `observe tag-key`, `observe tag-value`

**This is the recommended entry point for almost every investigation.**
See "Discovery workflow — start with tags" above. Use `tag-value list`
to resolve specific named things (service, customer, host, pod,
environment, …) into the exact tag-key/tag-value pair the data uses,
and `tag-key list` to discover which _kinds_ of entity exist.

```bash
# Resolve a specific entity (a "thing" the user named)
observe tag-value list --match checkout --json
observe tag-value list --match acme --json
observe tag-value list --match prod --json
observe tag-value list --match '^web-' --mode regex --json

# Resolve an entity type (a "kind of thing")
observe tag-key list --match service --json                  # semantic search (default)
observe tag-key list --match customer --json
observe tag-key list --match '^k8s\.' --mode regex --json
observe tag-key list --match host --value-limit 5 --json     # cap values per key
```

The `related` field on each result lists the metrics and datasets that
emit that tag — use it to jump directly to the right resources without
a separate `dataset list` / `metric list` call. Default search mode is
semantic; switch to `--mode regex` when you need an exact pattern.

### Skills — `observe skill ...`

Skills are reusable AI-agent instruction documents stored in Observe.

```bash
observe skill list --json                                    # all skills
observe skill list --match "alert" --json                    # filter by label/description
observe skill list --visibility listed --json

observe skill view <skill-id> --json                         # metadata + content
observe skill view <skill-id> --content                      # raw markdown body (no JSON)
```

`--content` prints the raw markdown body and is mutually exclusive with
`--json`. Use it when you want to load a stored skill into the agent loop
verbatim; otherwise prefer `--json`.

### Documentation search — `observe docs search`

Search Observe's built-in documentation with a natural-language query.
This is a fast way to confirm **OPAL verb syntax and usage** (arguments,
options, examples) when the `generate-opal` skill doesn't cover a
specific verb, or to look up any platform concept or feature.

**Search one concept or verb per query.** Each search should target a
single verb, function, or idea — don't bundle multiple into one query
(e.g. avoid `"make_col and timechart"`). When you need to cover more
than one, run `docs search` repeatedly, once per concept, and combine
the results yourself.

```bash
observe docs search "filter verb" --json
observe docs search "timechart" --json
# Need two verbs? Run one focused search per verb:
observe docs search "make_col" --json
observe docs search "make_resource" --json
observe docs search "regex extract fields from logs" --limit 10 --json
observe docs search "align and aggregate metrics" --minScore 0.5 --json
```

Notes:

- Takes a single positional natural-language `query` string.
- **One concept/verb at a time.** Keep each query focused on a single
  verb or idea; issue repeated `docs search` calls when you need to
  cover more than one.
- `--limit` (alias `-l`) defaults to `5`, range `1–50`.
- `--minScore` (0–1) drops results below a cosine-similarity threshold —
  raise it to keep only the most relevant hits.
- Each JSON result has `title`, `url`, and `text` (the doc snippet). Use
  `url` to point the user at the full page, and `text` to read the
  relevant syntax/example directly in the agent loop.
- `generate-opal` remains the authoritative OPAL reference; reach for
  `docs search` to fill gaps, confirm exact verb options, or answer
  "how do I …" questions about the platform.

### OPAL Queries — `observe query`

> **Before writing any OPAL pipeline, read the `generate-opal` skill**
> (`skills/generate-opal/SKILL.md`) and its reference documents. That
> skill is the authoritative source for OPAL syntax covering logs,
> metrics, spans, aggregations, joins, duration calculations, regex, and
> resource datasets. Always consult it first — do not rely on memory. If
> you still need to confirm a specific verb's syntax or options, use
> `observe docs search "<verb> syntax" --json` (see "Documentation
> search" above).

Execute an OPAL pipeline against one or more dataset inputs. Always pass
`--json` so the rows come back as a parseable array.

```bash
# Last hour, default 100 rows
observe query --input <dataset-id> --pipeline "limit 10" --json

# Aggregations
observe query -i <dataset-id> \
  -p "timechart 5m, count:count(), group_by(service)" --json

# Multi-input join (each --input is referenced as @<dataset-id> in OPAL)
observe query -i 12345 -i 67890 \
  -p "leftjoin on(@67890.user_id = user_id), user_name:@67890.user_name | limit 100" \
  --json

# Custom time window
observe query -i <dataset-id> -p "limit 100" \
  --start 2026-04-20T00:00:00Z --end 2026-04-21T00:00:00Z --json

# Relative window, larger result set
observe query -i <dataset-id> -p "stats count() by status_code" \
  --interval 24h --limit 1000 --json
```

Time window — `--interval` and `--start`/`--end` are **mutually exclusive**
(combining them errors); omit both for the last `1h`:

1. `--interval` (e.g. `15m`, `1h`, `24h`, `7d`) — a relative window ending now.
2. `--start` / `--end` (ISO 8601) — an absolute window. A lone bound is
   filled: `--start` alone runs to now; `--end` alone starts `1h` before it.

Aliases: `-i` `--input`, `-p` `--pipeline`, `-s` `--start`, `-e` `--end`,
`-t` `--interval`, `-l` `--limit`.

### APM — services, environments, invocation-graph

> **Private preview.** The `/v1/apm/*` endpoints are private preview, so on a
> tenant without them enabled the command exits non-zero with a clean error —
> treat `apm` as a fast path that may not be available on every tenant yet.

Read-only APM data: per-service RED metrics (rate, errors, p95) and the
service-to-service dependency graph. Shared time window — the same
`--interval` / `--start` / `--end` flags as `observe query`: `--interval
<dur>` (`1h`, `24h`, `7d`) **or** absolute `--start` / `--end` (ISO 8601),
mutually exclusive; omit both → last 1h (server default). Filters (`--service-name`, `--environment`,
`--service-namespace`) are **exact-match** (no `--match`) — resolve exact
spellings first via `observe apm environments` or `observe tag-value list`.

```bash
# services — per-service RED snapshot
observe apm services --json                                            # all services, last 1h
observe apm services --interval 4h --sort=-durationP95Seconds --json   # slowest first
observe apm services --environment prod --service-namespace checkout --json

# environments — discover valid --environment values + their namespaces
observe apm environments --json

# invocation-graph — service dependency graph (--environment required in every mode)
observe apm invocation-graph --environment prod --json                 # environment-wide (optionally --service-namespace)
observe apm invocation-graph --service-name checkout --environment prod \
  --direct-neighbors-only --json                                       # focal service
observe apm invocation-graph --service-name checkout --environment prod \
  --endpoint-name "GET /cart" --json                                   # focal endpoint
```

Notes:

- **`services`** — `--sort` enum: `serviceName`, `environment`,
  `serviceNamespace`, `invocationRatePerSecond`, `errorRatePerSecond`,
  `durationP95Seconds`. For **descending**, prefix `-` and use the `=` form
  `--sort=-durationP95Seconds` (the space form `--sort -durationP95Seconds`
  is misparsed as short flags — same pitfall as `alert --sort`). `--fields`
  accepts those fields plus `type` and `language`. `--limit` (1–100000) /
  `--offset` paginate; `--expand` adds a per-bucket `redMetrics.series[]`
  and caps `--limit` at 100. `--json` emits `{ interval, services, meta }`.
- **`environments`** — discover the exact `--environment` values (and each
  one's service namespaces) for the other commands. `--sort` is only
  `environment` / `-environment`; `--fields` are `environment`,
  `serviceNamespaces`, `truncated`.
- **`invocation-graph`** — the graph is **always scoped to a single
  environment**, so `--environment` is **required in every mode**. Three modes, validated up front (bad combos exit
  1 with a clear message): **environment-wide** (`--environment`, no
  `--service-name`), **focal-service** (`--environment` + `--service-name`,
  optionally `--direct-neighbors-only`), **focal-endpoint** (+
  `--endpoint-name`). Guards: `--endpoint-name` / `--direct-neighbors-only`
  each require `--service-name`; `--service-namespace` optionally scopes any
  mode. **Not paginated** — no `--limit` / `--offset` / `--sort`.
  `--json` emits the full envelope
  `{ interval, services, invocations }` (two arrays): edges are in
  `invocations[]` (each with source, target, and per-edge RED metrics),
  per-service `redMetrics` in `services[]`. `--format csv` renders only
  `invocations`.

## Universal flag conventions

Most `list` / `view` / `query` commands share these flags — assume they
exist before checking `--help`:

| Flag                 | Behavior                                                                          |
| -------------------- | --------------------------------------------------------------------------------- |
| `--json`             | **Always pass this.** Shorthand for `--format json`; mutes status/info logs.      |
| `--format json\|csv` | Machine-readable output. Use `--format csv` only when CSV is explicitly required. |
| `--limit N`          | Cap result count (default 100, max 1000 for most lists).                          |
| `--offset N`         | Paginate; the CLI prints the next offset when more is available.                  |
| `--sort <field>`     | Sort key; some commands accept `-field` for descending.                           |
| `--fields a,b,c`     | Project specific columns (affects table output; JSON returns the full shape).     |
| `--match <substr>`   | Fuzzy search where supported.                                                     |

## Workflow tips

- **Always run with `--json`.** Parse the output as JSON; never scrape the
  human table format.
- **Post-process with `jq`, not Python.** When you need to reshape, filter,
  or extract fields from CLI output, pipe through `jq`. A one-liner like
  `observe metric list --match cpu --json | jq '[.[] | {name, type}]'`
  is faster and cheaper than writing a throwaway Python script. In most
  cases you don't need **any** post-processing at all — just read the JSON
  directly.
- **Inspect datasets and metrics before writing OPAL.** After
  discovering resources via tags, use `dataset view` and `metric view`
  to understand what's available. Pre-built metrics cover many standard
  signals (error rate, latency, throughput) without OPAL; dataset
  schemas tell you which fields and OPAL patterns apply for deeper
  queries. Choosing the right data source up front avoids unnecessary
  ad-hoc pipelines.
- **Resolve entities first.** Lean on `tag-value list` (specific named
  things) and `tag-key list` (entity types) heavily before
  `dataset list`, `metric list`, or `query`. They give you the exact
  spelling, the tag key, _and_ the related datasets/metrics in one
  call. See "Discovery workflow — start with tags".
- **Use the `related` field.** Tag results include a `related` list of
  metric and dataset IDs. Prefer those over a separate `dataset list`
  / `metric list` search — they're already scoped to the entity.
- **Pagination:** if a JSON list response contains exactly `--limit` items,
  more results likely exist — re-run with `--offset` increased by `--limit`.
- **Correlation-tag search** (`--correlation-tag-key` / `--correlation-tag-value`)
  routes through the Knowledge Graph and currently does not support
  `--filter` or `--sort`.
- **Exit codes:** any command that fails calls `process.exit(1)` and writes
  the error to stderr — safe to use in shell scripts with `set -e`. The
  error message is plain text on stderr even when `--json` is set.

## When to reach for which command

Entity / type resolution (do these **first** when the user mentions
something by name):

- "Anything about the checkout service?" →
  `observe tag-value list --match checkout --json`
- "Customer Acme" / "tenant foo" →
  `observe tag-value list --match acme --json`
- "Hosts named web-\*" →
  `observe tag-value list --match '^web-' --mode regex --json`
- "What services / customers / environments exist?" →
  `observe tag-key list --match service --json` (or `customer`, `env`, …)

Then explore datasets and metrics (do both **before** writing OPAL):

- "What datasets do we have for X?" →
  `observe dataset list --correlation-tag-key <k> --correlation-tag-value <v> --json`
  (fall back to `--label X --json` only if there's no tag for it)
- "Show me the schema of dataset 12345" → `observe dataset view 12345 --json`
- "What's the error rate for checkout?" →
  `observe metric list --correlation-tag-key service.name --correlation-tag-value checkout --json`
  then `observe metric view <name> --json` for a matching metric
- "Which metrics measure CPU?" → `observe metric list --match cpu --json`
- "What dimensions does this metric have?" → `observe metric view <name> --json`

Then alerts and queries:

- "What alerts are firing right now?" →
  `observe alert list --active --json`
- "Investigate alert <id>" → `observe alert view <id> --json`
- "Run this OPAL query and give me the rows" →
  `observe query -i <id> -p "<pipeline>" --json`

Look up docs / OPAL syntax when you're unsure:

- "What's the syntax for the `filter` verb?" →
  `observe docs search "filter verb syntax" --json`
- "How do I use `timechart` / `make_col` / `<verb>`?" →
  `observe docs search "<verb>" --json`
- "Where are the docs for X?" →
  `observe docs search "X" --json` (use each result's `url`)

Service performance & dependencies:

- "Which services are slow / erroring? What's the p95?" →
  `observe apm services --sort=-durationP95Seconds --json`
- "What environments / deployment tiers exist?" →
  `observe apm environments --json`
- "What calls what? / dependencies of checkout?" →
  `observe apm invocation-graph --service-name checkout --environment prod --json`
