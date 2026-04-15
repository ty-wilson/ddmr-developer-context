# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This file is symlinked to the DDmR parent directory so it auto-loads in every sibling repo session. Keep under 100 lines. Docs-only repo — no build/lint/test commands. Line limits: this file <100, `docs/*.md` 100-300, `services/*.md` <200. Only document what's verified in code; update `Last reviewed` dates when verifying. Auto-sync scripts in `scripts/`; `setup` skill in `.claude/skills/`; `update-context` skill is installed globally (`~/.claude/skills/update-context/`).

# Blueprints Platform — Service Map

All services below participate in the Blueprints system (`system: blueprints` in Backstage). Team ownership is noted per service.

## Services

### DDmR Team
- **scoping-engine** — Scope membership, device-to-group resolution, device sync state. DynamoDB + Pulsar.
- **declaration-service** — Declaration CRUD: fragment building, validation, translation, activation. Stateless REST API.
- **declaration-storage-service** — Persists declaration data (DynamoDB). Publishes `declaration-assignment-changed` events.
- **blueprint-component-custom-declarations** — Custom declarations blueprint component. Stateless adapter to DSS.
- **ddmr-jwt-sidecar** — Micronaut JWT validation proxy, deployed as pod sidecar on port 7070.
- **ddmr-authorizer-tenant** — API gateway authorizer: resolves JWT claims → tenant ID via DynamoDB lookup.

### Ocean Team
- **spring-m2m-authentication** — Spring Boot autoconfiguration library for M2M JWT auth. Servlet-only.
- **blueprint-management-service** — Blueprint CRUD, component orchestration. PostgreSQL + Pulsar async deploys.
- **blueprint-components-registry-service** — Component library catalog: fragment names, types, icons.
- **blueprint-component-declarations-service** — Declarations fragment metadata for the component registry.
- **blueprint-component-sw-update-service** — Software update blueprint component backend.

### Goldminers Team
- **configuration-profile-service** — Config profile CRUD, plist handling. MongoDB + S3.
- **configuration-profile-plist-migrator** — Plist migration tooling. GraalVM native.
- **mdm-schema-ingest-inbound-adapter** — Hourly Lambda: ingests Apple DDM repo → transforms → S3.
- **mdm-schema-ingest-infrastructure** — Terraform/Terragrunt infra for the schema pipeline.
- **mdm-ui-schema** — UI-specific schema customizations.

### Other Teams
- **tenants-odin** (Angry Cockroaches) — Tenant management service. DynamoDB.
- **device-group-inventory-service** (Data Manager) — Platform proxy to Jamf Pro group API. PostgreSQL cache + pass-through. No division data currently.
- **device-declaration-reporting-service** (Jabberwocky) — Reporting backend for declaration status per device.
- **micro-frontend-hub** — Nx + pnpm monorepo: all MFE apps (declarations, config-profiles, JSFG, etc.)
- **json-schema-form-generator** — Standalone repo for the JSFG component.

### Infrastructure & Shared Libraries (DDmR)
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

### External Services (interact with Blueprints via events or HTTP)
- **blueprint-report-aggregation-service** (Ocean) — Consumes device-scope-membership-changed, device-management-channel-changed.
- **blueprint-deployment-service** (Ocean) — Produces blueprint-deployment-task, blueprint-deployment-changed.
- **compliance-benchmark-report-service** (Mars/Red) — Consumes device-group-changed, device-*-changed events.
- **compliance-benchmark-engine** (Red) — Produces verified-rules topic.
- **spaghetti-mux** (Pulsaroni) — Pulsar event relay. Consumes declaration-assignment-changed and SCIM events.
- **m2m-robocop** — M2M authentication library used by all services.
- **jamf-school-helm-apns** (Pixels) — Consumes declaration-assignment-changed.
- **Jamf Pro Server / jamf-messaging** — Produces platform events: device-group-changed, device-management-channel-changed, device-*-state events.
- **device-identity-mapping-service** (Team Rocket) — Consumes device-identity-certificate-issued, device-management-state.
- **scim-directory-service** (Orange) — Produces SCIM events.
- **mms-pigeon** (PowerPC) — Produces apple-media-* topics in pdd/mms/ namespace.
- **app-lifecycle-management-engine-client** — Client lib for VPP/app lifecycle service.

## Communication

- **Sync (HTTP):** Services communicate via Tyk API gateway (`tyk-gateway-management` repo). M2M auth via robocop.
- **Async (Events):** Apache Pulsar under the `pdd` tenant. Platform topics in `default` namespace (cannot change without cross-team consultation), service-owned topics in per-service namespaces (e.g., `scoping-engine`).

## Deep Dives

IMPORTANT: When answering questions about cross-service concerns, do NOT guess or infer from partial information. Read the relevant doc below BEFORE answering. These docs contain verified information about service interactions, event consumers, API contracts, and infrastructure that cannot be reliably inferred from a single repo's code.

- **HTTP calls between services, Tyk gateway, API contracts** → read `../ddmr-developer-context/docs/api-layer.md`
- **Pulsar events, topic routing, event schemas** → read `../ddmr-developer-context/docs/event-layer.md`
- **Authentication, JWT sidecar, tenant resolution** → read `../ddmr-developer-context/docs/auth-and-tenancy.md`
- **DynamoDB table designs, GSIs, key patterns** → read `../ddmr-developer-context/docs/database.md`
- **Test repos, component/system/perf/contract testing** → read `../ddmr-developer-context/docs/testing.md`
- **Grafana dashboards, metrics, logging, alerting** → read `../ddmr-developer-context/docs/observability.md`
- **Deployments, ArgoCD, shared values, release flow** → read `../ddmr-developer-context/docs/cicd-pipeline.md`
- **Terraform, AWS accounts, regions, IAM** → read `../ddmr-developer-context/docs/infrastructure.md`
- **Helm charts, values layering, pod topology, Backstage** → read `../ddmr-developer-context/docs/kubernetes.md`
- **Shared client libraries, messaging client, Gradle plugin** → read `../ddmr-developer-context/docs/shared-libraries.md`
- **Micro-frontends, schema pipeline, shell integration** → read `../ddmr-developer-context/docs/frontend.md`
- **Internals of a specific service** (data model, API details, design decisions) → read `../ddmr-developer-context/services/<service-name>.md`

**Accuracy note:** These docs are point-in-time snapshots. For important decisions, verify claims against the actual code — treat these as orientation, not source of truth.

If you discover that information in these docs is outdated or incorrect based on what you observe in the code, flag it to the user. To update the docs, use the `/update-context` skill (installed globally at `~/.claude/skills/update-context/SKILL.md`).
