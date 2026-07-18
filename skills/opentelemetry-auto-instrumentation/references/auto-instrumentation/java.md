# OpenTelemetry Java Auto-Instrumentation Reference

## Resolve latest versions

Before installing or downloading anything, resolve the latest stable versions of all required artifacts. Use these resolved versions in all subsequent steps.

## Setup using the OpenTelemetry Java Agent

### Install dependencies

Resolve the latest release from <https://api.github.com/repos/open-telemetry/opentelemetry-java-instrumentation/releases/latest> and save the agent using the resolved release tag in the filename (for example, `opentelemetry-javaagent-v2.28.0.jar`).

#### POSIX shell (`sh`, `bash`, `zsh`)

```sh
OTEL_JAVA_RELEASE_JSON="$(
  curl -fsSL "https://api.github.com/repos/open-telemetry/opentelemetry-java-instrumentation/releases/latest" |
    tr -d '\n'
)"
OTEL_JAVA_VERSION="$(
  printf '%s' "$OTEL_JAVA_RELEASE_JSON" |
    sed -n 's/.*"tag_name":"\([^"]*\)".*/\1/p'
)"
OTEL_JAVA_DOWNLOAD_URL="$(
  printf '%s' "$OTEL_JAVA_RELEASE_JSON" |
    sed -n 's/.*"browser_download_url":"\([^"]*opentelemetry-javaagent\.jar\)".*/\1/p'
)"

if [ -z "$OTEL_JAVA_VERSION" ] || [ -z "$OTEL_JAVA_DOWNLOAD_URL" ]; then
  echo "Failed to resolve the latest OpenTelemetry Java agent release." >&2
  exit 1
fi

curl -fsSL \
  -o "opentelemetry-javaagent-${OTEL_JAVA_VERSION}.jar" \
  "$OTEL_JAVA_DOWNLOAD_URL"
```

#### PowerShell

```powershell
$release = Invoke-RestMethod -Uri 'https://api.github.com/repos/open-telemetry/opentelemetry-java-instrumentation/releases/latest'
$version = $release.tag_name
$asset = $release.assets | Where-Object { $_.name -eq 'opentelemetry-javaagent.jar' } | Select-Object -First 1

if (-not $version -or -not $asset) {
    throw 'Failed to resolve the latest OpenTelemetry Java agent release.'
}

$output = "opentelemetry-javaagent-$version.jar"
Invoke-WebRequest -Uri $asset.browser_download_url -OutFile $output
```

### Update application startup

The downloaded JAR needs to be attached to applications using one of the following approaches:

#### Command line argument `-javaagent`

By specifying `java -javaagent:path/to/opentelemetry-javaagent-<version>.jar -jar myapp.jar` in application startup scripts or Dockerfiles.

#### Environment variables

The agent may be injected using one of the following environment variables:

- `JAVA_TOOL_OPTIONS=-javaagent:/path/opentelemetry-javaagent-<version>.jar`
- `JDK_JAVA_OPTIONS=-javaagent:/path/opentelemetry-javaagent-<version>.jar`
  Note: When both are provided, `JDK_JAVA_OPTIONS` takes precedence.

## Frameworks incompatible with the Java Agent

The following frameworks may not be directly compatible with the Java Agent, but offer native instrumentation bootstrappers which indirectly provide the same features as the java agent.

## Quarkus

Add the extension to `pom.xml` (version managed by the Quarkus BOM):

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-opentelemetry</artifactId>
</dependency>
```

Configure `application.properties`:

```properties
quarkus.otel.enabled=true
quarkus.otel.metrics.enabled=true
quarkus.otel.logs.enabled=true
quarkus.otel.exporter.otlp.endpoint=${OTLP_ENDPOINT}
quarkus.otel.exporter.otlp.logs.endpoint=${OTLP_ENDPOINT}
quarkus.datasource.jdbc.telemetry=true

%test.quarkus.otel.sdk.disabled=true
quarkus.test.continuous-testing=disabled
```

## Micronaut

Resolve the latest versions from Maven Central before adding dependencies:

#### POSIX shell (`sh`, `bash`, `zsh`)

```sh
maven_latest() {
  curl -fsSL "https://search.maven.org/solrsearch/select?q=g:${1}+AND+a:${2}&rows=1&wt=json" |
    sed -n 's/.*"latestVersion":"\([^"]*\)".*/\1/p'
}

MN_TRACING_VERSION=$(maven_latest "io.micronaut.tracing" "micronaut-tracing-opentelemetry")
OTEL_EXPORTER_VERSION=$(maven_latest "io.opentelemetry" "opentelemetry-exporter-otlp")
MN_MICROMETER_VERSION=$(maven_latest "io.micronaut.micrometer" "micronaut-micrometer-registry-otlp")

echo "io.micronaut.tracing:micronaut-tracing-opentelemetry:${MN_TRACING_VERSION}"
echo "io.opentelemetry:opentelemetry-exporter-otlp:${OTEL_EXPORTER_VERSION}"
echo "io.micronaut.micrometer:micronaut-micrometer-registry-otlp:${MN_MICROMETER_VERSION}"
```

#### PowerShell

```powershell
function Maven-Latest($group, $artifact) {
    $result = Invoke-RestMethod "https://search.maven.org/solrsearch/select?q=g:${group}+AND+a:${artifact}&rows=1&wt=json"
    $result.response.docs[0].latestVersion
}

$MN_TRACING_VERSION = Maven-Latest 'io.micronaut.tracing' 'micronaut-tracing-opentelemetry'
$OTEL_EXPORTER_VERSION = Maven-Latest 'io.opentelemetry' 'opentelemetry-exporter-otlp'
$MN_MICROMETER_VERSION = Maven-Latest 'io.micronaut.micrometer' 'micronaut-micrometer-registry-otlp'
```

If any version fails to resolve, do not proceed with installation.

Add the tracing modules to `build.gradle.kts` using the resolved versions:

```kotlin
implementation("io.micronaut.tracing:micronaut-tracing-opentelemetry:${MN_TRACING_VERSION}")
implementation("io.micronaut.tracing:micronaut-tracing-opentelemetry-http:${MN_TRACING_VERSION}")
implementation("io.micronaut.tracing:micronaut-tracing-opentelemetry-jdbc:${MN_TRACING_VERSION}")
implementation("io.opentelemetry:opentelemetry-exporter-otlp:${OTEL_EXPORTER_VERSION}")
implementation("io.micronaut.micrometer:micronaut-micrometer-registry-otlp:${MN_MICROMETER_VERSION}")
```

Configure `application.yml`. Traces are exported via the OTel SDK exporter (gRPC on port 4317). Metrics are exported via the Micrometer OTLP registry (HTTP on port 4318):

```yaml
micronaut:
  metrics:
    enabled: true
    binders:
      jvm:
        enabled: true
    export:
      otlp:
        enabled: true
        url: ${OTLP_ENDPOINT_HTTP}/v1/metrics
        step: PT30S

otel:
  traces:
    exporter: otlp
  metrics:
    exporter: otlp
  exporter:
    otlp:
      endpoint: ${OTLP_ENDPOINT}
```

Note that `micronaut.metrics.export.otlp.url` requires the HTTP endpoint with the `/v1/metrics` path, while `otel.exporter.otlp.endpoint` uses the gRPC endpoint.
