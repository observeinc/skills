# Configuring Auto-Instrumentation

## Environment Variables

The following environment variables are recommended and detected by auto-instrumentation:

| Variable                      | Recommended Value                                                                                                             | Description                                   |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| `OTEL_SERVICE_NAME`           | Desired application name specified by user                                                                                    | Identifier for the application in Observe APM |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf`                                                                                                               | OTLP protocol to use when shipping telemetry  |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4318` for bare metal or VMs, `http://observe-agent-forwarder.observe.svc.cluster.local:4318` for kubernetes | Observe Agent endpoint                        |

Note: some custom instrumentation bootstrappers (e.g. Quarkus for Java, Micronaut for Java) would require specifying some OTLP variables as part of the app configuration files.

### Configuring instrumented applications running on bare metal / VMs

- Instruct the user to set the environment variables according to the target platform. For example (Linux systemd):

  ```text
  [Service]
  Environment="OTEL_SERVICE_NAME=my-app"
  Environment="OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf"
  Environment="OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318"
  ExecStart=/usr/local/bin/my-app
  ```

- Additionally, instruct the user to set the `service.name` in the observe agent configuration file to match the OTEL_SERVICE_NAME so that infrastructure metrics are correlated to the application running on the host. For example, in Linux, `/etc/observe-agent/observe-agent.yaml` should have the following:

  ```yaml
  resource_attributes:
    service.name: my-app
  ```

### Configuring instrumented applications running on Kubernetes

- Instruct the user to add environment variables in the kubernetes manifest for their service. For example:

  ```yaml
  template:
    spec:
      containers:
        - name: app
          image: my-app:latest
          env:
            - name: OTEL_SERVICE_NAME
              value: my-app
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: http/protobuf
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://observe-agent-forwarder.observe.svc.cluster.local:4318
  ```

- Additionally, instruct the user to annotate their application pods so that infrastructure metrics collected by the Observe Agent are associated with the correct service in Service Explorer. The supported annotations are:

  | Annotation                                         | Maps To                  | Required |
  | -------------------------------------------------- | ------------------------ | -------- |
  | `resource.opentelemetry.io/service.name`           | `service_name`           | Yes      |
  | `resource.opentelemetry.io/service.namespace`      | `service_namespace`      | No       |
  | `resource.opentelemetry.io/deployment.environment` | `deployment_environment` | Yes      |

  The annotation values must match the values used in the application's OTel instrumentation for the same service.

  If the `resource.opentelemetry.io/service.name` annotation is not present, the Observe Agent will fall back to the `app.kubernetes.io/name` pod label to determine `service_name` (requires observe-agent Helm chart >= 0.52.0).

  Example pod template with both environment variables and annotations:

  ```yaml
  template:
    metadata:
      annotations:
        resource.opentelemetry.io/service.name: my-app
        resource.opentelemetry.io/service.namespace: my-team
        resource.opentelemetry.io/deployment.environment: production
    spec:
      containers:
        - name: app
          image: my-app:latest
          env:
            - name: OTEL_SERVICE_NAME
              value: my-app
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: http/protobuf
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://observe-agent-forwarder.observe.svc.cluster.local:4318
  ```
