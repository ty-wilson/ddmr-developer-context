# DDmR Authorizer Tenant

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** DDmR team

## Summary

DDmR Authorizer Tenant is a Spring Boot WebFlux service (not a Lambda despite the name) used by HAProxy as a sub-request authorizer. When a device or client presents a CSA (Cloud Services Architecture) access token, HAProxy calls this service's `/authorize` endpoint. The service validates the JWT, extracts the organization ID and customer/instance ID, looks up or generates a stable tenant ID from DynamoDB, and returns that tenant ID in the `X-TenantId` response header. HAProxy then forwards the header to the upstream service. It is the canonical source of truth for the `organizationId + instanceId → tenantId` mapping across the DDmR platform.

---

## Request Flow

1. HAProxy receives an inbound request and makes a sub-request to `GET /authorize`, forwarding the bearer token and an `x-customer-id` header.
2. Spring Security validates the JWT as an OAuth2 resource server token (CSA-issued, public key fetched from S3 via `csa.jwt.s3Host`).
3. `AuthorizeHandler` extracts:
   - `organizationId` from `jwt.subject` — 401 if missing.
   - `customerId` (HAProxy-supplied `x-customer-id` header) used as the `instanceId`.
   - Optional `tenant_id` claim directly from the JWT payload (`claimTenantId`).
4. The handler asserts `token_use == "access"` — 401 otherwise.
5. `DynamoDbTenantIdRepository.getTenantId` resolves the tenant ID (see Tenant Resolution Logic below).
6. On success, returns `HTTP 200` with:
   - `X-TenantId: <tenantId>` — forwarded by HAProxy to the upstream service.
   - `X-Auth-Src: CSA:<organizationId>:<customerId>` — for audit/tracing.
7. HAProxy proxies the original request upstream with both headers attached.

---

## DynamoDB Key Patterns

Single-table design with a single hash key (`pk`). No sort key, no GSIs.

| `pk` | Attributes | Description |
|------|-----------|-------------|
| `ORG#<organizationId>#<instanceId>` | `tenantId`, `organizationId`, `instanceId`, `createdOn`, `lastAccess` | Primary tenant mapping record |

- `createdOn` and `lastAccess` are stored as epoch milliseconds (Number).
- `lastAccess` is updated at most once every 6 hours (`A_TIME_GRANULARITY`) to reduce write traffic.
- A legacy attribute `platformTenant` may be present on older records. The lookup prefers `platformTenant` over `tenantId` when both exist.
- A `claimTenantMigration` attribute is written (once, conditionally) when a mismatch is detected between the stored tenant ID and the JWT `tenant_id` claim, flagging the record for reconciliation.

---

## Tenant Resolution Logic

`DynamoDbTenantIdRepository.getTenantId` follows this priority order:

1. **DynamoDB hit** — if a record exists for `ORG#<organizationId>#<instanceId>`, return the stored `tenantId` (or `platformTenant` if present). Update `lastAccess` if it is stale.
2. **Claim fallback** — if no DB record exists but the JWT carries a `tenant_id` claim, return the claim value directly without writing to DynamoDB.
3. **Generate** — if neither exists and `tenant-authorizer.generateTenantId=true`, generate a UUID, write it to DynamoDB with a conditional `attribute_not_exists(pk)` check to prevent races, and return it. If the conditional write races, a follow-up `getItem` retrieves the winner.
4. **Reject** — if neither exists and `generateTenantId=false` (the default/production setting), return 401.

Mismatch detection: if DynamoDB has a record but its stored tenant ID differs from the JWT `tenant_id` claim, the service logs a warning and writes a `claimTenantMigration` marker onto the record (once). The stored DB value is still returned — the mismatch does not cause a rejection.

---

## Configuration Properties

| Property | Default | Description |
|----------|---------|-------------|
| `aws.dynamodb.table` | (required) | DynamoDB table name |
| `aws.dynamodb.localPort` | `null` | Set to use a local DynamoDB instance |
| `csa.jwt.s3Host` | (required) | S3 bucket name used to fetch CSA public keys for JWT validation |
| `tenant-authorizer.generateTenantId` | `false` | Allow on-the-fly UUID generation for unknown tenants. Should be `false` in production |
| `tenant-authorizer.rejectRequestInStageHack` | `false` | Staging-only flag that rejects all requests for a given `(organizationId, instanceId)` pair until 1 hour has elapsed since first attempt, then requires `all-basic-cloud-services : allow` scope on the JWT |

---

## Gotchas

**`generateTenantId=false` in production.** Unknown tenants get a 401 rather than a new ID. If a device cannot get through and there is no record in DynamoDB, the tenant mapping must be seeded externally before requests will succeed.

**`platformTenant` vs `tenantId` attribute.** Some older DynamoDB records use `platformTenant` as the attribute name. The lookup reads `platformTenant` preferentially when present. New records are always written with `tenantId`.

**`rejectRequestInStageHack` is not yet removed.** This property exists as a staging test shim; it should be treated as dead code in production. If it is accidentally enabled, all requests for any new `(organizationId, instanceId)` pair will be rejected for at least one hour.

**JWT validation depends on CSA public keys in S3.** The `csa.jwt.s3Host` bucket must be reachable at startup and at validation time. Key-fetch failures will cause 401s for all requests. In local development, use `csa-public-key-store-development`.

**`x-customer-id` is required.** HAProxy is expected to always supply this header. If the header is missing, the handler binding will fail before reaching business logic.

**`lastAccess` throttles writes.** `recordAccess` skips the DynamoDB `updateItem` call if the stored `lastAccess` is within 6 hours of now. This means the `lastAccess` field may lag reality by up to 6 hours — do not rely on it for precise activity tracking.
