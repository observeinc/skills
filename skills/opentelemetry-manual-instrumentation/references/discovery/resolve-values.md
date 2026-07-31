# Resource Attributes & Config Values

Resource attributes describe the **entity that produces telemetry** — the service,
process, host, or container — and ride on _every_ signal it emits, so they must be
**stable for the process/instance lifetime**. A field that varies per request belongs on
a span or metric. This file also covers the delivery values instrumentation needs (OTLP
endpoint, auth).

**Resolve every value from the project; never invent or hardcode it.** Use the first
source that yields a real value; if none does, **ask the user**. Record where each value
came from in the final report. General order for every value: **existing `OTEL_*` config**
(`.env*`, compose, k8s manifests, Helm values) → **project/manifest source** → **ask**.
Prefer resource **detectors** and standard `OTEL_*` env vars over hand-set values, so the
same build runs everywhere and values stay out of source.

## `service.name`

1. Existing `OTEL_SERVICE_NAME` / `service.name` in `OTEL_RESOURCE_ATTRIBUTES`.
2. The package manifest name — Node `package.json` `name`; Python
   `pyproject.toml`/`setup.cfg`; Go module path; Java `artifactId`; .NET assembly;
   Ruby gemspec; PHP `composer.json`.
3. The project directory name.

Normalize to a stable convention (e.g. kebab-case) and keep casing consistent — `checkout`
and `CheckOut` are different services to a backend and split your data. A missing
`service.name` shows up as `unknown_service`. `OTEL_SERVICE_NAME` takes precedence over a
`service.name` set inside `OTEL_RESOURCE_ATTRIBUTES`.

## `service.version`

1. Existing OTEL config.
2. CI/CD version injection — reference the variable the pipeline sets
   (`.github/workflows`, `.gitlab-ci.yml`, `Jenkinsfile`) rather than copying its
   current value.
3. Manifest version field.
4. `git describe --tags --always` (or the short SHA).

## `deployment.environment.name`

1. Existing OTEL config.
2. A framework/runtime env var: `NODE_ENV`, `RAILS_ENV`/`RACK_ENV`,
   `DJANGO_SETTINGS_MODULE`, `SPRING_PROFILES_ACTIVE`, `ASPNETCORE_ENVIRONMENT`,
   `APP_ENV`.
3. The Kubernetes namespace, or a Docker/Compose target.

Resolve it from a real source or ask — a wrong environment silently merges prod and
non-prod telemetry, worse than a missing value.

## `service.namespace`

Existing config → a monorepo grouping directory → the org/product name; omit if none
resolves.

## `service.instance.id`

Unlike the others, **generate** this value rather than sourcing it. It must make the
triplet (`service.namespace`, `service.name`, `service.instance.id`) **globally unique**,
and be opaque and stable for the process lifetime: generate a UUID v4 at startup, or a
deterministic UUID v5 from the pod UID. `$(hostname)` collides across replicas;
multi-worker runtimes (Gunicorn/Puma/PHP-FPM) need a **per-worker** id. This is separate
from `k8s.pod.uid` (detectors set that for a different purpose); the instance id stays
opaque, not a pod or container name.

## Preserve operator / environment-provided values

When a Kubernetes operator, auto-instrumentation, or the environment already supplies
resource attributes (via `OTEL_RESOURCE_ATTRIBUTES` / `OTEL_SERVICE_NAME`), **merge app
defaults only for keys that are absent**; operator-provided `service.name`,
`deployment.environment[.name]`, and `service.version` win. Overwriting them silently
re-labels every signal. Add a focused **resource-merge test**, and confirm the
**effective** resource from real exported telemetry (source-level merge logic alone
doesn't prove what survived provider construction — see
`../verification/validation-and-debugging.md`).

## Keep resource attributes entity-level

Keep per-request and per-user data (the risky-value list in
`../review/cardinality.md`) off resource attributes. A resource attribute is
broadcast onto **every** signal from the process, so per-request data there is both wrong
(not entity-level) and a cardinality/cost problem. Put tenant ID here only when the
deployment is genuinely single-tenant per instance and it's explicitly approved and
bounded.

## OTLP endpoint and auth

- **Endpoint:** existing config → a sidecar/DaemonSet collector
  (`localhost:4317` gRPC / `localhost:4318` HTTP) → ask.
- **Auth:** existing config → localhost collector usually needs no auth header →
  ask. Read any token from the environment.

## State these before you edit

Before writing code, confirm and record: the target process/entrypoint (from the
real start surface — compose, k8s manifests, `package.json` scripts, Makefile,
Procfile, PM2, systemd), the resolved `service.name` and environment sources, and
whether this is an incremental change to existing OTel setup or a fresh one. Don't
defer these to the final report.
