# CI/CD Pipeline

Last reviewed: 2026-04-07

## Release Flow Overview

The end-to-end path from code merge to live deployment follows this sequence:

1. A PR is merged to `main` in the service repository (e.g. `scoping-engine`).
2. The `ci.yml` GitHub Actions workflow triggers on push to `main`.
3. CI runs tests, builds a container image via Jib, packages the Helm chart, and runs a release step.
4. The `release` job (using `jamf/github-actions-highway-to-prod/release@v2`) updates the `version` SHA in the service's ApplicationSet entry inside the `components` repo.
5. ArgoCD detects the change to the `components` repo and syncs the new version to each targeted cluster according to the ApplicationSet generators.
6. The container image is pulled from ECR (registry `359585083818.dkr.ecr.us-east-1.amazonaws.com`) and the Helm chart is deployed.

The short SHA (7 characters of `GITHUB_SHA`) is the canonical version token. It appears as the `version` field in the ApplicationSet and as the OCI chart tag `0.0.0+<sha>`.

## GitHub Actions Workflows

All workflows live in `.github/workflows/`. The main ones for a service like `scoping-engine`:

- **`ci.yml`** — triggers on push to `main` and `workflow_dispatch`. `reusable-java-build-test.yml` and `reusable-build-image.yml` run in parallel. `reusable-helm-validate-package.yml` waits on `reusable-build-image.yml`. The release step waits on all three. Also calls `reusable-api-export.yml` after release and `reusable-notify-failure.yml` on failure.
- **`reusable-java-build-test.yml`** — sets up Java 21 (Liberica), starts a local DynamoDB container on port 8000, runs `./gradlew test`, publishes a JUnit test report, and runs SonarQube analysis via `jacocoTestReport sonar`.
- **`reusable-build-image.yml`** — runs `./gradlew :jib` to build and push the container image to ECR, generates a short SHA, extracts the version tag from `build/resources/main/META-INF/build-info.properties`, and signs the image with cosign using KMS key `awskms:///arn:aws:kms:us-east-1:359585083818:alias/cosign`.
- **`reusable-helm-validate-package.yml`** — validates the Helm chart against ArgoCD using `jamf/github-actions-helm/validate-argocd@v3` (passes `ARGOCD_COMPONENTS_VIEWER` and `GHA_PSV_READ_PRIVATE_KEY`), then packages and pushes the chart OCI artifact to ECR via `jamf/github-actions-helm/package@v2`.
- **`check-shared-values.yml`** — runs on every PR using `jamf/github-actions-highway-to-prod/check-shared-values@v2`. Checks that the `shared-values-version` file in the repo matches the latest SHA from `platform-shared-values`. A failure blocks merging if branch protection rules require the check to pass.
- **`manual-deployment.yml`** — `workflow_dispatch` only. Deploys a specific image tag directly to the sandbox cluster (`use1-ddmr-sbox231207`, account `183197288009`) or perf namespace via `helm upgrade --install`, bypassing ArgoCD. Used for ad-hoc testing.
- **`branch-build.yml`** — builds images for feature branches without triggering a release or updating the components repo.

### `shared-values-version` file

Each service repo contains `values/shared-values-version`, a plain text file holding a 7-character SHA pinning the version of `platform-shared-values` the service expects. For example, `scoping-engine` currently pins `6ed3b09`. The `check-shared-values.yml` workflow verifies this on PRs.

## components repo: ApplicationSet per Service

The `components` repo holds one `ApplicationSet` YAML per service under `components/`. Each ApplicationSet drives ArgoCD to deploy the service across all targeted clusters.

### Generator blocks

Each generator block targets one cluster using a `clusters.selector.matchLabels` filter on `jamf.com/cluster-name`. The `values` stanza sets:

- `env` — logical environment name (dev, stage, integration, prod)
- `batch` — deployment batch (dev, stage, prod); controls metacluster rollout ordering
- `namespace` — Kubernetes namespace (e.g. `ddmr-stage`, `ddmr-prod`)
- `region` — AWS region string
- `version` — 7-char git SHA of the service image to deploy
- `sharedValuesVersion` — 7-char git SHA of `platform-shared-values` to use
- `enabled` — whether ArgoCD should actively sync this Application

For `scoping-engine` the generator blocks cover:

| Cluster | env | batch | namespace | region |
|---|---|---|---|---|
| `use2-platformsvc-dev240524` | dev | dev | ddmr-dev | us-east-2 |
| `use1-jprosvc-stage240110` | stage | stage | ddmr-stage | us-east-1 |
| `use1-jprosvc-stage240110` | integration | stage | ddmr-integration | us-east-1 |
| `use2-platformsvchc-stage260108` | stage | stage | ddmr-stage | us-east-2 |
| `use1-jprosvc-prod240110` | prod | prod | ddmr-prod | us-east-1 |
| `euc1-jprosvc-prod240110` | prod | prod | ddmr-prod | eu-central-1 |
| `apne1-jprosvc-prod240110` | prod | prod | ddmr-prod | ap-northeast-1 |

### Application template

The Application name is `<service>-<cluster-name>-<env>`. Labels include `batch`, `metacluster-automation: 'true'`, and `metacluster.sync-priority: '100'`.

Each Application uses three sources:

1. **Helm chart** from ECR: `359585083818.dkr.ecr.us-east-1.amazonaws.com`, chart path `helm/jamf/ga/ddm/<service>`, `targetRevision: 0.0.0+<version>`.
2. **Service repo** (`ref: component`): `https://github.com/jamf/<service>.git` at `targetRevision: <version>`. Provides per-env value files referenced as `$component/values/values-<env>-<region>.yaml`.
3. **platform-shared-values** (`ref: sharedvalues`): `https://github.com/jamf/platform-shared-values.git` at `targetRevision: <sharedValuesVersion>`. Provides cluster-level and service-specific shared values.

Value file layering order (last wins):
1. `$component/values/values-<env>-<region>.yaml` — service-level env/region overrides
2. `$sharedvalues/values/<cloud>/<metapartition>/<env>/<metaregion>/<region>/values.yaml` — cluster-level shared values
3. `$sharedvalues/values/<cloud>/<metapartition>/<env>/<metaregion>/<region>/<service>/values.yaml` — service-specific shared values for that cluster (optional, `ignoreMissingValueFiles: true`)

## components-resources repo: Per-Service Metadata

`components-resources/components/<service>/values.yaml` holds ArgoCD project and RBAC metadata consumed by the platform's ArgoCD project provisioner. A typical entry looks like:

```yaml
maintainers:
- "Engineering - DDMR devs"
- "Github_DDmR_Admin"
- "AWS-Engineering-DDMR-ArgoCD"
admins:
- "Engineering - DDMR devs"
nameOverride: "scoping-engine"
gitHubOrg: "jamf"
useSharedValues: "true"
```

`useSharedValues: "true"` tells the provisioner to wire up platform-shared-values access for the ApplicationSet. The `maintainers` and `admins` lists map to GitHub teams / AWS IAM groups that get ArgoCD RBAC permissions on the project. `nameOverride` sets the ArgoCD project name if it differs from the directory name.

## platform-shared-values: Shared Cluster Config

`platform-shared-values` is a central repo of Helm values that are independent of any single service. Values are organized by path:

```
values/aws/<metapartition>/<env>/<metaregion>/<region>/values.yaml
values/aws/<metapartition>/<env>/<metaregion>/<region>/<service>/values.yaml
```

`metapartition` is either `commercial` or `hc`. The root `values.yaml` for each partition sets `metaPartition: commercial` or `metaPartition: hc`.

Values at each level typically set AWS account IDs, regions, region-short names, cluster names, logging config, and OpenTelemetry endpoints. For example, `values/aws/commercial/stage/values.yaml` sets `aws.accountId: "767397914463"` and `applicationProperties.jamf.platform.deployment: STAGE`.

The `sharedValuesVersion` SHA in each ApplicationSet generator pins the exact commit of `platform-shared-values` to use. This is decoupled from the service version so shared infrastructure values can be updated independently.

## HC Pipeline Differences

The HC (High Compliance / FedRAMP) deployment track is a separate AWS account and cluster with its own constraints.

- **AWS account**: `604006981984` (vs. `767397914463` for commercial stage)
- **Cluster**: `use2-platformsvchc-stage260108`
- **Region**: `us-east-2` (commercial stage uses `us-east-1`)
- **Batch**: `stage` (same batch label as commercial stage, but different cluster)
- **Shared values path**: `values/aws/hc/stage/us/us-east-2/values.yaml`

The HC cluster's shared values set `aws.cluster: use2-platformsvchc-stage260108`, `aws.region: us-east-2`, and point OpenTelemetry to `tracing-collector.observability-tracing.svc.cluster.local:4318`.

Service-specific HC overrides live at `values/aws/hc/stage/us/us-east-2/<service>/values.yaml`. For scoping-engine this file configures the DynamoDB table name, Pulsar broker URL (`platform-event-bus.stage.platform-hc.jamflabs.io`), M2M token URLs pointing to `us1.stage.platform-hc.jamflabs.io`, and the IAM role ARN `arn:aws:iam::604006981984:role/ddmr-scoping-engine-role`. It also sets `auth.env: hc-stage` for OIDC verification.

The HC generator block in the ApplicationSet is structurally identical to other generators — same `version` and `sharedValuesVersion` SHAs — but targets the HC cluster name, so ArgoCD treats it as a distinct Application.

## Batch Strategy

The `batch` label on each ApplicationSet Application drives the metacluster automation rollout order. Batches in use for DDMR services:

- **`dev`** — development cluster only (`use2-platformsvc-dev240524`). Receives updates immediately.
- **`stage`** — stage cluster (`use1-jprosvc-stage240110`), integration namespace on the same cluster, and the HC stage cluster (`use2-platformsvchc-stage260108`). Updated after dev.
- **`prod`** — all production clusters (us-east-1, eu-central-1, ap-northeast-1). Updated after stage gates pass.

The `metacluster-automation: 'true'` label opts the Application into automated progressive delivery. `metacluster.sync-priority: '100'` controls relative ordering within a batch.

## ddmr-jenkins

`ddmr-jenkins` is a Jenkins shared library (Groovy). It lives at `/vars/*.groovy` and `/lib/`. The `vars/` scripts are the callable pipeline steps:

- `updateStaging.groovy` — expects the `ddmr-deployments` repo to already be checked out; does a `git pull` to get the latest, then uses `yq` to write new `container.repo` and `container.tag` into `values-stage.yaml` and `values-stable-dev.yaml`, then commits and pushes. Used by older Jenkins pipelines that write directly to `ddmr-deployments` instead of `components`.
- `updateProduction.groovy` — same pattern but updates `values-integration.yaml` and `values-prod.yaml`. Requires an `approver` parameter.
- `syncArgoCD.groovy` — downloads the `argocd` CLI from `argo.jamf.build`, then calls `argocd app sync <name>` followed by `argocd app wait <name>` to block the pipeline until the sync is healthy.
- `checkoutDeployments.groovy`, `runComponentTests.groovy`, `runSystemTests.groovy`, `captureComponentTestResults.groovy`, `captureSystemTestResults.groovy`, `createLocalDynamoTable.groovy`, `readDynamoJson.groovy`, `updatePerformance.groovy`, `updateSandbox.groovy` — additional pipeline utilities for test execution and non-ArgoCD deployments.

The `lib/` directory contains bundled Groovy JAR dependencies (Groovy 5.0.0-alpha-1).

## ddmr-deployments

`ddmr-deployments` is an older Helm-values repository that predates the `components` / ApplicationSet pattern. It is still used for a small number of tooling applications (scope-membership-tool, mdm-tool, tenant-migration jobs) that are not yet migrated to the components pattern.

Structure:
- `argo/apps/` — ArgoCD ApplicationSet YAMLs for the remaining tools. These use a `list` generator (explicit cluster/namespace entries) rather than the `clusters` selector pattern, and point directly at `helm/` subdirectories in this repo as their source. Example: `scope-membership-tool-appset.yaml`, `mdm-tool-appset.yaml`, `tenant-migration-job-appset.yaml`, `tenant-authorizer-appset.yaml`.
- `helm/` — Helm chart values organized by tool, with per-env value files (`values-all.yaml`, `values-prod.yaml`, `values-stable-dev.yaml`, etc.).

For tools in `ddmr-deployments`, updates flow through `updateStaging.groovy` / `updateProduction.groovy` in `ddmr-jenkins` (writing new image tags directly into these value files) rather than through the GitHub Actions CI release step.
