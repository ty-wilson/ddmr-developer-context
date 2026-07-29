# DDmR Authorizer Tenant

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the full `AuthorizeHandler` request flow and its four 401 message strings, the `TenantAuthorizerProperties` field set, `A_TIME_GRANULARITY`'s existence and effect, and the per-environment table names and `generate-tenant-id` settings in `ddmr-deployments/helm/auth/`. **Corrected on that date:** the `rejectRequestInStageHack` property described by the previous review no longer exists in the code (see Configuration). **Not re-verified and older:** the DynamoDB attribute list and the `ddmr-tenant-migration-job` interaction, which date from 2026-06-24. Commands to re-derive everything are in "Where to find the data" at the end.

**Owner:** DDmR team

**Trap: this repo and its Backstage component have different names.** The repo is `ddmr-authorizer-tenant`; `catalog-info.yaml` registers it as **`dss-csa-authorizer`** (`lifecycle: experimental`, `serviceTier: 5`), and SonarQube uses a third name, `com.jamf.ddm:service-authorizer-tenant`. Searching Backstage or Sonar for the repo name finds nothing. Its ArgoCD ApplicationSet is a fourth name again, `ddmr-tenant-authorizer`.

## Summary

DDmR Authorizer Tenant is a Spring Boot WebFlux service (not a Lambda despite the name) used by HAProxy as a sub-request authorizer. When a device or client presents a CSA (Cloud Services Architecture) access token, HAProxy calls this service's `/authorize` endpoint. The service validates the JWT, extracts the organization ID and customer/instance ID, looks up or generates a stable tenant ID from DynamoDB, and returns that tenant ID in the `X-TenantId` response header. HAProxy then forwards the header to the upstream service. It is the canonical source of truth for the `organizationId + instanceId → tenantId` mapping across the DDmR platform.

It is **only** on the CSA path. See `docs/auth-and-tenancy.md`, which owns the question of which ingress path a given service uses.

---

## Request Flow

1. HAProxy receives an inbound request and makes a sub-request to `GET /authorize`, forwarding the bearer token and an `x-customer-id` header.
2. Spring Security validates the JWT as an OAuth2 resource server token (CSA-issued, public keys fetched from S3, derived from `csa.jwt.s3-host`; audience `Jamf`).
3. `AuthorizeHandler` runs four checks in this order, each a 401 with a distinct message. The strings are the durable identifiers, grep for them rather than for line numbers:
   - `organizationId` from `jwt.subject`, else `"Missing subject on JWT"`.
   - Scope check, else `"Scope on JWT token is not allowed, was ..."` (see the trap below).
   - `token_use == "access"`, else `"Wrong token_use claim in JWT"`.
   - Tenant resolution, else `"No tenant ID found for organizationId: ..., instanceId: ... and no claim tenant ID provided."`
4. `customerId` (the HAProxy-supplied `x-customer-id` header) is used as the `instanceId`. An optional `tenant_id` claim is read straight off the JWT payload as `claimTenantId`.
5. `DynamoDbTenantIdRepository.getTenantId` resolves the tenant ID (see Tenant Resolution Logic).
6. On success, returns `HTTP 200` with:
   - `X-TenantId: <tenantId>`, forwarded by HAProxy to the upstream service.
   - `X-Auth-Src: CSA:<organizationId>:<customerId>`, for audit/tracing.
7. HAProxy proxies the original request upstream with both headers attached.

**Trap: the required scope is hardcoded, not configurable.** `AuthorizeHandler` holds a single `scopeRegex` of `all-basic-cloud-services\s*:\s*allow` and rejects any token whose `scope` claim has no authority matching it. A CSA token that is otherwise perfectly valid gets a 401 if it lacks that one scope, and the failure looks like a credential problem rather than a scope problem. `MDC` carries `organizationId` and `customerId` on the log line, so the rejected pair is identifiable.

---

## DynamoDB Key Patterns

Single-table design with a single hash key (`pk`). No sort key. One GSI: `claimTenantMigration_index` (hash key `claimTenantMigration`, projection ALL), scanned by the `ddmr-tenant-migration-job` to find records flagged for tenant reconciliation.

| `pk` | Attributes | Description |
|------|-----------|-------------|
| `ORG#<organizationId>#<instanceId>` | `tenantId`, `organizationId`, `instanceId`, `createdOn`, `lastAccess` | Primary tenant mapping record |

`createdOn` and `lastAccess` are stored as epoch milliseconds (Number).

**Trap: `tenantId` only ever holds an authorizer-*generated* UUID.** A JWT `tenant_id` claim is never persisted into `tenantId`; the claim-fallback path returns the claim without writing anything. So a stored `tenantId` is always a value this service minted, and the table is effectively a registry of instances that once hit the generate path. Reading `tenantId` as "this instance's platform tenant" is the single most common misreading of this table.

**`platformTenant`** is the canonical *real* platform tenant. It is written by the **`ddmr-tenant-migration-job`** after it successfully remaps a tenant's data (promoting the flagged `claimTenantMigration` value to `platformTenant`). When present, the lookup prefers it over `tenantId`; migrated records carry both (the original generated `tenantId` plus the real `platformTenant`).

A **`claimTenantMigration`** attribute is written (once, conditionally) when the stored tenant differs from the JWT `tenant_id` claim, flagging the record for reconciliation. The migration job scans the `claimTenantMigration_index` GSI for flagged records, remaps their DSS data old→new, then sets `platformTenant` and clears the flag.

**Trap: mismatch flagging only ever fires on the CSA path, so an M2M-only divergence is invisible here.** The M2M path (Tyk → the DSS in-pod `JwtFilter`) takes the tenant straight from the JWT claim and never touches this table. A tenant change seen only over M2M is therefore never flagged, never migrated, and never appears in this table at all. That is the root of the DSS `401 "Tenant mismatch"` chain: see `services/declaration-storage-service.md` and the step-by-step triage in `docs/dss-401-tenant-mismatch-playbook.md`.

---

## Tenant Resolution Logic

`DynamoDbTenantIdRepository.getTenantId` follows this priority order:

1. **DynamoDB hit**: if a record exists for `ORG#<organizationId>#<instanceId>`, return the stored `tenantId` (or `platformTenant` if present). Update `lastAccess` if it is stale.
2. **Claim fallback**: if no DB record exists but the JWT carries a `tenant_id` claim, return the claim value directly **without writing to DynamoDB**.
3. **Generate**: if neither exists and `tenant-authorizer.generateTenantId=true`, generate a UUID, write it with a conditional `attribute_not_exists(pk)` check to prevent races, and return it. If the conditional write races, a follow-up `getItem` retrieves the winner.
4. **Reject**: if neither exists and `generateTenantId=false`, return 401.

Mismatch detection: if DynamoDB has a record but its stored tenant ID differs from the JWT `tenant_id` claim, the service logs a warning and writes a `claimTenantMigration` marker onto the record (once). **The stored DB value is still returned**; the mismatch does not cause a rejection, so the divergence is silent from the caller's point of view.

---

## Configuration

`TenantAuthorizerProperties` (prefix `tenant-authorizer`) has exactly **one** field on `origin/main`: `generateTenantId`, defaulting to `false`.

The previous review of this page also documented `tenant-authorizer.rejectRequestInStageHack`, a staging shim that rejected new `(organizationId, instanceId)` pairs for an hour. **That property no longer exists.** What survives of it is the now-unconditional `all-basic-cloud-services : allow` scope requirement in `AuthorizeHandler` described above. If you are reading an older doc, runbook, or values file that references the flag, it is stale.

Other configuration is Spring/AWS standard: `aws.dynamodb.table` (required), `aws.dynamodb.localPort` (set to target a local DynamoDB), `aws.dynamodb.region` plus pool/timeout knobs in `AwsProperties`, and `csa.jwt.s3-host` for CSA key fetch. Read defaults from `AwsProperties.kt` and `src/main/resources/application.yaml`; do not quote them from here.

Deployment values are **not in this repo**. They live in `ddmr-deployments/helm/auth/values-*.yaml`, applied by `ddmr-deployments/argo/apps/tenant-authorizer-appset.yaml`. As of 2026-07-29:

| Environment | Table | `generate-tenant-id` |
|---|---|---|
| prod (us-east-1, eu-central-1, ap-northeast-1) | `ddmr-tenant-authorizer` | not set, so `false` |
| stage | `ddmr-tenant-authorizer` | **`true`** |
| integration | `ddmr-integration-tenant-authorizer` | **`true`** |
| dev (sandbox) | `sandbox-authorizer` | not set, so `false` |
| perf | `performance-authorizer` | not set, so `false` |

**Trap: generation is enabled in stage and integration, which is how generated tenants get minted in the first place.** "Generation is off in production" is true and is often quoted as if it meant generated tenants cannot exist. Records with a generated `tenantId` and no `platformTenant` are created in the lower environments and are the population that later diverges. Re-read the values files before asserting the setting for any environment.

---

## Gotchas

**`generateTenantId=false` means unknown tenants get a 401, not a new ID.** If a device cannot get through and there is no record in DynamoDB, the mapping must be seeded externally before requests will succeed.

**`platformTenant` versus `tenantId`.** `tenantId` is always an authorizer-generated UUID; `platformTenant` (when present) is the real platform tenant added by the `ddmr-tenant-migration-job`, and the lookup prefers it. A record with a generated `tenantId` and **no** `platformTenant` was never migrated. If its real tenant later diverges, and only ever appears over M2M (which bypasses this service), it stays unreconciled forever. This is the root pattern behind DSS "Tenant mismatch" 401s.

**JWT validation depends on CSA public keys in S3.** The `csa.jwt.s3-host` bucket must be reachable at startup and at validation time; key-fetch failures cause 401s for every request. Local development uses `csa-public-key-store-development`; production uses `csa-public-key-store-production`.

**`x-customer-id` is required.** HAProxy is expected to always supply it. If the header is missing, the handler *binding* fails before any business logic runs, so you get a framework-level error rather than one of the four 401 messages above.

**`lastAccess` can lag reality, so never use it for activity tracking.** `recordAccess` skips the DynamoDB `updateItem` entirely when the stored `lastAccess` is recent enough, where "recent enough" is the `A_TIME_GRANULARITY` constant in `DynamoDbTenantIdRepository.kt`. Read the value there; the durable point is that a record's `lastAccess` is a coarse "seen at some point in the last window", not a timestamp. Treating it as a last-seen for cleanup or churn analysis will drop live tenants.

**Do not confuse this `lastAccess` with DSS's `last_device_access`.** They are different attributes in different tables with different owners: `lastAccess` is on the authorizer's `ORG#...` record, `last_device_access` is on a DSS assignment row (`MDM#<tenant>|<device>|<channel>` / `A#<identifier>`). DSS's handling of that field is in flux: there is an unmerged `origin/ddmr-1249-move-last-device-access` branch on `declaration-storage-service` as of 2026-07-28. Check it before reasoning about DSS access timestamps.

---

## Where to find the data (verify rather than trust)

```bash
AT=~/Projects/DDmR/ddmr-authorizer-tenant; git -C $AT fetch origin -q
git -C $AT log --oneline origin/main --since=2026-07-29
git -C $AT for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# The whole authorization contract: check order, scope regex, all four 401 strings
git -C $AT show origin/main:src/main/kotlin/com/jamf/serviceauthorizertenant/handlers/AuthorizeHandler.kt

# Every configurable property the service actually has (currently one)
git -C $AT show origin/main:src/main/kotlin/com/jamf/serviceauthorizertenant/config/properties/TenantAuthorizerProperties.kt

# The lastAccess write-throttle window, and the resolution priority order
git -C $AT grep -n 'A_TIME_GRANULARITY\|platformTenant\|claimTenantMigration' origin/main \
  -- src/main/kotlin/com/jamf/serviceauthorizertenant/repository/DynamoDbTenantIdRepository.kt
```

Deployment reality is in a different repo, and is what actually decides behaviour per environment:

```bash
DEP=~/Projects/DDmR/ddmr-deployments; git -C $DEP fetch origin -q
grep -rn 'generate-tenant-id\|table:' $DEP/helm/auth/values-*.yaml
git -C $DEP show origin/main:argo/apps/tenant-authorizer-appset.yaml
```

Before concluding a record is unmigrated, confirm the migration job can even run. `docs/database.md` records that it queries a `tenant_index` GSI removed under DDMR-1035, so check the cluster rather than the GitOps config: `kubectl get cronjob -A | grep tenant-migration`.
