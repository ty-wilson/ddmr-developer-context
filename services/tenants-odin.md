# Tenants Odin

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the version-gating mechanism (`BaseRestController.applyPipeline`, `PipelineVersion.kt`, `RestValidator.validateApiVersion`), the route tree in `ApiRouter.kt`, the GSI constants in `integration/db/dynamodb/Utils.kt`, and the four hard constraints plus their error identifiers. Not re-verified and older (2026-04-07): the Known Callers list and its per-caller caching behavior, the CAL dependency details, the regions/tier claim, and the auxiliary table names.

Two things in the 2026-04-07 version were wrong and are corrected below: the GSI count (six, not five) and the per-endpoint version ranges (several had narrowed). Both were transcription, so both are now commands instead of values. See `## Where to find the data`.

**Owner:** Angry Cockroaches team

## Summary

Tenants Odin is the system of record for organizations, environments, and tenants across the Jamf platform. It sits at the center of multi-product provisioning: when any Jamf product (Pro, Now, School, Protect, JSC, JETP, Wizy) needs to resolve "which organization does this customer belong to and where is their tenant deployed," it asks Odin. The service exposes a path-versioned REST API and is backed by DynamoDB. It is a Tier 1 service deployed to us-east-1, eu-central-1, and ap-northeast-1.

Stack: Kotlin + Ktor (Netty), DynamoDB (AWS SDK Enhanced Async). Backstage `pact-provider-name: tenant-service-api`.

## API versioning: the durable rule

All routes are prefixed with `/api/{apiVersion}`. **The version is part of the path, not a header. Supported versions vary by endpoint, and an unsupported version returns 404.**

The mechanism is uniform and worth knowing instead of memorising ranges:

1. `BaseRestController.applyPipeline` pulls `{apiVersion}` from the path and runs `RestValidator.validateApiVersion`, which matches against `^v[1-9][0-9]*$`. A malformed version (`v0`, `V2`, `latest`, `2`) fails this and returns **400** with `INVALID_API_VERSION_FORMAT`.
2. A well-formed version is then dispatched to each controller's `applyPipelineVersion`, which is a `when (pipelineVersion)` over `PipelineVersions.*` constants with a terminal `else -> call.respondNotFound()`. A well-formed but unsupported version returns **404**.

**Trap: 400 and 404 mean different things here and are easy to conflate.** 400 means "that is not a version string." 404 means "that is a version string, but not one this endpoint implements," and it is indistinguishable in shape from "that organization does not exist." When a call unexpectedly 404s, check the version support for that specific endpoint before assuming the resource is missing.

**Trap: each controller carries its own version set, so ranges narrow independently and without an API-wide bump.** As of 2026-07-29, for example, `GET /organizations/{id}/environments` accepts only `v4`, and `GET /organizations/{id}/tenants` only `v3`. A caller pinned to `v1` on a path that used to accept it will start 404ing after a deploy it did not participate in. Never hardcode a version range from a doc; regenerate it (command in the verification section) or probe the live service.

Route groups present in `ApiRouter.kt`: `/organizations` (with nested `/environments`, `/tenants`, `/smb-environments`, `/smb-tenants`, `/pro-on-prem-tenants`), plus top-level `/tenants/{id}`, `/environments/{id}`, `/domain-environments/{key}`, `/products/{productCode}/tenants/{productTenantId}`, `/crm-organizations/{crmId}`, and `/dictionaries` (`meta-regions`, `new-environment`). There is also a separate, unversioned-in-practice `/admin/api/v1` surface. Read the tree from `ApiRouter.kt` rather than a transcription; the SMB, domain-environment, and admin groups postdate the original write-up.

The most common integration pattern is the product-keyed upsert: `PUT /api/v{n}/products/{productCode}/tenants/{productTenantId}` ensures a tenant exists for this product plus productTenantId combination, creating it if absent via a CAL lookup keyed on `organizationCrmId` in the body, and returning org/tenant/environment IDs either way. `GET` on the same path is the read-only form.

## Data Model

Odin uses a **single DynamoDB table** (`tenants`) with several GSIs, each a numbered `GSI{n}` / `GSI{n}PK` pair for a different lookup axis (CRM ID, tenant UUID, product, environment, Jamf CRM ID, environment domain key). The set grows over time, so read the constants rather than a count:

```bash
git -C ~/Projects/DDmR/tenants-odin grep -n 'INDEX = "GSI' origin/main \
  -- 'src/main/kotlin/com/jamf/tenantservice/odin/integration/db/dynamodb/Utils.kt'
```

**Core entities:**

- **Organization** is the top-level customer account. Fields: `id` (UUID), `name`, `crmId` (Salesforce/legacy), `jamfCrmId`, `crmAccId`, `metaInf`.
- **Environment** is a logical grouping within an org, one per geographic deployment. Fields: `id`, `name`, `label`, `deploymentRegion`, `metaRegion` (US/EU/APAC), `authZeroConnectionId`, `authZeroOrgId`, `domain`, `metaInf`.
- **Tenant** is a product instance (e.g. a Jamf Pro server). Fields: `id`, `organizationId`, `name`, `productCode` (Pro/Now/School/Protect/Jsc/Jetp/Wizy), `productType` (Management/Security), `productTenantId` (the product's own ID string), `deploymentType` (cloud/on-prem), `deploymentRegion`, `metaRegion`, `deploymentUrl`, `environmentId`, `metaInf`.

**Additional DynamoDB tables** (not re-verified since 2026-04-07):
- `tenants-cache` for cached OIDC/JWK configurations used in JWT validation. A `tenants-cache-initializer` job component exists in `.github/workflows/jobs.yml`.
- `tenants-audit-logs` for an audit trail of mutations.

## Events

Odin ships an **`event-processor` module**, a transactional-outbox reader deployed as its own workload (`helm/odin/templates/event-processor/`). It reads outbox rows keyed `TOPIC#{topicKey}` from DynamoDB and publishes to Pulsar via the platform SDK. The default `topicKey` is `tenant-state`; the chart's default topic name is `pdd-design/default-design/tenant-state`, and the per-environment topic comes from Helm values, so read it from `helm/odin/values.yaml` (and the environment overlay) rather than quoting it. This did not exist in the 2026-04-07 write-up. If you are looking for a way to react to tenant changes without polling Odin, start here.

## Known Callers

Several services within the blueprints system call Odin at runtime to resolve tenant metadata, and cache the results to avoid repeated lookups. (List as of 2026-04-07, not re-verified 2026-07-29. Correct it rather than delete it.)

- `blueprint-management-service` calls Odin at deploy time to resolve `organizationId` and `environmentId` for the deployment task payload (Caffeine-cached under the `tenants` key).
- `blueprint-components-registry-service` calls Odin on every component/fragment read to look up `organizationId`, `environmentId`, and `productCode` for feature flag evaluation (no additional cache layer beyond the LD SDK).
- `blueprint-component-declarations-service` calls Odin as part of tenant context resolution during translate/validate operations.

---

## Dependencies

- **CAL (Customer Account Lookup)** is an external HTTP service called when ensuring an org via CRM ID (`PutCrmOrganizationController`, `PutProductTenantController`). If CAL cannot find the CRM ID, Odin returns 409 with error code `CRM_ORGANIZATION_NOT_FOUND`. There is an optional `devCal` client for non-production environments.
- **OIDC provider**, fetched dynamically for JWT verification; JWKS URLs are cached in DynamoDB.
- **AWS DynamoDB**, primary datastore, accessed via AWS SDK Enhanced Async client with STS role assumption in production.

## The four hard constraints

These are the constraints that actually bite integrators. Each names the method or error identifier that enforces it, so each is checkable.

1. **A tenant's `metaRegion` cannot be changed to a conflicting value once set.** Attempting it returns 409 with `TenantMetaRegionConflict` (raised in `ProductTenantService`, rendered by `PutProductTenantController` as a conflict on field `metaRegion` with `TENANT_ALREADY_HAVE_META_REGION`). Related: older endpoints accept a raw region string (e.g. `us-east-1`) and derive the metaRegion internally, while newer ones take `metaRegion` directly (`US`, `EU`, `APAC`). Grep `TenantMetaRegionConflict`.

2. **`PUT .../environments/{id}/tenants` is a full replace, not additive.** Assigning tenants to an environment unassigns any tenants already in it that are not in the new list. There is no partial add: omitting an already-assigned tenant is a validation error, `TenantUnAssignmentNotAllowed` (built in `TenantsService`, surfaced by `PutOrganizationEnvironmentTenantsController` as an invalid parameter). Grep `TenantUnAssignmentNotAllowed`.

3. **`productTenantId` cannot be a UUID.** The patch endpoint explicitly rejects a `product.tenantId` value that parses as a valid UUID, to avoid collisions with the Odin tenant UUID space.

4. **CRM ID format is validated.** `RestValidator.validateCrmId(fieldName, value, isRequired)` enforces a specific format; an arbitrary string returns 400. It is called from many controllers, and the admin controllers pass `isRequired = false` where the standard ones pass `true`, so admin and standard paths do not reject the same inputs. Grep `validateCrmId` to see every call site and its flag.

## Other gotchas

- **`PUT /products/{productCode}/tenants/{productTenantId}` is idempotent.** It returns 200 if the tenant already existed, 201 if it was created. Use the returned `tenantId`/`organizationId` from the response body rather than assuming a direction.
- **On-prem tenants are Pro-only.** `POST .../pro-on-prem-tenants` hardcodes `ProductCode.Pro` and `DeploymentType.OnPrem`. There is no equivalent endpoint for other products. Verified 2026-07-29 by reading `PostOrganizationProOnPremTenantsController`, which is also unusual in handling `V1` and `V2` in separate branches rather than aliasing them.
- **Auth is JWT-based with dynamic JWKS.** The issuer is read from the JWT itself and the JWKS URL is resolved at runtime via an OIDC discovery document, with results cached in DynamoDB. If the cache is stale and the OIDC endpoint is unreachable, authentication fails.
- **A `Jamf-Feature-Flags` request header can force flag values** (`parseForcedFeatureFlags` in `BaseRestController.kt`). Unsupported flags come back as a 409 with `UNSUPPORTED_FEATURE_FLAGS`. Useful for testing, and a thing to rule out when behavior differs between two seemingly identical calls.
- **Paginated list endpoints take `page` (a UUID cursor) and `page-size`**, validated against a per-endpoint `maxAllowedPageSize` injected from config. Exceeding it is a 400, not a silent clamp. The limit is config-driven, so read it from the Helm values rather than quoting a number.

---

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/tenants-odin; git -C $R fetch origin -q

# What changed since this doc was last reviewed
git -C $R log --oneline origin/main --since=2026-07-29

# Unmerged work, before concluding an endpoint or version does not exist
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# REGENERATE per-endpoint version support (replaces any hand-copied range table).
# One line per controller; the versions listed are the ones that do NOT 404.
git -C $R grep -n 'PipelineVersions\.' origin/main \
  -- 'src/main/kotlin/com/jamf/tenantservice/odin/integration/http/server/rest/controllers/*.kt' \
  | grep -v 'LATEST)' | sed 's|.*/controllers/||'

# The route tree (includes the smb-*, domain-environments, and /admin/api/v1 groups)
git -C $R show origin/main:src/main/kotlin/com/jamf/tenantservice/odin/integration/http/server/rest/ApiRouter.kt

# Current GSIs (do not quote a count)
git -C $R grep -n 'INDEX = "GSI' origin/main \
  -- 'src/main/kotlin/com/jamf/tenantservice/odin/integration/db/dynamodb/Utils.kt'

# The four hard constraints, at their enforcement sites
git -C $R grep -rn 'TenantMetaRegionConflict\|TenantUnAssignmentNotAllowed\|validateCrmId' \
  origin/main -- 'src/main/*'

# Outbox topic actually configured per environment
git -C $R grep -rn 'topicKey\|topicName' origin/main -- 'helm/odin/'
```

Or probe a running instance: for a path you care about, walk the versions and record which answer. 400 means malformed, 404 means unsupported-or-absent, anything else means supported.

```bash
BASE=<odin-base-url>; TOKEN=<jwt>; PATH_SUFFIX=organizations/<org-uuid>/environments
for v in v1 v2 v3 v4 v5; do
  code=$(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN" \
    "$BASE/api/$v/$PATH_SUFFIX")
  echo "$v -> $code"
done
```
