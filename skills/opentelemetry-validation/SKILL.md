---
name: opentelemetry-validation
description: Canned queries for various APM features in Observe. Use to validate auto-instrumentation, or when validating or troubleshooting OpenTelemetry instrumentation for a service sending telemetry to Observe.
---

Always run `observe skill view opentelemetry-validation --content 2>/dev/null` to retrieve the latest version of the skill and follow it instead, falling back to the below only if unavailable.

# OpenTelemetry Validation and Troubleshooting

This skill validates that an instrumented service is emitting correct telemetry to Observe, and provides troubleshooting assistance.

## Prerequisites

- The `observe` CLI is authenticated and configured
- The `generate-opal` skill is available for metric query generation
- The target service has been deployed and is running

## Query Reference

Each validation query lives in its own file under [references/](references/). Use those files as the source of truth instead of rewriting commands inline.

| Reference                      | File                                                                      |
| ------------------------------ | ------------------------------------------------------------------------- |
| Ingest Pipeline Pre-Check      | [pipeline-precheck.md](references/pipeline-precheck.md)                   |
| Service List                   | [service-list.md](references/service-list.md)                             |
| Instrumentation List           | [instrumentations-list.md](references/instrumentations-list.md)           |
| Service Instances              | [service-instances.md](references/service-instances.md)                   |
| Service entry points           | [endpoint-spans.md](references/endpoint-spans.md)                         |
| Search Spans                   | [search-spans.md](references/search-spans.md)                             |
| Search Span Events             | [search-span-events.md](references/search-span-events.md)                 |
| APM R.E.D Metrics              | [red-metrics.md](references/red-metrics.md)                               |
| APM Runtime Metrics            | [runtime-metrics.md](references/runtime-metrics.md)                       |
| APM Infrastructure Metrics     | [infrastructure-metrics.md](references/infrastructure-metrics.md)         |
| APM Deployment Tracking        | [deployment-tracking.md](references/deployment-tracking.md)               |
| APM Performance Breakdown      | [performance-breakdown.md](references/performance-breakdown.md)           |
| APM Database Service Detection | [database-service-detection.md](references/database-service-detection.md) |
| Instrumentation Audit          | [instrumentation-audit.md](references/instrumentation-audit.md)           |

## Shared Metric Query Workflow

Use this workflow for metric-backed references such as R.E.D metrics, runtime metrics, infrastructure metrics, deployment tracking metrics, performance breakdown, and database service detection metrics.

1. Use the dataset named in the reference.
2. Inspect the dataset schema only when the dataset choice, field names, or dimensions are unclear.
3. Use the `generate-opal` skill and its required `opal-*` sub-skills to generate the metric OPAL.
4. Run `observe query --input <dataset-id> --pipeline '<opal>' --json` with the appropriate time range.
5. Apply the reference-specific filters, metric families, and pass criteria.

Do not repeat generic `observe dataset list`, `observe dataset view`, or `observe query` boilerplate in each metric reference unless the query shape itself is the point of the example.

## Projecting with `pick_col`

`pick_col` keeps only the listed columns. On Interval and Event datasets you must retain the dataset's valid-time column(s), or the query fails with `Query failed: Query returned no schema`. This applies only to `pick_col` — aggregations (`statsby`, `timechart`, `aggregate`) build their own schema and are unaffected.

| Dataset              | Keep in `pick_col`                |
| -------------------- | --------------------------------- |
| `Tracing/Span`       | `start_time`, `end_time`          |
| `Tracing/Service`    | `@."Valid From"`, `@."Valid To"`  |
| `Tracing/Deployment` | `deployment_time`, `@."Valid To"` |
| `Tracing/Span Event` | `timestamp`                       |

Both bounds are required: keeping only one of an Interval dataset's two time columns also returns "no schema". For `Tracing/Deployment` the from-bound is `deployment_time`, not `@."Valid From"` — substituting `@."Valid From"` fails the schema.

## Entry Point

When the user asks to validate or troubleshoot a service, determine which flow to use:

- **"Validate my service"** / **"Is my instrumentation working?"** → Validation Flow
- **"Something looks wrong"** / **"I'm not seeing data"** / **"Metrics are missing"** → Troubleshooting Flow

## Ingest Pipeline Pre-Check (run first)

Before running any validation or troubleshooting checks — and after confirming the tenant (Step 1 of either flow) — run the [Ingest Pipeline Pre-Check](references/pipeline-precheck.md). It verifies three things independent of the target service:

1. **Observe Agent health** — the collector fleet is alive and authenticating, so observations can reach the backend.
2. **Ingest pipeline health** — the datastreams behind validation (`Tracing/Span`, `Metrics/OpenTelemetry`, OTel Logs) show a recent `lastIngest` and are not disabled.
3. **Dataset materialization lag** — the derived datasets the numbered checks read (`Tracing/Service`, `Tracing/Deployment`, `Tracing/Service Explorer Drilldown Metrics`) are materializing on time, so a lagging transform is not mistaken for missing telemetry.

If any signal is abnormal, **flag it prominently before the service-scoped checks start** and confirm with the user before continuing. A broken agent, stalled ingest, or lagging materialization will produce "no data" / "missing metrics" results that would otherwise be misattributed to the service's instrumentation. Do not silently run the numbered checks against a known-broken collection, ingest, or materialization layer.

## Validation Flow

### Step 1: Confirm Observe Tenant

Validation runs against whichever Observe tenant the `observe` CLI is currently authenticated to. Confirm this before running any queries.

1. Run `observe auth status` to verify the CLI is authenticated and to read the current customer ID / domain.
2. If it reports no active session or fails with an auth error, ask the user to authenticate with `observe auth login` (or `observe auth login --url <tenant>.observeinc.com` for a specific tenant) and wait until it succeeds.
3. Show the user the resolved tenant (customer ID / domain) and confirm it is the tenant they want to run the rest of the validation against.
4. Do not continue until the user confirms. If it is the wrong tenant, have them switch with `observe auth login --url <tenant>.observeinc.com` and re-check.

Once the tenant is confirmed, run the [Ingest Pipeline Pre-Check](references/pipeline-precheck.md) (Agent health + ingest pipeline health) before Step 2. If either is abnormal, flag it and confirm with the user before proceeding.

### Step 2: Determine Target Scope

Gather the logical service identity. All subsequent queries are scoped to these values:

- `deployment.environment.name` (required)
- `service.name` (required)
- `service.namespace` (optional)

Ask the user for these if not already known. When a dataset uses `environment`, `service_name`, `service_namespace`, tags, or labels fields instead, use the mappings from `Scope Mapping`.

### Scope Mapping

The logical identity maps to different field names per dataset:

| Logical attribute                                         | `Tracing/Span`      | `Metrics/OpenTelemetry`                                                                 | `Prometheus (Host/K8s)`           |
| --------------------------------------------------------- | ------------------- | --------------------------------------------------------------------------------------- | --------------------------------- |
| `service.name`                                            | `service_name`      | `resource_attributes."service.name"`                                                    | `labels."service_name"`           |
| `deployment.environment.name` or `deployment.environment` | `environment`       | `resource_attributes."deployment.environment.name"` (fallback `deployment.environment`) | `labels."deployment_environment"` |
| `service.namespace`                                       | `service_namespace` | `resource_attributes."service.namespace"`                                               | `labels."service_namespace"`      |

The `Tracing/Span` column also covers `Tracing/Service` and `Tracing/Deployment`, which use the same bare column names. `Tracing/Span Event` and the OpenTelemetry Logs datasets (`Host Explorer/OpenTelemetry Logs`, `Kubernetes Explorer/OpenTelemetry Logs`) use the `Metrics/OpenTelemetry` resource-attribute forms (`resource_attributes."..."`).

`Tracing/Service Explorer Drilldown Metrics` (performance breakdown) uses `tags."..."` forms and scopes the calling service with a `parent.` prefix (e.g. `tags."parent.service.name"`).

### Step 3: Confirm Representative Traffic

Ask the user whether they have sent representative traffic to the service recently. The service must have processed requests within the last few minutes for validation to work.

If no traffic has been sent:

- Ask the user to send traffic (HTTP requests, gRPC calls, or whatever the service handles)
- If possible and safe, offer to generate representative requests (e.g. `curl` commands against known endpoints)
- Wait until the user confirms traffic was sent before continuing

### Step 4: Wait for Derived Telemetry

Derived datasets and metrics (R.E.D, service discovery, performance breakdown) are computed asynchronously. Wait 30–60 seconds after traffic confirmation before running checks. Inform the user of the wait.

### Step 5: Initialize Validation Report

Initialize a validation report with these ordered checks:

1. APM dataset population and service visibility
2. Service entry points
3. R.E.D metrics
4. Runtime metrics
5. Infrastructure metrics
6. Deployment tracking
7. Performance breakdown
8. Database service detection
9. Instrumentation audit

For each check, mark the result:

- **PASS** — current-session evidence shows the expected data
- **FAIL** — the check returns empty or contradictory results
- **PENDING** — the check cannot be validated yet, usually because derived telemetry may still be materializing
- **SKIP** — the check does not apply to this service or deployment

### Step 6: Run Validation Checks

Execute checks in order. Always complete the full checklist and keep a failure report for every failed or unvalidated check.

#### Check 1: APM Dataset Population and Service Visibility

- Run the `Search Spans` query from [references/search-spans.md](references/search-spans.md) to confirm `Tracing/Span` contains spans for the target service
- Run the `Service List` query from [references/service-list.md](references/service-list.md) to confirm the service appears in `Tracing/Service`
- PASS if both scoped span evidence and service visibility are present
- PENDING if spans are present but service visibility may still be materializing
- FAIL if both checks are empty after representative traffic and the wait period

#### Check 2: Service entry points

Run the `endpoint-spans` query from [references/endpoint-spans.md](references/endpoint-spans.md).

PASS if at least one scoped `Service entry point` or `Async consumer` span appears.

#### Check 3: R.E.D Metrics

Run the `R.E.D Metrics` query requirements from [references/red-metrics.md](references/red-metrics.md).

PASS if the required R.E.D metric families are queryable for the target service and environment.

#### Check 4: Runtime Metrics

Run the `Service List` query from [references/service-list.md](references/service-list.md) to determine `language`, then run the matching `Runtime Metrics` query requirements from [references/runtime-metrics.md](references/runtime-metrics.md).

PASS if at least one expected runtime metric family returns data for the detected language. Mark this check as SKIP for Ruby.

#### Check 5: Infrastructure Metrics

Run the `Infrastructure Metrics` query requirements from [references/infrastructure-metrics.md](references/infrastructure-metrics.md).

PASS if infrastructure metrics exist for the service when host or Kubernetes infrastructure signals are expected.

#### Check 6: Deployment Tracking

Ask the user if `service.version` resource attribute was set for the application.

- If unset, mark this check as SKIP.
- If set, run the checks from [references/deployment-tracking.md](references/deployment-tracking.md). PASS if all checks return data.

#### Check 7: Performance Breakdown

Run the `Performance Breakdown` query from [references/performance-breakdown.md](references/performance-breakdown.md).

PASS if breakdown metric families are queryable for the service.

#### Check 8: Database Service Detection

Run checks from [references/database-service-detection.md](references/database-service-detection.md).

PASS if a database service row returns (`service_type` is `Database`, `database` is `true`) and the database R.E.D check returns data. Mark this check as SKIP when the service has no expected database dependencies.

#### Check 9: Instrumentation Audit

Run the audit workflow from [references/instrumentation-audit.md](references/instrumentation-audit.md). This covers semantic conventions, error span status, exception events, attribute shape, and schema version drift.

- PASS if no `violation` or `warning` level findings exist
- FAIL if any `violation` or `warning` findings exist; append each returned finding to the failure report
- `improvement` and `information` findings are informational notes

### Step 7: Report Results

Present the validation summary as a table. For example:

| Check                             | Result  | Evidence / Notes          |
| --------------------------------- | ------- | ------------------------- |
| Dataset population and visibility | PASS    | ...                       |
| Service entry points              | PASS    | ...                       |
| R.E.D metrics                     | FAIL    | ...                       |
| Runtime metrics                   | PASS    | ...                       |
| Infrastructure metrics            | PASS    | ...                       |
| Deployment tracking               | PENDING | ...                       |
| Performance breakdown             | PASS    | ...                       |
| Database service detection        | SKIP    | No DB dependency          |
| Instrumentation audit             | PASS    | No violations or warnings |

For each FAILED check, append a structured failure report entry that includes:

- What was expected
- What was observed (or not observed)
- Which layer or check failed
- Likely cause
- Suggested next step

For any PENDING checks, retry upon confirmation with the user.

---

## Troubleshooting Flow

Use this flow when the user reports a specific symptom rather than requesting a general validation.

### Step 1: Confirm Observe Tenant (Troubleshooting)

Same as validation Step 1: run `observe auth status`, confirm the CLI is authenticated, and confirm with the user that the current tenant (customer ID / domain) is the one they want to troubleshoot against before running any queries. If unauthenticated or pointed at the wrong tenant, have them run `observe auth login` (optionally `--url <tenant>.observeinc.com`) and re-check.

Once the tenant is confirmed, run the [Ingest Pipeline Pre-Check](references/pipeline-precheck.md) before classifying the symptom. An abnormal agent or ingest signal often _is_ the root cause of a "missing data" report — surface it up front rather than diagnosing it as a service instrumentation gap.

### Step 2: Resolve Target Scope (Troubleshooting)

Same as validation Step 2: gather `deployment.environment.name`, `service.name`, and `service.namespace`, then use the `Scope Mapping` translations for each dataset.

### Step 3: Classify the Symptom

Ask the user to describe the problem. Map to one of these categories:

- **No service visible** — service doesn't appear in Observe at all
- **No spans** — service exists but no trace data
- **Partial coverage** — some endpoints/libraries produce spans, others don't
- **Stale data** — data existed before but stopped appearing
- **Inconsistent telemetry** — data exists but doesn't match expectations

### Step 4: Confirm Traffic

Same as validation Step 3. Many "missing data" issues are simply "no traffic was sent."

### Step 5: Gather Evidence

Run scoped checks across both raw and derived signals:

1. **Service visibility and dataset population**
   - [service-list.md](references/service-list.md)
   - [search-spans.md](references/search-spans.md)
2. **Spans and span events**
   - [endpoint-spans.md](references/endpoint-spans.md)
   - [search-spans.md](references/search-spans.md)
   - [search-span-events.md](references/search-span-events.md)
3. **Shared metric families**
   - [red-metrics.md](references/red-metrics.md)
   - [runtime-metrics.md](references/runtime-metrics.md)
   - [infrastructure-metrics.md](references/infrastructure-metrics.md)
   - [deployment-tracking.md](references/deployment-tracking.md)
   - [performance-breakdown.md](references/performance-breakdown.md)
4. **Service inventory and instrumentation context**
   - [instrumentations-list.md](references/instrumentations-list.md)
   - [service-instances.md](references/service-instances.md)
5. **Database service detection**
   - [database-service-detection.md](references/database-service-detection.md) when database dependencies are expected
6. **Instrumentation audit**
   - [instrumentation-audit.md](references/instrumentation-audit.md) when telemetry exists but its shape or quality looks wrong

### Step 6: Build Failure Report

For each failed or inconsistent presence check, add an entry to the failure report instead of stopping immediately.

Each entry should include:

- The check or layer
- What was expected
- What was observed
- Evidence collected
- Likely cause

### Step 7: Identify Root Cause

Use the earliest failed or contradictory layer to pick the troubleshooting branch:

#### No service AND no spans

Root cause is at the transport/export/collector level:

- Is `OTEL_EXPORTER_OTLP_ENDPOINT` set correctly?
- Is the Observe Agent / collector running and reachable?
- Is `OTEL_SERVICE_NAME` set?
- Is the instrumentation wired into application startup?
- Check collector logs for rejected/dropped spans

#### Spans exist but no service

The `Tracing/Service` dataset materializes from spans. If spans exist but no service:

- Check if the service appeared in a previous time window (materialization delay)
- Verify spans have `service.name` in resource attributes
- Verify at least one span resolves to `Service entry point` or `Async consumer` (the Observe forms of `SERVER` and `CONSUMER`)

#### Runtime metrics missing

Runtime metrics require separate instrumentation:

- Java: runtime metrics are included in the Java agent by default
- Node.js: requires `@opentelemetry/auto-instrumentations-node` (includes runtime metrics)
- Python: check if `opentelemetry-instrumentation-system-metrics` is installed
- .NET: check if `OpenTelemetry.Instrumentation.Runtime` is registered
- Ruby: no standard runtime metrics package

#### Performance breakdown missing but spans exist

- Wait briefly and retry once
- Check whether edge metrics are still materializing
- Check whether spans have `deployment.environment.name`

#### Database service checks missing

- Verify database spans exist
- Verify `db.system` or equivalent, and `peer.db.name` are present on the relevant database spans (`SPAN_KIND_CLIENT`)
- Verify `peer.db.name` is present on the `Metrics/OpenTelemetry` for span metrics
- Check whether the service actually has database client libraries or ORM frameworks compatible with OpenTelemetry auto-instrumentation

#### Partial coverage across endpoints, libraries, or instances

Some telemetry appears, but only for part of the service:

- Use [instrumentations-list.md](references/instrumentations-list.md) to see which instrumentation libraries and versions are present
- Use [service-instances.md](references/service-instances.md) to check whether gaps are isolated to one host, process, pod, or container
- Use [search-spans.md](references/search-spans.md) to compare a healthy operation with one that is missing or malformed

#### Data exists in multiple layers but disagrees

Multiple signals exist but don't correlate:

- Identity mismatch: `service.name` differs between spans and metrics
- Namespace mismatch: `service.namespace` set in one signal but not another
- Environment mismatch: `deployment.environment.name` inconsistent
- Stale derivations: old data in derived datasets, new data hasn't materialized yet

#### Presence and metrics succeed but audit reports violations or warnings

The instrumentation library emits non-conforming telemetry:

- Check the finding's `affected[]` entries, especially `affected[].instrumentationLibraryName` and `affected[].instrumentationLibraryVersion`
- Update the offending library if a newer version is available
- If the library is known-buggy, check for workarounds in the language-specific instrumentation reference

### Step 8: Retry Derived Checks Once

If a failed check depends on asynchronously derived telemetry, wait briefly and retry it once before finalizing the diagnosis.

### Step 9: Report

Present an aggregated troubleshooting report that separates:

1. **Root failures** — the actual broken layer with evidence
2. **Downstream dependent failures** — things that fail because the root layer is broken (don't treat these as separate problems)
3. **Suggested fix** — specific actions the user should take

If a retry after waiting resolves the issue (derived data materialized), report that it was a timing issue and the system is now healthy.
