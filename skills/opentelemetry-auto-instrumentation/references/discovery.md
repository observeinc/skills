# Application Discovery

Scan the target directory and return a structured summary of the application stack, existing instrumentation, and candidate runnable services.

## Scan Procedure

### 1. Identify Candidate Applications

Look for runnable application entrypoints in the target directory. A repository may contain multiple services (monorepo). For each candidate, record:

- **Path**: relative directory containing the application
- **App name**: inferred from directory name, `package.json` `name` field, build artifact name, or module name
- **Language**: Java, Python, Node.js, .NET, Ruby
- **Framework**: the primary web/server framework (e.g. Spring Boot, Django, FastAPI, Express, NestJS, ASP.NET Core, Rails)
- **Run command**: if inferrable from Dockerfile CMD, package.json scripts, Makefile targets, or Procfile
- **Existing instrumentation status**: none, OTel, or vendor (summarize)

If more than one plausible candidate exists, stop after presenting the candidate summary and ask the developer to choose the target app or directory. Do not continue with stack-specific instrumentation until they do.

**Entrypoint indicators by language:**

| Language | Indicators                                                                                                                              |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Java     | `pom.xml`, `build.gradle`, `build.gradle.kts`, `src/main/java/**/Application.java`, `@SpringBootApplication`, `public static void main` |
| Python   | `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile`, `manage.py` (Django), `app.py` / `main.py` with ASGI/WSGI app objects      |
| Node.js  | `package.json` with `main` or `scripts.start`, `server.js`, `index.js`, `app.js`                                                        |
| .NET     | `*.csproj`, `*.sln`, `Program.cs`, `Startup.cs`                                                                                         |
| Ruby     | `Gemfile`, `config.ru`, `Rakefile`, `bin/rails`                                                                                         |

### 2. Identify Runtime and Framework

For the selected target (or each candidate if multiple):

- **Language version**: from lockfiles, `.tool-versions`, `.python-version`, `.node-version`, `.ruby-version`, `global.json`, or Dockerfile base image
- **Framework and version**: from dependency declarations (e.g. `spring-boot-starter-web` in pom.xml, `django` in requirements.txt, `express` in package.json)
- **Package manager**: which tool manages dependencies (Maven, Gradle, pip, poetry, uv, npm, yarn, pnpm, bun, NuGet, Bundler)
- **Lockfiles present**: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb`, `poetry.lock`, `uv.lock`, `Gemfile.lock`, etc.
- **Test framework**: JUnit, pytest, Jest, Mocha, xUnit, RSpec, Minitest, etc.
- **Container hints**: Dockerfile, docker-compose.yml, Containerfile, `.dockerignore`
- **Deployment hints**: Kubernetes manifests, Helm charts, Procfile, systemd unit files, Terraform

### 3. Detect Existing OpenTelemetry Instrumentation

Search for signs of existing OTel setup:

**Dependencies:**

- Java: `opentelemetry-javaagent*.jar`, `io.opentelemetry` in pom.xml/build.gradle, `quarkus-opentelemetry`, `micronaut-tracing-opentelemetry`
- Python: `opentelemetry-distro`, `opentelemetry-sdk`, `opentelemetry-instrumentation-*`, `opentelemetry-exporter-otlp`
- Node.js: `@opentelemetry/auto-instrumentations-node`, `@opentelemetry/sdk-node`, `@opentelemetry/exporter-trace-otlp-*`
- .NET: `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.*`, `OpenTelemetry.Exporter.OpenTelemetryProtocol`
- Ruby: `opentelemetry-sdk`, `opentelemetry-instrumentation-all`, `opentelemetry-exporter-otlp`

**Startup wiring:**

- Java: `-javaagent:` flag in startup scripts, Dockerfiles, `JAVA_TOOL_OPTIONS`, `JDK_JAVA_OPTIONS`
- Python: `opentelemetry-instrument` command wrapper, `opentelemetry.instrumentation.auto_instrumentation.initialize()` call
- Node.js: `--require @opentelemetry/auto-instrumentations-node/register`, `--import` flag, `instrumentation.{js,ts,mjs,cjs}` setup file
- .NET: `AddOpenTelemetry()` in Program.cs or startup code
- Ruby: `config/initializers/opentelemetry.rb`, `OpenTelemetry::SDK.configure`

**Environment variables:**

- `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_TRACES_EXPORTER`, `OTEL_METRICS_EXPORTER`
- Look in `.env` files, docker-compose.yml, Kubernetes manifests, systemd unit files, Procfile

**Configuration files:**

- `otel-collector-config.yaml`, `observe-agent.yaml`, `opentelemetry-collector.yaml`

### 4. Detect Vendor (Non-OTel) Instrumentation

Search for competing APM agents that may conflict with OpenTelemetry:

| Vendor      | Sample Indicators                                                                                          |
| ----------- | ---------------------------------------------------------------------------------------------------------- |
| Datadog     | `dd-trace`, `ddtrace`, `datadog-agent`, `DD_AGENT_HOST`, `DD_TRACE_ENABLED`, `com.datadoghq:dd-java-agent` |
| New Relic   | `newrelic`, `@newrelic/`, `newrelic-agent.jar`, `NEW_RELIC_LICENSE_KEY`, `newrelic.yml`, `newrelic.js`     |
| Splunk      | `splunk-otel-*`, `SPLUNK_ACCESS_TOKEN`, `splunk-otel-javaagent.jar`                                        |
| Dynatrace   | `@dynatrace/oneagent`, `dynatrace-oneagent`, `DT_TENANT`, `oneagentctl`                                    |
| AppDynamics | `appdynamics`, `APPDYNAMICS_*`, `AppServerAgent`                                                           |
| Elastic APM | `elastic-apm-node`, `elastic-apm-agent`, `elasticapm`, `ELASTIC_APM_*`                                     |

Record: vendor name, where detected (dependency file, env var, config file, startup script), and version if visible.

## Output Format

Return findings as structured text in this format:

```text
## Discovery Results

### Candidate Applications
- [list each candidate with: path, name, language, framework, run command, instrumentation status]

### Runtime (for primary candidate, or each candidate if multiple)
- Language: <language> <version>
- Framework: <framework> <version>
- Package manager: <tool>
- Lockfiles: <list>
- Test framework: <name>
- Container: <Dockerfile path or "none">
- Deployment: <type or "unknown">

### Existing OpenTelemetry
- Status: <none | partial | complete>
- Dependencies: <list>
- Startup wiring: <description or "not found">
- Environment variables: <list of OTEL_* vars found>
- Configuration: <description or "not found">

### Vendor Instrumentation
- Status: <none | detected>
- Vendor: <name>
- Indicators: <where detected>
- Version: <if known>
```

When multiple candidate applications are present, end with a direct target-selection question instead of silently choosing one.
