# Validation and Debugging

Pre-flight delivery checks and the failure-classified **debug loop** for emitted
telemetry. Semantic-convention conformance is a review-gate concern (semantic review).

Verdict vocabulary and proof-strength rules: [`final-report-format.md`](final-report-format.md).

## Pre-flight checks (before trusting "no telemetry")

When a signal seems missing, confirm the delivery basics before concluding it's an
instrumentation bug:

- `OTEL_SERVICE_NAME` (or a `service.name` resource attr) is set — else it shows as
  `unknown_service`.
- The OTLP endpoint is set and reachable (e.g. `curl -v <endpoint>/v1/traces`).
- The exporter **protocol matches the receiver** (gRPC↔4317, http/protobuf↔4318) —
  a mismatch drops data silently (see **OTLP delivery reference** below).
- If a Collector is in the path, it's **accepting** (`otelcol_receiver_accepted_*`
  > 0) and **exporting** (`otelcol_exporter_sent_* > 0`, `send_failed_* == 0`).
- Each signal is exercised **separately** — a working trace export does not prove
  metrics or logs are delivered.

## OTLP delivery reference

Prefer **OTLP**; configure it through environment variables so instrumentation stays
portable, and read endpoints, tokens, and headers from the environment rather than
source. Delivery is a **(protocol, endpoint, path)** tuple that must match the receiver
**per signal** — traces, metrics, and logs each resolve their own exporter:

- **gRPC** → default port **4317**; `OTEL_EXPORTER_OTLP_PROTOCOL=grpc`.
- **HTTP/protobuf** → default port **4318**, path `/v1/{traces,metrics,logs}`;
  `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`.
- A protocol/receiver mismatch (e.g. `http/protobuf` aimed at a gRPC 4317 port) makes
  export **fail silently** or with a parse error.
- Set each exporter explicitly — `OTEL_TRACES_EXPORTER`, `OTEL_METRICS_EXPORTER`,
  `OTEL_LOGS_EXPORTER` = `otlp`. Some manual and older SDK setups default a signal to
  **`none`**, dropping it with no error.

Keep the SDK sampler at **`AlwaysOn`** and defer sampling to the Collector. In-app head
sampling discards traces before the request outcome is known and skews span-derived RED
metrics.

## Conformance vs coverage

Semantic review (the review gate — `../review/semantic-review.md`) validates the
**conformance** of telemetry that is **actually emitted**; a **missing** signal isn't
a violation (e.g. an opt-in metric like `http.client.response.body.size` you wanted but
never enabled). Confirm **coverage** separately: if an expected opt-in signal is
missing, enable it or record it manually. Record coverage gaps in the final report.

## Debug loop

Classify the failure **before** fixing — match it to one of the classes below
(use the failing command and stage as context), then apply that row's remediation:

Failure classes and targeted remediation:

| Class                                                      | Remediation summary                                                                                                                                                                                                                      |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `environment_detection_failure` / `scope_analysis_failure` | Re-run detection (confirm language/runtime, don't fabricate); respect user scope, else pick a high-value/low-risk/testable candidate.                                                                                                    |
| `dependency_failure`                                       | Install the required OpenTelemetry packages with the detected package manager (**ask first** — installing deps / modifying lockfiles mutates the project; propose the exact command and get consent); match versions; report if blocked. |
| `build_failure` / `typecheck_failure`                      | Fix the first error; check the bootstrap matches the module format and API types resolve at call sites; rerun.                                                                                                                           |
| `unit_test_failure`                                        | Separate new-telemetry vs pre-existing; if behavior broke, revert the behavioral change.                                                                                                                                                 |
| `telemetry_unit_test_failure`                              | Check provider/exporter setup, span lifecycle, async context, that fn executed, error-path asserts.                                                                                                                                      |
| `app_start_failure`                                        | Don't block whole flow; confirm bootstrap loads first; mark runtime telemetry `Blocked`.                                                                                                                                                 |
| `collection_health_failure`                                | Delegate to debug-k8s/-linux-host skill.                                                                                                                                                                                                 |
| `validation_failure`                                       | Semantic review found nonconforming fields; fix actionable semconv/attrs/naming/units per `../review/semantic-conventions.md`.                                                                                                           |
| `backend_query_failure`                                    | Check CLI auth/tenant/syntax/time window and that the path was exercised; report `Failed` vs `Blocked`. Only relevant when the user takes the optional live check.                                                                       |

**Bounded loop:** apply at most ~3 targeted fixes per failing stage, re-running the
specific failed command each time. If still failing, stop and report the remaining
blocker clearly.

## Honesty rules for this stage

- "Passed" is only for commands/skills that actually succeeded.
- A missing or unstartable app → runtime telemetry is **inconclusive**, not failed.
- Collection-health and validation outcomes are reported exactly as their skills
  returned them.
