# Existing OpenTelemetry Audit

If discovery finds an existing OpenTelemetry setup, audit it before changing anything. The goal is to decide whether to leave the setup alone, improve it in place, or pause because the situation is unclear or risky.

## Audit checklist

### 1. Service identity

Check where the service identity is defined:

- `OTEL_SERVICE_NAME`
- `OTEL_RESOURCE_ATTRIBUTES`
- framework config files
- `.env` files
- Docker or Kubernetes manifests

Verify whether the intended `service.name` is present and whether `deployment.environment.name` and `service.namespace` are set consistently when the project uses them.

### 2. Startup wiring

Check whether instrumentation is actually activated at process startup:

- Java: `-javaagent:` flags, `JAVA_TOOL_OPTIONS`, framework-native OTel bootstrapping
- Python: `opentelemetry-instrument`, distro bootstrap, startup hooks
- Node.js: `--require`, `--import`, preload registration, dedicated instrumentation bootstrap file
- .NET: `AddOpenTelemetry()` and related startup registration
- Ruby: `config/initializers/opentelemetry.rb`, `OpenTelemetry::SDK.configure`

Having dependencies without startup wiring usually means the setup is incomplete.

### 3. OTLP export configuration

Check whether telemetry has a plausible export path:

- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_PROTOCOL`
- headers or auth settings if the project uses them
- application config that overrides exporter defaults

If the project has OTel libraries but no valid export path, treat the setup as incomplete.

### 4. Duplicate or conflicting instrumentation

Look for:

- multiple OTel bootstrappers for the same runtime
- overlapping framework-native and generic auto-instrumentation that would likely emit duplicate spans
- multiple exporters for the same signal without a clear reason
- vendor instrumentation that would conflict with OTel

### 5. Library and agent health

Check whether the current OTel packages or agents look:

- current enough to be worth preserving
- partially installed but not wired
- obviously stale, deprecated, or inconsistent with the runtime
- mismatched across package files, startup scripts, and environment variables

The point is not to force the newest version every time, but to spot versions or combinations that make the setup look fragile or broken.

## Outcome logic

### `healthy`

Use this outcome when:

- service identity looks intentional
- startup wiring is present
- OTLP export configuration looks valid
- there are no obvious duplicates or conflicts
- the installed setup is coherent enough that changing it would add more risk than value

Recommendation: no code changes. Tell the developer the setup already looks healthy and point them to `opentelemetry-validation` for Observe-side confirmation.

### `improve`

Use this outcome when OTel is present but one or more of these gaps exist:

- missing or incorrect `service.name`
- missing startup wiring
- missing or invalid OTLP configuration
- duplicate or conflicting instrumentation
- stale or clearly unhealthy packages

Recommendation: improve the existing setup in place instead of starting over.

### `uncertain`

Use this outcome when the evidence is incomplete or the safest path is unclear, for example:

- multiple competing instrumentation patterns are present
- the repo suggests generated or external startup scripts you cannot inspect
- the project appears to rely on infrastructure-side injection outside the repo

Recommendation: pause, explain what is unknown, and ask before making disruptive changes.

## Output format

Use this summary before proposing edits:

```text
## Existing OTel Audit
- Status: <healthy | improve | uncertain>
- Service identity: <summary>
- Startup wiring: <summary>
- Export config: <summary>
- Duplicate/conflicting instrumentation: <summary>
- Library health: <summary>
- Recommendation: <no-op | improve existing | pause>
```
