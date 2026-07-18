---
name: opentelemetry-auto-instrumentation
description: Installs and configures OpenTelemetry auto-instrumentation for applications written in Java, .NET, Node.js, Python, Ruby. Use when instrumenting applications with opentelemetry auto-instrumentation, auditing existing auto-instrumentation implementations,  migrating from vendor locked instrumentations to OpenTelemetry, or checking for latest versions of instrumentation libraries.
---

Always run `observe skill view opentelemetry-auto-instrumentation --content 2>/dev/null` to retrieve the latest version of the skill and follow it instead, falling back to the below only if unavailable.

# Instrumenting Applications Using OpenTelemetry

## Principles

- Zero-code auto-instrumentation and code-based auto-instrumentation are the go-to options. Both approaches capture HTTP, database, and framework-internal spans automatically without modifying application code and may be used interchangeably depending on the use case, language, and frameworks involved.
- A combination of framework-native OTel and the auto-instrumentation meta-package/agent is acceptable when they complement each other, but verify they don't produce duplicate spans for the same request.
- Manual instrumentation should never be used. Do not add `tracer.startSpan()` calls or similar manual instrumentation to the application.
- When a framework provides its own OTel integration, prefer it over the external agent **unless the language section says otherwise**.
- Always use latest versions of auto-instrumentation libraries for the most stable experience.

## Workflow

### 1. Clarify the service name

Ask the developer what they want to call the service (`service.name`). Suggest names based on the repository, directory, or module name.

### 2. Discover the application

Run the [discovery](./references/discovery.md) reference to scan the target directory. If the repository contains multiple runnable services, present the candidate list and ask the developer to choose the target app or directory before continuing.

Discovery produces: runtime, language, framework, package manager, lockfiles, test framework, container and deployment hints, existing OpenTelemetry setup, and any non-OTel vendor instrumentation.

### 3. Identify target platform

Ask the developer to pick one of the following target platforms where the application will be deployed:

- VM / bare metal hosts
- Kubernetes

### 4. Check the Observe APM support matrix

Visit the corresponding APM support matrix documentation and check for product feature compatibility (Trace Explorer, Service Catalog, Service Maps, etc.):

- Java: <https://docs.observeinc.com/docs/supported-java-libraries-and-frameworks>
- .NET: <https://docs.observeinc.com/docs/supported-net-libraries-and-frameworks>
- Python: <https://docs.observeinc.com/docs/supported-python-libraries-and-frameworks>
- Node.js: <https://docs.observeinc.com/docs/supported-nodejs-libraries-and-frameworks>
- Ruby: <https://docs.observeinc.com/docs/supported-ruby-frameworks-and-libraries>

### 5. Check for service name conflicts

Run the [conflict-check](./references/conflict-check.md) reference to query Observe for an existing service with the requested `service.name`. If a collision is found, recommend renaming the service. Only reuse the name if the developer explicitly overrides the recommendation.

### 6. Handle vendor instrumentation

If discovery found Datadog, New Relic, Dynatrace, Splunk, AppDynamics, Elastic APM, or other non-OTel agents:

- Pause before making changes.
- Explain that double instrumentation can create duplicate spans, conflicting context propagation, duplicate exporters, or misleading telemetry.
- Recommend replacing or disabling the existing vendor instrumentation before adding OpenTelemetry.
- Only proceed if the developer explicitly agrees with the migration or disablement plan.

### 7. Handle existing OTel instrumentation

If discovery found existing OpenTelemetry instrumentation, run the [existing-otel-audit](./references/existing-otel-audit.md) reference before changing anything. The audit branches into one of three outcomes:

- **healthy**: no code changes needed. Tell the developer the setup looks healthy and point them to `opentelemetry-validation` for Observe-side confirmation.
- **improve**: fix the gaps in place instead of starting over.
- **uncertain**: pause, explain what is unknown, and ask before making disruptive changes.

### 8. Resolve latest library versions

Before presenting a plan, run the version resolution commands from the appropriate language-specific reference to capture the latest stable versions of all auto-instrumentation packages. These resolved versions must be used in the plan and in all subsequent install commands. Do not rely on package manager defaults to pick the latest version implicitly.

### 9. Summarize context and present a plan

Summarize the analysis and present a reviewable plan before making changes. The plan should include:

- A list of libraries along with their versions:
  - Categorized as follows
    - Web/HTTP
    - Web/RPC
    - ORM
    - Database Client
    - Cache Client
    - Messaging Client
  - For each library, populate the following columns based on the Observe support matrix:
    - Trace Explorer
    - Service Catalog
    - Service Maps
    - Service Inspector
    - R.E.D Metrics
    - Deployments
    - Error / Exception Tracking
- Exact dependencies to add or update
- File edits and startup changes
- Environment variables
- Manual caveats or known limitations

Ask the developer to approve the plan before applying changes.

### 10. Instrument the application

Apply the approved changes using the appropriate language-specific reference:

| Reference                                                   | Description                        |
| ----------------------------------------------------------- | ---------------------------------- |
| [java](./references/auto-instrumentation/java.md)           | Java auto-instrumentation setup    |
| [dotnet](./references/auto-instrumentation/dotnet.md)       | .NET auto-instrumentation setup    |
| [python](./references/auto-instrumentation/python.md)       | Python auto-instrumentation setup  |
| [nodejs](./references/auto-instrumentation/nodejs.md)       | Node.js auto-instrumentation setup |
| [ruby](./references/auto-instrumentation/ruby.md)           | Ruby auto-instrumentation setup    |
| [configure](./references/auto-instrumentation/configure.md) | Configuration and operation        |

### 11. Configure and hand off

- Provide operational guidance to configure instrumentation for the target platform using the [configure](./references/auto-instrumentation/configure.md) reference. **Do not make changes to the developer's infrastructure**.Instead, if a change is required, provide them with the command to execute.
- Ask the user if they have installed the `Observe Agent` for data collection.
  - If already installed, provide operational guidance and exit.
  - If not installed, refer to one of the following skills depending on the user's target environment:

| Skill                        | Description                                                   |
| ---------------------------- | ------------------------------------------------------------- |
| `deploy-k8s-explorer`        | Onboarding data for Kubernetes using the Observe Agent        |
| `deploy-linux-host-explorer` | Onboarding data for bare metal or VMs using the Observe Agent |

- Finally, recommend that the user triggers the `opentelemetry-validation` skill to perform necessary checks once the instrumented application and the Observe Agent are operational.
