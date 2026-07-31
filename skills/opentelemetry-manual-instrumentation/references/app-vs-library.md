# Application vs Library Instrumentation

Whether you're an application or a reusable library determines what you may
configure. Decide this **first**.

## Application / service responsibilities

An application or service is the process that runs and owns the telemetry
pipeline. Applications:

- **initialize the SDK** (exactly once, before application modules run)
- **configure exporters** (prefer OTLP via environment variables)
- **set resource attributes** (service name, version, environment, etc.)
- **configure sampling**
- **implement graceful shutdown / flushing** of telemetry
- create their own manual spans and metrics for domain logic

## Library / reusable package responsibilities

A reusable library is consumed by other applications and emits through the host's
telemetry pipeline. Libraries:

- emit spans/metrics **through the OpenTelemetry API only**
- use **stable tracer and meter scope names** (typically the library/package
  name) and a meaningful version
- add semantic attributes describing their operations
- **do nothing** if the host has not configured an SDK (the API is a no-op by default)

## Why SDK initialization inside a library is an anti-pattern

If a library initializes the SDK or configures exporters, it:

- fights the host application for ownership of the global providers
- can cause **double initialization** and conflicting configuration
- forces an exporter/endpoint the host may not want
- couples consumers to dependency versions and pipeline choices they did not pick
- breaks when multiple such libraries are combined in one process

## Division of ownership

| Concern                        | Application | Library       |
| ------------------------------ | ----------- | ------------- |
| SDK initialization             | ✅ owns     | ❌ never      |
| Exporters / endpoints          | ✅ owns     | ❌ never      |
| Resource attributes            | ✅ owns     | ❌ never      |
| Sampling                       | ✅ owns     | ❌ never      |
| Shutdown / flush               | ✅ owns     | ❌ never      |
| Emitting spans/metrics via API | ✅          | ✅            |
| Stable tracer/meter scope name | ✅          | ✅ (required) |
