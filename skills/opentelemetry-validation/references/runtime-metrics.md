# Runtime Metrics Query

Returns the runtime metrics for a given instrumented service.

Prerequisites:

- Observe CLI
- `generate-opal` skill for producing metric OPAL
- `Service List` query to determine the language / runtime of the service

Data source:

- `Metrics/OpenTelemetry` for OpenTelemetry runtime metrics

Use `generate-opal` to build the metric query for the dataset below, then run it with the filters and metric families in this reference.

## Dataset: `Metrics/OpenTelemetry`

Required filters:

- `resource_attributes."service.name"` = `<service-name>`
- `resource_attributes."deployment.environment.name"` = `<environment>`
- add `resource_attributes."service.namespace"` when known

Expected metric families by language:

- `Java`: `jvm.memory.used`, `jvm.memory.limit`, `jvm.memory.committed`
- `Node.js`: `v8js.gc.duration`, `v8js.memory.heap.used`, `v8js.memory.heap.limit`, `nodejs.eventloop.delay.p99`
- `Python`: `process.runtime.cpython.gc_count`, `process.runtime.cpython.thread_count`
- `.NET`:
  - If the app uses .NET 9+:
    - `dotnet.gc.collections`,
    - `dotnet.gc.pause.time`,
    - `dotnet.thread_pool.thread.count`,
    - `dotnet.exceptions`
  - If the app uses .NET <9:
    - `process.runtime.dotnet.gc.collections.count`,
    - `process.runtime.dotnet.gc.pause.time`,
    - `process.runtime.dotnet.thread_pool.thread.count`,
    - `process.runtime.dotnet.exceptions.count`
- `Go`:
  - `process.runtime.go.mem.heap_alloc`,
  - `process.runtime.go.mem.heap_objects`,
  - `process.runtime.go.gc.pause_ns.sum`,
  - `process.runtime.go.gc.pause_ns.count`,
  - `process.runtime.go.goroutines`,
  - `process.runtime.go.cgo.calls`
- `Ruby`: no standard runtime metrics. Skip this check.
