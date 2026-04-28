# API Layer

Last reviewed: 2026-04-28

## Overview

All DDmR HTTP traffic flows through a Tyk API gateway. Tyk handles JWT authentication, token-to-M2M translation, path routing, and CORS before requests reach service pods. Configuration lives in `tyk-gateway-management`, a GitOps repo that applies Kubernetes CRDs (ApiDefinitions and SecurityPolicies) via ArgoCD. Services themselves are not directly reachable from outside the cluster.

---

## Tyk Gateway Configuration Repo

Repository: `tyk-gateway-management`

### Directory Layout

```
<environment>/
  api-products/
    <product>/
      api-definitions[-region].yaml    # ApiDefinition CRDs
      access-policy[-region].yaml      # SecurityPolicy CRDs
  plans/
    default.yaml                       # Internal plan (jwt scope → policy mapping)
    default-external.yaml              # External plan
```

Each file is a Kubernetes CRD with `kind: ApiDefinition` or `kind: SecurityPolicy`. Multiple definitions can appear in a single file, separated by `---`. When a PR is merged to the repo, ArgoCD deploys the updated CRDs to the target cluster. There is no automatic deployment; an ArgoCD sync must be triggered after merge.

### Environments

| Directory | Purpose |
|---|---|
| `dev` | Development / sandbox (internal to engineering) |
| `sbox` | Sandbox (ad-hoc integration testing) |
| `stage` | Pre-production staging |
| `stable-stage` | Stable staging variant used for performance testing |
| `hc-stage` | Healthcare / StateRAMP isolated staging |
| `prod` | Production (US, EU, APAC regions) |

### Regions

Production and staging products use per-region files:
- `use1` / `use2` — US East (commercial)
- `euc1` — EU (Frankfurt)
- `apne1` — Asia-Pacific (Tokyo)

HC stage uses a single-region US deployment.

### Contributing

Changes require two separate PRs: one for non-prod environments (`dev/`, `sbox/`, `hc-stage/`, `stable-stage/`, `stage/`) and a second for `prod/`. Both PRs must be squashed to one commit. Raise changes in the `#help-api-gateway` Slack channel for approval.

---

## ApiDefinition Structure

Each ApiDefinition specifies:

- **`tags`**: Which gateway data plane receives the definition. Tag format is `<internal|external>-<region>` (e.g., `internal-use1`, `external-use1`). Internal gateways are cluster-internal; external gateways are internet-facing.
- **`proxy.target_url`**: The upstream service. In-cluster services use `http://<service>.<namespace>:<port>`. Some services are fronted by a regional platform URL (e.g., `https://device-inventory.use2.platform.jamfapps.io`).
- **`proxy.listen_path`**: The path prefix the gateway exposes. It is stripped (`strip_listen_path: true`) before forwarding.
- **`custom_middleware_bundle`**: A zip file identifying the Tyk plugin to run. DDmR uses two main bundles:
  - `jwt-to-m2m-*.zip` — validates an internal M2M JWT and injects an M2M header for the upstream service
  - `auth0-to-m2m-with-authorization-*.zip` — validates an external (Auth0/CSA) JWT and performs tenant authorization before injecting M2M headers
- **JWT configuration**: `enable_jwt: true`, `jwt_signing_method: rsa`, with `jwt_scope_to_policy_mapping` driving access control per JWT scope claim.
- **`version_data.versions.Default.global_headers_remove`**: Tyk strips `x-tenantid` from inbound client requests before forwarding, preventing spoofing. The JWT sidecar injects the verified header.

---

## DDmR Service Route Mapping

Listen paths are what external callers use; the gateway strips the prefix before forwarding.

| Tyk Product | Listen Path | Upstream Service (prod) | Internal / External |
|---|---|---|---|
| `scope-eng` | `/scoping` (prod, stage→integration); `/scope-eng-prerelease` (stage→ddmr-stage) | `scoping-engine-svc.ddmr-<env>:7070` | Both |
| `dss` | `/dss` | `declaration-storage-svc.ddmr-<env>:7070` | Both |
| `declaration-service` | `/declaration-service` | `declaration-service.ddmr-<env>:7070` (internal); `declaration-service-svc.ddmr-<env>:7070` (external) | Both |
| `blueprint-components` | `/blueprints/components/{sw-update,declarations,cps,declaration-service,app-service,sls}` | Multiple — `/blueprints/components/declaration-service` targets `declaration-service.ddmr-<env>:7070` with `X-Forwarded-Prefix` header | Internal (UI variant for app-service is external) |
| `app-declaration-service` | `/app-declaration-service` | `bcads.use2.platform.jamfapps.io` | Internal |
| `declaration-reporting-service` | `/ddm/report` | `drs.use2.platform.jamfapps.io` | Internal |
| `blueprint-management` | `/blueprints/management` | `blueprint-management-service.ocean-prod:8080` | Both |
| `configuration-profile-service` | `/cps` | `cps.use2.platform.jamfapps.io` | Both |
| `device-inventory-service` | `/management/devices` | `device-inventory.use2.platform.jamfapps.io` | Both |
| `device-group-inventory-service` | `/management/device-groups` | `device-group-inventory.use2.platform.jamfapps.io` | Both |
| `mdm-protocol-handler` | `/mdm-protocol-handler` | `mdm-protocol-handler.use2.platform.jamfapps.io` | Internal |
| `school-declaration-scoping` | `/school-scoping` | `school-declaration-scoping-service.ocean-prod:8080` | Internal |

Services running as in-cluster pods (scoping-engine, declaration-storage-service, declaration-service) use Kubernetes DNS names. Services that live outside the cluster (e.g., device-inventory, CPS) are referenced by their platform hostname. In stage, `scope-eng` exposes two listen paths: `/scope-eng-prerelease` targets `ddmr-stage` (the latest prerelease build) and `/scoping` targets `ddmr-integration`.

**Dual-product routes:** declaration-service is reachable through two Tyk products at two different listen paths backed by the same pod. Each product has its own SecurityPolicy and therefore its own scope:

| Caller path | Tyk product / scope name | Notes |
|---|---|---|
| `/declaration-service/...` | `declaration-service-product` | Direct route. Used by tooling and direct callers. |
| `/blueprints/components/declaration-service/...` | `blueprint-components-api-product` | Bundle scope shared across `sw-update`, `declarations`, `cps`, `declaration-service`, `app-service`, `sls`. Used by callers (e.g. blueprint-management-service, MFE traffic, the declaration-service component-test suite when run against stable-dev) that traverse the unified blueprint-components surface. |

The two scopes are not interchangeable on a given URL — Tyk's SecurityPolicy is bound to specific ApiDefinitions/listen paths.

Both the gateway and the in-pod `JwtFilter` enforce scope. The pod's `JwtScopeValidator` (`src/main/kotlin/com/jamf/declaration/auth/JwtFilter.kt`) accepts the JWT if it carries **any** of the configured `requiredScopes`. The default set is `{declaration-service-product, blueprint-components-api-product}`, so a request that survives Tyk's path-bound check on either route also satisfies the pod-level check. Pre-PR-#158 (DDMR-1088), `JwtScopeValidator` accepted only a single scope, so traffic via the blueprint-components route would have been rejected by the pod even though Tyk authorized it; PR #158 broadened it to ANY-match.

---

## Scoping Engine Route Reference

All scoping-engine routes are under `/api/v1`. Tyk strips `/scoping`, so the full external path is `/scoping/api/v1/...`.

| Method | Path | Handler |
|---|---|---|
| `HEAD` | `/api/v1` | Connectivity check (unauthenticated) |
| `POST` | `/api/v1/scope` | Create scope |
| `PUT` | `/api/v1/scope/{scopeId}` | Update scope groups |
| `GET` | `/api/v1/scope/{scopeId}` | Get scope |
| `DELETE` | `/api/v1/scope/{scopeId}` | Delete scope |
| `PUT` | `/api/v1/scope/{scopeId}/assignment` | Update declaration/app assignments |
| `GET` | `/api/v1/scope/{scopeId}/assignment` | Get assignments |
| `DELETE` | `/api/v1/scope/{scopeId}/assignment` | Delete assignments |
| `GET` | `/api/v1/scope/{scopeId}/devices/count` | Device count for scope |
| `GET` | `/api/v1/scope/{scopeId}/devices/publish/{action}` | Trigger device sync publish |
| `PUT` | `/api/v1/toggle-scopes` | Bulk enable/disable scopes |
| `GET` | `/api/v1/group/{groupId}/used` | Check if group is referenced by any scope |

The external Tyk product for scoping-engine exposes a whitelist of routes (`/api/v1/scope`, `/api/v1/scope/{id}`, `/api/v1/scope/{id}/devices/count`) and blocks all others. The internal product allows all paths, with only actuator endpoints blacklisted.

---

## API Contract Conventions

### Required Headers

Every request to a DDmR service must carry an `X-TenantId` header. The JWT sidecar injects this from the verified JWT before it reaches the application. If the header is absent, `AbstractApiRequest` throws `ResponseStatusException(401, "No tenant identifier")`.

`X-EnvironmentId` is optional and treated as a tenant-scoped environment discriminator. If absent, it resolves to `null` and most handlers proceed without filtering by environment. Neither header is set by API clients — both are injected by the authentication infrastructure.

### Request Serialization

Services use `kotlinx.serialization`, not Jackson. The JSON codec is configured with `ignoreUnknownKeys = true` and `encodeDefaults = true`. All request and response models must be annotated `@Serializable`. Path variables and query parameters are extracted in the request class constructor.

### Error Response Format

Validation and business-logic errors return a JSON body with an `errors` array:

```json
{
  "errors": [
    { "description": "Scope does not exist" }
  ]
}
```

Additional key-value pairs may appear alongside `description` for structured errors. HTTP 4xx errors from Jakarta Validation or Spring are mapped through `WebExceptionMapper` implementations registered with `MappingErrorWebExceptionHandler` (ordered `-2`).

### Unauthenticated Endpoints

The following paths are configured as `ignored` in Tyk (bypassing JWT auth) and in the sidecar `open` list:

- `HEAD /api/v1` — connectivity check
- `GET /api-docs(.*)`  — OpenAPI docs (dev/stage only)
- `GET /api-schema(.*)` — OpenAPI schema (dev/stage only)

Actuator endpoints (`/actuator`, `/actuator-jwt`) are blacklisted at the Tyk layer, preventing external access even though Spring exposes them internally.

---

## Service-to-Service HTTP Calls

**Scoping Engine → DSS**: Uses the `DeclarationAssignments` starter client (`declaration-product-springboot-starter`) configured via `declaration.client.host`. In production this resolves to `https://us.int.apigw.jamf.com/dss`.

**Scoping Engine → VPP App Service**: `VppAppClient` calls `POST /v1/components/app-service/assignment/device/{device}/{channel}` when a device sync triggers app assignment. Requires `vpp-app.client.host` to be configured; skips silently if absent.

**Other**: Blueprint Management, Declaration Reporting Service, and App Declaration Service all call DSS for declaration content or device data.

---

## M2M Auth for Outbound Calls

M2M tokens are fetched via the `robocop` library (`com.jamf.stratus.m2m.robocop`). `M2MService.getRestToken(tenantId, credentials)` fetches a per-tenant token; credentials come from AWS Secrets Manager. `m2m.env` selects the Robocop environment (`DEV`, `STAGE`, `PROD_US`, `PROD_EU`, `PROD_AP`); `m2m.customEnv.url` overrides the endpoint for HC and perf environments. Tenant must be a valid UUID — `M2MTokenAcquisitionException(retryable=false)` is thrown otherwise. See `auth-and-tenancy.md` for the full JWT sidecar description.

---

## Gateway URLs by Environment

### Commercial

| Environment | Internal Gateway Base URL | External Gateway Base URL |
|---|---|---|
| Dev / Sbox | `https://us.api.dev.platform.jamflabs.io` | (same) |
| Stage | `https://us1.api.stage.platform.jamflabs.io` | `https://us1.api.stage.platform.jamflabs.io` |
| Prod US | `https://us.int.apigw.jamf.com` | `https://us.int.apigw.jamf.com` |
| Prod EU | `https://eu.int.apigw.jamf.com` | `https://eu.int.apigw.jamf.com` |
| Prod APAC | `https://apac.int.apigw.jamf.com` | `https://apac.int.apigw.jamf.com` |

### HC (Healthcare / StateRAMP)

| Environment | Internal Gateway Base URL |
|---|---|
| HC Stage | `https://us1.stage.platform-hc.jamflabs.io` |

HC stage is a fully isolated AWS deployment (`604006981984`) using `hc-stage/` CRDs. Service DNS names resolve within the `ddmr-stage` namespace; M2M JWKS and Pulsar broker use `platform-hc.jamflabs.io` hostnames.

---

## External vs. Internal Gateway

Every DDmR service with public API access has two Tyk definitions in prod:

- **Internal** (tag `internal-<region>`): accepts internal M2M JWTs; all routes open. Used by service-to-service calls and tooling. The `jwt-to-m2m` middleware bundle translates the incoming JWT to an M2M token before forwarding.
- **External** (tag `external-<region>`): accepts external (Auth0/CSA) JWTs; routes are restricted via a whitelist. Uses `auth0-to-m2m-with-authorization` middleware which also enforces tenant-level authorization. CORS headers are enabled on external definitions.

For services like DSS where the external API is intentionally limited, the external Tyk definition uses `white_list` to expose only the device-facing MDM protocol paths (e.g., `GET /api/v1/device/{device}/{channel}/tokens`, `PUT /api/v1/declaration`).
