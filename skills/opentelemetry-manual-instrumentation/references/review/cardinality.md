# Cardinality

Unbounded cardinality destabilizes collectors and backends: each unique attribute
set is a new time series. Check every emitted field. The never-record list and
sanitization patterns live in `sensitive-data.md`.

## Cardinality discipline

- **Avoid** high-cardinality **span** attributes where possible (spans tolerate
  more than metrics, but unbounded values still cost storage and indexing).
- **Strictly avoid** high-cardinality **metric** labels — each unique attribute
  set is a separate time series.

## Risky values (treat as unbounded by default)

- user ID
- request ID
- trace ID
- session ID
- full URL with path/query parameters
- order ID
- cart ID
- IP address (unless explicitly approved and normalized)
- tenant ID (unless explicitly approved and bounded)
- arbitrary exception messages when they may contain user input

## Safer alternatives

| Risky                          | Safer                                            |
| ------------------------------ | ------------------------------------------------ |
| Full URL with IDs              | **Route template** (`/orders/{id}`)              |
| Free-form string               | **Bounded enum**                                 |
| Exact numeric value as a label | **Bucketed** value / histogram                   |
| Raw dynamic status detail      | **Normalized status class** (`2xx`, `error`)     |
| Raw identifier                 | **Hashed value** _only when explicitly approved_ |

A hashed value is still potentially high-cardinality and may be reversible for
small input spaces — use it only when explicitly approved, and prefer it on spans,
not metric labels.

## Metrics: keep dimensions bounded

Because metrics aggregate into per-series storage, a single unbounded label can
explode one metric into millions of series, destabilizing collectors/backends and
inflating cost. When a dimension might be unbounded, put it on a span attribute
instead (see `../signals/metrics.md`).

## Document approved exceptions

If the project owner explicitly approves recording a normally-risky value (e.g. a
bounded tenant ID, a normalized IP), **document it**: which field, where it
appears, the approved transformation (bounding/normalization/hashing), and the
justification. Record this in the cardinality review of the final response.
Unapproved risky values are rejected or revised.
