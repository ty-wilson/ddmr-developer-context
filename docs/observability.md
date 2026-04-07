# Observability

Last reviewed: 2026-04-07 (updated metrics accuracy)

## Grafana Dashboards

Dashboard JSON files live in the `grafana-dashboards` repo under `grafana-dashboards/DDmR/`. Each file corresponds to one logical view:

| File | Contents |
|---|---|
| `Scoping-Engine.json` | HTTP request latency/throughput per endpoint, JVM heap/non-heap, CPU, running pod count |
| `Scoping-Engine-Events.json` | Pulsar inbound/outbound event rates, per-topic backlogs, group-change and device-sync processing detail |
| `Declaration-Service.json` | Declaration Service HTTP and JVM metrics |
| `Declaration-Storage.json` | Declaration Storage Service HTTP and JVM metrics |
| `Declaration-Storage-Events.json` | Declaration Storage event processing metrics |
| `Tenant-Authorizer.json` | Tenant Authorizer metrics |
| `Service-Logs.json` | Cross-service log viewer (Loki datasource), filterable by service, namespace, and pod |

Dashboards use Prometheus as their primary datasource and Loki for logs. Common label dimensions are `env`, `namespace`, `region`, `service`, and `pod`.

### Linking a Service to Its Dashboards

The Backstage `catalog-info.yaml` annotation `grafana/dashboard-selector` controls which dashboards appear on a service's Backstage page. The value is a Grafana tag query:

```yaml
grafana/dashboard-selector: "tags @> 'scoping-engine'"
```

Each dashboard that should appear for a service must have the matching tag applied in Grafana. The scoping-engine dashboards use the tag `scoping-engine`; declaration-storage-service dashboards use `dss`; declaration-service dashboards use `declaration-service`.

## Metrics

### Stack

Services use Micrometer with the Prometheus registry (`io.micrometer:micrometer-registry-prometheus`) exposed via Spring Boot Actuator. Distributed tracing uses `io.micrometer:micrometer-tracing-bridge-otel`.

### HTTP Metrics

Spring Boot auto-instruments HTTP handlers. The standard metric is `http_server_requests_seconds`, labeled with `uri`, `method`, `status`, and `outcome`. Dashboards show average latency, p90 latency, max latency, and response code breakdowns per endpoint.

### Event Processing Metrics

Each Pulsar event handler has a dedicated top-level metrics class (`DeviceGroupChangedMetrics`, `DeviceSyncMetrics`, `DeviceChannelChangedMetrics`, `ApiRequestEventMetrics`) that records:

- **Event age** — time between when the message was published and when processing started, recorded as a `DistributionSummary` histogram. Metric names follow the pattern `<service>.<event-type>.event.age` (e.g., `device.group.event.age`, `device.sync.event.age`, `device.channel.event.age`, `api.request.event.age`).
- **Handler duration** — total wall-clock time for `processEvent()`, recorded as a `Timer`. Metric names follow the pattern `<service>.<event-type>.event.process.duration` (e.g., `device.group.event.process.duration`, `device.sync.event.process.duration`, `device.channel.event.process.duration`).
- **Distribution summaries** — histograms over per-event payload sizes, such as group counts (`device.sync.event.group.distribution`), scope counts (`device.sync.event.scope.distribution`), assignment counts (`device.sync.event.assignment.distribution`), sync request detail (`device.group.event.sync.request.distribution`), and sync action detail (`device.group.event.sync.detail.distribution`).
- **Counters** — discrete outcome counts such as `device.sync.event.deployable.type.skipped.count` and `device.sync.event.deployable.type.sync.count` (tagged with `type` and `result`).

All histograms are registered with `publishPercentileHistogram()` so Prometheus stores bucket data for quantile queries. Timers additionally call `publishPercentiles()`.

Metric names use dot-separated lowercase words and follow the convention `<domain>.<event-type>.<measurement>.<unit-or-kind>`.

### Pulsar Backlog Metrics

Pulsar broker metrics (`pulsar_subscription_back_log_no_delayed`, `pulsar_subscription_delayed`) are scraped separately and shown in the Events dashboards alongside application-level metrics. Subscriptions are named `scoping-engine-<topic>-<env>`, e.g., `scoping-engine-device-group-changed-dev`.

### JVM / Infrastructure Metrics

Standard JVM metrics (`jvm_memory_used_bytes`, `process_cpu_usage`, `kube_pod_status_phase`) are surfaced in the main service dashboard.

## Logging

### MDC Context

`MDCFilter` (a Spring `WebFilter`) populates the MDC at the start of every HTTP request:

- `tenantId` — from `X-TenantId` header
- `trace_id` — from the incoming trace-id header
- `span_id` — from the incoming span-id header

For Pulsar event handlers, `AbstractEventHandler` (in `service/`) propagates the MDC into coroutines via `MDCContext()` and calls `MDCHelper.addTenantId()` from the event payload before any handler logic runs. This ensures that tenant context is present on every log line throughout the lifetime of an event.

### Log Shipping

Logs are shipped to Loki. The `Service-Logs.json` Grafana dashboard provides a cross-service log viewer using the Loki datasource. Queries filter on `service`, `namespace`, and `pod` labels and support free-text filtering.

### Conventions

- Use `LoggerFactory.getLogger(ClassName::class.java)` for per-class loggers.
- `EventTraceLogger` emits a `TRACE`-level line for every received event, including the full event payload. This is off by default and intended for debugging.
- Structured fields flow via MDC into whatever JSON log format is configured; avoid embedding tenant or trace identifiers as raw string interpolation inside log messages.

## SonarQube

Code quality analysis runs in CI and is linked to Backstage via the `sonarqube.org/project-key` annotation. Project keys follow the pattern `com.jamf.ddm:<service-name>`:

```yaml
sonarqube.org/project-key: com.jamf.ddm:scoping-engine
```

Quality gate results are visible on the Backstage component page. All production services are expected to pass the quality gate before merging.

## ReportPortal

Smoke test results are published to ReportPortal under the `jamf_capabilities` project. Two annotations on `catalog-info.yaml` control the integration:

```yaml
reportportal.io/project-name: "jamf_capabilities"
reportportal.io/launch-name: "scoping-engine-stable-dev-smoke-tests"
```

`reportportal.io/project-name` identifies the ReportPortal project. `reportportal.io/launch-name` identifies which test launch to surface on the Backstage component page. Launch names typically follow the pattern `<service>-<environment>-smoke-tests` (e.g., `DSS-production-smoke-tests` for declaration-storage-service).

## Backstage Catalog

`catalog-info.yaml` at the root of each service repository is the single place that ties a service to all its observability tooling. The annotations used across DDmR services:

| Annotation | Purpose |
|---|---|
| `grafana/dashboard-selector` | Tag query that links Grafana dashboards to this component |
| `sonarqube.org/project-key` | SonarQube project for code quality gate results |
| `reportportal.io/project-name` | ReportPortal project for smoke test results |
| `reportportal.io/launch-name` | Specific launch name within the ReportPortal project |
| `argocd/app-selector` | ArgoCD application selector for deployment status |
| `jira/project-key` | Jira project for issue tracking (all DDmR services use `DDMR`) |

Other non-observability annotations present in most services include `argo-workflows/namespace`, `github.com/project-slug`, and `jsm/team`.

When adding a new service, register all of the above annotations so that Grafana dashboards, SonarQube results, and ReportPortal launches are reachable from a single Backstage component page.
