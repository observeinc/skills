# Anti-Patterns

These are the recurring mistakes that make manual instrumentation wrong,
expensive, unsafe, or misleading. Reject them in review and avoid them when
writing code.

## Setup and ownership

- **Initializing SDKs inside reusable libraries.** Libraries must use the API only
  and let the host application own the pipeline. (See `app-vs-library.md`.)
- **Initializing telemetry after application modules have already been imported.**
  Bootstrap must run **first**, or instrumentation may be missing or no-op.
- **Double-initializing SDKs.** One SDK init per application process; competing
  inits cause conflicting/duplicate configuration.
- **Hardcoding vendor-specific endpoints or credentials.** Use OTLP and `OTEL_*`
  environment variables; never bake in endpoints, tokens, or headers.

## Spans

- **Creating a span around every tiny function.** Produces noise and overhead;
  instrument meaningful operations, not trivial calls.
- **Forgetting to end spans.** Unended spans leak and never export — always end in
  `finally` or via a scope-managed helper.
- **Swallowing exceptions after recording them.** Recording an error then
  suppressing it hides real failures. Rethrow unless intentionally handled.
- **Putting per-request data into resource attributes.** Resources are
  entity-level; per-request data belongs on spans.

## Metrics

- **Creating metric instruments inside request handlers / hot paths.** Create them
  once at startup and reuse.
- **Using user IDs or request IDs (or any unbounded value) as metric labels.**
  Causes cardinality explosions.
- **Using full URLs with IDs instead of route templates.** Same cardinality
  problem; use the template.

## Logs

- **Emitting `trace_id`/`span_id` as log attributes.** These are dedicated fields in
  the OTLP LogRecord proto. An attribute named `trace_id` does not satisfy OOTB
  trace-log correlation. On an OTLP-native path (OTel logs SDK / appender bridge),
  the bridge sets them from context automatically. On a legacy stdout path, the
  collector must map them to the record fields.

## Data safety

- **Recording raw PII or secrets.** Never record bodies, tokens, cookies, auth
  headers, or emails.

## Conventions and correctness

- **Creating custom attributes when official semantic conventions already exist.**
  Prefer the convention; reserve custom fields (namespaced, documented) for
  genuinely project-specific concepts.

## Process and behavior

- **Copying examples without adapting to the project's module system.** A CJS
  bootstrap pasted into an ESM project (or vice versa) silently fails to
  instrument. Match the project's module system and startup path.
- **Adding telemetry that changes application behavior.** Instrumentation must be
  side-effect-free: no control-flow changes, no swallowed errors, no unacceptable
  overhead.
- **Treating successful export as proof of good instrumentation.** Successful
  export proves only delivery; validate the telemetry's correctness, boundedness,
  and privacy (see `verification/validation-and-debugging.md`).
