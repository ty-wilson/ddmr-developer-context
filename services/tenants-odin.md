# Tenants Odin

Last reviewed: 2026-04-07

**Owner:** Angry Cockroaches team

## Summary

Tenants Odin is the system of record for organizations, environments, and tenants across the Jamf platform. It sits at the center of multi-product provisioning: when any Jamf product (Pro, Now, School, Protect, JSC, JETP, Wizy) needs to resolve "which organization does this customer belong to and where is their tenant deployed," it asks Odin. The service exposes a versioned REST API (v1–v4 on most endpoints) and is backed by DynamoDB. It is owned by the Angry Cockroaches team and is a Tier 1 service deployed to us-east-1, eu-central-1, and ap-northeast-1.

Stack: Kotlin + Ktor (Netty), DynamoDB (AWS SDK Enhanced Async), Java 21.

## API Endpoints

All routes are prefixed with `/api/{apiVersion}`. The version is part of the path, not a header. Supported versions vary by endpoint — unsupported versions return 404.

**Organizations**
- `POST /api/v1/organizations` — create a new organization (requires `name` + `crmId`)
- `GET /api/v1/organizations?crmId=` — find organization by CRM ID
- `GET /api/v1/organizations/{organizationId}` — get a single organization
- `DELETE /api/v1/organizations/{organizationId}` — delete an organization

**Environments** (under an organization)
- `GET /api/v{1-4}/organizations/{organizationId}/environments` — list environments
- `POST /api/v{1-4}/organizations/{organizationId}/environments` — create environment (v1/v2 use legacy region string; v3/v4 use `metaRegion` enum + optional `domain`)
- `GET /api/v{1-4}/organizations/{organizationId}/environments/{environmentId}` — get environment with its assigned tenants
- `PATCH /api/v{1-4}/organizations/{organizationId}/environments/{environmentId}` — patch environment
- `DELETE /api/v1/organizations/{organizationId}/environments/{environmentId}` — delete environment
- `PUT /api/v{1-4}/organizations/{organizationId}/environments/{environmentId}/tenants` — assign tenants to an environment (v1: typed object with management/security slots; v2+: list of tenant UUIDs)

**Tenants** (under an organization)
- `GET /api/v{1-3}/organizations/{organizationId}/tenants` — list tenants (v3 supports pagination)
- `POST /api/v{1-3}/organizations/{organizationId}/tenants` — create a cloud tenant
- `PATCH /api/v{1-3}/organizations/{organizationId}/tenants/{tenantId}` — patch tenant (name, environmentId, metaRegion, product.tenantId)
- `DELETE /api/v1/organizations/{organizationId}/tenants/{tenantId}` — delete tenant
- `PUT /api/v1/organizations/{organizationId}/tenants/{tenantId}/name` — rename tenant
- `POST /api/v{1-2}/organizations/{organizationId}/pro-on-prem-tenants` — create an on-prem Pro tenant

**Cross-resource lookups**
- `GET /api/v1/tenants/{tenantId}` — fetch any tenant by its Odin UUID
- `GET /api/v1/environments/{environmentId}` — fetch an environment by its UUID
- `GET /api/v1/tenants/{tenantId}/migrate-organization` — move a tenant to a different organization (by target CRM ID)

**Product-keyed tenant lookup / upsert** (most common integration pattern)
- `PUT /api/v{1-2}/products/{productCode}/tenants/{productTenantId}` — "ensure" a tenant exists for this product+productTenantId combo; creates if absent, returns org/tenant/environment IDs either way. Uses `organizationCrmId` in the body to find or create the org via CAL.
- `GET /api/v{1-2}/products/{productCode}/tenants/{productTenantId}` — read-only version of above

**CRM organization management**
- `PUT /api/v1/crm-organizations/{crmId}` — ensure an org exists for this CRM ID (creates via CAL lookup if missing)
- `GET /api/v1/crm-organizations/{crmId}` — get org by CRM ID
- `POST /api/v1/crm-organizations/{crmId}/migrate` — update a CRM ID on an org (old → new)

**Dictionaries / metadata**
- `GET /api/v1/dictionaries/meta-regions` — list valid meta-regions (US, EU, APAC)
- `GET /api/v1/dictionaries/new-environment` — list valid region values for legacy region field

## Data Model

Odin uses a **single DynamoDB table** (`tenants`) with two GSIs that allow CRM-keyed and product-keyed lookups.

**Core entities:**

- **Organization** — top-level customer account. Fields: `id` (UUID), `name`, `crmId` (Salesforce/legacy), `jamfCrmId`, `crmAccId`, `metaInf`.
- **Environment** — a logical grouping within an org (one per geographic deployment). Fields: `id`, `name`, `label`, `deploymentRegion`, `metaRegion` (US/EU/APAC), `authZeroConnectionId`, `authZeroOrgId`, `domain`, `metaInf`.
- **Tenant** — a product instance (e.g., a Jamf Pro server). Fields: `id`, `organizationId`, `name`, `productCode` (Pro/Now/School/Protect/Jsc/Jetp/Wizy), `productType` (Management/Security), `productTenantId` (the product's own ID string), `deploymentType` (cloud/on-prem), `deploymentRegion`, `metaRegion`, `deploymentUrl`, `environmentId`, `metaInf`.

**Additional DynamoDB tables:**
- `tenants-cache` — cached OIDC/JWK configurations for JWT validation
- `tenants-audit-logs` — audit trail of mutations

## Dependencies

- **CAL (Customer Account Lookup)** — external HTTP service called when ensuring an org via CRM ID (`PutCrmOrganizationController`, `PutProductTenantController`). If CAL cannot find the CRM ID, Odin returns a 409 with error code `CRM_ORGANIZATION_NOT_FOUND`. There is an optional `devCal` client for non-production environments.
- **OIDC provider** — fetched dynamically for JWT verification; JWKS URLs are cached in DynamoDB.
- **AWS DynamoDB** — primary datastore, accessed via AWS SDK Enhanced Async client with STS role assumption in production.

## Gotchas

- **API versioning in path, not header.** `PUT /api/v2/products/...` and `PUT /api/v1/products/...` behave differently (v1 uses legacy region strings, v2 uses `metaRegion` enum). Sending the wrong version silently routes to a different code path.
- **`metaRegion` vs `deploymentRegion`.** Older endpoints accept a raw region string (e.g., `us-east-1`) and derive the metaRegion internally. Newer endpoints (v3/v4) take the metaRegion directly (`US`, `EU`, `APAC`). Once set, a tenant's `metaRegion` cannot be changed to a conflicting value — you will get a `TenantMetaRegionConflict` 409.
- **`PUT .../tenants` is a full replace, not additive.** Assigning tenants to an environment with this endpoint unassigns any tenants already in that environment that are not in the new list. There is no partial add — omitting an already-assigned tenant is a validation error (`TenantUnAssignmentNotAllowed`).
- **`PUT /products/{productCode}/tenants/{productTenantId}` is idempotent.** It returns 200 if the tenant already exists, 201 if it was created. Callers should use the returned `tenantId`/`organizationId` from the response body rather than assuming one direction.
- **`productTenantId` cannot be a UUID.** The patch endpoint explicitly rejects a `product.tenantId` value that parses as a valid UUID, to avoid collisions with the Odin tenant UUID space.
- **On-prem tenants are Pro-only.** The `POST .../pro-on-prem-tenants` endpoint hardcodes `ProductCode.Pro` and `DeploymentType.OnPrem`. There is no equivalent endpoint for other products.
- **CRM ID format is validated.** The `RestValidator.validateCrmId` method enforces a specific format; passing an arbitrary string will return 400.
- **Auth is JWT-based with dynamic JWKS.** The issuer is read from the JWT itself, and the JWKS URL is resolved at runtime via an OIDC discovery document, with results cached in DynamoDB. If the cache is stale and the OIDC endpoint is unreachable, authentication will fail.
