# Kubernetes

Last reviewed: 2026-07-29. This pass removed drifting tuning constants (resource sizes, JVM percentages, scaling bounds, utilization thresholds) and replaced transcribed file trees with the commands that regenerate them. Confirmed on that date: the `helm/<service>/` + `values/values-<env>-<region>.yaml` layout and the template set in scoping-engine. NOT re-verified and older (carried from the 2026-04-07 review): the Backstage annotation table, the IAM/ServiceAccount section, and the logging section.

## Helm Chart Structure

Each DDmR service owns its chart under `helm/<service-name>/`: `Chart.yaml`, a base `values.yaml`, and a `templates/` directory. The template set is per-service and drifts. Some services carry templates others don't (e.g. declaration-storage-service has both a general `ingress.yaml` and `ingress-sbox-m2m.yaml`, where scoping-engine has only the sbox one). List it rather than trusting a tree here:

```bash
ls helm/*/templates/
```

**Trap: `Chart.yaml` version lies in the repo.** It is committed as `version: 0.0.0+versionHash` / `appVersion: versionHash`, literal placeholder strings, not a real semver. CI substitutes the commit-based version at release time. A local `helm template` or `helm lint` therefore shows `0.0.0`, and any tooling that parses the chart version off `main` gets the placeholder, not what is deployed. Get the deployed version from ArgoCD or the pod image tag instead.

## Per-Environment Values Files

Environment-specific overrides live in `values/` at the repo root, named `values-<env>-<region>.yaml` (plus a `shared-values-version` file). Each file opens with a comment directing readers to `helm/<service>/values.yaml` for non-env-specific config. The set of environments differs per service:

```bash
ls values/
```

## Values Layering

Helm merges values in order (later files win):

1. `helm/<service>/values.yaml`: base defaults (resource sizes, metric filters, sidecar image tag, scaling defaults)
2. `values/values-<env>-<region>.yaml`: env-specific overrides (DynamoDB table name, m2m env, Pulsar credentials, broker/crypto secret references, IAM role ARN, Loki/Tempo URLs, scaling min/max)
3. Platform-shared-values overlay: injected by ArgoCD at deploy time; fills in `aws.cluster`, `aws.region`, and other platform-wide values

The per-env file is the primary place to change things like replica bounds, tracing endpoints, consumer group suffix, and CloudWatch/Loki logging targets. This precedence rule is the durable part. Every concrete number below layer 1 is expected to drift, so read it from the file, never from this doc.

## shared-values-version

The file `values/shared-values-version` contains a single git commit SHA. It pins which version of the platform-shared-values chart overlay ArgoCD should apply on top of the service's own values. Bumping this SHA pulls in platform-level changes (new labels, updated platform defaults, etc.), so a service can pick up a platform change with no diff in its own chart, and two services on different SHAs can behave differently with identical charts. Check the pin before concluding a platform default applies:

```bash
cat values/shared-values-version
```

## Pod Topology

A pod runs the service container plus, **only if `.Values.auth` is set**, the `ddmr-jwt-sidecar` container. Services have been migrating to an in-pod `JwtFilter` instead (since DDMR-1088), so do not assume two containers. `auth-and-tenancy.md` owns that question; check a specific pod with `kubectl get pod -o jsonpath='{.spec.containers[*].name}'`.

**service** (`container.*` values)
- Spring Boot application on port 8080
- `SPRING_PROFILES_ACTIVE` set to `<service-name>,jsonlog`
- `JAVA_TOOL_OPTIONS` injected from `container.options`: this is where `InitialRAMPercentage` / `MaxRAMPercentage` are set (`grep -n options helm/*/values.yaml`)
- `SPRING_PULSAR_CONSUMER_NAME` / `SPRING_PULSAR_PRODUCER_NAME` sourced from `metadata.name` (pod name) for unique Pulsar consumer identity
- Liveness/readiness via `/actuator/health/liveness` and `/actuator/health/readiness`; a startup probe guards against premature liveness checks
- `appProperties` rendered as a ConfigMap and mounted at `/app/config`
- Resource requests/limits come from the layering above: base chart defaults, overridden per env

**Trap: some services set memory request == memory limit, so there is no burst headroom.** When request and limit are equal the pod cannot borrow memory from the node under a spike. It OOMKills instead. That makes `MaxRAMPercentage` load-bearing: it must be sized against the *limit*, and a heap percentage that is safe on a service with headroom will kill one without it. Before changing either, read both `container.resources` and `container.options` from the same values file and check whether request and limit match.

**auth** (`auth.*` values): JWT sidecar
- Runs the shared `jamf/ga/ddm/jwt` image on port 7070
- `MICRONAUT_ENVIRONMENTS` set to `k8s,<auth.env>` (e.g. `k8s,stage,stage-alt` or `k8s,prod-use1`)
- Acts as a reverse proxy: terminates JWT/M2M/CSA authentication before traffic reaches the service container
- `authProperties` rendered as a separate ConfigMap and mounted at `/config`
- Sized much smaller than the service container; values in `auth.resources`

Replicas are not set in the Deployment. They are controlled entirely by a KEDA `ScaledObject` (see `autoscale.yaml`).

## Autoscaling (KEDA)

The `ScaledObject` in `autoscale.yaml` drives horizontal pod autoscaling. The trigger mix spans several kinds, and the count differs per service and per environment (`observability.md` notes scoping-engine's `scoping-engine-scaling` object carries 9 triggers). Kinds in use:

- CPU and memory utilization
- HTTP requests per second (Prometheus query against Thanos)
- Pulsar messages per second, one trigger per topic with its own threshold
- **Event-age lag detectors**: average age of in-flight events (e.g. `device_group_event_age`, `device_sync_event_age`)
- Event processing duration per topic

**Trap: the age-based triggers are the non-obvious ones.** They fire on *backlog growth*, which a `msg/sec` trigger cannot see. A slow consumer with a flat arrival rate keeps every rate trigger under threshold while the age trigger climbs. Conversely, do not conclude "KEDA isn't scaling" from one metric being quiet; the HPA aggregates all triggers. See `observability.md` for the `ScalingLimited: TooManyReplicas` reading.

Never enumerate triggers from this doc. Read them from the cluster:

```bash
kubectl --context <ctx> -n <ns> get scaledobject <service>-scaling -o yaml
```

`scaling.min` and `scaling.max` in the per-env values file control replica bounds. Prod bounds are roughly an order of magnitude above dev; read the actual numbers from `values/values-<env>-<region>.yaml`.

## IAM / Service Account

Each service has a `ServiceAccount` (`<service>-acct`) annotated with an IRSA role ARN:

```yaml
eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<service-role>
```

The ARN is set per environment in the `serviceAccount.role` value. This grants the pod AWS permissions (DynamoDB, Secrets Manager) without static credentials. `infrastructure.md` owns account IDs and role names.

## Logging

The `logging.yaml` template creates Banzai Cloud `Flow` and `Output` resources:
- **CloudWatch**: enabled in all deployed environments; log group path is `/aws/eks/<cluster>/<namespace>/<service>`
- **Loki**: enabled when `logging.loki.url` is set; URL varies by environment (e.g. `http://loki-internal.us-east-1.stage.observability.jamf.build`)

Both outputs are active simultaneously in deployed environments.

**Trap: a Loki `Output` existing does not mean DDmR logs are searchable in Loki.** Per `reference_ddmr_logs_opensearch`, DDmR service logs are queried through Unified Logging OpenSearch in practice. Check `observability.md` before building a log query around Loki.

## Namespace

The namespace comes from the ApplicationSet generator (the `namespace` value in `components/<service>.yaml`), not from the chart. Observed values include `ddmr-dev`, `ddmr-stage`, `ddmr-integration`, and `ddmr-prod`; read the generator block for the authoritative mapping rather than trusting a list here (`cicd-pipeline.md` covers the generator).

**Trap: namespace alone does not identify the partition.** `ddmr-stage` is used by *both* commercial stage and HC stage, on different clusters. Always pair the namespace with the cluster when reasoning about where something runs, and note that other docs give per-env namespaces for their own purposes (`observability.md` for log queries, `infrastructure.md` for IAM) which may reflect a single environment rather than the full set.

The `Release.Namespace` variable in templates always resolves to the correct namespace for the target cluster, so don't hardcode it.

## Backstage catalog-info.yaml

Every service has a `catalog-info.yaml` at the repo root that registers it in Backstage. Key annotations:

| Annotation | Purpose |
|---|---|
| `argocd/app-selector` | Links to ArgoCD application (e.g. `app=scoping-engine`) |
| `grafana/dashboard-selector` | Finds dashboards by tag (e.g. `tags @> 'scoping-engine'`) |
| `jira/project-key` | Links to Jira project (e.g. `DDMR`) |
| `jsm/team` | JSM team assignment (e.g. `DDmR`) |
| `sonarqube.org/project-key` | Links to SonarQube analysis (e.g. `com.jamf.ddm:scoping-engine`) |
| `reportportal.io/project-name` | ReportPortal project for test results (`jamf_capabilities`) |
| `reportportal.io/launch-name` | ReportPortal launch name for smoke tests |
| `github.com/project-slug` | GitHub repository slug |
| `argo-workflows/namespace` | Namespace for Argo workflow runs |

The `pact-consumer-name` label (e.g. `scoping-engine`) is used by Pact contract testing. `spec.providesApis` and `spec.consumesApis` document the service's API contracts within the Backstage service catalog.

---

## Where to find the data (verify rather than trust)

Every number this doc used to quote is one command away. Run these instead.

```bash
# 1. What templates and environments does this chart actually have?
ls helm/*/templates/ values/

# 2. Resource requests/limits and JVM heap percentages, base vs per-env.
#    Check whether memory request == limit (the no-headroom trap above).
grep -n -A6 'resources:' helm/*/values.yaml
grep -rn 'MaxRAMPercentage\|InitialRAMPercentage\|memory:\|cpu:' values/*.yaml

# 3. Replica bounds per environment: never quote these, diff them.
grep -rn -A3 'scaling:' values/*.yaml

# 4. Live KEDA triggers (count and kind), plus what the HPA is actually deciding.
kubectl config get-contexts -o name          # e.g. dev, stage, integ, prod-use1, hc-stage
kubectl --context prod-use1 -n ddmr-prod get scaledobject scoping-engine-scaling -o yaml
kubectl --context prod-use1 -n ddmr-prod get hpa

# 5. Real pod topology: one container or two (service + ddmr-jwt-sidecar)?
kubectl --context stage -n ddmr-stage get pods -l app=scoping-engine \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'

# 6. The platform overlay pin: a platform change can land with no chart diff.
cat values/shared-values-version
```

If a rendered manifest disagrees with this doc, the manifest wins. Render it locally to see the merge result:

```bash
helm template helm/<service> -f helm/<service>/values.yaml -f values/values-<env>-<region>.yaml
```

Note that the local render omits the ArgoCD-injected platform-shared-values layer, so `aws.cluster` / `aws.region` and other platform values will be empty.
