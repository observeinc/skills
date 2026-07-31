# Logs

Manual log instrumentation is about **shape and delivery**. (Log API maturity
varies by SDK — see `telemetry-signals.md`.)

## Structured, key-value, one record per line

- Emit **structured** logs (key/value), never string interpolation
  (`"processed order " + id`). Structure is what makes logs queryable.
- **Pick fields explicitly.** Never spread a whole request/response/user object
  into a log — you leak data and explode field cardinality.
- **One record per line.** A multi-line record (e.g. a raw stack trace) is split by
  line-based collectors (filelog, Fluentd) into unrelated records. Serialize any
  stack trace as a **single string** with escaped newlines in `exception.stacktrace`
  (or `error.stack_trace`).

## Severity

Always set a **severity** (number + text). An unset severity loses filtering and
alerting. Access/audit logs are typically `INFO` (severity number 9). Note that
some collector receivers don't parse severity from the body by default — set it
explicitly on the record.

## Log events vs regular records

A **log event** (a record with a stable `event_name` / `otel.event.name`) is for a
**business or operational milestone with a stable schema** — deploys, payments,
signups. Use a regular log record for diagnostic, variable-attribute logging. Don't
turn every log line into an event.

Keep exceptions off log events. APM error tracking reads the span-event format, so
record exceptions with `recordException` on the active span (see `traces.md`).

## Trace correlation

`trace_id` and `span_id` are **LogRecord data-model fields** in the OTLP proto.
Backends read trace-log correlation from those fields, so an attribute named
`trace_id` breaks it.

Two paths:

- **OTLP-native** (OTel logs SDK or a log-appender bridge): the bridge reads the
  active span context and sets the record fields automatically.
- **Legacy logger on stdout/file**: emit them in the JSON payload, but the collector
  must **map** them from the JSON fields into the LogRecord's `trace_id`/`span_id`
  fields. Without that mapping they land as plain string attributes and OOTB
  correlation breaks. This is a collector configuration change; surface it to the
  user rather than assuming it is already in place. If the user owns the logging
  stack, offer to migrate to an OTel-native library or appender bridge. That drops
  the collector mapping entirely.

Confirm which path the logging stack uses before adding trace correlation. See
`../sdks/<lang>/` for the language-specific mechanism.

## Log ↔ span decision

A log that marks the start/end of an operation, carries a hand-rolled correlation
id, or reports a duration is usually better as a **span** (or a span event). Keep a
genuine discrete record (audit line, config dump) as a log and rely on trace
correlation. See `../instrumentation-rules.md` (logs vs spans).

## Delivery: stdout vs OTLP

How logs reach the backend is a real decision with cost and coverage tradeoffs:

| Approach                         | Pros                                                                                                                          | Cons                                                                                                                                      |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Stdout/stderr only** (default) | Captures **all** logs incl. bootstrap, crash, and third-party library output; `kubectl logs` works; a collector forwards them | Needs a collector/agent to forward                                                                                                        |
| **OTLP only**                    | Native OTel log records straight to the pipeline                                                                              | Bypasses the runtime log path — **loses bootstrap/crash/library logs**, breaks `kubectl logs`, and an endpoint outage silently drops logs |
| **Both**                         | —                                                                                                                             | **Duplicate** records and **double cost**; needs dedup                                                                                    |

Default to **structured single-line JSON on stdout/stderr** and let a collector
forward it. Don't enable both paths without deliberate dedup.

## Stdout trace-context is NOT OTLP log export

Injecting `trace_id`/`span_id` (or MDC/context fields) into **stdout** logs adds
**correlation** only; OTLP log export is a separate capability. Adding an OTLP log
bridge changes cost and privacy and can duplicate ingestion, so make it an explicit
choice rather than a silent add-on to "send logs to OpenTelemetry." If OTLP log
export was requested but isn't configured, say so plainly — the capability isn't
present (a scope note, not a `Failed` verification).

## Privacy

Log PII/redaction rules — including that a **formatter**, adapter, MDC, framework
**access log**, or **exception renderer** can re-add an ID you stripped from a
`logger.*` call, so review the final rendered output — live in
`../review/sensitive-data.md`.
