# Custom Node Observability Platform

## 1. Background

We need a control plane that can launch custom workload nodes and expose health and performance data in a uniform way. OpenTelemetry (OTel) supplies the APIs and SDKs required to emit traces, metrics, logs, and baggage while keeping cross-node correlations intact via shared context propagation (specification/overview.md:49, specification/overview.md:351).

## 2. Goals

- Instrument every node type with supported OTel APIs and SDKs so telemetry remains portable and conformant (specification/overview.md:61).
- Provide centralized configuration that bootstraps nodes with consistent telemetry defaults while permitting per-node overrides.
- Build a pipeline that aggregates, enriches, and exports telemetry without losing resource context (specification/resource/sdk.md:20, specification/overview.md:355).
- Deliver visual workflows that highlight component slowdowns by combining RED metrics, traces, and logs.

## 3. Non-Goals

- Creating a proprietary telemetry protocol. We rely on OTLP and supported exporters (specification/metrics/data-model.md:205).
- Replacing existing storage or dashboard backends; assume an external observability stack.
- Managing third-party auto-instrumentation lifecycles beyond configuration hooks.

## 4. Requirements

### 4.1 Functional

- Bootstrap nodes with mandatory resource attributes (service, environment, version) and optional launch metadata using `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_SERVICE_NAME` (specification/resource/sdk.md:37, specification/configuration/sdk-environment-variables.md:172).
- Ensure every node exports traces, metrics, and logs to a co-located collector agent through OTLP/gRPC (specification/overview.md:353).
- Supply declarative configuration profiles that can be selected per node class (specification/configuration/README.md:34).
- Surface latency and error hotspots via derived metrics (histograms, rate/error/duration) using views (specification/metrics/data-model.md:140).

### 4.2 Non-Functional

- Keep telemetry overhead under 5% CPU and 10% network, enforced through batching and sampling policies (specification/trace/sdk.md:561).
- Support multi-region deployments with highly available collector clusters.
- Make all configuration changes auditable and version-controlled.

## 5. Architecture Overview

Telemetry data path: Node → Local Collector Agent → Regional Collector Tier → Observability Backends.  
Configuration path: Repository-managed declarative configs → Launcher renders per-node settings → Node bootstrap scripts consume rendered config.

## 6. Component Design

### 6.1 Node Launcher & Runtime

- Launcher accepts a node descriptor covering runtime image, environment variables, exporter endpoints, and sampling rules.
- Bootstrap sequence: provision compute, inject environment-derived resource attributes, start the application with OTel SDK wiring, and launch a sidecar or host daemon collector agent.
- Combine default resource detector output with launcher-supplied attributes using merge semantics so high-cardinality data can be excluded when necessary (specification/resource/sdk.md:71).

### 6.2 Telemetry Instrumentation

- Wrap critical component operations in spans with descriptive names and enforce parent-based sampling so remote sampling decisions propagate (specification/trace/sdk.md:563).
- Emit metrics via synchronous and asynchronous instruments, registering views to standardize histogram buckets and attribute projections (specification/metrics/data-model.md:140).
- Bridge existing logging libraries into the OTel log SDK so logs include trace context and resource metadata (specification/logs/README.md:365).

### 6.3 Configuration Control Plane

- Author declarative configuration files that describe providers, propagators, exporters, processors, and views; store them alongside launch recipes (specification/configuration/README.md:44).
- A configuration service validates schemas, resolves defaults, and renders per-node manifests (YAML or JSON) consumed at startup.
- Provide CLI and API workflows for promoting configurations through environments with validation hooks (for example, linting views and detecting resource collisions).

### 6.4 Collector Tier

- **Agent mode:** runs beside each node, receiving OTLP from the application, performing batching, optional tail-based sampling, log parsing, and secure forwarding (specification/overview.md:363, specification/logs/README.md:395).
- **Regional collectors:** a horizontally scaled cluster that exposes receivers (OTLP, Prometheus scrape, filelog), processors (memory limiter, attributes scrubber, spanmetrics), and exporters (OTLP, Prometheus Remote Write, log sinks) (specification/metrics/data-model.md:152).
- Centralize schema translation so downstream systems receive normalized telemetry (specification/schemas/README.md:189).

### 6.5 Storage & Visualization

- Integrate collectors with existing observability stacks (for example Tempo/Jaeger, Prometheus/Mimir, Loki/Elasticsearch).
- Build dashboards keyed on resource attributes to compare nodes by role, region, and release.
- Use exemplars to link metric outliers with representative traces when available.

### 6.6 Observability & Alerting

- Health checks on collectors (receiver backlog, dropped telemetry counts).
- Alerts on missing heartbeats (no telemetry from a node for N minutes) and RED metric thresholds.
- Surface SDK and collector warnings (invalid environment variables, unknown enum values) to operations consoles (specification/configuration/sdk-environment-variables.md:161).

## 7. Deployment & Operations

- Package node runtime and collector agent together (sidecar or host daemon model).
- Use infrastructure-as-code to roll out collectors with environment-specific but consistent configurations.
- Version resource schemas and configuration bundles; require compatibility testing before promotion.
- Apply blue/green rollouts for configuration changes, using canary nodes to verify telemetry output before global rollout.

## 8. Risks & Mitigations

- **High-cardinality resource attributes** can inflate storage. Enforce naming guidelines and run lint checks on configs (specification/resource/sdk.md:37).
- **Misconfigured propagators** break trace linkage. Default to `tracecontext,baggage` and validate overrides (specification/configuration/sdk-environment-variables.md:177, specification/overview.md:349).
- **Sampling gaps** can obscure incidents. Monitor sampled-versus-total span counts and adjust ratios with remote sampling endpoints (specification/trace/sdk.md:592).

## 9. Open Questions

- Which visualization stack(s) should we standardize on for latency heatmaps?
- Do we need custom collector processors or is the community set sufficient?
- How should remote sampling configurations be distributed—through the collector, a dedicated service, or a vendor backend?

## 10. Next Steps

1. Prototype the node bootstrap flow for one node type and validate telemetry export end to end.
2. Draft initial collector configurations (agent and regional) in the repository and run integration tests.
3. Define dashboards and alerts for RED metrics and trace sampling health.
