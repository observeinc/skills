# Traces

Spans are the most common manual instrumentation. Good spans are **useful, stable,
and safe**.

## Span naming principles

- Name the **class of operation** with a route template, operation name, or method
  name (`GET /orders/{id}`) — a small, stable set of values, so the name stays
  **low-cardinality**.
- Match semantic-convention naming where one exists (HTTP, DB, messaging, RPC).

## Span kind principles

Set the kind that reflects the span's role:

- `SERVER` — handling an inbound request
- `CLIENT` — making an outbound request
- `PRODUCER` — enqueuing/publishing a message
- `CONSUMER` — processing a received message
- `INTERNAL` — in-process work (the default for most domain spans)

Kind affects how backends build service topology and interpret latency. Re-derive it
whenever you add, wrap, move, or edit a span.

## Parent/child relationships

- Child spans attach to the **active span** in the current context. Start spans as
  active so nested work is captured automatically.
- Across async boundaries, the active context must be carried explicitly — see
  `context-propagation.md`.

## Span attributes

- Describe **the operation**: what kind, against what resource, with what result.
- **Prefer semantic conventions** before inventing custom attributes.
- Keep attributes **low-cardinality** and free of PII/secrets.
- Custom business attributes use a bounded namespace (e.g. `app.*`, `business.*`)
  and must be documented.
- Keep any user identifier in a controlled, justified place, off routine spans.

## Span events

A span event is a timestamped annotation inside a span (e.g. "cache miss", "retry
scheduled", "validation failed"). Prefer a **span event** over a **new span** when
the moment:

- is instantaneous or very short,
- does not need its own duration/timing, and
- is a notable point _within_ an existing operation.

Prefer a **child span** when the work has a meaningful duration worth measuring and
its own attributes/status. Exception recording (below) is a specialized event.

**Favor span events for anything APM reads.** OTel is moving general events to the
log-based Events API, but APM features (exceptions, error tracking) still read span
events. Keep exceptions and APM-consumed signals on span events until that changes.

## Span links

Use links to reference related spans that are **not** in a direct parent/child
chain — for example, a batch-processing span that links to each message that
triggered it, or a span continuing work started in a different trace.

## Per-signal naming formats

Match the semantic-convention name shape for the operation:

| Signal          | Span name                                                                                         |
| --------------- | ------------------------------------------------------------------------------------------------- |
| HTTP **server** | `{method} {route}` (low-cardinality route template)                                               |
| HTTP **client** | `{method}` — or `{method} {url.template}` only when a low-cardinality template exists             |
| Database        | `{db.operation} {db.collection}` (fallbacks: `db.query.summary` → `{collection}` → `{db.system}`) |
| RPC             | `{rpc.service}/{rpc.method}`                                                                      |
| Messaging       | `{operation} {destination}`                                                                       |

### HTTP client span names

A client span named just `GET`/`POST` is **correct** — the OTel HTTP convention is
`{method}`, or `{method} {url.template}` when a low-cardinality template exists
(usually it doesn't on clients).

- ✅ `POST` — compliant when no template is known.
- ✅ `POST /v1/meta/graphql` — only if that path is a stable low-cardinality template.
- ❌ `POST https://host/v1/things/abc-123?x=…` — high-cardinality.

To make client spans friendlier without breaking cardinality, set the `url.template`
attribute, or rename using the low-cardinality `server.address`
(e.g. `${method} ${server.address}`).

## Span status (default UNSET) — client vs server differ

Status is **UNSET** by default. Set **ERROR** only on a genuine failure; set **OK**
only when a success must be explicitly asserted. The same HTTP status maps
differently by span kind:

| Situation                       | CLIENT span                                 | SERVER span                                                              |
| ------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------ |
| 4xx response                    | **ERROR** (the call the client made failed) | **UNSET** (the server correctly rejected a bad request — it did its job) |
| 5xx response                    | **ERROR**                                   | **ERROR**                                                                |
| No response / transport failure | **ERROR**                                   | **ERROR**                                                                |

An ERROR status message carries the **error class + a short explanation**; put stack
traces in a recorded exception or log. For a handled or retried error, record each
failed attempt as a span event and set ERROR only once retries are exhausted.

## Root and parent hygiene

- A **root span must not be `CLIENT` or `PRODUCER`.** Roots should be `SERVER`,
  `CONSUMER`, or `INTERNAL`. A `CLIENT`/`PRODUCER` root usually means missing
  inbound instrumentation, broken context propagation, or a headless operation with
  no wrapping span.
- **Headless operations** (cron jobs, CLIs, workers, queue consumers) have no
  inbound request, so the first outbound call would otherwise become a `CLIENT`
  root. Wrap the whole operation in a manual `SERVER` (or `CONSUMER`) span so the
  trace has a proper root.
- **`CLIENT`/`PRODUCER` spans must have a parent** — an outbound call span should be
  a child of a `SERVER`/`CONSUMER`/`INTERNAL` span, so the trace shows _why_ the
  call was made.
- **One trace per independent operation.** Each independent top-level request starts its
  own root span and trace id; only work causally within one operation shares a trace. A
  driver that makes several independent calls (a `search`, then a `result` lookup, then
  another) opens a new root span per call, not one span in `main()` wrapping them all. That
  gives one trace per call (N calls, N trace ids), each linking its own client and server
  spans through propagation.

## Span-count hygiene

- **No orphan spans** — every non-root span's parent must exist in the trace.
- Keep **≤ ~10 `INTERNAL` spans** per service per trace; don't span trivial helpers.
- Avoid **> ~20 sub-5ms spans** per trace — they add noise and cost without insight.
- Instrumenting a loop over N items → prefer **one span with a `batch.size` attribute**
  over N tiny spans.

## Error handling

On failure:

1. **Record the exception** with `recordException` on the active span, capturing
   `exception.type`/`.message`/`.stacktrace`. APM error tracking reads this
   span-event format, so prefer it over a log-based event; a log-only exception may
   not surface in APM error views. Messages can carry user input, so mind privacy
   (see `../review/cardinality.md`). Record it once, not at every catch layer.
2. **Set the span status to error** (with a short class + explanation, per above).
3. **Rethrow** the error unless it is intentionally handled — recording an error
   and then swallowing it hides real failures and changes control flow.

## Always end spans reliably

End every span on every path, including errors and early returns. Use a `finally`
block or the SDK's scope-managed active-span helper. A span that is never ended
leaks and never exports.
