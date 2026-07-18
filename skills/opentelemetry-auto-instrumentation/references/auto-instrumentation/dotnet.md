# Instrumenting .NET Apps with OpenTelemetry

All modern .NET apps use the same shape:

1. Install the OpenTelemetry packages that match the host and libraries you use.
2. Register telemetry once at startup in an `AddAppTelemetry()` helper.
3. For each library, use either its OTel instrumentation package, its native `ActivitySource` / `Meter`, or its documented opt-in hook.
4. Export with standard OTLP environment variables.

This pattern works for ASP.NET Core, Minimal APIs, MVC, gRPC services, worker services, background jobs, and console apps. Do not add manual spans just to make auto-instrumentation work.

## Resolve latest versions

Before installing anything, query the NuGet registry for the latest stable version of each required package. Use these resolved versions in all subsequent `dotnet add package` commands.

#### POSIX shell (`sh`, `bash`, `zsh`)

```sh
nuget_latest() {
  local pkg_lower
  pkg_lower=$(echo "$1" | tr '[:upper:]' '[:lower:]')
  curl -fsSL "https://api.nuget.org/v3-flatcontainer/${pkg_lower}/index.json" |
    tr '[],"' '\n' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | tail -1
}

OTEL_HOSTING_VERSION=$(nuget_latest "OpenTelemetry.Extensions.Hosting")
OTEL_OTLP_VERSION=$(nuget_latest "OpenTelemetry.Exporter.OpenTelemetryProtocol")

echo "OpenTelemetry.Extensions.Hosting ${OTEL_HOSTING_VERSION}"
echo "OpenTelemetry.Exporter.OpenTelemetryProtocol ${OTEL_OTLP_VERSION}"
```

Resolve additional packages based on the application's needs:

```sh
OTEL_ASPNETCORE_VERSION=$(nuget_latest "OpenTelemetry.Instrumentation.AspNetCore")
OTEL_HTTP_VERSION=$(nuget_latest "OpenTelemetry.Instrumentation.Http")
OTEL_RUNTIME_VERSION=$(nuget_latest "OpenTelemetry.Instrumentation.Runtime")
OTEL_PROCESS_VERSION=$(nuget_latest "OpenTelemetry.Instrumentation.Process")
# OTEL_GRPC_VERSION=$(nuget_latest "OpenTelemetry.Instrumentation.GrpcNetClient")
# OTEL_SQL_VERSION=$(nuget_latest "OpenTelemetry.Instrumentation.SqlClient")
```

#### PowerShell

```powershell
function NuGet-Latest($package) {
    $versions = (Invoke-RestMethod "https://api.nuget.org/v3-flatcontainer/$($package.ToLower())/index.json").versions
    $versions | Where-Object { $_ -notmatch '-' } | Select-Object -Last 1
}

$OTEL_HOSTING_VERSION = NuGet-Latest 'OpenTelemetry.Extensions.Hosting'
$OTEL_OTLP_VERSION = NuGet-Latest 'OpenTelemetry.Exporter.OpenTelemetryProtocol'

Write-Host "OpenTelemetry.Extensions.Hosting $OTEL_HOSTING_VERSION"
Write-Host "OpenTelemetry.Exporter.OpenTelemetryProtocol $OTEL_OTLP_VERSION"
```

Resolve additional packages based on the application's needs:

```powershell
$OTEL_ASPNETCORE_VERSION = NuGet-Latest 'OpenTelemetry.Instrumentation.AspNetCore'
$OTEL_HTTP_VERSION = NuGet-Latest 'OpenTelemetry.Instrumentation.Http'
$OTEL_RUNTIME_VERSION = NuGet-Latest 'OpenTelemetry.Instrumentation.Runtime'
$OTEL_PROCESS_VERSION = NuGet-Latest 'OpenTelemetry.Instrumentation.Process'
# $OTEL_GRPC_VERSION = NuGet-Latest 'OpenTelemetry.Instrumentation.GrpcNetClient'
# $OTEL_SQL_VERSION = NuGet-Latest 'OpenTelemetry.Instrumentation.SqlClient'
```

If any version fails to resolve, do not proceed with installation.

## Package matrix

Install the base packages in every app using the resolved versions:

```bash
dotnet add package OpenTelemetry.Extensions.Hosting --version "$OTEL_HOSTING_VERSION"
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol --version "$OTEL_OTLP_VERSION"
```

Add only the packages that match the app (using their resolved versions):

- Web server, Minimal API, MVC, gRPC server: `OpenTelemetry.Instrumentation.AspNetCore`
- Outbound `HttpClient`: `OpenTelemetry.Instrumentation.Http`
- Runtime metrics: `OpenTelemetry.Instrumentation.Runtime`
- Process metrics: `OpenTelemetry.Instrumentation.Process`
- Outbound gRPC client: `OpenTelemetry.Instrumentation.GrpcNetClient`
- Database or cache libraries: the library-specific package if one exists

If a library does not have an `OpenTelemetry.Instrumentation.<Library>` package, look for a native `ActivitySource` or `Meter` instead of inventing wrapper code.

## One registration method

Keep all telemetry registration in one helper so every host uses the same pattern:

```csharp
using System.Reflection;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

namespace MyApp.Observability;

public static class OpenTelemetryExtensions
{
    public static IServiceCollection AddAppTelemetry(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var serviceName =
            configuration["OTEL_SERVICE_NAME"] ??
            Assembly.GetEntryAssembly()?.GetName().Name ??
            "app";
        var resourceBuilder = ResourceBuilder.CreateDefault()
            .AddService(serviceName)
            .AddEnvironmentVariableDetector();

        services.AddOpenTelemetry()
            .ConfigureResource(resource => resource
                .AddService(serviceName)
                .AddEnvironmentVariableDetector())
            .WithTracing(tracing => tracing
                // Keep only the lines that match this app.
                .AddAspNetCoreInstrumentation(options => options.RecordException = true)
                .AddHttpClientInstrumentation()
                // .AddGrpcClientInstrumentation()
                // .AddSqlClientInstrumentation()
                // .AddSource("Npgsql")
                .AddOtlpExporter())
            .WithMetrics(metrics => metrics
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddRuntimeInstrumentation()
                .AddProcessInstrumentation()
                .AddOtlpExporter());

        services.AddLogging(logging => logging.AddOpenTelemetry(options =>
        {
            options.SetResourceBuilder(resourceBuilder);
            options.AddOtlpExporter();
        }));

        return services;
    }
}
```

Web apps call it like this:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAppTelemetry(builder.Configuration);
```

Worker or console apps call it the same way:

```csharp
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddAppTelemetry(builder.Configuration);
```

If the app is not an ASP.NET Core host, remove the ASP.NET Core lines. The rule is simple: only register the instrumentations that match the host and libraries actually in use. Keep the `AddLogging(...AddOpenTelemetry(...))` bridge when you want log export and trace/log correlation.

## Library rule

For each library, pick the first matching case:

1. **`OpenTelemetry.Instrumentation.<Library>` exists**
   Install it and call the matching registration method.

   ```csharp
   .AddSqlClientInstrumentation()
   .AddGrpcClientInstrumentation()
   ```

2. **The library emits a native `ActivitySource` or `Meter`**
   Subscribe to it directly.

   ```csharp
   .AddSource("Npgsql")
   .AddMeter("Some.Library")
   ```

3. **The library needs an SDK-specific opt-in**
   Enable that opt-in where the client is constructed, then subscribe to the source it emits.

That is the default .NET decision tree. Most libraries fit one of those three cases.

Browse supported contrib instrumentations at <https://github.com/open-telemetry/opentelemetry-dotnet-contrib/tree/main/src>.

## Short appendix: exceptions worth remembering

### Entity Framework Core

Do **not** add `OpenTelemetry.Instrumentation.EntityFrameworkCore`. It duplicates the provider span. Instrument the actual provider instead:

- PostgreSQL: `.AddSource("Npgsql")`
- SQL Server: `.AddSqlClientInstrumentation()`
- MongoDB: subscribe the Mongo driver diagnostics source

### Npgsql

Default to:

```csharp
.AddSource("Npgsql")
```

Use `Npgsql.OpenTelemetry` only when you build `NpgsqlDataSource` yourself and need its extra tracing configuration. For EF Core `UseNpgsql()`, plain `.AddSource("Npgsql")` is the safer default.

### gRPC server

There is no separate gRPC server instrumentation package. gRPC server spans come from `OpenTelemetry.Instrumentation.AspNetCore`.

If you want RPC attributes such as `rpc.system`, enable:

```bash
OTEL_DOTNET_EXPERIMENTAL_ASPNETCORE_ENABLE_GRPC_INSTRUMENTATION=true
```

Outbound gRPC is separate and uses `OpenTelemetry.Instrumentation.GrpcNetClient`.

### MongoDB.Driver

MongoDB needs a driver hook when the client is built:

```bash
dotnet add package MongoDB.Driver.Core.Extensions.DiagnosticSources
```

```csharp
settings.ClusterConfigurator = cb =>
    cb.Subscribe(new DiagnosticsActivityEventSubscriber());
```

Then subscribe in OTel:

```csharp
.AddSource("MongoDB.Driver.Core.Extensions.DiagnosticSources")
```

### Azure Cosmos DB SDK

Cosmos spans do not appear unless tracing is enabled on the client:

```csharp
new CosmosClient(endpoint, key, new CosmosClientOptions
{
    CosmosClientTelemetryOptions = new CosmosClientTelemetryOptions
    {
        DisableDistributedTracing = false
    }
});
```

Then subscribe to:

```csharp
.AddSource("Azure.Cosmos.Operation", "Azure.Cosmos.Request")
```

`Microsoft.EntityFrameworkCore.Cosmos` is still a special limitation: EF-mediated Cosmos calls do not currently produce db spans.

## When a library is unclear

Check in this order:

1. `OpenTelemetry.Instrumentation.<Library>` in OTel .NET contrib
2. The library docs for `ActivitySource` / `Meter` names
3. A library-owned `*.OpenTelemetry` or `*.DiagnosticSources` package

If none of those exist, the library is not covered by standard auto-instrumentation.
