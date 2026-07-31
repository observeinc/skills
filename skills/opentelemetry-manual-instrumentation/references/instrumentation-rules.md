# Manual Instrumentation Rules

The canonical, **language-neutral** rules for editing code to add manual
instrumentation. Keep changes **bounded, idiomatic, and side-effect-free**.
For idiomatic code, bootstrap, and the test harness, see the language unit
under `sdks/<lang>/`.

## Golden rules

1. **Inspect first.** Use the environment checkpoint. Reuse what exists.
2. **No duplicate SDK init.** If an SDK is already initialized, never add another.
3. **Preserve existing instrumentation** — vendor (e.g. `dd-trace`) and OTel
   auto-instrumentation stay. Do not replace unless explicitly required.
4. **Use the API, not the SDK, in business logic.** Emit through the OpenTelemetry
   API; keep SDK wiring in the bootstrap (see `app-vs-library.md`).
5. **Bounded, idiomatic change.** Touch only what the instrumentation goal
   requires — no unrelated refactors, formatting churn, or renames — and write it
   the way the code's owners would, matching existing structure and reusing existing
   patterns. Prefer reuse over a new abstraction; add a helper or module only when it
   removes real duplication or the app-vs-library split needs it. Between two
   idiomatic placements, pick the one at the architectural seam the telemetry's
   intent points to. If a consequential structural choice stays ambiguous (a new
   abstraction or module, or placements that answer the intent differently), present
   the options and ask the user. A minimal, behavior-preserving change made to make the
   telemetry testable — resolving a tracer/meter lazily, extracting a testable inner
   function, or an injectable provider — counts as bounded and is preferred over
   labeling a telemetry test `Blocked`.
6. **Telemetry must not change behavior.** No swallowed errors, no control-flow
   changes, no added latency in hot paths.

## Application vs library

Application owns the pipeline; a library depends on the OpenTelemetry API only. Full
rule: `app-vs-library.md`. SDK-side specifics below.

## SDK wiring rules (applications)

- **One global provider per signal.** Don't initialize the SDK twice in a process.
  Find the existing setup (including lazy providers created on first instrument use)
  and **extend** it rather than adding a second — never let auto-instrumentation and
  app code race to set the provider.
- **Preserve operator/environment resource values.** The merge rule (app defaults
  fill only absent keys) and its test live in `discovery/resolve-values.md`.
- **HTTP server instrumentation must emit the request-duration metric.** Wrap the
  outermost handler so `http.server.request.duration` (stable semconv) is emitted
  alongside spans. If the SDK needs a semconv-stability opt-in, set it in the launch
  environment before the instrumentation initializes.
- **Fast metric export for short-lived runs.** Default metric export intervals
  (~60s) are too slow for CLIs, jobs, and eval runs — metrics never flush before the
  process exits, producing false "no metrics" results. Shorten the export interval
  (and keep the export timeout ≤ the interval). Concrete values are language-specific
  — see `sdks/<lang>/`.

## Enriching auto-instrumentation (don't duplicate)

When a path is **already** covered by auto-instrumentation (HTTP, graphql, web
framework), enrich the existing span rather than wrapping it in a redundant manual
span, deciding in this order:

1. **Instrumentation hook** — add attributes / rename on the auto-created span via
   the instrumentation's hook options (configured where the SDK is initialized).
2. **A custom span processor** — normalize names or stamp attributes uniformly
   across many spans.
3. **A manual child span** — only for sub-operations auto-instrumentation cannot
   see (domain steps, custom protocols) that have meaningful duration.

The hook / span-processor APIs are language-specific — see `sdks/<lang>/`.

## Logs vs spans (when a log should be a span/event)

A log line that marks the **start/end of an operation**, carries a hand-rolled
correlation id (`[${transactionId}] …`), or reports a duration is usually better as
a span (or a span event on an existing span):

- The operation has meaningful duration / can fail → **span** (or child span);
  trace/span IDs replace the hand-rolled correlation id.
- A notable moment inside an operation → **span event** on the active span.
- A genuine discrete record with no operation boundary (audit line, config dump) →
  keep it a **log**, and rely on log↔trace correlation rather than manual ids.

Keep useful logs: enrich the active span or add a child span, and drop a
now-redundant manual correlation id only once spans carry it natively. Leave the
existing logging framework intact.

## Patch checkpoint (produce this)

```text
Files changed:
Dependencies added/changed:
SDK setup changed:
Manual spans added:
Metrics/logs added:
Existing instrumentation preserved:
Startup script changes:
Risky changes:
Reasoning:
```
