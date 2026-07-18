# OpenTelemetry Python Auto-Instrumentation Reference

Python auto-instrumentation is powered by [`opentelemetry-distro`](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/opentelemetry-distro) and the [`opentelemetry-instrument`](https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/opentelemetry-instrumentation) CLI.

The CLI (`opentelemetry-bootstrap`) automatically detects installed libraries (Django, Flask, psycopg2, requests, etc.) and instruments them at runtime — **do NOT manually list individual instrumentation packages** like `opentelemetry-instrumentation-django` as dependencies.

## Resolve latest versions

Before installing anything, query PyPI for the latest stable versions of the core auto-instrumentation packages. Use these resolved versions in all subsequent install commands.

#### POSIX shell (`sh`, `bash`, `zsh`)

```sh
OTEL_DISTRO_VERSION=$(curl -fsSL https://pypi.org/pypi/opentelemetry-distro/json |
  python3 -c "import sys,json; print(json.load(sys.stdin)['info']['version'])")
OTEL_EXPORTER_VERSION=$(curl -fsSL https://pypi.org/pypi/opentelemetry-exporter-otlp/json |
  python3 -c "import sys,json; print(json.load(sys.stdin)['info']['version'])")

echo "opentelemetry-distro==${OTEL_DISTRO_VERSION}"
echo "opentelemetry-exporter-otlp==${OTEL_EXPORTER_VERSION}"
```

#### PowerShell

```powershell
$OTEL_DISTRO_VERSION = (Invoke-RestMethod 'https://pypi.org/pypi/opentelemetry-distro/json').info.version
$OTEL_EXPORTER_VERSION = (Invoke-RestMethod 'https://pypi.org/pypi/opentelemetry-exporter-otlp/json').info.version

Write-Host "opentelemetry-distro==$OTEL_DISTRO_VERSION"
Write-Host "opentelemetry-exporter-otlp==$OTEL_EXPORTER_VERSION"
```

If either version fails to resolve, do not proceed with installation.

## Step 1: Install Dependencies

Install `opentelemetry-distro` and `opentelemetry-exporter-otlp` using the resolved versions, then bootstrap the auto-instrumentation packages for all detected libraries. The bootstrap step must be re-run whenever dependencies change.

**pip:**

```bash
pip install opentelemetry-distro=="${OTEL_DISTRO_VERSION}" opentelemetry-exporter-otlp=="${OTEL_EXPORTER_VERSION}"
opentelemetry-bootstrap -a install
```

**poetry:**

```bash
poetry add "opentelemetry-distro==${OTEL_DISTRO_VERSION}" "opentelemetry-exporter-otlp==${OTEL_EXPORTER_VERSION}"
poetry run opentelemetry-bootstrap -a install
```

**uv:**

```bash
uv add "opentelemetry-distro==${OTEL_DISTRO_VERSION}" "opentelemetry-exporter-otlp==${OTEL_EXPORTER_VERSION}"
uv run opentelemetry-bootstrap -a requirements | uv add --requirement -
```

`opentelemetry-bootstrap -a install` does not work with uv's package management. The piped command generates the requirements dynamically and feeds them to `uv add`. Re-run this after every `uv sync` or package update.

If a Dockerfile is present for the app, the bootstrap command must also be part of the image build process.

## Step-2: Production Deployment with Pre-fork Servers

Pre-fork servers like Gunicorn spawn multiple worker processes. Forking creates inconsistencies in the background threads and locks used by OpenTelemetry SDK components (specifically `PeriodicExportingMetricReader`), which breaks metric export. Traces and logs are unaffected.

### Option-1 (Recommended): Gunicorn with UvicornWorker (ASGI apps)

For ASGI apps (FastAPI, Starlette, aiohttp), deploy with `uvicorn.workers.UvicornWorker` which preserves background threads across forks:

```bash
opentelemetry-instrument gunicorn \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  myapp.main:app
```

### Option-2 (Alternative): Programmatic auto-instrumentation

Initialize OpenTelemetry inside each worker process after the fork instead of using the `opentelemetry-instrument` CLI wrapper:

```python
from opentelemetry.instrumentation.auto_instrumentation import initialize
initialize()

from your_app import app
```

For FastAPI, `initialize()` must be called **before** importing `FastAPI` because instrumentation patches are applied at import time:

```python
from opentelemetry.instrumentation.auto_instrumentation import initialize
initialize()

from fastapi import FastAPI

app = FastAPI()
```

With this approach, the app can run without the `opentelemetry-instrument` wrapper. For example:

```bash
uvicorn main:app --workers 2
```

## Known Issues

| Issue                                      | Affected Frameworks                         | Impact                                                                                                                       |
| ------------------------------------------ | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Dev server reloader breaks instrumentation | Django, Flask, FastAPI (uvicorn `--reload`) | No spans emitted from reloaded process; use `--noreload` / `use_reloader=False`                                              |
| Gunicorn multi-worker breaks metrics       | All (via Gunicorn)                          | `PeriodicExportingMetricReader` thread not forked; use UvicornWorker or single worker                                        |
| `gcc` / `gcc-c++` required on slim Linux   | All                                         | `pip install` fails for native extensions; install `python3-dev` and `build-essential` (Debian/Ubuntu) or `gcc-c++` (CentOS) |
