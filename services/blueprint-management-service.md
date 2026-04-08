# Blueprint Management Service

Last reviewed: 2026-04-07

**Owner:** Ocean team

## Summary

Blueprint Management Service (BMS) is a Spring Boot 3.5 / Java 21 service that owns the lifecycle of blueprints on the Jamf OCEAN platform. A blueprint is a named, versioned configuration object composed of ordered steps, where each step contains components (typed configuration payloads) and optional activation rules. BMS handles CRUD for blueprints, tracks a full version history on every edit, and coordinates deployments asynchronously via Apache Pulsar. It persists everything to PostgreSQL, enforces per-tenant isolation on every query, and integrates with the Component Registry, Tenant Service, and a downstream deployment executor.

---

## API Endpoints

All routes are under `/v1/blueprints` and require an M2M JWT. Tenant isolation is enforced via the `@TenantId` annotation, which extracts the tenant from the JWT on every request.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/blueprints` | List blueprints (paginated, sortable, filterable by `search`) |
| `GET` | `/v1/blueprints/{id}` | Get a single blueprint |
| `POST` | `/v1/blueprints` | Create a blueprint. Accepts a `validation-profile` query param (`SAVE` or `DEPLOY`). Returns `201` with `Location` header. |
| `PATCH` | `/v1/blueprints/{id}` | Update a blueprint via RFC 7386 JSON Merge Patch. Accepts `application/merge-patch+json` or `application/json`. Also accepts a `validation-profile` query param. Returns `204`. |
| `DELETE` | `/v1/blueprints/{id}` | Soft-delete a blueprint. Automatically triggers undeploy first. |
| `POST` | `/v1/blueprints/{id}/deploy` | Deploy the latest version. Returns `202 Accepted` — actual work is async. Returns `409` if the blueprint fails DEPLOY-profile validation. |
| `POST` | `/v1/blueprints/{id}/undeploy` | Undeploy a blueprint. Returns `202 Accepted`. |
| `GET` | `/v1/blueprints/{id}/versions` | List version history (paginated). |
| `GET` | `/v1/blueprints/{id}/deployments/latest` | Get latest deployment state and deployed object info. Returns `404` if never deployed. |
| `GET` | `/v1/blueprints/{id}/versions/latest-deployed/deployable-objects` | **Deprecated.** List deployed objects for the current version. |

A Swagger UI is available at `/swagger-ui/index.html` and an OpenAPI spec at `https://docs.jamf.build/blueprint-management-service/current/openapi.json`.

There are also two Spring Actuator endpoints:
- `POST /actuator/redeployblueprints` — force-redeploys all currently deployed blueprints; used for operational recovery.
- `DELETE /actuator/blueprints/{id}?undeploy={true|false}` — internal force-delete of a blueprint by ID, bypassing tenant-from-JWT extraction. `undeploy` defaults to `true`.

---

## Data Model (PostgreSQL)

Flyway manages schema migrations in `src/main/resources/db/migration/`.

### Key Tables

**`blueprint`** — the root entity, one row per blueprint per tenant.
- `id` (UUID PK), `tenant_id`, `name` (optional, max 200), `description` (optional, max 2000)
- `deployed_version_id` — FK to `blueprint_version`; `NULL` means not deployed
- `deleted_at` — soft-delete timestamp; all queries filter `WHERE deleted_at IS NULL` via `@SQLRestriction`
- `created_at`, `updated_at`

**`blueprint_version`** — append-only; a new row is created on every edit that changes `scope_id` or the step structure. Marked `@Immutable`.
- `id`, `blueprint_id` (FK), `version` (monotonically increasing integer, unique per blueprint), `scope_id`, `created_at`

**`blueprint_step`** — ordered steps within a version. Immutable.
- `id`, `version_id` (FK), `name`, `order` (integer, unique per version — managed by JPA `@OrderColumn` on the parent `BlueprintVersion.steps` collection, not a named field on the entity)

**`component`** — components within a step.
- `id`, `step_id` (FK), `identifier` (max 100), `configuration` (JSONB), `private_configuration` (encrypted JSONB, added in V13)

**`activation`** — one optional activation rule set per step (JSONB `rules`).

**`blueprint_deployment`** — one row per deploy/undeploy attempt.
- `id`, `version_id` (FK), `state` (`PENDING`, `DEPLOYING`, `SUCCEEDED`, `FAILED`), `created_at`, `updated_at`

**`blueprint_deployable_object`** — records of objects that were materialized during a deployment (DSS declarations or VPP apps). Single-table inheritance keyed on `deployable_object_type` (`DSS` or `VPP_APP`).

**`blueprint_activation_declaration`** — activation declarations associated with a blueprint or deployment.

**`blueprint_activation_declaration_deployable_object`** — join table linking activations to their deployable objects.

### Relationships

```
blueprint 1──* blueprint_version 1──* blueprint_step 1──* component
         |                    └──* blueprint_deployment
         |──* blueprint_deployable_object  (also FK'd to blueprint_deployment)
         └──* blueprint_activation_declaration  (also FK'd to blueprint_deployment)
```

Both `blueprint_deployable_object` and `blueprint_activation_declaration` carry direct FKs to `blueprint` as well as to `blueprint_deployment`.

---

## Events

### Produced

| Topic | Type | When |
|-------|------|------|
| `pdd/blueprints/blueprint-deployment-task` | `BlueprintDeployDeploymentTask` or `BlueprintUndeployDeploymentTask` | After a deploy or undeploy transaction commits |

The deploy task payload carries `tenantId`, `deploymentId`, `blueprintId`, `organizationId`, `environmentId`, and the full blueprint structure (scope ID + steps with component identifiers, configurations, and private configurations). The message is sent **after transaction commit** to avoid publishing before the DB write is durable.

### Consumed

| Topic | Listener | Action |
|-------|----------|--------|
| `pdd/blueprints/blueprint-component-translation-changed` | `BlueprintComponentTranslationChangedListener` | Re-deploys all currently deployed blueprints that reference the changed component identifier. |
| `pdd/default/blueprint-deployment-changed` | `BlueprintDeploymentChangedListener` | Updates the `blueprint_deployment.state` to `SUCCEEDED` or `FAILED` based on the result from the downstream executor. |

Both listeners use `Key_Shared` subscription and have DLQ policies configured in `MessagingProperties`. The `blueprint-deployment-changed` consumer applies a tenant security context before processing.

---

## External Service Dependencies

**Component Registry (`blueprint-components-registry-service`)** — BMS calls this service to look up component metadata and validate component configurations before save. Results are cached in-memory (Caffeine cache, key `blueprint-component` and `blueprint-global-component`). If a component is not found, `BlueprintComponentNotFoundException` is thrown and surfaces as a 4xx to callers.

**Tenant Service (`tenants-odin`)** — BMS calls this to resolve `organizationId` and `environmentId` for a given `tenantId` at deploy time. These values are passed along in the `blueprint-deployment-task` message so the downstream executor can route the deployment. Results are Caffeine-cached (`tenants`).

**Blueprint Deployment Service (downstream)** — Not called directly. BMS publishes `blueprint-deployment-task` events and receives results on `blueprint-deployment-changed`. BMS never directly invokes the deployment executor.

**Declaration Storage Service (DSS) / Configuration Profile Service** — Referenced in `catalog-info.yaml` as consumed APIs, and reflected in the `DSSDeployableObject` entity. BMS records DSS declaration identifiers and version hashes that result from deployments but does not own DSS data.

---

## Key Design Decisions and Gotchas

**Soft deletes.** `Blueprint` uses `@SQLRestriction("deleted_at IS NULL")`. This means all JPA queries automatically exclude deleted records. If you need to query deleted blueprints (e.g., for audit), you must use native SQL or remove the restriction explicitly. Deletion also triggers an undeploy before the soft-delete stamp is set.

**Versioning is append-only.** `BlueprintVersion` and `BlueprintStep` are annotated `@Immutable`. A new version row is only created when `scope_id` or step structure changes — metadata-only edits (name, description) do not bump the version. The `deployedVersion` FK on `blueprint` always points to the version that was last deployed, not necessarily the latest.

**Async deploys.** `POST /deploy` and `POST /undeploy` return `202 Accepted` immediately. The actual work happens in a separate service listening on `blueprint-deployment-task`. BMS creates the `BlueprintDeployment` row in state `PENDING` and transitions it to `SUCCEEDED` or `FAILED` when it receives a `blueprint-deployment-changed` event back. (`DEPLOYING` is a valid enum value but BMS never sets it — it is used by the downstream executor.) Do not assume a `202` means the blueprint is deployed — poll `GET /deployments/latest` and check `state`.

**Two validation profiles.** `SAVE` allows partial blueprints (components without fully valid configurations). `DEPLOY` enforces stricter rules — blueprints must be fully valid to deploy. Both PATCH and POST accept a `?validation-profile=` param, so clients can stage incomplete blueprints and validate strictly at deploy time.

**JSON Merge Patch.** PATCH uses RFC 7386 semantics. Setting a field to `null` in the patch removes it. Partial step updates follow merge-patch rules, not array-position-preserving upserts. Be precise when patching step arrays.

**Component configuration transformation and private config.** At save time, component configurations are validated against the Component Registry's schema. If a component returns sensitive fields, they are split into `private_configuration` (encrypted at rest via `SensitiveConfigurationConverter`) and the sanitized `configuration`. Private configuration is carried forward across version bumps using a three-tier lookup: (1) exact positional match (same step index and component index with matching identifier), (2) identifier match within the same step, (3) identifier match across all steps. The first match found wins. The private config is included in the `blueprint-deployment-task` payload so the executor has it during deployment.

**`OriginSystemType` (BMS vs BDS).** Deployable objects and activation declarations carry an `origin_system_type` field (`BMS` or `BDS`). This is being populated as a migration from an older system and is marked TODO as required after migration completes. Do not rely on it always being non-null in older records.

**Cache invalidation.** Component Registry and Tenant Service responses are cached with Caffeine but have no explicit eviction hooks. Stale cache entries can cause component validation to use outdated schemas until the cache TTL expires (configured in `CacheProperties`).
