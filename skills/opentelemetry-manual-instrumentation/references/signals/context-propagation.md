# Context Propagation

Context propagation is how OpenTelemetry knows that one piece of work is the
**child** of another — within a process and across service boundaries. Without it,
spans become orphaned roots.

## What context propagation is

**Context** is an immutable, propagated container that carries the currently
active span (and other values, e.g. baggage). **Propagation** is moving that
context to wherever the next unit of work runs — another function, another async
task, another process.

## Why active context matters for traces

When you start a span, it becomes a child of whatever span is **active in the
current context**. If the active context is lost (e.g. across an async hop), a new
span has no parent and appears as a disconnected root. Keeping the right context
active is what produces correct parent/child trees.

## In-process propagation

Within a process, the SDK uses a context mechanism to keep the active span
available across synchronous calls and, with proper support, across async
callbacks and promises. Most async flows are handled automatically when you start
spans as **active**, but manual re-attachment is sometimes required for custom
async primitives — see the SDK rule for specifics.

## Cross-service propagation

Across services, context travels in **carriers** (typically request headers). The
sender **injects** the current context into the carrier; the receiver **extracts**
it to continue the trace. The standard format is **W3C Trace Context**
(`traceparent` / `tracestate`); baggage uses the `baggage` header.

## Header / carrier injection and extraction (conceptually)

- **Inject:** before sending, write the active context into outgoing carrier
  fields (e.g. request headers, message metadata).
- **Extract:** on receipt, read those carrier fields into a context, then run the
  handler with that context active so new spans become children of the remote
  span.

Automatic instrumentation does this for supported HTTP/messaging clients. You do
it manually for unsupported transports.

## When manual propagation may be needed

Manual injection/extraction or explicit context re-attachment is commonly required
for:

- custom protocols
- message queues
- background jobs
- worker pools
- event emitters
- callbacks
- libraries without automatic instrumentation
- cross-process work dispatch

## SDK specifics

This file is **language-neutral**. For how to capture, attach,
inject, and extract context in a given language — including async-context
mechanisms — see the relevant `../sdks/<lang>/` file.
