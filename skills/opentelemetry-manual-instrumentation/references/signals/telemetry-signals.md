# Choosing a Signal

OpenTelemetry defines several signals — **traces**, **metrics**, **logs** (plus
**resources**, which describe the producing entity). Manual instrumentation can touch
all of them; this file maps a need to a signal. For the mechanics of each, see
`traces.md`, `metrics.md`, `logs.md`, and `../discovery/resolve-values.md`.

## The primary axis: debug vs aggregate

How will the telemetry be used — inspect one operation (→ span/log, high-cardinality
OK) or aggregate over time (→ metric, bounded and queryable)? See
`../discovery/intent.md` for the full treatment; the table below maps a need to a
signal.

## Selection table

| You need to…                                                  | Use                    | Notes                                                                 |
| ------------------------------------------------------------- | ---------------------- | --------------------------------------------------------------------- |
| Time an operation, see where time went, trace across services | **span**               | The primary signal; set kind + status (`traces.md`)                   |
| Mark a notable moment _inside_ an operation (no own duration) | **span event**         | Cheaper than a child span (`traces.md`)                               |
| Relate work not in a parent/child chain (batch fan-in)        | **span link**          | (`traces.md`)                                                         |
| Count / rate / level / distribution over time, cheaply        | **metric**             | Pick the instrument by shape (`metrics.md`)                           |
| Record a discrete diagnostic or audit record                  | **log**                | Structured, trace-correlated (`logs.md`)                              |
| Describe the producing service/process/host                   | **resource attribute** | Entity-level, set once per process (`../discovery/resolve-values.md`) |

A high-cardinality value that is valuable per-operation goes on a **span attribute**
(`metrics.md`). A log that marks an operation's start/end or carries a hand-rolled
correlation id is usually better as a **span** or span event
(`../instrumentation-rules.md`).

## Cross-SDK reality (check before relying on a signal)

Signal availability, instrument names, **log-API maturity**, and
semantic-convention package layout differ between SDKs and versions — some SDKs
expose only a log bridge/appender rather than a first-class logging API. Always
confirm what the installed SDK/API and convention packages actually support before
depending on a specific signal or constant.
