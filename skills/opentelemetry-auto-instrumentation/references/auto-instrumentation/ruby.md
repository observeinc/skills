# OpenTelemetry Ruby Auto-Instrumentation Reference

Ruby auto-instrumentation is provided by the [`opentelemetry-instrumentation-all`](https://github.com/open-telemetry/opentelemetry-ruby-contrib/tree/main/instrumentation/all) gem. For Rails apps, the meta-package installs `opentelemetry-instrumentation-rails`, `opentelemetry-instrumentation-active_record`, `opentelemetry-instrumentation-pg`, Rack, ActionPack, and related framework instrumentations.

## Resolve latest versions

Before adding gems, query RubyGems for the latest stable versions. Use these resolved versions to pin dependencies in the Gemfile.

#### POSIX shell (`sh`, `bash`, `zsh`)

```sh
OTEL_SDK_VERSION=$(gem search '^opentelemetry-sdk$' --remote --exact | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)
OTEL_ALL_VERSION=$(gem search '^opentelemetry-instrumentation-all$' --remote --exact | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)
OTEL_OTLP_VERSION=$(gem search '^opentelemetry-exporter-otlp$' --remote --exact | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)

echo "opentelemetry-sdk ${OTEL_SDK_VERSION}"
echo "opentelemetry-instrumentation-all ${OTEL_ALL_VERSION}"
echo "opentelemetry-exporter-otlp ${OTEL_OTLP_VERSION}"
```

#### PowerShell

```powershell
$OTEL_SDK_VERSION = (Invoke-RestMethod 'https://rubygems.org/api/v1/versions/opentelemetry-sdk/latest.json').version
$OTEL_ALL_VERSION = (Invoke-RestMethod 'https://rubygems.org/api/v1/versions/opentelemetry-instrumentation-all/latest.json').version
$OTEL_OTLP_VERSION = (Invoke-RestMethod 'https://rubygems.org/api/v1/versions/opentelemetry-exporter-otlp/latest.json').version

Write-Host "opentelemetry-sdk $OTEL_SDK_VERSION"
Write-Host "opentelemetry-instrumentation-all $OTEL_ALL_VERSION"
Write-Host "opentelemetry-exporter-otlp $OTEL_OTLP_VERSION"
```

If any version fails to resolve, do not proceed with installation.

## Ruby Dependencies

The production app should include, pinned to the resolved versions:

```ruby
gem "opentelemetry-sdk", "${OTEL_SDK_VERSION}"
gem "opentelemetry-instrumentation-all", "${OTEL_ALL_VERSION}"
gem "opentelemetry-exporter-otlp", "${OTEL_OTLP_VERSION}"
```

Unlike Java/Node/Python zero-code agents, Ruby configures instrumentation in-process via an initializer, so these gems are production dependencies. They still must not appear in business logic: controllers, models, services, jobs, and serializers should contain no `OpenTelemetry` calls, no manual spans, and no manual exception recording.

## Rails initializer

Create `config/initializers/opentelemetry.rb`:

```ruby
require "opentelemetry/sdk"
require "opentelemetry/instrumentation/all"
require "opentelemetry/exporter/otlp"

OpenTelemetry::SDK.configure do |c|
  c.use_all("OpenTelemetry::Instrumentation::Rake" => { enabled: false })
end
```

**Always disable Rake instrumentation.** The `opentelemetry-instrumentation-rake` gem (included transitively by `opentelemetry-instrumentation-all`) wraps every `Rake::Task#invoke` in a span. For daemon-like rake tasks such as `rake jobs:work` (Delayed Job), `rake resque:work`, or similar long-running workers, this creates a root span that never ends because the task body runs an infinite loop. The `BatchSpanProcessor` only exports spans when they close, so the root span is never exported — all child spans (DB polling, ActiveRecord queries) reference a `parent_span_id` that doesn't exist in the backend, producing orphaned traces.

Require the OTLP exporter explicitly. Rails may auto-require Gemfile entries via `Bundler.require(*Rails.groups)`, but the initializer should not depend on that side effect; non-Rails Ruby apps and gems declared with `require: false` need the explicit require for real exports.
