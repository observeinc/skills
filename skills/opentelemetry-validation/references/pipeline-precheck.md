# Ingest Pipeline Pre-Check

Run before any validation or troubleshooting check. Confirms three things: the Observe Agent fleet is alive, the ingest pipeline is receiving observations, and the downstream datasets are materialized. A "no data" result from a validation check can mean any of these three are broken. This pre-check distinguishes them.

Run immediately after confirming the Observe tenant and before service scope, traffic, and the numbered checks. Not scoped to a single service.

Pre-requisites:

- Observe CLI, authenticated to the intended tenant

## Check A: Observe Agent Health

Confirms the Observe Agent / OpenTelemetry collector fleet is alive and authenticating, so observations can be sent to the backend.

Data source: `Observe Agent/Events` (source event dataset)

1. Find the events dataset ID. `--label` is a fuzzy match; pick the row with label exactly `Observe Agent/Events`:

```bash
observe dataset list --label "Observe Agent" --json
```

2. Query per-agent status:

```bash
observe query --input <events-dataset-id> --interval 1h --json --pipeline '
filter kind = "AgentLifecycleEvent" or kind = "AuthErrorEvent"
make_col agent:string(coalesce(identifiers."host.name", identifiers."k8s.daemonset.name", identifiers."k8s.deployment.name")), version:string(facets."observe.agent.version"), auth_passed:parse_json(data).authCheck.passed
make_col status:case(
  control.eventType = "SHUTDOWN", "Shutdown",
  kind = "AuthErrorEvent" or auth_passed = false, "Error",
  true, "Healthy")
statsby status:last_not_null(status), last_seen:max(valid_from), version:last_not_null(version), group_by(agent)
make_col last_seen:format_time(last_seen, "YYYY-MM-DD HH24:MI:SS")
sort desc(last_seen)'
```

Interpretation:

- **Healthy**: agent is emitting lifecycle events and passing its auth check.
- **Error**: `AuthErrorEvent` or `authCheck.passed = false`. The agent cannot authenticate; observations are rejected. **Flag.**
- **Shutdown**: agent reported a `SHUTDOWN` event and has not come back. **Flag** if it covers the host/cluster under validation.
- **No rows**: no agent reported in the last hour. Nothing is collecting. **Flag.**

If the service exports OTLP directly to Observe without an Observe Agent, skip this check and rely on Check B.

## Check B: Ingest Pipeline Health

Confirms raw observations are arriving at the backend for each signal. `stats.lastIngest` on a datastream is updated at arrival time, before dataset materialization.

| Signal  | Datastream name                                                                                      |
| ------- | ---------------------------------------------------------------------------------------------------- |
| Traces  | `Tracing/Span`                                                                                       |
| Metrics | `Metrics/OpenTelemetry`                                                                              |
| Logs    | `Kubernetes Explorer/OpenTelemetry Logs` or `Host Explorer/OpenTelemetry Logs` (per deployment type) |

Check only the signals the target service is expected to emit.

1. List datastreams to find IDs:

```bash
observe datastream list
```

2. View each relevant datastream:

```bash
observe datastream view <datastream-id>
```

Interpretation of `stats`:

- `lastIngest` within the last few minutes: data is arriving. **OK.**
- `lastIngest` stale: no recent data is reaching the backend for this signal. **Flag.**
- `disabled: true`: the datastream is paused; no data can arrive. **Flag.**

## Check C: Dataset Materialization Lag

The numbered validation checks never read the raw datastreams. They query APM datasets that Observe materializes from those streams asynchronously. A span can be fully ingested (Check B OK) yet not appear in `Tracing/Service` for another minute or two while materialization catches up. During that window a healthy service looks empty, so measure the lag before the checks run.

Read `accelerationInfo.stalenessSeconds` from `observe dataset view` — the number of seconds between now and the newest data the dataset can return. Only _derived_ datasets report it:

- **Source** datasets (raw ingest, `alwaysAccelerated: true`) report `null`. The write is the materialization, so there is no lag — their freshness is already covered by Check B.
- **Views** such as `Tracing/Span` report `null`. They have no materialization of their own and read through to their source; that freshness is also Check B.
- **Derived** datasets run OPAL transforms on a schedule and can fall behind. These are the only ones worth checking here.

Check the derived datasets the validation flow depends on:

| Dataset                                      | Feeds check                              |
| -------------------------------------------- | ---------------------------------------- |
| `Tracing/Service`                            | Service visibility (1), DB detection (8) |
| `Tracing/Deployment`                         | Deployment tracking (6)                  |
| `Tracing/Service Explorer Drilldown Metrics` | Performance breakdown (7)                |

Check only the datasets whose checks you intend to run.

1. Find dataset IDs:

```bash
observe dataset list --label "<dataset name>" --json
```

2. View each dataset and read `accelerationInfo.stalenessSeconds` (compare it against `effectiveTargetStalenessSeconds`, the dataset's own lag target):

```bash
observe dataset view <dataset-id>
```

Interpretation:

- `stalenessSeconds` at or under its target (typically a few minutes): materialization is current. **OK.**
- `stalenessSeconds` well above target and climbing: the dataset is behind. A check reading it returns empty even though spans are arriving. **Flag** — expect to wait and retry once it catches up.
- `stalenessSeconds: null` on a dataset you expected to be derived: acceleration is off. Note it and rely on Check B.

## Pre-Check Verdict

Report a short pre-check summary before starting validation:

| Check                            | Result | Evidence                                                              |
| -------------------------------- | ------ | --------------------------------------------------------------------- |
| Agent health                     | OK     | N agents Healthy, latest seen \<timestamp\>                           |
| Ingest — traces                  | OK     | lastIngest 12s ago (Tracing/Span)                                     |
| Ingest — metrics                 | FLAG   | lastIngest 3h ago (Metrics/OpenTelemetry)                             |
| Materialization — service        | OK     | stalenessSeconds 62, target 240 (Tracing/Service)                     |
| Materialization — perf breakdown | FLAG   | stalenessSeconds 900, target 240 (Tracing/Service Explorer Drilldown) |

- **All OK**: proceed to the validation/troubleshooting flow.
- **Any FLAG**: surface it first. State which layer is broken (agent, ingest, or materialization), explain that it will cause "no data" downstream, and confirm with the user before continuing.
