# Telemetry Unit Tests

Telemetry unit tests are **required** for newly added manual instrumentation; only
tests that assert telemetry behavior count. They verify emission **locally**, with the
Observe backend checked separately. The language- and test-framework-specific harness
(in-memory exporter setup, provider construction, common pitfalls) lives in
`../sdks/<lang>/`.

## Test the app's emission

Prove the **application** emits the telemetry. A test that constructs an SDK span,
metric, or log itself and then asserts on it proves nothing about the instrumented
code; it tests the SDK. Every test must call into the real instrumented code path
(function, route handler, middleware, worker, startup hook) with fake inputs, then
assert on what that code emitted.

Assert the full shape the code promised: name, attributes, parentage, status, and
(where added) metric datapoints and log fields.

## Prefer stronger proof

In order, strongest first:

1. An existing app test that already exercises the code path; add telemetry
   assertions to it.
2. A new focused unit test that calls the instrumented function/handler with fakes.
3. A new integration test that boots enough of the app to exercise real wiring.
4. A temporary in-line harness that runs app code, useful before deciding to persist.
5. A synthetic SDK-only emission. Use only for exporter/schema smoke; label it as not
   app-code proof.

## Cover paths, not just names

Instrumentation shapes differ by path: success, error, empty result, timeout,
streaming, retry. Cover each **distinct** path, and write one test per distinct
emitted span name; a test of a shared helper does not prove each caller's span. One
aggregate test that emits every signal does not prove per-path coverage.

Split a path into its own test only when the name, parentage, status, events,
attributes, metric datapoints, or log shape differ; micro-branches emitting identical
telemetry share one test.

## Test shape

- Use the repo's existing test framework. Match the installed OpenTelemetry version.
- Register the provider **before** the code under test resolves its tracer/meter/logger,
  or the exporter captures **0 signals** (a common, language-specific pitfall; see
  `../sdks/<lang>/`).
- Use an **in-memory / test exporter** (span processor → in-memory exporter, plus the
  metric-reader and log equivalents). A permanent unit test must pass without a
  collector; add OTLP export only as a paired check when explorer-visible proof is also
  requested.
- **Fake external dependencies**: network, databases, queues, LLMs, credentials.
- **Reset provider/exporter state** between tests (clear the exporter; shut down or
  replace the provider in teardown) to avoid cross-test contamination.

## What to assert

Per signal, where applicable:

- **Spans**: the instrumented code executed; span created with the exact, stable,
  low-cardinality name (no IDs/PII); expected parentage; required/semantic attributes
  present with correct values; error paths record the exception and set status = ERROR;
  span started and ended (not leaked).
- **Metrics** (only if metrics were added): instrument name, type, and unit; datapoint
  value/count; bounded attributes/dimensions.
- **Logs** (only if logs were added): body and severity; `trace_id`/`span_id`
  correlation when emitted inside a span; sensitive fields redacted.
- **Resources**: `service.name`, environment, and version resolve as expected.
- **Behavior**: existing return values and side effects are unchanged.

## Before you call a test `Blocked`

`Blocked` marks a test that cannot run in isolation. Don't reach for it because a
test is harder than your first attempt, and don't escalate to a bigger requirement
(e.g. "needs a full DB integration test") to justify stopping. Before claiming
`Blocked`, rule out each of these in one line — most "blocked" telemetry tests fall to
one of them:

- **Synthesize the parent, not the span under test.** To prove parentage, start the
  real parent span in the test and call the real instrumented code with that context —
  you do **not** need to drive the whole outer entrypoint. (Building the parent span is
  a valid test; hand-building the span you then assert on is not.)
- **Call the inner function directly.** If the entrypoint is unreachable in isolation
  but the instrumented function isn't, test the function.
- **Use an existing fake.** Look for a repo in-memory/fake (fake DB, fake client,
  seedable fixture) before concluding the path needs real infra.
- **Add a minimal testability seam.** A small, behavior-preserving change made to
  enable the test — resolving the tracer/meter lazily, extracting a testable inner
  function, an injectable provider — is in scope (see `instrumentation-rules.md`
  golden rule 5) and preferred over `Blocked`. Fix testability at the source rather
  than building a heavier test-only harness around it.

Only after ruling these out, record it in the final report rather than skipping
silently. State:

- exactly **why** (which of the alternatives above don't apply, and the hard
  dependency that remains — e.g. the success path spawns a live goroutine needing real
  infra),
- **what was attempted**,
- the **closest safe coverage** achieved (e.g. the error branch reachable without
  wiring, or asserting parentage via a synthesized parent span),
- and label it a **serious limitation** — `Blocked` (see `final-report-format.md`).

A `Blocked` cell whose receipt doesn't list the ruled-out alternatives reads as
`Failed`, not a legitimate block.
