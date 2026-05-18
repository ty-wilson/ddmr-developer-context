# Observability

Last reviewed: 2026-05-18 (added Pulsar/KEDA/dev quirks)

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

## Troubleshooting Playbook: Tenant-Scoped Investigation in stable-dev

Use this when a tenant is reported as having an issue (M2M errors, declarations not delivered, sync stalled). Symptoms cross service boundaries — start with logs across all relevant services before forming a hypothesis.

### 1. Get access

- AWS SSO: `aws sso login --profile stable_dev` (interactive). The DDmR dev AWS account is `381491946762`.
- `kubectl config use-context platformsvc-dev` — the dev EKS cluster (`use2-platformsvc-dev240524`). All DDmR services run in namespace `ddmr-dev`.
- VPN must be connected for the cluster API and internal Jamf endpoints to resolve.

### 2. Locate the tenant across services

Always check **all three** services before assuming where the problem lives. Different consumer paths exercise different code.

**For dev**, prefer Unified Logging OpenSearch over `kubectl logs` — DDmR services in dev ship to OpenSearch, not Loki, and the per-pod kubelet log ring buffer is small enough (often only 1–4 h) that anything past a few hours has rolled. The Grafana datasource is `unified-logging-opensearch`. Lucene query format: `component:"scoping-engine"` (hyphenated — `scopingengine` will return nothing), `environment:"dev"`, `tenantId:"<uuid>"`. Loki shards `loki-dev-us` and `loki-dev-eu` do **not** contain the `ddmr-dev` namespace.

`kubectl logs` is fine for recent-only diagnostics but treat anything older than ~4 h as gone unless the pod's been quiet enough to keep its buffer:

```bash
TENANT=<tenant-uuid>
NS=ddmr-dev
CTX=platformsvc-dev
for svc in scoping-engine declaration-service declaration-storage; do
  for pod in $(kubectl --context $CTX -n $NS get pods -o name | grep "^pod/$svc"); do
    count=$(kubectl --context $CTX -n $NS logs --since=24h "$pod" 2>/dev/null | grep -c "$TENANT")
    echo "$pod : $count"
  done
done
```

Tenants that appear in DSS but **not** in scoping-engine indicate a direct-product-API caller (e2e tests, scope tools); not all paths involve scoping-engine.

### 3. Distinguish inbound vs outbound auth failures

| Log signature | Direction | Source |
|---|---|---|
| 401 with `JwtProxyFilter` / `No tenant identifier` | Inbound (caller's JWT rejected) | `ddmr-jwt-sidecar` (sidecar pod), or in-pod `JwtFilter` for DSS/declaration-service |
| `Received unexpected status code '<X>' from M2M service when requesting an access token` | Outbound (this service can't get a token) | Robocop `HalClient` (logger `com.jamf.stratus.m2m.robocop.clients.HalClient`) |
| `Failed to acquire M2M token` + `M2M error, retrying` | Outbound, scoping-engine | `M2MService.kt:32`, `DeployableObjectSyncAdapter.kt:42` |

HAL response codes carry meaning: **400** = malformed request (bad scope, unknown tenant binding, bad scopes); **401** = client credentials rejected. Robocop 0.0.20's `HalClient` only logs the status code — to see the body you must either curl HAL directly or read HAL pod logs (owned by Stratus).

### 4. Watch for retry-storm patterns

Pulsar consumers use `Key_Shared`. When one event for one device-channel triggers a permanent failure (e.g., `M2MTokenAcquisitionException(retryable=true)`), it loops on the same pod, generating thousands of identical log lines for one tenant. Don't confuse a retry storm with a wider outage:

```bash
# Distinct devices in the failure stream — if it's one, it's a stuck event
kubectl --context $CTX -n $NS logs --since=24h <pod> | grep "$TENANT" | grep 'Exception during sync' | jq -r .message | grep -oE "device='[^']+'" | sort -u
```

Workaround for a single stuck event without redeploying: invalidate the per-tenant M2M cache (`M2MCache.invalidate(tenant)` in `scoping-engine/service/M2MCache.kt`) — exposed via actuator if enabled, otherwise restart the pod that holds that key in `Key_Shared`.

### 5. Inspect persisted state in DynamoDB

Dev tables (commercial, us-east-2):

- `ddmr-dev-scoping-engine` — scope, scope-group, membership, device-channel, device-sync (key patterns in `database.md`)
- `ddmr-dev-declaration-storage` — declarations + assignments (`MDM#<tenant>|<device>|<channel>` / `A#<identifier>`)
- `tenant-authorizer` — CSA-tenant resolution (per-env table name; check helm values)

```bash
aws dynamodb query --profile stable_dev --region us-east-2 \
  --table-name ddmr-dev-scoping-engine \
  --key-condition-expression "pk = :pk" \
  --expression-attribute-values '{":pk":{"S":"MEMBERSHIP#<tenant>|<device>"}}' \
  --max-items 5
```

For DSS use the per-tenant assignment key pattern; for declarations use the `DECLARATION_INDEX` GSI by declaration ID.

### 6. Probe M2M / HAL directly (when needed)

The dev scoping-engine M2M credentials are at Secrets Manager path `ddmr/dev/scoping/sync-credentials` (region `us-east-2`). HAL token endpoint pattern is derived from `M2MProperties.M2MEnv` — for `DEV` it's `Environment.devUS(Realm.PLATFORM)` (resolves to `us.api.dev.platform.jamflabs.io/m2m/...`). Scopes requested by scoping-engine: `tenant` + `declaration-storage-product` + `app-declaration-service-api-product`.

Reading the credentials and curling HAL is a **credential-handling operation** and is blocked by the auto-mode classifier by default. Either:
1. Add an explicit Bash permission rule for `aws secretsmanager get-secret-value:*` in your Claude settings, OR
2. Run the curl yourself in a `!` shell and paste the relevant pieces (redact the secret).

### 7. Log signature reference (scoping-engine M2M path)

| Line | Logger / File |
|---|---|
| `Received unexpected status code '<X>' from M2M service when requesting an access token` | Robocop `HalClient` |
| `Failed to acquire M2M token` (wrapped) | `M2MService.kt:32` |
| `Exception during sync for {...} - M2M error, retrying (...)` | `DeployableObjectSyncAdapter.kt:39-42` |
| `No sync needed on (...) for app` | `DeviceSyncHandler.kt:155` (per-adapter idempotency) |
| `No syncs needed for channel(s) [...] on device '...' (successfully synced after event trigger time)` | `DeviceSyncHandler.kt:259` (top-level idempotency) |
| `Failed to remove assignment ... (wrong tenant or tag)` | DSS `ProductApiHandler.kt:360` — **not** auth; means the stored assignment's tenant/tag doesn't match the request |

### 8. Common false leads

- **One tenant in a 10-line log paste ≠ only that tenant is affected.** Sample size matters. Compare cardinality in a window via an OpenSearch `terms` aggregation on `tenantId.keyword`, not by inspecting log line samples.
- **DSS "wrong tenant or tag" WARNs are cleanup-path issues**, not delivery failures.
- **High WARN-line rate ≠ high produce rate.** Pulsar redelivery of a NACKed message produces fresh log lines on every redelivery but no new produces — the actual queue can be stable even when log volume is enormous. Look at `pulsar_rate_in` for the topic, not log line counts, to know the true produce rate.
- **Don't conflate the retry-storm tenant with the backlog-growth tenant.** A retry self-feedback loop (see `services/scoping-engine.md` gotcha) recirculates events 1:1 — it dominates log volume but contributes zero net backlog growth. Net growth comes from a separate source (typically REST API write fan-out via `deviceService.syncDevices` over scope groups). They're usually unrelated tenants.

### 9. Pulsar backlog & rate metric distinctions

These metrics measure subtly different things — pick carefully when answering "is the backlog growing":

| Metric | Scope | Includes delayed? | What it counts |
|---|---|---|---|
| `pulsar_subscription_back_log_no_delayed` | per subscription | no | ready-to-deliver messages |
| `pulsar_subscription_back_log` | per subscription | yes | all unacked |
| `pulsar_subscription_delayed` | per subscription | only delayed | scheduled future deliveries |
| `pulsar_msg_backlog` | per topic | yes (all subs, incl. zombies) | retained storage |
| `pulsar_rate_in` | per topic | yes (publish time) | produce rate |
| `pulsar_subscription_msg_rate_out` | per subscription | yes (delivery time) | consume rate (incl. redeliveries) |

A "reset cursor to latest" (e.g. by EMB) drops the visible cursor but **does not clear the delay queue** — delayed messages continue maturing and re-feed the retry loop. The cleanest force-drain is `pulsar-admin topics skip-all-messages`.

### 10. Dev Tyk metric label limitations

In `use2-platformsvc-dev240524`, `tyk_http_requests_total` series for DDmR APIs **omit the `oauth_id`, `alias`, and `path` labels** that are present in prod. To identify a caller in dev you have to look at traces (`net.sock.peer.addr` on the Tyk root span), not Tyk metrics. The `host`/`pod` labels on the metric series identify only the gateway pod, not the upstream caller.

### 11. KEDA on scoping-engine: read the actual triggers, not just one

The `scoping-engine-scaling` ScaledObject has **9 triggers**, not just rate-based ones. Inspect with:

```bash
kubectl --context platformsvc-dev -n ddmr-dev get scaledobject scoping-engine-scaling -o yaml
```

Important: the HPA reports `ScalingLimited: TooManyReplicas` when *any* trigger demands more than `maxReplicaCount`. Don't conclude "KEDA isn't scaling" from a single metric being under threshold — the *aggregate* desired-replica calculation may be hitting the cap even if no single trigger looks active. The age-based triggers (`device_group_event_age`, `device_sync_event_age`) act as lag detectors and respond to growing backlog where pure `msg/sec` triggers would not.
