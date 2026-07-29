# Blueprint Management Service

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the Flyway migration list (the doc's "added in V13" was correct but there are now migrations past it, including division support this page did not mention), the `@Size` constraints, the `DeploymentState` enum and which members BMS actually sets, and where the per-component base URI comes from (**the previous claim pointed at the wrong class**). Not re-verified and older, carried forward from the 2026-04-07 review: the endpoint table, the produced/consumed event payloads, the private-configuration three-tier lookup, and the cache-invalidation behaviour. Treat those as a pointer.

Run the commands in **Where to find the data** at the end before acting on any value here. **This page is about another team's database**, so it is authority-shaped and worth distrusting: read the migrations and entities rather than the tables below.

**Owner:** Ocean team

## Summary

Blueprint Management Service (BMS) is a Spring Boot / Java service that owns the lifecycle of blueprints on the Jamf OCEAN platform. A blueprint is a named, versioned configuration object composed of ordered steps, where each step contains components (typed configuration payloads) and optional activation rules. BMS handles blueprint CRUD, tracks a full version history on every edit, and coordinates deployments asynchronously over Apache Pulsar. It persists to PostgreSQL, enforces per-tenant isolation on every query, and integrates with the Component Registry, Tenant Service, and a downstream deployment executor.

---

## API surface

All routes under `/v1/blueprints`, all requiring an M2M JWT. Tenant isolation comes from the `@TenantId` annotation, which extracts the tenant from the JWT on every request.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/blueprints` | List (paginated, sortable, filterable by `search`) |
| `GET` | `/v1/blueprints/{id}` | Get one |
| `POST` | `/v1/blueprints` | Create. Accepts `?validation-profile=SAVE\|DEPLOY`. `201` with `Location`. |
| `PATCH` | `/v1/blueprints/{id}` | RFC 7386 JSON Merge Patch (`application/merge-patch+json` or `application/json`). Also accepts `validation-profile`. `204`. |
| `DELETE` | `/v1/blueprints/{id}` | Soft-delete. Triggers undeploy first. |
| `POST` | `/v1/blueprints/{id}/deploy` | `202 Accepted`, async. `409` if DEPLOY-profile validation fails. |
| `POST` | `/v1/blueprints/{id}/undeploy` | `202 Accepted`. |
| `GET` | `/v1/blueprints/{id}/versions` | Version history (paginated). |
| `GET` | `/v1/blueprints/{id}/deployments/latest` | Latest deployment state and deployed object info. `404` if never deployed. |
| `GET` | `/v1/blueprints/{id}/versions/latest-deployed/deployable-objects` | **Deprecated.** |

Swagger UI at `/swagger-ui/index.html`. The published OpenAPI spec is a live document; fetch it rather than reading this table (URL in the verification section).

Two Spring Actuator operations, both operational escape hatches:

- `POST /actuator/redeployblueprints`: force-redeploys every currently deployed blueprint. Recovery tool, fleet-wide effect.
- `DELETE /actuator/blueprints/{id}?undeploy={true|false}`: internal force-delete **bypassing tenant-from-JWT extraction**, so it is not tenant-scoped. `undeploy` defaults to `true`.

---

## Data Model (PostgreSQL)

Flyway migrations in `src/main/resources/db/migration/`. Do not read a migration number or a column length out of this page; both drift and both are one command away (see the verification section). Notably the previous version of this page stopped at V13, and the schema has since gained division support (a `division_id` column plus a `divisionId` field on the `Blueprint` entity) that this page never mentioned. Length constraints are declared as `@Size` on the entities and DTOs, not only in SQL, so grep for `@Size` to get the authoritative pair.

Entities and how they relate, which is the part that is expensive to rediscover:

```
blueprint 1──* blueprint_version 1──* blueprint_step 1──* component
         |                    └──* blueprint_deployment
         |──* blueprint_deployable_object  (also FK'd to blueprint_deployment)
         └──* blueprint_activation_declaration  (also FK'd to blueprint_deployment)
```

- **`blueprint`**: root entity, one row per blueprint per tenant. Holds `deployed_version_id` (FK to `blueprint_version`, `NULL` means not deployed) and `deleted_at` for soft deletes.
- **`blueprint_version`**: append-only, `@Immutable`. A new row appears on every edit that changes `scope_id` or the step structure. `version` is a monotonically increasing integer, unique per blueprint.
- **`blueprint_step`**: ordered steps within a version, also `@Immutable`. **Ordering is managed by JPA `@OrderColumn` on `BlueprintVersion.steps`, not by a named field on the entity**, so the column exists in the table but you will not find a getter for it.
- **`component`**: components within a step. Carries `configuration` (JSONB) and `private_configuration` (encrypted JSONB).
- **`activation`**: at most one activation rule set per step, JSONB `rules`.
- **`blueprint_deployment`**: one row per deploy/undeploy attempt, with a `state` from `DeploymentState`.
- **`blueprint_deployable_object`**: objects materialized during a deployment (DSS declarations or VPP apps). Single-table inheritance on `deployable_object_type`.
- **`blueprint_activation_declaration`** and its join table to deployable objects.

Both `blueprint_deployable_object` and `blueprint_activation_declaration` carry direct FKs to `blueprint` **as well as** to `blueprint_deployment`, so a join through the wrong one silently changes the result set.

---

## Events

### Produced

| Topic | Type | When |
|-------|------|------|
| `pdd/blueprints/blueprint-deployment-task` | `BlueprintDeployDeploymentTask` or `BlueprintUndeployDeploymentTask` | After a deploy or undeploy transaction commits |

The deploy task payload carries `tenantId`, `deploymentId`, `blueprintId`, `organizationId`, `environmentId`, and the full blueprint structure (scope ID plus steps with component identifiers, configurations, and private configurations). It is sent **after transaction commit**, so the DB write is durable before anyone can act on the message.

### Consumed

| Topic | Listener | Action |
|-------|----------|--------|
| `pdd/blueprints/blueprint-component-translation-changed` | `BlueprintComponentTranslationChangedListener` | Re-deploys all currently deployed blueprints referencing the changed component identifier |
| `pdd/default/blueprint-deployment-changed` | `BlueprintDeploymentChangedListener` | Updates `blueprint_deployment.state` to `SUCCEEDED` or `FAILED` from the downstream executor's result |

Both use `Key_Shared` subscriptions with DLQ policies in `MessagingProperties`. The `blueprint-deployment-changed` consumer applies a tenant security context before processing.

**Note the blast radius of the first one:** a single component-translation event can trigger a re-deploy of every deployed blueprint that uses that component, across tenants.

---

## External Service Dependencies

**Component Registry (`blueprint-components-registry-service`)**: BMS calls it to look up component metadata and validate component configurations before save. Results are Caffeine-cached under keys `blueprint-component` and `blueprint-global-component`. A missing component raises `BlueprintComponentNotFoundException` and surfaces as a 4xx. See `blueprint-components-registry-service.md`.

**Tenant Service (`tenants-odin`)**: resolves `organizationId` and `environmentId` for a `tenantId` at deploy time. Those values ride along in the `blueprint-deployment-task` message so the downstream executor can route the deployment. Caffeine-cached under `tenants`.

**Blueprint Deployment Service (downstream)**: **never called directly.** BMS publishes `blueprint-deployment-task` and receives results on `blueprint-deployment-changed`. There is no synchronous path to the executor.

**Declaration Storage Service / Configuration Profile Service**: listed in `catalog-info.yaml` as consumed APIs and reflected in the `DSSDeployableObject` entity. BMS records DSS declaration identifiers and version hashes resulting from deployments but does not own DSS data.

### Component services BMS calls during deployment

BMS calls each component service's translate, validate, and cleanup endpoints. **Which service handles which component type is registry-driven, not hardcoded**, so treat the following as a snapshot: as of 2026-07-29 the set was `configuration-profile-service` (`ddm-profile`), `blueprint-component-declarations-service` (DDmR-owned declaration components), `blueprint-component-sw-update-service`, and `blueprint-component-custom-declarations`.

**Trap: the per-component base URI does not come from `ComponentClientProperties`.** That record holds only `connectTimeout` and `readTimeout`. `ComponentServiceClientProvider.getBaseUri(identifier)` fetches the component from the registry (`ComponentRegistryAdapter.getGlobalComponent`) and uses `component.translator().baseUri()`, and the resulting client is cached under `component-service-clients`. So the authoritative answer to "which host will BMS call for component X" lives in the **registry's** data, not in BMS config, and a stale entry in that cache will keep BMS calling the old host after the registry changes.

---

## Traps and design decisions

**Trap: `DEPLOYING` is a valid `DeploymentState` that BMS never sets.** BMS writes only `PENDING` (in `service/DeploymentService.java`) and `SUCCEEDED` / `FAILED` (in `service/BlueprintDeploymentChangedProcessor.java`, mapping the inbound result). `DEPLOYING` exists in `model/DeploymentState.java` for the downstream executor. Code or a dashboard that waits for `DEPLOYING` as an intermediate state will wait forever.

**Trap: `202 Accepted` does not mean deployed.** `POST /deploy` and `/undeploy` return immediately; the work happens in another service listening on `blueprint-deployment-task`. Poll `GET /deployments/latest` and check `state`.

**Trap: `deployedVersion` is not the latest version.** The FK on `blueprint` points at the version that was last *deployed*, which is frequently behind the newest `blueprint_version` row. And a new version row is only created when `scope_id` or the step structure changes: metadata-only edits (name, description) do **not** bump the version. So "latest version" and "what is running" and "what the user last saved" are three different things. `db/entity/Blueprint.java` is the file that proves the FK semantics; `BlueprintVersion` and `BlueprintStep` are `@Immutable`.

**Trap: soft deletes are invisible to JPA.** `Blueprint` carries `@SQLRestriction("deleted_at IS NULL")`, so every JPA query silently excludes deleted rows. Querying deleted blueprints (for audit, say) requires native SQL or removing the restriction. Deletion also triggers an undeploy before the soft-delete stamp is set.

**Trap: JSON Merge Patch does not preserve array positions.** PATCH uses RFC 7386, so setting a field to `null` removes it, and partial step updates follow merge-patch rules rather than array-position-preserving upserts. Patching a step array imprecisely rewrites it.

**Two validation profiles.** `SAVE` allows partial blueprints (components without fully valid configurations); `DEPLOY` requires full validity. Both PATCH and POST accept `?validation-profile=`, which is what lets clients stage incomplete blueprints and validate strictly only at deploy.

**Private configuration is carried forward heuristically.** At save time, component configurations are validated against the registry's schema; sensitive fields are split into `private_configuration` (encrypted at rest via `SensitiveConfigurationConverter`) and a sanitized `configuration`. Across version bumps, private config is matched by a three-tier lookup, first match wins: (1) exact positional match (same step index and component index with matching identifier), (2) identifier match within the same step, (3) identifier match across all steps. **Tier 3 can attach a secret to a different step's component** if identifiers repeat, so reordering steps is not a no-op for secrets. The private config is included in the `blueprint-deployment-task` payload.

**Trap: Caffeine caches have no eviction hooks.** Component Registry and Tenant Service responses expire only by TTL (configured in `CacheProperties`), so component validation can keep using an outdated schema after the registry changes, with no way to force a refresh short of a restart.

**`OriginSystemType` (BMS vs BDS).** Deployable objects and activation declarations carry `origin_system_type` (`BMS` or `BDS`). May be null in older records, so filter accordingly.

---

## Where to find the data (verify rather than trust)

The local checkout is often far behind and parked on a ticket branch (it was on `OCEAN-205-sensitive-data-phase1` from February on 2026-07-29, ~5 months stale), so read through `origin/main`.

```bash
R=~/Projects/DDmR/blueprint-management-service; git -C $R fetch origin -q
git -C $R log --oneline origin/main --since=2026-07-29
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Schema: the migration list replaces the column tables and any "added in Vnn" claim
git -C $R ls-tree --name-only origin/main src/main/resources/db/migration/ | sed 's|.*/||' | sort -V | tail -12

# Length limits, from the annotations rather than from prose
git -C $R grep -n '@Size' origin/main -- 'src/main/java/**/db/entity/*' 'src/main/java/**/rest/dto/*'

# Which DeploymentState members BMS actually writes (DEPLOYING should not appear)
git -C $R show origin/main:src/main/java/com/jamf/blueprint/management/model/DeploymentState.java
git -C $R grep -n 'DeploymentState\.' origin/main -- 'src/main/java/**'

# Where a component's base URI really comes from, and the deployedVersion FK
git -C $R show origin/main:src/main/java/com/jamf/blueprint/management/client/ComponentServiceClientProvider.java
git -C $R grep -n 'deployedVersion\|divisionId' origin/main -- 'src/main/java/**/db/entity/Blueprint.java'

# The live API contract, which beats the endpoint table above
curl -s https://docs.jamf.build/blueprint-management-service/current/openapi.json | python3 -m json.tool | head -40
```

The registry, not BMS, decides which component services get called. Confirm the current set there rather than from the snapshot list on this page:

```bash
git -C ~/Projects/DDmR/blueprint-components-registry-service show \
  origin/main:src/main/resources/application.yml | grep -nE 'standaloneComponents|fragmentedComponents|baseUri|apiBaseUri' -A3
```
