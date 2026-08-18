# Semantic Review (Mandatory)

**Every** manually emitted field — span name, span kind, status, attribute, event,
metric name/unit/instrument, resource attribute, log field — must pass this review
before instrumentation is complete.

The agent **must reject or revise** any field that fails a check below.

This gate is about **convention conformance** (names/units/placement/cardinality/
privacy). Whether the telemetry actually answers the user's question — the right
signal/instrument/dimensions for their goal — is a **separate axis**; see
`../discovery/intent.md`.

## Review checklist

For each manually emitted telemetry field, ask:

1. **Convention first.** Does an official semantic convention already exist for
   this attribute, span name, metric, unit, or resource field? If so, use it
   instead of a custom field.
2. **Correct level.** Is the field placed at the right level?
    - resource (entity producing telemetry)
    - span (one operation)
    - span event (a moment inside an operation)
    - metric datapoint (a bounded dimension)
    - log record (a single log line, ideally trace-correlated)
3. **Span name.** Is the span name **stable and low-cardinality** (operation
   class, not instance)?
4. **Span kind.** Is the span kind correct (`SERVER`/`CLIENT`/`PRODUCER`/
   `CONSUMER`/`INTERNAL`) for its role?
5. **Span status.** Is status set correctly — `Error` on failure, with the
   exception recorded?
6. **Instrument type.** Is the metric instrument type correct (counter vs
   up-down counter vs histogram vs observable)?
7. **Unit.** Is the metric unit correct and canonical (e.g. `s`, `By`, `1`,
   `{request}`)?
8. **Bounded labels.** Are metric attributes **bounded and low-cardinality**?
9. **No sensitive data.** Are any attributes raw PII, secrets, tokens, cookies,
   request bodies, response bodies, auth headers, or unbounded identifiers? If so,
   remove or transform them.
10. **Custom necessity.** Is a custom attribute **truly necessary**, or does a
    convention cover it?
11. **Namespaced if custom.** If custom, does it use a **bounded project
    namespace** (e.g. `app.*`, `business.*`)?
12. **Documented if custom.** Is the custom attribute documented (key, meaning,
    value space, why it exists)?
13. **Better representation?** Could this value be better represented as a span
    event, span attribute, metric attribute, resource attribute, or log field?
14. **Stability.** Would this field remain stable across deployments and versions?
15. **Cost / cardinality / privacy.** Could this field cause cost, cardinality, or
    privacy problems? If yes, revise it (route template, bounded enum, bucket,
    drop).

## The gate's receipt

The required receipt is the **per-field convention review** recorded in the final
response (`Semantic convention review outcome`): list each emitted field (span name,
kind, status, attribute, event, metric name/unit/instrument, resource attribute) with
either the **standard convention it matches**, or (if custom) its **bounded
namespace + justification + where it's documented**; plus any field that was
**rejected or revised** and why. The per-field list is required; a bare "reviewed"
claim does not satisfy the gate.

This is a **conformance** check (is the shape correct); backend arrival is proven
separately by the backend-verification step.

## Quick failure → fix table

| Failure                                  | Fix                                                    |
| ---------------------------------------- | ------------------------------------------------------ |
| Span name contains an ID                 | Use the operation/route template                       |
| Custom attribute duplicates a convention | Use the convention attribute                           |
| High-cardinality metric label            | Replace with a bounded enum / route template           |
| Raw PII or secret in any field           | Remove; if a class is needed, use a bounded derivative |
| Wrong level (per-request on resource)    | Move to span/metric attribute                          |
| Duration as counter/gauge                | Use a histogram with unit `s`                          |
| Custom attribute without namespace       | Move under `app.*`/`business.*` and document           |
