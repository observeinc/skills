# Fundamentals of Manual Instrumentation

## What manual instrumentation is

Manual instrumentation is telemetry you write **by hand** using the OpenTelemetry
API: you create spans, record metrics, set attributes and events, and propagate
context explicitly in your own code. It contrasts with **automatic / zero-code
instrumentation**, where agents or instrumentation libraries wrap known
frameworks and clients for you without source changes.

## When manual instrumentation is useful

Manual instrumentation is worth adding where automatic instrumentation cannot see
**business meaning** or **custom code paths**. Reach for it to capture:

- meaningful business operations (e.g. "checkout", "reconcile invoice")
- custom protocols not covered by an instrumentation library
- background jobs and scheduled tasks
- queue/message handlers
- expensive operations worth timing and counting
- important async boundaries where context would otherwise be lost
- external calls not covered by automatic instrumentation
- critical domain workflows you must be able to detect, localize, and explain

## How it complements automatic instrumentation

Automatic instrumentation gives broad coverage of HTTP servers/clients, database
drivers, messaging clients, and runtimes with little effort. Manual
instrumentation **fills the gaps** and **adds domain context**: it enriches
auto-created spans with business attributes, adds child spans for internal steps,
and measures things only your application understands. Let automatic instrumentation
handle the plumbing; add manual spans only for what it cannot infer, and enrich its
spans rather than re-wrapping them.

## What should not be manually instrumented

- every tiny function
- trivial getters/setters
- hot loops, unless carefully sampled, aggregated, or otherwise justified
- code paths where telemetry would change behavior or add unacceptable overhead

Over-instrumentation creates noise, cost, and cardinality risk, and can obscure
the signal you actually need.

## General lifecycle of a span

1. Acquire a tracer from the API (scoped by instrumentation name and version).
2. Start a span, ideally as the **active** span so children attach automatically.
3. Set semantic attributes describing the operation.
4. Add events for notable moments; add links for related-but-not-parent work.
5. On error, record the exception and set the span status to error.
6. **End the span reliably** (e.g. in a `finally` block) — always, on every path.

## General lifecycle of a metric

1. Acquire a meter from the API (scoped by instrumentation name and version).
2. Create instruments **once** (counter, up-down counter, histogram, observable
   gauge / observable instruments) with a name, unit, and description.
3. Record measurements with **bounded, low-cardinality** attributes.
4. For observable instruments, register a callback that reports the current value.

## Relationship between traces, metrics, logs, and resources

- **Traces** explain _what happened in one request/operation_ and _where time
  went_ across services.
- **Metrics** aggregate _how the system behaves over time_ (rates, durations,
  counts) cheaply and continuously.
- **Logs/events** capture _discrete records_, ideally correlated to traces.
- **Resources** describe _the entity producing the telemetry_ (the service /
  process / host) and are attached to all signals from that entity.

The four are complementary: metrics detect a problem, traces localize it, logs
explain it, and resources tell you which entity it came from.

## Signal density principle

Every signal you add should help you **detect**, **localize**, **explain**, or
**validate** system behavior. Drop any proposed span, metric, or attribute that
does none of those.
