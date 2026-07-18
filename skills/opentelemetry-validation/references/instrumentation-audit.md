# Instrumentation Audit

Queries span data for a service and evaluates conformance to OpenTelemetry semantic conventions. Produces findings at four severity levels: `violation`, `warning`, `improvement`, `information`.

Prerequisites:

- Observe CLI

Data sources:

- `Tracing/Span`
- `Tracing/Span Event`

## Setup

Resolve dataset IDs before running checks:

1. `observe dataset list --filter 'label == "Tracing/Span"' --json`
2. `observe dataset list --filter 'label == "Tracing/Span Event"' --json`

All queries below use `<tracing-span-id>` and `<tracing-span-event-id>` as placeholders.

If `service.namespace` is known, add `filter service_namespace = "<service-namespace>"` to every `Tracing/Span` query below.

For the `Tracing/Span Event` query in Check 7, extract `resource_attributes."service.namespace"` and add a matching filter when `service.namespace` is known.

---

## Convention Conformance Checks

Each check filters to a span category, extracts expected attributes, and aggregates attribute coverage per instrumentation library using `sum(if(..., 1, 0))` inside `statsby`. Checks accept both old and stable attribute names. When only the old form is present, report as `improvement`.

### Check 1: HTTP Spans

Covers both server and client HTTP spans.

```bash
observe query --input <tracing-span-id> --interval 4h --limit 50 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col http_method:string(attributes."http.request.method"), http_method_old:string(attributes."http.method")
filter not is_null(http_method) or not is_null(http_method_old)
make_col lib_name:string(instrumentation_library.name)
make_col http_status:string(attributes."http.response.status_code"), http_status_old:string(attributes."http.status_code")
make_col http_route:string(attributes."http.route")
make_col url_scheme:string(attributes."url.scheme")
make_col has_status:if(not is_null(http_status) or not is_null(http_status_old), 1, 0)
make_col has_route:if(not is_null(http_route), 1, 0)
make_col has_url_scheme:if(not is_null(url_scheme), 1, 0)
make_col uses_stable:if(not is_null(http_method), 1, 0)
make_col uses_old:if(is_null(http_method) and not is_null(http_method_old), 1, 0)
statsby total:count(), with_status:sum(has_status), with_route:sum(has_route), with_url_scheme:sum(has_url_scheme), stable_conv:sum(uses_stable), old_conv:sum(uses_old), group_by(service_name, lib_name, span_type)
sort desc(total)'
```

Evaluation:

| Condition                 | Severity    | Finding                                                                     |
| ------------------------- | ----------- | --------------------------------------------------------------------------- |
| `with_status < total`     | violation   | HTTP spans missing response status code                                     |
| `with_route < total`      | warning     | HTTP spans missing `http.route` — endpoint grouping degraded                |
| `with_url_scheme < total` | improvement | HTTP spans missing `url.scheme`                                             |
| `old_conv > 0`            | improvement | Using old HTTP conventions (`http.method` instead of `http.request.method`) |

If zero rows returned, no HTTP spans exist — skip this check.

### Check 2: Database Spans

Group by the effective `db.system` so each system class can be evaluated against rules appropriate for that technology.

```bash
observe query --input <tracing-span-id> --interval 4h --limit 50 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col db_system:string(attributes."db.system"), db_system_new:string(attributes."db.system.name")
filter not is_null(db_system) or not is_null(db_system_new)
make_col db_system_effective:coalesce(db_system_new, db_system)
make_col lib_name:string(instrumentation_library.name)
make_col db_name:string(attributes."db.name"), db_namespace:string(attributes."db.namespace")
make_col db_statement:string(attributes."db.statement"), db_query:string(attributes."db.query.text")
make_col server_addr:string(attributes."server.address"), net_peer:string(attributes."net.peer.name")
make_col has_db_name:if(not is_null(db_name) or not is_null(db_namespace), 1, 0)
make_col has_statement:if(not is_null(db_statement) or not is_null(db_query), 1, 0)
make_col has_server:if(not is_null(server_addr) or not is_null(net_peer), 1, 0)
make_col uses_stable:if(not is_null(db_system_new), 1, 0)
make_col uses_old:if(is_null(db_system_new) and not is_null(db_system), 1, 0)
statsby total:count(), with_db_name:sum(has_db_name), with_statement:sum(has_statement), with_server:sum(has_server), stable_conv:sum(uses_stable), old_conv:sum(uses_old), group_by(service_name, lib_name, db_system_effective)
sort desc(total)'
```

#### Per-system evaluation

Required attributes depend on `db.system` because OTel's database conventions mean different things for different technologies. Pick the row class for `db_system_effective` and apply only those rules.

**SQL and SQL-like systems** — `postgresql`, `mysql`, `mssql`, `oracle`, `mariadb`, `cockroachdb`, `db2`, `sqlite`, `hive`, `spanner`, `cassandra`, `clickhouse`

| Condition                | Severity    | Finding                                                                                 |
| ------------------------ | ----------- | --------------------------------------------------------------------------------------- |
| `with_server < total`    | warning     | SQL spans missing server address — database peer identity at risk                       |
| `with_db_name < total`   | warning     | SQL spans missing database name (`db.namespace` / `db.name`) — service detection broken |
| `with_statement < total` | improvement | SQL spans missing query text — query-level visibility reduced                           |
| `old_conv > 0`           | improvement | Using old DB conventions (`db.system` instead of `db.system.name`)                      |

**Document and search systems** — `mongodb`, `elasticsearch`, `opensearch`, `couchdb`, `couchbase`

| Condition                | Severity    | Finding                                                                                                  |
| ------------------------ | ----------- | -------------------------------------------------------------------------------------------------------- |
| `with_server < total`    | warning     | Document/search spans missing server address                                                             |
| `with_db_name < total`   | warning     | Document/search spans missing namespace (collection / index) — `db.namespace` carries that identity here |
| `with_statement < total` | improvement | Document/search spans missing operation/query text                                                       |
| `old_conv > 0`           | improvement | Using old DB conventions (`db.system` instead of `db.system.name`)                                       |

**Key-value and cache systems** — `redis`, `memcached`, `dynamodb`, `etcd`, `hazelcast`

| Condition                | Severity    | Finding                                                                                                                                                     |
| ------------------------ | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `with_server < total`    | warning     | Key-value spans missing server address — peer identity at risk                                                                                              |
| `with_db_name < total`   | improvement | Key-value spans missing `db.namespace` — for Redis this is the numeric DB index and is optional when default DB is used; not required for service detection |
| `with_statement < total` | improvement | Key-value spans missing command text — operation-level visibility reduced                                                                                   |
| `old_conv > 0`           | improvement | Using old DB conventions (`db.system` instead of `db.system.name`)                                                                                          |

**Messaging-as-DB and other** — `kafka` (when surfaced under `db.system`), unknown / unrecognized values

| Condition                | Severity    | Finding                                                                                     |
| ------------------------ | ----------- | ------------------------------------------------------------------------------------------- |
| `with_server < total`    | warning     | DB spans missing server address                                                             |
| `with_db_name < total`   | improvement | Unrecognized `db.system` — `db.namespace` semantics unclear, so flagged at improvement only |
| `with_statement < total` | improvement | DB spans missing statement text                                                             |
| `old_conv > 0`           | improvement | Using old DB conventions (`db.system` instead of `db.system.name`)                          |

If zero rows returned, no database spans exist — skip this check.

#### `db.system` value validity

In addition to the per-system rules above, evaluate `db_system_effective` itself against the OTel registry. Canonical values include `postgresql`, `mysql`, `mssql`, `oracle`, `mariadb`, `cockroachdb`, `db2`, `sqlite`, `hive`, `spanner`, `cassandra`, `clickhouse`, `mongodb`, `elasticsearch`, `opensearch`, `couchdb`, `couchbase`, `redis`, `memcached`, `dynamodb`, `etcd`, `hazelcast`, and `other_sql`.

| Condition                                          | Severity  | Finding                                                                                                                                                                                         |
| -------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `db_system_effective` is not in the canonical list | violation | Non-canonical `db.system` value — should be a registry identifier (e.g. `redis`), not an instance / peer name. Routes spans to the wrong system bucket and breaks per-system audit assumptions. |

When this fires, the offending row should be reported separately from the per-system evaluation since the row's bucket is itself untrusted.

### Check 3: RPC Spans

```bash
observe query --input <tracing-span-id> --interval 4h --limit 50 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col rpc_system:string(attributes."rpc.system")
filter not is_null(rpc_system)
make_col lib_name:string(instrumentation_library.name)
make_col rpc_service:string(attributes."rpc.service")
make_col rpc_method:string(attributes."rpc.method")
make_col has_service:if(not is_null(rpc_service), 1, 0)
make_col has_method:if(not is_null(rpc_method), 1, 0)
statsby total:count(), with_service:sum(has_service), with_method:sum(has_method), group_by(service_name, lib_name, rpc_system)
sort desc(total)'
```

Evaluation:

| Condition              | Severity | Finding                         |
| ---------------------- | -------- | ------------------------------- |
| `with_service < total` | warning  | RPC spans missing `rpc.service` |
| `with_method < total`  | warning  | RPC spans missing `rpc.method`  |

If zero rows returned, no RPC spans exist — skip this check.

### Check 4: Messaging Spans

```bash
observe query --input <tracing-span-id> --interval 4h --limit 50 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col msg_system:string(attributes."messaging.system")
filter not is_null(msg_system)
make_col lib_name:string(instrumentation_library.name)
make_col msg_dest:string(attributes."messaging.destination.name")
make_col msg_op:string(attributes."messaging.operation"), msg_op_type:string(attributes."messaging.operation.type")
make_col has_dest:if(not is_null(msg_dest), 1, 0)
make_col has_op:if(not is_null(msg_op) or not is_null(msg_op_type), 1, 0)
statsby total:count(), with_dest:sum(has_dest), with_op:sum(has_op), group_by(service_name, lib_name, msg_system)
sort desc(total)'
```

Evaluation:

| Condition           | Severity    | Finding                                              |
| ------------------- | ----------- | ---------------------------------------------------- |
| `with_dest < total` | warning     | Messaging spans missing `messaging.destination.name` |
| `with_op < total`   | improvement | Messaging spans missing operation type               |

If zero rows returned, no messaging spans exist — skip this check.

### Check 5: Span Naming

```bash
observe query --input <tracing-span-id> --interval 4h --limit 100 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col lib_name:string(instrumentation_library.name)
statsby span_count:count(), group_by(service_name, lib_name, span_name, span_type)
sort desc(span_count)'
```

Inspect the `span_name` values in the results:

| Pattern                                           | Severity    | Finding                                                             |
| ------------------------------------------------- | ----------- | ------------------------------------------------------------------- |
| Span name contains a UUID                         | warning     | High-cardinality span name — route parameterization missing         |
| Span name contains a long numeric ID              | warning     | High-cardinality span name — route parameterization missing         |
| Span name is a full URL                           | warning     | Span name should be a low-cardinality route, not a full URL         |
| Span name exceeds 120 characters                  | improvement | Unusually long span name                                            |
| Many unique span names each with `span_count = 1` | warning     | High-cardinality span names — likely missing route parameterization |

### Check 6: Error and Status Consistency

```bash
observe query --input <tracing-span-id> --interval 4h --limit 50 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
filter error = true or response_status = "Error"
make_col lib_name:string(instrumentation_library.name)
make_col has_error_attr:if(error = true, 1, 0)
make_col has_status_error:if(response_status = "Error", 1, 0)
make_col both:if(error = true and response_status = "Error", 1, 0)
statsby total:count(), with_error_attr:sum(has_error_attr), with_status_error:sum(has_status_error), consistent:sum(both), group_by(service_name, lib_name)
sort desc(total)'
```

Evaluation:

| Condition                             | Severity    | Finding                                                                          |
| ------------------------------------- | ----------- | -------------------------------------------------------------------------------- |
| `with_error_attr > with_status_error` | warning     | Spans marked `error=true` but span status (`response_status`) not set to `Error` |
| `with_status_error > with_error_attr` | information | Spans with `response_status=Error` but `error` attribute not set                 |

If zero rows returned, no error spans exist in the interval — note as informational.

### Check 7: Exception Events

```bash
observe query --input <tracing-span-event-id> --interval 4h --limit 50 --json --pipeline '
filter event_name = "exception"
make_col svc:string(resource_attributes."service.name"), env:string(resource_attributes."deployment.environment.name"), svc_ns:string(resource_attributes."service.namespace")
filter env = "<environment>"
filter svc = "<service-name>"
make_col exc_type:string(attributes."exception.type"), exc_msg:string(attributes."exception.message"), exc_stack:string(attributes."exception.stacktrace")
make_col has_type:if(not is_null(exc_type), 1, 0)
make_col has_msg:if(not is_null(exc_msg), 1, 0)
make_col has_stack:if(not is_null(exc_stack), 1, 0)
statsby total:count(), with_type:sum(has_type), with_msg:sum(has_msg), with_stack:sum(has_stack), group_by(svc)
sort desc(total)'
```

If `service.namespace` is known, add `filter svc_ns = "<service-namespace>"` after `filter svc = "<service-name>"`.

Evaluation:

| Condition            | Severity    | Finding                                         |
| -------------------- | ----------- | ----------------------------------------------- |
| `with_type < total`  | violation   | Exception events missing `exception.type`       |
| `with_msg < total`   | warning     | Exception events missing `exception.message`    |
| `with_stack < total` | improvement | Exception events missing `exception.stacktrace` |

If zero rows returned but error spans exist (Check 6), report as `improvement` — exception events provide richer error context than status alone.

---

## Schema Version Drift Checks

Compare the schema version the service emits against the latest published OTel schema. Summarize attribute renames, removals, and additions between the two versions.

### Check 8: Span Schema URL

```bash
observe query --input <tracing-span-id> --interval 4h --limit 50 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col lib_name:string(instrumentation_library.name), lib_version:string(instrumentation_library.version), schema:string(fields."schema_url")
statsby span_count:count(), group_by(service_name, lib_name, lib_version, schema)
sort desc(span_count)'
```

Evaluation procedure (do not skip — if you cannot complete a step, mark the finding as `pending` with a reason, do not guess):

1. **Discover the latest published schema version.** Fetch with `curl` in the shell, never a web-page/markdown fetch tool — markdown conversion collapses JSON and YAML formatting and makes the result unparseable.
   - Run `curl -sS "https://api.github.com/repos/open-telemetry/semantic-conventions/contents/schemas" | jq -r '.[].name' | sort -V | tail -1`. This lists every published schema version and returns the highest; `sort -V` orders versions numerically, so `1.40.0` / `1.41.0` rank above `1.9.0` (a plain sort wrongly picks `1.9.0`).
   - Do NOT parse `https://github.com/open-telemetry/semantic-conventions/tree/main/schemas` — it is JS-rendered and the file list may come back empty or partial.
   - Do NOT use `releases/latest` — it returns the repo release tag (e.g. `v1.41.1`), which can be ahead of the schemas directory and will 404 when fetched as a schema file.
   - Set `latest_version` to that value, then confirm the raw file exists: `curl -sS -o /dev/null -w '%{http_code}\n' "https://raw.githubusercontent.com/open-telemetry/semantic-conventions/main/schemas/<latest_version>"` should print `200`.
   - If either `curl` fails (non-2xx, rate limited, or empty), mark all schema-drift findings as `pending` and report the failure. Do NOT guess the latest version.

2. **Classify each library's `schema` value:**
   - `schema` is null/empty → `warning` (`missing_schema_url`). Stop; skip steps 3–5 for this row.
   - `schema` equals `latest_version` → `information` (`schema_current`). Stop.
   - `schema` is older than `latest_version` → continue to step 3.

3. **Fetch the latest schema file ONCE per run with `curl`** (never a web-page/markdown fetch — it collapses the YAML indentation that defines the `versions:` hierarchy). Run `curl -sS "https://raw.githubusercontent.com/open-telemetry/semantic-conventions/main/schemas/<latest_version>"`. OTel schema files are cumulative — the file at `latest_version` contains a `versions:` map covering every prior release back to 1.4.0, so you do NOT need to fetch intermediate schemas. Cache the parsed YAML and reuse it for every library evaluated in this run.

4. **For each library with an older `schema`, extract the change chain** from the cached YAML:
   - Walk the `versions:` map from the version just above `emitted_version` through `latest_version` (inclusive).
   - From each version block, collect entries under `all.changes`, `spans.changes`, `metrics.changes`, and `resources.changes`. Relevant entry kinds: `rename_attributes`, `rename_metrics`, `remove_attributes`.
   - Tally `attribute_renames`, `metric_renames`, and `attribute_removals` across that range.

5. **Required `evidence` shape** for every schema-drift finding (level `improvement`, id `schema_drift`):
   - `emitted_schema_url`
   - `latest_schema_url`
   - `minor_version_gap` (latest minor − emitted minor)
   - `attribute_renames`, `metric_renames`, `attribute_removals` (integer counts)
   - If the SUM of those three counts is ≤ 25, also include `changes[]` with `{from_version, to_version, kind, mapping}` entries verbatim from the YAML. If the sum is > 25, omit `changes[]` to keep the finding readable — the counts plus an upgrade recommendation are sufficient.

   A finding that lacks the required counts is invalid — re-run the step or downgrade to `pending`.

### Check 9: Resource Schema URL

```bash
observe query --input <tracing-span-id> --interval 4h --limit 50 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col lib_name:string(instrumentation_library.name), lib_version:string(instrumentation_library.version), resource_schema:string(fields."resource_schema_url")
statsby span_count:count(), group_by(service_name, lib_name, lib_version, resource_schema)
sort desc(span_count)'
```

Same evaluation as Check 8 but using the `resource_schema` value.

---

## Severity Levels

- **violation** — required attribute missing
- **warning** — conditionally required attribute missing, attribute absence degrades derived data quality, or schema URL missing entirely
- **improvement** — recommended attribute missing, old convention used where stable exists, or schema behind latest but within one release
- **information** — neutral observations (schema is current, no deprecated attributes found, etc.)

Overall audit result: **PASS** if no `violation` or `warning` findings, **FAIL** otherwise. `improvement` and `information` findings are included in the report but do not cause failure.

## Findings Report Format

When reporting manual audit results, use a shape compatible with the future `apm services instrumentations audit` output:

- **id** — stable finding type such as `missing_attribute`, `old_convention`, or `schema_drift`
- **title** — short human-readable finding title
- **level** — `violation`, `warning`, `improvement`, or `information`
- **message** — summary of what was observed
- **context** — optional structured details such as attribute name, span type, or schema version
- **affected[]** — one or more affected targets
- **evidence** — the query result values that support the finding (for example, "42 of 50 DB spans missing database name")

Use these `affected[]` shapes:

- **instrumentation**
  - `environment`
  - `serviceName`
  - `serviceNamespace`
  - `serviceVersion`
  - `instrumentationLibraryName`
  - `instrumentationLibraryVersion`
- **service**
  - `environment`
  - `serviceName`
  - `serviceNamespace`
  - `serviceVersion`

If a finding is tied to a specific library version, prefer an `affected[]` entry of type `instrumentation`. If the finding applies to the whole service, use an entry of type `service`.
