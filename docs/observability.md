# Observability

Last reviewed: 2026-07-29. Re-verified against code on that date: every log-message string below (grepped against `origin/main` of `scoping-engine` and `declaration-storage-service`), the metric-name registration sites, the `grafana-dashboards/DDmR/` file list, and the KEDA ScaledObject source. Not re-verified and older: the Backstage/SonarQube/ReportPortal annotation sections, the dev-cluster names and AWS/Secrets Manager paths, and the OpenSearch-vs-Loki dev routing. Treat those as a pointer only.

## Grafana Dashboards

Dashboard JSON lives in the `grafana-dashboards` repo under `grafana-dashboards/DDmR/`. One file per logical view:

| File | Contents |
|---|---|
| `Scoping-Engine.json` | HTTP request latency/throughput per endpoint, JVM heap/non-heap, CPU, running pod count |
| `Scoping-Engine-Events.json` | Pulsar inbound/outbound event rates, per-topic backlogs, group-change and device-sync processing detail |
| `Declaration-Service.json` | Declaration Service HTTP and JVM metrics |
| `Declaration-Storage.json` | Declaration Storage Service HTTP and JVM metrics |
| `Declaration-Storage-Events.json` | Declaration Storage event processing metrics |
| `Tenant-Authorizer.json` | Tenant Authorizer metrics. **Only integration reports data**: the service was removed from prod/stage/dev/perf in DDMR-1085, so an empty panel with `env` set to anything else is expected, not an outage. |
| `Service-Logs.json` | Cross-service log viewer (Loki datasource), filterable by service, namespace, and pod |

Dashboards use Prometheus (via Thanos) as their primary datasource and Loki for logs. Common label dimensions are `env`, `namespace`, `region`, `service`, and `pod`.

### Linking a service to its dashboards

The Backstage `catalog-info.yaml` annotation `grafana/dashboard-selector` holds a Grafana tag query:

```yaml
grafana/dashboard-selector: "tags @> 'scoping-engine'"
```

Each dashboard that should appear for a service must carry the matching tag **in Grafana**. The tag is not in the JSON-repo filename. Tags in use: `scoping-engine`, `dss` (declaration-storage-service), `declaration-service`.

## Metrics

### Stack

Micrometer with the Prometheus registry (`io.micrometer:micrometer-registry-prometheus`) exposed via Spring Boot Actuator. Tracing via `io.micrometer:micrometer-tracing-bridge-otel`.

### HTTP metrics

Spring Boot auto-instruments HTTP handlers. The standard metric is `http_server_requests_seconds`, labeled `uri`, `method`, `status`, `outcome`.

### Event processing metrics

Micrometer registration lives **inline in each Pulsar handler class** (`DeviceGroupChangedHandler`, `DeviceSyncHandler`, `DeviceChannelChangedHandler`, `ApiRequestEventHandler`, `DeviceGroupDivisionChangedHandler`, …), not in separate metrics classes. Don't transcribe the list, regenerate it:

```bash
git -C ~/Projects/DDmR/scoping-engine grep -n 'event\.age\|event\.process\.duration\|\.distribution\|\.count"' origin/main -- '*.kt' | grep -v test
```

The durable part is the **naming convention**, which every handler follows:

- `<domain>.<event-type>.event.age`: publish-to-processing-start latency, a `DistributionSummary`. e.g. `device.group.event.age`, `device.sync.event.age`, `device.channel.event.age`, `api.request.event.age`.
- `<domain>.<event-type>.event.process.duration`: wall-clock `processEvent()` time, a `Timer`.
- `<domain>.<event-type>.event.<thing>.distribution`: histograms over per-event payload sizes (group counts, scope counts, assignment counts, sync request/detail).
- `<domain>.<event-type>.event.<outcome>.count`: discrete counters, usually tagged `type` and `result`.

Dot-separated lowercase words throughout. Histograms register `publishPercentileHistogram()`; timers add `publishPercentiles()`.

**Trap: the convention keeps growing, so a hand-copied list goes stale silently.** Priority and dimension variants get their own metric *name* rather than a tag. For example `device.sync.bulk.event.age` sits alongside `device.sync.event.age`, and `device.group.division.event.age` was added later. A dashboard or KEDA trigger written against the base name will not see the variant's traffic at all.

**Trap: Micrometer dot-names are mangled on the Prometheus side.** `device.group.event.age` is scraped as `device_group_event_age_sum` / `_count` / `_bucket`. PromQL and the KEDA queries in `helm/scoping-engine/templates/autoscale.yaml` use the underscore form; the Kotlin source uses the dot form. Grepping for one will not find the other.

### Pulsar backlog metrics

**`docs/event-layer.md` is the canonical reference for the backlog/rate metric distinctions** (`pulsar_subscription_back_log_no_delayed` vs `pulsar_subscription_back_log` vs `pulsar_subscription_delayed` vs `pulsar_msg_backlog` vs `pulsar_rate_in` vs `pulsar_subscription_msg_rate_out`). Read that table there rather than a second copy here.

Two nuances that matter when you are debugging rather than designing:

- `pulsar_msg_backlog` is **per topic and includes every subscription, including abandoned/zombie ones**. A topic can show a large backlog while the subscription you actually care about is empty.
- **Trap: "reset cursor to latest" does not clear the delay queue.** Resetting a cursor (e.g. via EMB) drops the visible cursor position, but delayed messages keep maturing and re-feed a retry loop, so the backlog reappears. The cleanest force-drain is `pulsar-admin topics skip-all-messages`.

Subscriptions are named `scoping-engine-<topic>-<env>`, e.g. `scoping-engine-device-group-changed-dev`.

### JVM / infrastructure metrics

`jvm_memory_used_bytes`, `process_cpu_usage`, `kube_pod_status_phase`, surfaced in the main service dashboards.

### Alerting

Prometheus alerts and recording rules are defined **in each service's own Helm chart**, not centrally. For scoping-engine, `helm/scoping-engine/templates/alerts.yaml` renders a `monitoring.coreos.com/v1` `PrometheusRule`. Two things to know:

- The whole template is wrapped in `{{ if .Values.alertsEnabled }}`, so **alerts exist only in environments whose values set it.** "No alert fired" in a lower env usually means no rule was rendered.
- Recording rules use the `ddmr:se:<subject>:<agg>:<window>` prefix (e.g. `ddmr:se:podrestarts:increase:30m`). Grep `alerts.yaml` for the current rule and threshold list instead of quoting them.

## Logging

### MDC context

`MDCFilter` (a Spring `WebFilter`) populates the MDC at the start of every HTTP request: `tenantId` (from the `X-TenantId` header), `trace_id`, and `span_id` (from the incoming trace/span headers).

For Pulsar handlers, `AbstractEventHandler` propagates the MDC into coroutines via `MDCContext()` and calls `MDCHelper.addTenantId()` from the event payload before any handler logic runs, so tenant context is present on every log line for the life of an event.

### Shipping and conventions

Logs ship to Loki, viewable via the `Service-Logs.json` dashboard, filterable on `service` / `namespace` / `pod`. See the dev caveat under the playbook below: dev does not follow this.

- Per-class loggers via `LoggerFactory.getLogger(ClassName::class.java)`.
- `EventTraceLogger` emits a `TRACE` line per received event including the full payload. Off by default; debugging only.
- Structured fields flow via MDC. Do not interpolate tenant or trace identifiers into message strings.

## SonarQube

Analysis runs in CI and links to Backstage via `sonarqube.org/project-key`, following `com.jamf.ddm:<service-name>`:

```yaml
sonarqube.org/project-key: com.jamf.ddm:scoping-engine
```

Gate results show on the Backstage component page. Whether the gate actually *blocks* a merge is per-repo branch protection, not a platform-wide rule, so check the required status checks rather than assuming:

```bash
gh api repos/jamf/scoping-engine/branches/main/protection/required_status_checks --jq '.contexts'
```

## ReportPortal

Smoke test results publish to ReportPortal under the `jamf_capabilities` project. Two `catalog-info.yaml` annotations drive it: `reportportal.io/project-name: "jamf_capabilities"` and `reportportal.io/launch-name`, which selects which launch surfaces on the Backstage page (e.g. `"scoping-engine-stable-dev-smoke-tests"`). Launch names generally follow `<service>-<environment>-smoke-tests`, e.g. `DSS-production-smoke-tests`.

## Backstage catalog

`catalog-info.yaml` at each service repo root is the single place tying a service to all of its observability tooling:

| Annotation | Purpose |
|---|---|
| `grafana/dashboard-selector` | Tag query linking Grafana dashboards to this component |
| `sonarqube.org/project-key` | SonarQube project for quality-gate results |
| `reportportal.io/project-name` | ReportPortal project for smoke test results |
| `reportportal.io/launch-name` | Specific launch within the ReportPortal project |
| `argocd/app-selector` | ArgoCD application selector for deployment status |
| `jira/project-key` | Jira project (all DDmR services use `DDMR`) |

Also present in most services: `argo-workflows/namespace`, `github.com/project-slug`, `jsm/team`.

When registering a new service, set all of the above so dashboards, Sonar results, and ReportPortal launches are all reachable from one Backstage page.

---

## Troubleshooting playbook: tenant-scoped investigation in stable-dev

Use this when a tenant is reported broken (M2M errors, declarations not delivered, sync stalled). Symptoms cross service boundaries, so read logs across all relevant services before forming a hypothesis.

### 1. Get access

- `aws sso login --profile stable_dev` (interactive). DDmR dev AWS account: `381491946762`.
- `kubectl config use-context platformsvc-dev`: dev EKS cluster `use2-platformsvc-dev240524`. DDmR services run in namespace `ddmr-dev`.
- VPN must be up for the cluster API and internal Jamf endpoints to resolve.

### 2. Locate the tenant across services

Check **all three** of scoping-engine, declaration-service, and declaration-storage before assuming where the problem lives. Different consumer paths exercise different code.

**Trap: in dev, DDmR logs are in Unified Logging OpenSearch, not Loki.** The Loki shards `loki-dev-us` and `loki-dev-eu` do **not** contain the `ddmr-dev` namespace. Use the `unified-logging-opensearch` Grafana datasource.

**Trap: the OpenSearch `component` value is hyphenated.** Lucene `component:"scoping-engine"` works; `component:"scopingengine"` silently returns zero hits and looks like "no logs, so it never ran." Other fields: `environment:"dev"`, `tenantId:"<uuid>"`.

`kubectl logs` is fine for recent-only diagnostics, but the per-pod kubelet ring buffer rolls in **hours, not days** (a busy pod can lose a morning's logs by lunch), so treat anything beyond a few hours as gone unless the pod has been quiet:

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

Tenants present in DSS but **not** in scoping-engine indicate a direct-product-API caller (e2e tests, scope tools). Not all paths involve scoping-engine.

### 3. Distinguish inbound vs outbound auth failures

| Log signature | Direction | Source |
|---|---|---|
| 401 with `JwtProxyFilter` / `No tenant identifier` | Inbound (caller's JWT rejected) | `ddmr-jwt-sidecar` (sidecar pod), or in-pod `JwtFilter` for DSS/declaration-service |
| `Received unexpected status code '<X>' from M2M service when requesting an access token` | Outbound (this service can't get a token) | Robocop `HalClient` (logger `com.jamf.stratus.m2m.robocop.clients.HalClient`) |
| `Failed to acquire M2M token` + `M2M error, retrying` | Outbound, scoping-engine | `M2MService.kt` and `DeployableObjectSyncAdapter.kt` |

**Trap: HAL's status code is the only diagnostic you get from the client side.** Robocop's `HalClient` logs the status code and **not the response body**, so to see *why* HAL refused you must curl HAL directly or read HAL pod logs (owned by Stratus). The codes do carry meaning:

- **400** = malformed request (bad scope, unknown tenant binding, bad scopes).
- **401** = client credentials rejected.

Reading 400 as "credentials are wrong" sends you looking at the wrong secret.

### 4. Discriminate a retry storm from an outage

Pulsar consumers use `Key_Shared`. When one event for one device-channel hits a permanent failure (e.g. `M2MTokenAcquisitionException(retryable=true)`), it loops on the same pod and emits thousands of identical lines for a single tenant.

```bash
# Distinct devices in the failure stream. If it's one, it's a stuck event, not an outage
kubectl --context $CTX -n $NS logs --since=24h <pod> | grep "$TENANT" \
  | grep 'Exception during sync' | jq -r .message | grep -oE "device='[^']+'" | sort -u
```

Workaround for a single stuck event without redeploying: invalidate the per-tenant M2M cache (`M2MCache.invalidate(tenant)` in `scoping-engine/.../service/M2MCache.kt`), which is exposed via actuator if enabled. Otherwise restart the pod holding that key in `Key_Shared`.

### 5. Inspect persisted state in DynamoDB

Dev tables (commercial, us-east-2):

- `ddmr-dev-scoping-engine`: scope, scope-group, membership, device-channel, device-sync (key patterns in `database.md`)
- `ddmr-dev-declaration-storage`: declarations + assignments (`MDM#<tenant>|<device>|<channel>` / `A#<identifier>`)
- `tenant-authorizer`: CSA-tenant resolution (per-env table name; check helm values). Read in prod by DSS in-pod, not by the authorizer service, which runs only in integration (see `database.md`).

```bash
aws dynamodb query --profile stable_dev --region us-east-2 \
  --table-name ddmr-dev-scoping-engine \
  --key-condition-expression "pk = :pk" \
  --expression-attribute-values '{":pk":{"S":"MEMBERSHIP#<tenant>|<device>"}}' \
  --max-items 5
```

For DSS use the per-tenant assignment key pattern; for declarations use the `DECLARATION_INDEX` GSI by declaration ID. `database.md` is authoritative on key names and which GSIs currently exist.

### 6. Probe M2M / HAL directly (when needed)

Dev scoping-engine M2M credentials: Secrets Manager path `ddmr/dev/scoping/sync-credentials` (region `us-east-2`). The HAL token endpoint is derived from `M2MProperties.M2MEnv`. For `DEV` it's `Environment.devUS(Realm.PLATFORM)` (resolving to `us.api.dev.platform.jamflabs.io/m2m/...`). Scopes requested by scoping-engine: `tenant` + `declaration-storage-product` + `app-declaration-service-api-product`.

**Trap: reading the credentials and curling HAL is a credential-handling operation and the auto-mode classifier blocks it by default.** Either add an explicit Bash permission rule for `aws secretsmanager get-secret-value:*` in your Claude settings, or run the curl yourself in a `!` shell and paste the relevant pieces (redacting the secret). Expect to be stopped here rather than treating it as a tooling failure.

### 7. Log signature reference

Line numbers are deliberately omitted: they drift, and the string is the durable identifier. Grep for the string.

| Log string | Meaning | Find it with |
|---|---|---|
| `Received unexpected status code '<X>' from M2M service when requesting an access token` | Robocop `HalClient` got a non-2xx from HAL. Status code only, no body. | logger `com.jamf.stratus.m2m.robocop.clients.HalClient` (library, not DDmR source) |
| `Failed to acquire M2M token` | scoping-engine wrapping the HAL failure in `M2MTokenAcquisitionException` | `grep -rn 'Failed to acquire M2M token' src/main/` (scoping-engine, `M2MService.kt`) |
| `Exception during sync for {...} - M2M error, retrying (...)` | retryable M2M failure; the event will be redelivered | `grep -rn 'Exception during sync for' src/main/` (scoping-engine, `DeployableObjectSyncAdapter.kt`) |
| `No sync needed on (...) for <adapter>` | per-adapter idempotency short-circuit, DEBUG level | `grep -rn 'No sync needed on' src/main/` (scoping-engine, `DeviceSyncHandler.kt`) |
| `No syncs needed for channel(s) [...] on device '...' (successfully synced after event trigger time)` | top-level idempotency short-circuit, DEBUG level | `grep -rn 'No syncs needed for' src/main/` (scoping-engine, `DeviceSyncHandler.kt`) |
| `... (wrong tenant or tag)` | DSS cleanup path: **not auth.** The stored assignment's tenant/tag doesn't match the request, so the row was left alone. | `grep -rn 'wrong tenant or tag' src/main/` (DSS) |

**Trap: `wrong tenant or tag` is emitted from several DSS sites with different verbs.** `Failed to remove assignment`, `Failed to delete assignment`, and `Failed to update assignment` all exist across `ProductApiHandler.kt` and `V2AssignmentApiHandler.kt`, one site interpolates the verb (`Failed to $failedAction ...`), and at least one has a stray leading space. It also appears as a `reason` field in API error responses, not only in logs. **Grep the suffix `(wrong tenant or tag)`, never a full prefix**. A prefix search finds a subset and makes the problem look narrower than it is.

### 8. Common false leads

- **One tenant in a 10-line log paste ≠ only that tenant is affected.** Sample size matters. Compare cardinality over a window with an OpenSearch `terms` aggregation on `tenantId.keyword`, not by eyeballing log samples.
- **DSS "wrong tenant or tag" WARNs are cleanup-path issues**, not delivery failures.
- **High WARN-line rate ≠ high produce rate.** Pulsar redelivery of a NACKed message emits fresh log lines on every redelivery but produces nothing new, so the queue can be perfectly stable while log volume is enormous. Read `pulsar_rate_in` for the topic to get the true produce rate; never infer it from log line counts.
- **Don't conflate the retry-storm tenant with the backlog-growth tenant.** A retry self-feedback loop (see the gotcha in `services/scoping-engine.md`) recirculates events 1:1, so it dominates log volume but contributes zero *net* backlog growth. Net growth comes from a separate source, typically REST API write fan-out via `deviceService.syncDevices` over scope groups. They are usually different tenants.

### 9. Dev Tyk metric label limitations

**Trap: in `use2-platformsvc-dev240524`, `tyk_http_requests_total` series for DDmR APIs omit the `oauth_id`, `alias`, and `path` labels that prod has.** You cannot identify a caller from Tyk metrics in dev. Use traces instead: `net.sock.peer.addr` on the Tyk root span. The `host`/`pod` labels identify only the gateway pod, never the upstream caller.

### 10. KEDA on scoping-engine: read the rendered triggers, not one metric

The `scoping-engine-scaling` ScaledObject has far more triggers than the rate-based ones people assume: CPU, memory, and several Prometheus triggers, some of which are Helm loops over topics, so the *rendered* trigger count varies by environment values. Source of truth is `helm/scoping-engine/templates/autoscale.yaml`; rendered truth is the cluster:

```bash
kubectl --context platformsvc-dev -n ddmr-dev get scaledobject scoping-engine-scaling -o yaml
```

**Trap: the HPA reports `ScalingLimited: TooManyReplicas` when *any* trigger demands more than `maxReplicaCount`.** Do not conclude "KEDA isn't scaling" because the one metric you checked is under threshold, because the *aggregate* desired-replica calculation can be pinned at the cap while no single trigger looks active.

The age-based triggers (`device_group_event_age`, `device_sync_event_age`) act as lag detectors and respond to growing backlog where pure `msg/sec` triggers would not. They query the Prometheus (underscore) form of the Micrometer names, against the delegated Thanos scaling endpoint, not the main Grafana datasource.

---

## Where to find the data (verify rather than trust)

Every metric name, log string, and threshold above is cheaply re-derivable. Prefer these over the prose.

```bash
SE=~/Projects/DDmR/scoping-engine
DSS=~/Projects/DDmR/declaration-storage-service

# Current metric names + registration sites (regenerates the convention list)
git -C $SE grep -n 'event\.age\|event\.process\.duration\|\.distribution' origin/main -- '*.kt' | grep -v test

# Every emitter of a log signature (use the suffix, verbs vary)
git -C $DSS grep -n 'wrong tenant or tag' origin/main -- '*.kt' | grep -v test
git -C $SE  grep -n 'Exception during sync for\|Failed to acquire M2M token' origin/main -- '*.kt' | grep -v test

# KEDA triggers: template source, then what actually rendered in the cluster
git -C $SE show origin/main:helm/scoping-engine/templates/autoscale.yaml
kubectl --context platformsvc-dev -n ddmr-dev get scaledobject scoping-engine-scaling -o yaml

# Alert + recording rules, and whether they are enabled in this env
git -C $SE show origin/main:helm/scoping-engine/templates/alerts.yaml
kubectl --context platformsvc-dev -n ddmr-dev get prometheusrule -o name

# Dashboards that exist, and the tags Backstage resolves against
ls ~/Projects/DDmR/grafana-dashboards/DDmR/
grep -rn 'grafana/dashboard-selector' $SE/catalog-info.yaml
```

Before concluding a metric, alert, or dashboard "doesn't exist", check unmerged work. These repos frequently carry observability changes on ticket branches:

```bash
git -C <repo> fetch origin --quiet
git -C <repo> for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -20
```
