# Application Instrumentation (Optional)

If `traces` was selected (or the user otherwise plans to send OTLP from app pods), the agent is already set up to receive that telemetry — this skill's responsibility ends here, at the OTLP endpoints below. Wiring the application to emit OTLP is the job of a separate, language-specific skill.

The agent exposes (in-cluster) OTLP at:

| Endpoint                                                        | Protocol  | Use                                                         |
| --------------------------------------------------------------- | --------- | ----------------------------------------------------------- |
| `http://observe-agent-forwarder.observe.svc.cluster.local:4318` | OTLP HTTP | App SDKs that prefer HTTP, or auto-instrumentation defaults |
| `http://observe-agent-forwarder.observe.svc.cluster.local:4317` | OTLP gRPC | App SDKs that prefer gRPC                                   |

## Hand off

- **Do not modify the application or its build/deploy pipeline yourself.** If a change is required, hand off to the appropriate skill so the user gets a complete, tested workflow rather than partial advice.
- Ask the user whether they want to instrument an application now, set it up later, or just confirm what they already have:
    - If they want to set up new instrumentation — refer them to `opentelemetry-auto-instrumentation`.
    - If they already have OpenTelemetry instrumentation and want to verify it lands in Observe correctly — refer them to `opentelemetry-validation`.
    - If they're not ready to instrument anything yet — exit; the cluster-side setup is complete and the OTLP endpoints will be ready whenever they are.

| Skill                                | Description                                                                                                               |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `opentelemetry-auto-instrumentation` | Add OpenTelemetry to an existing app (auto-instrumentation for Java, Python, Node.js, .NET, Ruby)                         |
| `opentelemetry-validation`           | Verify instrumentation is wired correctly and the expected spans, RED metrics, and runtime metrics are landing in Observe |

> **Backend datastream prerequisite:** if `setup-k8s-backend` was not run with traces selected, the `Tracing/Span` and `Metrics/OpenTelemetry` datastreams won't exist and OTLP data will be accepted by the ingest endpoint but routed nowhere. Re-run `setup-k8s-backend` to add them before instrumenting the app.
