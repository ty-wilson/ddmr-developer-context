# Kubernetes

Last reviewed: 2026-04-07

## Helm Chart Structure

Each DDmR service owns its chart under `helm/<service-name>/`:

```
helm/scoping-engine/
├── Chart.yaml          # name, description, version (set to 0.0.0+versionHash at release)
├── values.yaml         # base defaults — env-agnostic config
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── autoscale.yaml
    ├── application-properties.yaml
    ├── service.yaml
    ├── service-account.yaml
    ├── service-monitor.yaml
    ├── logging.yaml
    ├── alerts.yaml
    ├── goldensignals.yaml
    └── ingress-sbox-m2m.yaml
```

`Chart.yaml` uses `version: 0.0.0+versionHash` and `appVersion: versionHash` as placeholders; the CI pipeline substitutes the real commit-based version at release time.

## Per-Environment Values Files

Environment-specific overrides live in `values/` at the repo root:

```
values/
├── shared-values-version               # commit SHA of platform-shared-values overlay
├── values-dev-us-east-2.yaml
├── values-integration-us-east-1.yaml
├── values-perf-us-east-1.yaml
├── values-sbox-us-east-1.yaml
├── values-stage-us-east-1.yaml
├── values-prod-us-east-1.yaml
├── values-prod-eu-central-1.yaml
└── values-prod-ap-northeast-1.yaml
```

Naming convention: `values-<env>-<region>.yaml`. Each file opens with a comment directing readers to `helm/<service>/values.yaml` for non-env-specific config.

## Values Layering

Helm merges values in order — later files win:

1. `helm/<service>/values.yaml` — base defaults (resource sizes, metric filters, sidecar image tag, scaling defaults)
2. `values/values-<env>-<region>.yaml` — env-specific overrides (DynamoDB table name, m2m env, Pulsar credentials, broker/crypto secret references, IAM role ARN, Loki/Tempo URLs, scaling min/max)
3. Platform-shared-values overlay — injected by ArgoCD at deploy time; fills in `aws.cluster`, `aws.region`, and other platform-wide values

The per-env file is the primary place to change things like replica bounds, tracing endpoints, consumer group suffix, and CloudWatch/Loki logging targets.

## shared-values-version

The file `values/shared-values-version` contains a single git commit SHA (e.g. `6ed3b09`). It pins which version of the platform-shared-values chart overlay ArgoCD should apply on top of the service's own values. Bumping this SHA pulls in platform-level changes (new labels, updated platform defaults, etc.).

## Pod Topology

Each pod runs two containers:

**service** (`container.*` values)
- Spring Boot application on port 8080
- `SPRING_PROFILES_ACTIVE` set to `<service-name>,jsonlog`
- `JAVA_TOOL_OPTIONS` injected from `container.options` (e.g. `-XX:InitialRAMPercentage=40 -XX:MaxRAMPercentage=40`)
- `SPRING_PULSAR_CONSUMER_NAME` / `SPRING_PULSAR_PRODUCER_NAME` sourced from `metadata.name` (pod name) for unique Pulsar consumer identity
- Liveness/readiness via `/actuator/health/liveness` and `/actuator/health/readiness`; a startup probe guards against premature liveness checks
- Default resources: 500m CPU request / 1500m limit, 550Mi mem request / 650Mi limit (overridden per service — scoping-engine uses 900Mi for both)
- `appProperties` rendered as a ConfigMap and mounted at `/app/config`

**auth** (`auth.*` values) — JWT sidecar
- Runs the shared `jamf/ga/ddm/jwt` image on port 7070
- `MICRONAUT_ENVIRONMENTS` set to `k8s,<auth.env>` (e.g. `k8s,stage,stage-alt` or `k8s,prod-use1`)
- Acts as a reverse proxy: terminates JWT/M2M/CSA authentication before traffic reaches the service container
- `authProperties` rendered as a separate ConfigMap and mounted at `/config`
- Default resources: 250m CPU (request and limit), 64Mi memory (request and limit)

Replicas are not set in the Deployment — they are controlled entirely by a KEDA `ScaledObject` (see `autoscale.yaml`).

## Autoscaling (KEDA)

The `ScaledObject` in `autoscale.yaml` drives horizontal pod autoscaling. Triggers include:
- CPU and memory utilization (default 85%)
- HTTP requests per second (Prometheus query against Thanos)
- Pulsar messages per second per topic (separate threshold per topic)
- Event age (average age of in-flight group events) — used in prod to catch processing lag
- Event processing duration per topic — catches extreme degradation

`scaling.min` and `scaling.max` in the per-env values file control replica bounds (e.g. prod-us-east-1 uses min 4, max 45; dev uses min 2, max 5).

## IAM / Service Account

Each service has a `ServiceAccount` (`<service>-acct`) annotated with an IRSA role ARN:

```yaml
eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<service-role>
```

The ARN is set per environment in the `serviceAccount.role` value. This grants the pod AWS permissions (DynamoDB, Secrets Manager) without static credentials.

## Logging

The `logging.yaml` template creates Banzai Cloud `Flow` and `Output` resources:
- **CloudWatch** — enabled in all deployed environments; log group path is `/aws/eks/<cluster>/<namespace>/<service>`
- **Loki** — enabled when `logging.loki.url` is set; URL varies by environment (e.g. `http://loki-internal.us-east-1.stage.observability.jamf.build`)

Both outputs are active simultaneously in deployed environments.

## Namespace

- **HC (dev/integration/stage/perf):** `ddmr-stage` namespace
- **Commercial prod:** namespace varies by region/cluster; injected via the platform-shared-values overlay at deploy time

The `Release.Namespace` variable in templates always resolves to the correct namespace for the target cluster — don't hardcode it.

## Backstage catalog-info.yaml

Every service has a `catalog-info.yaml` at the repo root that registers it in Backstage. Key annotations:

| Annotation | Purpose |
|---|---|
| `argocd/app-selector` | Links to ArgoCD application (e.g. `app=scoping-engine`) |
| `grafana/dashboard-selector` | Finds dashboards by tag (e.g. `tags @> 'scoping-engine'`) |
| `jira/project-key` | Links to Jira project (e.g. `DDMR`) |
| `sonarqube.org/project-key` | Links to SonarQube analysis (e.g. `com.jamf.ddm:scoping-engine`) |
| `reportportal.io/project-name` | ReportPortal project for test results (`jamf_capabilities`) |
| `reportportal.io/launch-name` | ReportPortal launch name for smoke tests |
| `github.com/project-slug` | GitHub repository slug |
| `argo-workflows/namespace` | Namespace for Argo workflow runs |

The `pact-consumer-name` label (e.g. `scoping-engine`) is used by Pact contract testing. `spec.providesApis` and `spec.consumesApis` document the service's API contracts within the Backstage service catalog.
