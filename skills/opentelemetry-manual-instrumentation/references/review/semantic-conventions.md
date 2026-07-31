# Semantic Conventions

Semantic conventions are OpenTelemetry's standard names and meanings for spans,
metrics, attributes, units, and resource fields. Using them makes telemetry
**portable**: dashboards, alerts, and backends understand it across tools and
vendors, and auto-build service maps, latency views, and error rates from the
conventional fields — without per-service customization.

## Where to look it up

Never guess a convention — look it up in the official source, and use the exact
key, value format, and unit it prescribes:

- **[All semantic conventions](https://opentelemetry.io/docs/specs/semconv/)** —
  every domain, rendered and versioned (this is the entry point).
- **[Attribute registry](https://opentelemetry.io/docs/specs/semconv/registry/)** —
  search here for an existing attribute key before inventing one.
- **[Spec repository](https://github.com/open-telemetry/semantic-conventions)** —
  raw YAML definitions and change history, when the docs are ambiguous.

## Prefer standard over custom

Before adding any field, **search for an existing convention**. Prefer standard
attributes, span names, span kinds, metric names, metric units, and resource
attributes wherever applicable. Introduce a custom field only when no convention
covers the concept.

## Guidance by domain

Consult the [official conventions](https://opentelemetry.io/docs/specs/semconv/)
for the specifics; the categories you will most often need:

- **[HTTP](https://opentelemetry.io/docs/specs/semconv/http/)** — request method,
  route (template), response status code, etc.
- **[Database](https://opentelemetry.io/docs/specs/semconv/db/)** — system,
  operation/statement kind, target table/collection.
- **[Messaging](https://opentelemetry.io/docs/specs/semconv/messaging/)** — system,
  destination name, operation (publish/receive/process).
- **[RPC](https://opentelemetry.io/docs/specs/semconv/rpc/)** — system, service,
  method.
- **[Exceptions](https://opentelemetry.io/docs/specs/semconv/exceptions/)** —
  exception type, message, stacktrace (recorded as an event).
- **[Service](https://opentelemetry.io/docs/specs/semconv/resource/)** — name,
  version, namespace, instance id (resource level).
- **[Deployment](https://opentelemetry.io/docs/specs/semconv/resource/)** —
  environment (resource level).
- **[Host / Process / Container / Cloud](https://opentelemetry.io/docs/specs/semconv/resource/)**
  — host, process, container, and cloud metadata (resource level).

Use the exact attribute keys, value formats, and units the convention prescribes.

## Attribute placement

Put each field at the level where it belongs:

- **Resource attributes** describe the **telemetry-producing entity** (service,
  process, host, container, cloud). Stable for the process/instance.
- **Span attributes** describe a **single operation**.
- **Span events** describe **notable moments inside** an operation.
- **Metric attributes** describe **bounded dimensions** of a measurement.
- **Log attributes** describe a **log record** and should support correlation
  (e.g. trace/span IDs) when the SDK provides it.

Wrong placement (e.g. per-request data on a resource, or operation data as a
resource attribute) breaks aggregation and inflates cardinality.

## Custom business attributes

When you genuinely need a project-specific field, use a **bounded, namespaced**
key such as `app.*`, `business.*`, or an organization-specific namespace. Keep the
value space bounded and document the attribute (see `semantic-review.md`).

## Conventions evolve — verify versions

Semantic conventions **change over time** (attributes get added, stabilized,
renamed, or deprecated). Before introducing new constants:

- inspect the **installed semantic-convention package version(s)** in the project
- check how the project **already uses** conventions (don't mix incompatible
  versions or invent keys that conflict with existing usage)
- prefer constants exported by the installed convention package over hand-typed
  string literals, so renames are caught at build time where possible

## Attribute naming style

- Use **dot-delimited namespaces**: `order.id`, `http.request.method`.
- Check the OpenTelemetry
  [**attribute registry**](https://opentelemetry.io/docs/specs/semconv/registry/)
  before inventing a key; prefer an existing one (`enduser.id` over a bespoke
  `user.id`) when it fits.
- Custom keys get an **organization/project prefix** (`com.acme.order.priority`) so a
  future convention can't collide with a bare word.

## Library lag and validator findings

Auto-instrumentation may emit **outdated** convention names or units (e.g.
`http.server.duration` in ms and `http.method`, versus the stable
`http.server.request.duration` in seconds with `http.request.method` /
`http.response.status_code`). Prefer the library's **stable-semconv opt-in** when
it has one; otherwise document the gap instead of emitting your own duplicate.

When a backend or conformance check reports issues, **classify each finding**
before treating it as a defect:

- **actionable** — a real violation in telemetry you own; fix it.
- **registry mismatch** — the validator's registry moved/omits a field, or it
  rejects a legitimate app-owned custom or framework-owned attribute.
- **library-owned** — an official instrumentation's shape the validator reads
  differently (e.g. an omitted default `server.port`); not yours to fix.
- **stale** — emitted by a still-running periodic exporter from a prior run.

Judge verification by the classified findings, not the raw red/violation count.
