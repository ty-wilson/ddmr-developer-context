# DSS 401 "Tenant Mismatch" Playbook

Last reviewed: 2026-06-24 (Tyler Wilson)

How to diagnose DSS **401 "Tenant mismatch"** / orphaned-tenant problems — how tenant IDs are created, stored, and referenced, how the Jan-2026 Tyk-routing change exposed a latent class of instances, where to find the data, and a step-by-step triage. Cross-refs: `auth-and-tenancy.md`, `database.md`, `services/declaration-storage-service.md`, `services/ddmr-authorizer-tenant.md`.

Terms: **CSA** = Client-Side Auth (end-user Jamf-ID/Auth0 token path); **M2M** = machine-to-machine OAuth token (Keycloak/m2m-foundry); **Tyk** = internal API gateway (`{us|eu|apac}.int.apigw.jamf.com`); **Atlas** = platform deployment/tenant provisioning; **generated tenant** = a UUID DDmR minted itself, not a real platform tenant.

## How the 401 happens
DSS looks up a declaration **by UUID alone**, then `DeclarationApiHelper.validateDeclarationAppropriate` compares the **request's tenant** to the declaration's stored **`tenant`** attribute. Differ → **`401 "Tenant mismatch"`** (not 403/404). UUID not found → `204` (GET) / `400` (assign). So a 401 here is never a credential failure — it means "this token's tenant doesn't own this declaration": the declarations were stamped with one tenant and the request now carries a different one.

## How tenant IDs are created / stored / referenced
Two auth paths resolve the tenant differently:
- **M2M (Tyk gateway):** DSS reads the tenant straight off the JWT claim `https://www.jamf.com/tenant.tenantId` — the **real Atlas tenant**. No DynamoDB lookup, no migration flagging. (DSS validates JWTs in-pod via `JwtFilter`, not the sidecar.)
- **CSA (HAProxy → `ddmr-authorizer-tenant`):** resolves via the `ddmr-tenant-authorizer` DynamoDB table.

`ddmr-tenant-authorizer` (prod acct 613358915025, us-east-1): key `pk = ORG#<organizationId>#<instanceId>` where organizationId = CSA JWT `sub` (Salesforce/CRM org id) and instanceId = the `x-customer-id` header (the JSS id). One GSI: `claimTenantMigration_index`. No tenant GSI → find a row by tenant via a filtered **scan** on `tenantId`/`platformTenant`.
- `tenantId` = **only ever an authorizer-generated UUID** (`UUID.randomUUID()`); a CSA `tenant_id` claim is never persisted into it. Generation only happens when `generateTenantId=true` (historically on; default **false** now → unknown tenant returns 401). The table is effectively a registry of instances that hit the claimless-CSA generate path; always-claim instances get **no row**.
- `platformTenant` = the **real Atlas tenant**, written only by the `ddmr-tenant-migration-job` after a successful migration (promotes the flagged value). Lookup returns `platformTenant` if present, else `tenantId`. Migrated records carry both.
- `claimTenantMigration` = transient flag, written once when a CSA token's `tenant_id` claim differs from the stored tenant. The migration job (scans `claimTenantMigration_index`) remaps the tenant's DSS rows old→new, then sets `platformTenant`. **This job is idle and the `tenant_index` GSI it needs on the DSS table was removed from prod** — it cannot run now.
- The **M2M path never writes any flag**, so a tenant change seen only over M2M is invisible to the authorizer and the migrator.

Where generated/diverged tenants come from: on-prem→cloud migrations (the JPRO-19169 cohort, Dec 2025) arrived with **claimless CSA tokens** and blank `declarative_management_organization_information`. While generation was on, the authorizer minted a synthetic `tenantId` and the instance created its DDM declarations under it. The real Atlas tenant (lost/changed during migration, separately fixed via the "Register a Deployment (CSA)" change control) then diverges from it.

## Timeline of routing changes (states at points in time)
- **Before 2026-01-22:** Jamf Pro → DSS went **direct** to the regional DSS service URL (`dss.{use1|euc1|apne1}.services.jamfcloud.com`) via the `dss.properties` file / `dss_url`. Claimless-CSA instances resolved to their generated tenant, declarations matched, everything worked.
- **2026-01-22:** `jamf/atlas-global-resources` commit `6fe0fe71f` ("update dss url for Tyk gateway") changed the **default** DSS URLs in `pipeline.yaml` to the **internal Tyk gateway** (`https://us.int.apigw.jamf.com/dss`, etc.). **This is the trigger:** any instance with **no per-instance `dss_url` override** picks up the Tyk default on its next roll; going through Tyk surfaces the **real Atlas tenant** to DSS → mismatch → 401. (Instances with a custom `dss_url` were pre-loaded via JPPRC-203 / change control CO-14705 and were spared.)
- **Jamf Pro 11.26.0** (JPCFM-5293 / jss PR #490425; planned for 11.25 but **missed the 11.25 code cut**; cloud ~Mar/Apr 2026): the **app code** that moved DSS URL resolution from `dss.properties` → the DB knob `com.jamfsoftware.dme.dss.url.override` with Tyk fallback. It *formalized* the routing but was **not** the original trigger (the Jan-22 config was), which is why there's no 401 step at its rollout.
- **Jamf Pro 11.28.0** (CERB-123 / jss PR #491441): `com.jamfsoftware.dme.dss.use.m2m.auth` — token-level CSA→M2M for the DSS client, default **false**. A separate, even-later step.

## Where to find the data
- **Declarations + their tenant:** `ddmr-declaration-storage` DynamoDB (acct 613358915025, us-east-1). Declaration payload: `pkey=DECL#<uuid>`, `psort=PAYLOAD`, attrs `tenant`, `type`, `created`, `last_device_access` (hour-rounded; last successful device read — proxy for when it stopped working). Assignments: `pkey=MDM#<tenant>|<deviceId>|<channel>`. (`tenant_index` GSI exists in staging, removed in prod.)
- **Authorizer mapping:** `ddmr-tenant-authorizer` (same acct/region). Find a row by tenant: `aws dynamodb scan --table-name ddmr-tenant-authorizer --filter-expression "tenantId = :t OR platformTenant = :t"`.
- **AWS access:** `aws sso login --profile prod` (= acct 613358915025).
- **Instance version / deployment record / `dss_url` override:** AWS Managed Grafana workspace `g-8204171cbc.grafana-workspace.us-east-1.amazonaws.com` (separate IAM Identity Center / Okta SSO), backed by **Athena** (DataOps-Ciliation). `atlaslocal.__t_deployment where fqdn='...'` → `version`, `version_track`, `tenant_id`, `dss_url` (null = uses the default → exposed to the Tyk switch), `account_id`. Current-state snapshots, not deep history. Customer JSS pods run in acct **628897759306** (`jamf-jpro-external-prod`).
- **401 rate over time:** observability Grafana (`grafana.observability.jamf.build`), Thanos: `tyk_http_requests_total{api_name="Declaration Storage (US)", response_code="401"}`. Caveat: caller `alias`/`oauth_id` labels are unreliable historically and `path` is templated (`/declaration/{uuid}`), so you can't isolate one instance from Tyk metrics. HWTP deploy annotations track platform-component deploys, NOT per-customer JSS version.
- **DSS server logs:** CloudWatch `/aws/eks/<jpro-cluster>/ddmr-prod/declaration-storage` (~90d retention; goes stale when the cluster rotates). DDmR logs also in Unified Logging OpenSearch (~27d).
- **Config/code history:** `jamf/atlas-global-resources` (default `dss_url`, `pipeline.yaml`), `jamf/k8s-jamfpro-startup` (`functions/jamfpro/jamfpro.sh` writes `dss.properties`), `jamf/jss` (app code; branches `release/<ver>`, tags `<ver>-t<epochms>`). To prove which release a commit shipped in: `gh api repos/jamf/jss/compare/<sha>...<tag>` → `behind_by:0` means the tag contains the commit.

## Triage playbook for a new 401 / tenant-mismatch case
1. Get the instance's fqdn, JSS id, CRM/org id, and **current tenant** (Athena `__t_deployment`).
2. Read its declaration rows in `ddmr-declaration-storage`; compare each declaration's stored `tenant` to the current tenant. **Differ → orphaned-tenant root cause**; the stored value is the old/generated tenant.
3. Check the authorizer row (scan by tenant): a generated `tenantId` + no `platformTenant` + no `claimTenantMigration` + current live tenant differs = a latent case **never migrated and never flagged** (M2M-only / claimless-CSA).
4. Check `dss_url` override (deployment record / CAPS): **null = on the Tyk default**, so it's exposed.
5. **Fix** (per-instance, support-run on the Jamf Pro DB): `UPDATE declarative_management_settings SET csa_subject='', dss_migrated=0 WHERE id=1;` + `DELETE FROM declarative_management_dss_declaration WHERE declaration_uuid IN (<the system-declaration uuids from the 401 logs>);` + restart → Jamf Pro recreates its system declarations under the current tenant. (Leaves the old-tenant declarations orphaned in DSS as harmless litter.)
6. To find latent cases proactively: scan `ddmr-tenant-authorizer` for rows with a generated `tenantId`, no `platformTenant`, whose live M2M/platform tenant differs.
