# Service Name Conflict Check

After discovery, check whether a service with the requested `service.name` already exists in Observe. Two unrelated services can share the same name (including across teams), so a match is a possible conflict, not proof that this codebase is already instrumented.

## When to Run

Run this check after the discovery sub-agent returns and before making any instrumentation changes. The check requires:

- The developer's chosen `service.name`
- The target `deployment.environment.name` (ask the developer if not already known)
- Working `observe` CLI access to the target tenant

If the CLI is unavailable or unauthenticated, say so explicitly and treat the conflict check as skipped. Do not imply that the name is clear unless the check actually ran.

## CLI Commands

### List services matching the name

```bash
observe dataset list --filter 'label == "Tracing/Service"' --json
observe dataset view <tracing-service-id> --json
observe query --input <tracing-service-id> --interval 4h --limit 20 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
pick_col @."Valid From", @."Valid To", service_name, environment, service_namespace, language, host_type, service_type, database
sort asc(service_name)'
```

Treat any returned row as a possible name collision.

If `service.namespace` is relevant (the developer specified one, or discovery found one), add `filter service_namespace = "<service-namespace>"`.

### Get more context on a matching service

If a match is found, gather additional context to help the developer decide:

```bash
observe dataset list --filter 'label == "Tracing/Span"' --json
observe dataset view <tracing-span-id> --json
observe query --input <tracing-span-id> --interval 4h --limit 10 --json --pipeline '
filter environment = "<environment>"
filter service_name = "<service-name>"
make_col host_name:string(resource_attributes."host.name"), process_pid:int64(resource_attributes."process.pid"), pod_name:string(resource_attributes."k8s.pod.name"), namespace_name:string(resource_attributes."k8s.namespace.name"), cluster_uid:string(resource_attributes."k8s.cluster.uid"), container_name:string(resource_attributes."container.name"), sdk_lang:string(resource_attributes."telemetry.sdk.language"), sdk_version:string(resource_attributes."telemetry.sdk.version")
make_col kind:if(pod_name != "" and not is_null(pod_name), "kubernetes", "host")
statsby span_count:count(), group_by(kind, host_name, process_pid, pod_name, namespace_name, cluster_uid, container_name, sdk_lang, sdk_version, service_name, service_namespace, service_version, environment)'
```

If `service.namespace` is relevant, add `filter service_namespace = "<service-namespace>"`.

This reveals the runtime language (`sdk_lang`), SDK version, host or pod information, service version, and recent activity. Use that context to determine whether the match is the same codebase or a different service sharing the name.

## Decision Logic

### No match found

Proceed with instrumentation. The name is available.

### Match found

Present the collision to the developer with this information:

1. **Existing service context**: language, SDK version, host/pod, last seen timestamp
2. **Local project context**: language and framework from discovery
3. **Comparison**: highlight whether the runtime stacks match or differ

**Default recommendation:** Rename the service. Suggest alternatives:

- `<name>-<environment>` (e.g. `cart-service-dev`)
- `<name>-<team>` (e.g. `cart-service-billing`)
- `<name>-v2` if this is a rewrite

**Override:** Only continue with the same name if the developer explicitly confirms after seeing the conflict. Document their acknowledgment before proceeding.

## Example Interaction

```text
I found an existing service named "cart-service" in the "production" environment:
- Language: java
- SDK: opentelemetry-java 1.32.0
- Host: ip-172-31-4-50.us-west-2.compute.internal

Your local project is a Python/FastAPI application, which suggests this is a
different service sharing the same name.

I recommend choosing a different service.name to avoid telemetry collision.
Suggestions:
  - cart-service-py
  - cart-api
  - cart-service-v2

Would you like to pick one of these, enter a custom name, or override and
keep "cart-service"?
```
