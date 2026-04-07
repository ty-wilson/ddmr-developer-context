# API Layer

Last reviewed: 2026-04-07

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

The table below covers DDmR services that have Tyk products. Listen paths are what external callers use; the gateway strips the prefix before forwarding.

| Tyk Product | Listen Path | Upstream Service (prod) | Internal / External |
|---|---|---|---|
| `scope-eng` | `/scoping` | `scoping-engine-svc.ddmr-<env>:7070` | Both |
| `dss` | `/dss` | `declaration-storage-svc.ddmr-<env>:7070` | Both |
| `declaration-service` | `/declaration-service` | `declaration-service.ddmr-<env>:7070` | Both |
| `app-declaration-service` | `/app-declaration-service` | `bcads.use2.platform.jamfapps.io` | Internal |
| `declaration-reporting-service` | `/ddm/report` | `drs.use2.platform.jamfapps.io` | Internal |
| `blueprint-management` | `/blueprints/management` | `blueprint-management-service.ocean-prod:8080` | Both |
| `configuration-profile-service` | `/cps` | `cps.use2.platform.jamfapps.io` | Both |
| `device-inventory-service` | `/management/devices` | `device-inventory.use2.platform.jamfapps.io` | Both |
| `device-group-inventory-service` | `/management/device-groups` | `device-group-inventory.use2.platform.jamfapps.io` | Both |
| `mdm-protocol-handler` | `/mdm-protocol-handler` | `mdm-protocol-handler.use2.platform.jamfapps.io` | Internal |
| `school-declaration-scoping` | `/school-scoping` | `school-declaration-scoping-service.ocean-prod:8080` | Internal |

Services running as in-cluster pods (scoping-engine, declaration-storage-service, declaration-service) use Kubernetes DNS names. Services that live outside the cluster (e.g., device-inventory, CPS) are referenced by their platform hostname.

The `scope-eng` product does not appear in `prod/api-products/` — scoping-engine has no direct Tyk product in prod under that name. It is present in `hc-stage` and `dev/stage` as `scope-eng`.

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

`X-EnvironmentId` is optional and treated as a tenant-scoped environment discriminator. If absent, it resolves to `null` and most handlers proceed without filtering by environment.

Neither header is set by API clients directly. They are injected by the authentication infrastructure.

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

DDmR services make synchronous HTTP calls to other services for real-time data. All outbound calls use an M2M Bearer token (see M2M Auth section below).

### Scoping Engine Outbound Calls

**Declaration Storage Service (`dss`)**: Scoping-engine calls DSS via the `DeclarationAssignments` Spring Boot starter client (`declaration-product-springboot-starter`). The client is configured via `declaration.client.host`, which points to the DSS gateway URL. Example (local dev targeting stage): `https://us.int.stage.apigw.jamfnebula.com/dss`. In production the host resolves to `https://us.int.apigw.jamf.com/dss`.

**VPP App Service**: Scoping-engine calls a Deployable VPP service at `vpp-app.client.host` to push app assignments when a device sync is triggered. The `VppAppClient` hits `POST /v1/components/app-service/assignment/device/{device}/{channel}` with an M2M Bearer token. This client is conditional — if no `vpp-app.client.host` is configured the client logs an error and skips the call rather than failing.

### Other Known Inter-Service Calls

- **Blueprint Management Service** calls Declaration Storage Service for declaration retrieval.
- **Declaration Reporting Service** calls Declaration Storage Service for device/declaration data.
- **App Declaration Service** calls DSS for declaration content.

---

## M2M Auth for Outbound Calls

All service-to-service calls use M2M tokens fetched via the `robocop` library (`com.jamf.stratus.m2m.robocop`). The token is scoped to a specific tenant UUID and a set of service scopes. In scoping-engine:

- `M2MService.getRestToken(tenantId, credentials)` fetches a token for a given tenant.
- Credentials (client ID and secret) are loaded from AWS Secrets Manager.
- The `m2m.env` config property selects the Robocop environment: `DEV`, `STAGE`, `PROD_US`, `PROD_EU`, or `PROD_AP`.
- A `m2m.customEnv.url` override allows pointing at an arbitrary M2M endpoint (used in HC and perf environments).

Tenant must be a valid UUID; if not, `M2MTokenAcquisitionException(retryable=false)` is thrown before Robocop is called. See `auth-and-tenancy.md` for a full description of M2M auth and the JWT sidecar.

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

The legacy `apigw.jamfnebula.com` hostname (`tyk-gateway.stage.apigw.jamfnebula.com`) appears in older performance test scripts but is superseded by the `platform.jamflabs.io` and `apigw.jamf.com` patterns above.

### HC (Healthcare / StateRAMP)

| Environment | Internal Gateway Base URL |
|---|---|
| HC Stage | `https://us1.stage.platform-hc.jamflabs.io` |

HC stage is a fully isolated AWS deployment (`604006981984`). It uses the same Tyk CRDs as `hc-stage/` in the gateway management repo but points to cluster-internal service DNS names within `ddmr-stage` namespace. The M2M JWKS endpoint and Pulsar broker also resolve to `platform-hc.jamflabs.io` hostnames.

---

## External vs. Internal Gateway

Every DDmR service with public API access has two Tyk definitions in prod:

- **Internal** (tag `internal-<region>`): accepts internal M2M JWTs; all routes open. Used by service-to-service calls and tooling. The `jwt-to-m2m` middleware bundle translates the incoming JWT to an M2M token before forwarding.
- **External** (tag `external-<region>`): accepts external (Auth0/CSA) JWTs; routes are restricted via a whitelist. Uses `auth0-to-m2m-with-authorization` middleware which also enforces tenant-level authorization. CORS headers are enabled on external definitions.

For services like DSS where the external API is intentionally limited, the external Tyk definition uses `white_list` to expose only the device-facing MDM protocol paths (e.g., `GET /api/v1/device/{device}/{channel}/tokens`, `PUT /api/v1/declaration`).
