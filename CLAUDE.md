# DDmR Platform — Service Map

## Services

### Scoping & Targeting
- **scoping-engine** — Scope membership, device-to-group resolution, device sync state. DynamoDB + Pulsar.

### Declarations (DDM)
- **declaration-service** — Declaration CRUD: fragment building, validation, translation, activation. Stateless REST API.
- **declaration-storage-service** — Persists declaration data (DynamoDB). Publishes `declaration-assignment-changed` events.
- **device-declaration-reporting-service** — Reporting backend for declaration status per device.

### Blueprints (UI Orchestration)
- **blueprint-management-service** — Blueprint CRUD, component orchestration. PostgreSQL + Pulsar async deploys.
- **blueprint-components-registry-service** — Component library catalog: fragment names, types, icons.
- **blueprint-component-declarations-service** — Declarations fragment metadata for the component registry.
- **blueprint-component-sw-update-service** — Software update blueprint component backend.
- **blueprint-component-custom-declarations** — Custom declarations component.

### Configuration Profiles
- **configuration-profile-service** — Config profile CRUD, plist handling.
- **configuration-profile-plist-migrator** — Plist migration tooling.

### MDM Schema
- **mdm-schema-ingest-inbound-adapter** — Hourly Lambda: ingests Apple DDM repo → transforms → S3.
- **mdm-schema-ingest-infrastructure** — Terraform/Terragrunt infra for the schema pipeline (ALB, Lambdas, S3, EventBridge).
- **mdm-ui-schema** — UI-specific schema customizations.

### Auth & Tenancy
- **ddmr-jwt-sidecar** — Micronaut JWT validation proxy, deployed as pod sidecar on port 7070.
- **ddmr-authorizer-tenant** — API gateway Lambda: resolves JWT claims → tenant ID via DynamoDB lookup.
- **tenants-odin** — Tenant management service.

### Frontend
- **micro-frontend-hub** — Nx + pnpm monorepo: all MFE apps (declarations, config-profiles, JSFG, app-switcher, etc.)
- **json-schema-form-generator** — Standalone repo for the JSFG component.

### Infrastructure & Shared Libraries
- **ddmr-infrastructure** — OpenTofu + Terragrunt IaC (HC environment). GitHub Actions → Highway-to-Prod.
- **ddmr-terraform** — Legacy Terragrunt IaC (commercial environments). Jenkins.
- **tyk-gateway-management** — Tyk API gateway route definitions (YAML CRDs, per-environment).
- **platform-messaging-client-java** — Pulsar client abstraction (core + Spring integration).
- **declaration-storage-client-core** — Shared client interfaces for Declaration Storage Service.
- **declaration-storage-product-client** / **declaration-storage-mdm-client** — Product and MDM client variants.
- **ddmr-gradle-coordinate-plugin** — Artifact naming, versioning, ECR image helpers.

### CI/CD & Deployment
- **components** — ArgoCD ApplicationSet definitions (one per service, generator blocks per cluster).
- **components-resources** — Per-service metadata (maintainers, useSharedValues, verifyOIDC).
- **platform-shared-values** — Environment-specific config overlaid by ArgoCD (HC values, auth, IAM).
- **ddmr-deployments** — Legacy ApplicationSets for tooling (scope-membership-tool, tenant-migration).
- **ddmr-jenkins** — Groovy shared library for older Jenkins pipelines.

### Cross-Team Services (not DDmR-owned, but interact with DDmR via events or HTTP)
- **app-lifecycle-management-engine-client** — Client lib for VPP/app lifecycle service; called by scoping-engine for app assignments.
- **blueprint-report-aggregation-service** (Ocean) — Consumes device-scope-membership-changed, device-management-channel-changed, apple-ddm/statusreport.
- **blueprint-deployment-service** (Ocean) — Produces blueprint-deployment-task, blueprint-deployment-changed. Handles async blueprint deploys.
- **compliance-benchmark-report-service** (Mars/Red) — Consumes device-group-changed, device-*-changed, device-management-state.
- **compliance-benchmark-engine** (Red) — Produces verified-rules topic.
- **spaghetti-mux** (Pulsaroni) — Pulsar event relay. Consumes declaration-assignment-changed and SCIM events.
- **m2m-robocop** — M2M authentication library used by all DDmR services.
- **jamf-school-helm-apns** (Pixels) — Consumes declaration-assignment-changed.
- **Jamf Pro Server / jamf-messaging** — Produces platform events: device-group-changed, device-management-channel-changed, device-*-state events.
- **device-identity-mapping-service** (Team Rocket) — Consumes device-identity-certificate-issued, device-management-state, enrollment-ca-changed.
- **scim-directory-service** (Orange) — Produces SCIM events (scim-group-state, scim-user-*).
- **mms-pigeon** (PowerPC) — Produces 6 apple-media-* topics in pdd/mms/ namespace.

## Communication

- **Sync (HTTP):** Services communicate via Tyk API gateway (`tyk-gateway-management` repo). M2M auth via robocop.
- **Async (Events):** Apache Pulsar under the `pdd` tenant. Platform topics in `default` namespace (cannot change without cross-team consultation), service-owned topics in per-service namespaces (e.g., `scoping-engine`).

## Deep Dives

IMPORTANT: When answering questions about cross-service concerns, do NOT guess or infer from partial information. Read the relevant doc below BEFORE answering. These docs contain verified information about service interactions, event consumers, API contracts, and infrastructure that cannot be reliably inferred from a single repo's code.

- **HTTP calls between services, Tyk gateway, API contracts** → read `docs/api-layer.md`
- **Pulsar events, topic routing, event schemas** → read `docs/event-layer.md`
- **Authentication, JWT sidecar, tenant resolution** → read `docs/auth-and-tenancy.md`
- **DynamoDB table designs, GSIs, key patterns** → read `docs/database.md`
- **Test repos, component/system/perf/contract testing** → read `docs/testing.md`
- **Grafana dashboards, metrics, logging, alerting** → read `docs/observability.md`
- **Deployments, ArgoCD, shared values, release flow** → read `docs/cicd-pipeline.md`
- **Terraform, AWS accounts, regions, IAM** → read `docs/infrastructure.md`
- **Helm charts, values layering, pod topology, Backstage** → read `docs/kubernetes.md`
- **Shared client libraries, messaging client, Gradle plugin** → read `docs/shared-libraries.md`
- **Micro-frontends, schema pipeline, shell integration** → read `docs/frontend.md`
- **Internals of a specific service** (data model, API details, design decisions) → read `services/<service-name>.md`

If you discover that information in these docs is outdated or incorrect based on what you observe in the code, flag it to the user. To update the docs, read the instructions at `ddmr-developer-context/.claude/skills/update-context/SKILL.md` (sibling of the repo you're in).
