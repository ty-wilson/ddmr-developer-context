# Auth And Tenancy

Last reviewed: 2026-06-24

## Overview

DDmR services use two ingress mechanisms, and traffic does not always pass through the sidecar:

- **Tyk API Gateway (M2M path):** All M2M service-to-service calls route through Tyk, which forwards to the `ddmr-jwt-sidecar` on port 7070. The sidecar validates the M2M JWT and injects HTTP headers (primarily `X-TenantId`) before proxying to the application on port 8080.
- **HAProxy Ingress (CSA/legacy path):** Some services (notably DSS) define a separate HAProxy-based Kubernetes ingress that routes directly to the application on port 8080, **bypassing the sidecar**. Authentication is handled by the `ddmr-authorizer-tenant` via HAProxy's `auth-url` annotation — the authorizer validates the CSA JWT and returns `X-TenantId`, which HAProxy injects into the forwarded request.

Services read `X-TenantId` regardless of which path delivered it. The `spring-m2m-authentication` library provides in-process M2M JWT validation as an alternative to the sidecar; declaration-service and DSS each ship their own in-pod `JwtFilter` (see "In-Pod JwtFilter" below). The migration off the sidecar started with declaration-service under DDMR-1088 and DSS has since followed; scoping-engine still uses the sidecar.

---

## JWT Sidecar (`ddmr-jwt-sidecar`)

The sidecar is a Micronaut-based proxy that runs as a second container in each pod. All external traffic enters the pod on port 7070 (the sidecar), which then proxies to the application on port 8080 after authentication.

### What It Does

The filter (`JwtProxyFilter`) processes every incoming request:

1. Actuator requests (`/actuator-jwt/...`) are passed directly to Micronaut's own handlers.
2. Requests matching the configured `open` endpoint list are proxied without authentication, but any proxy headers that would normally be set are stripped from the request first to prevent spoofing.
3. All other requests must carry a `Bearer` token in the `Authorization` header. The token is parsed, its issuer is looked up in `JwtProxySignatures`, and the signature is verified via JWKS. If the issuer is not in the configured map the request is rejected with 401.
4. The `scope` claim on the token is checked against a configured value. When scope is configured as `"*"` the check is bypassed entirely and the request is allowed regardless of the token's scope claim — no regex is applied. For CSA tokens scoping-engine requires `scope: "all-basic-cloud-services"`.
5. JWT claims are extracted into HTTP headers per the `proxy.headers` configuration. The transformation supports three modes: `REQUIRED` (401 if claim absent), `OPTIONAL` (strip the header if claim absent), and `BOOLEAN` (sets `true`/`false` based on claim presence).
6. The mutated request (with injected headers and rewritten URI) is forwarded to `localhost:8080`.

### Deployment

The sidecar is conditionally included in the Helm deployment template. In `scoping-engine/helm/scoping-engine/templates/deployment.yaml`, if `.Values.auth` is set a second container named `auth` is added to the pod. The image is taken from `auth.repo`/`auth.tag` in `values.yaml`. The MICRONAUT_ENVIRONMENTS variable selects the environment-specific TOML config inside the sidecar image. The ECR repo is `jamf/ga/ddm/jwt`.

### Auth Type Support

`JwtProxySignatures` maps JWKS URLs to token type (CSA or M2M). The sidecar supports multiple JWKS endpoints simultaneously:

- `m2m.jwksInternal` — internal M2M (required)
- `m2m.jwksExternal` — external M2M (optional)
- `m2m.jwksInternalAlt` / `m2m.jwksExternalAlt` — alternate endpoints used during infrastructure migrations

Different `.toml` environment profiles bake the correct JWKS URLs into the image:
- `application-stage.toml` — `us1.api.stage.platform.jamflabs.io`
- `application-prod-use1.toml` — `us.int.apigw.jamf.com`
- `application-fi.toml` — Tyk gateway at `tyk-gateway.stage.apigw.jamfnebula.com` for `jwksInternal`, with `internalIssuer` override pointing to `us.int.stage.apigw.jamfnebula.com` for the `iss` claim

### Sidecar Configuration via Helm

The `authProperties` block in a service's Helm values becomes a ConfigMap mounted at `/config` inside the sidecar container. Scoping-engine configures:

```yaml
authProperties:
  m2m:
    scope: "*"
  csa:
    scope: "all-basic-cloud-services"
  proxy:
    headers:
      "x-tenantid":
        csa-claim: "tenant_id"
        csa-claim-field: ""
        m2m-claim: "https://www.jamf.com/tenant"
        m2m-claim-field: "tenantId"
    open:
      - method: "HEAD"
        path-prefix: "*"
```

This tells the sidecar to map the CSA `tenant_id` claim or the M2M `https://www.jamf.com/tenant.tenantId` JSON field to the `x-tenantid` header.

---

## Tenant Resolution Flow (CSA path)

For user-facing requests (CSA tokens), the `ddmr-authorizer-tenant` service resolves an opaque customer/org identity into a stable UUID tenant ID before the request reaches downstream services. The authorizer is invoked as a subrequest by HAProxy (via `auth-url` annotation), not as an inline proxy.

```
CSA path (e.g. DSS with ingress.legacyEnabled):
  Client -> HAProxy Ingress ({release}-authorized, path /api)
         -> HAProxy calls tenant-authorizer-svc:8080/authorize as auth subrequest
            * validates CSA JWT, resolves tenant in DynamoDB
            * returns X-TenantId and X-Auth-Src headers
         -> HAProxy injects response headers, forwards to service port 8080 (no sidecar)
         -> service reads X-TenantId from request headers

M2M path (all services via Tyk):
  Service A -> Tyk Gateway (strips X-TenantId, validates M2M JWT)
            -> service pod port 7070 (sidecar validates JWT, extracts X-TenantId from claim)
            -> service port 8080 reads X-TenantId
```

The `ddmr-authorizer-tenant` is a Spring Boot WebFlux service acting as a Lambda-style authorizer. It:

- Validates the CSA JWT via Spring Security's `oauth2ResourceServer().jwt()` with the CSA JWKS URI from S3.
- Requires `token_use == "access"` on the JWT.
- When the `rejectRequestInStageHack` feature flag is enabled: temporarily rejects the first request per `(organizationId, customerId)` pair for one hour (simulating a missing `tenant_id` claim), and also checks that the JWT scope matches `all-basic-cloud-services:allow`.
- Looks up the mapping `ORG#<organizationId>#<instanceId>` in a DynamoDB table (`tenant-authorizer`) keyed by `pk`.
- If no entry exists and `generateTenantId` is enabled, generates a UUID and writes it with a condition expression to prevent race conditions.
- If the JWT carries a `tenant_id` claim and it disagrees with the stored record, logs a warning and flags the record for migration (`claimTenantMigration` attribute).
- Returns `X-TenantId` (the tenant UUID) and `X-Auth-Src` (a `CSA:<organizationId>:<customerId>` string) as response headers to the API gateway.

**Generated vs real tenant (root of "Tenant mismatch" 401s).** The `tenantId` the authorizer stores is only ever a UUID it *generated* (when `generateTenantId` was enabled and no claim was present); a `tenant_id` claim is never persisted into `tenantId`. The real platform tenant lands in a separate `platformTenant` attribute, written by the `ddmr-tenant-migration-job` after a successful migration, and is preferred by the lookup when present. Crucially, **only the CSA path consults this table.** The M2M path (Tyk → DSS in-pod `JwtFilter`) reads the tenant straight from the `https://www.jamf.com/tenant.tenantId` claim and never touches the authorizer — so if an instance's real tenant only ever arrives over M2M, the authorizer never sees the divergence, never flags it (`claimTenantMigration`), and the migrator never reconciles it. That orphaned-generated-tenant state is what produces DSS "Tenant mismatch" 401s once the instance's DSS traffic starts presenting the real tenant. See `services/ddmr-authorizer-tenant.md` and `services/declaration-storage-service.md`.

---

## Service-Side Header Extraction (`AbstractApiRequest`)

`AbstractApiRequest` is defined in `ApiRequests.kt`. Every HTTP handler in scoping-engine extends it, and it extracts the tenancy attributes in its constructor:

- `X-TenantId` (constant `TENANT_HEADER`): required. If absent or blank, throws `ResponseStatusException(401, "No tenant identifier")`.
- `X-EnvironmentId` (constant `ENVIRONMENT_HEADER`): optional. Returns `null` if absent or blank.
- `X-DivisionId` (constant `DIVISION_HEADER`): optional. Returns `null` if absent or blank — see "Division Context" below.

These values are available as `tenant`, `tenantEnv`, and `callerDivision` properties on any request object. None of these are documented in the OpenAPI spec as direct client concerns because they are injected by the sidecar/authorizer/`JwtFilter`, not set by API callers.

---

## M2M Auth via Robocop

When DDmR services call other platform services (e.g., declaration-storage-service, VPP app service), they use M2M tokens fetched from the Robocop library (`com.jamf.stratus.m2m.robocop`).

`M2MService` (scoping-engine) wraps `M2MToken` from Robocop:

```
M2MService.getRestToken(tenantId, credentials)
  -> M2MToken.fetchToken(clientId, clientSecret, scopes, tenantServiceIdentity)
     -> returns Bearer token string
```

The `M2MProperties` enum selects the Robocop environment: `DEV`, `STAGE`, `PROD_US`, `PROD_EU`, `PROD_AP`, or a `customEnv` with an arbitrary URL. Only one of `env` or `customEnv` can be set; validation fires at startup via `InitializingBean`.

Credentials (clientId/clientSecret) are loaded from AWS Secrets Manager at runtime via `LoadablePropertyResolver`. The scoping-engine uses the secret path `ddmr/stage/scoping/sync-credentials`.

The tenant must be a valid UUID for Robocop. If the tenant string is not a valid UUID, `M2MService` throws `M2MTokenAcquisitionException(retryable=false)` before attempting the fetch.

---

## Division Context (AD-16 Milestone 1)

Division is a platform-wide logical subsection of an environment — similar to Sites in Jamf Pro or Locations in Jamf School. Division-aware filtering is each service's responsibility (see [AD-16](https://jamfsoftware.atlassian.net/wiki/spaces/ARCH/pages/5577703604)). The platform's contract is: services receive a `divisionId` UUID on the M2M JWT claim; everything else (storage, filtering, reference-consistency) is up to the service.

### Claim shape

`divisionId` is added as a sibling field on both `https://www.jamf.com/tenant` and `https://www.jamf.com/environment` claims:

```json
"https://www.jamf.com/tenant": {
  "tenantId": "64427c7b-...",
  "environmentId": "a3f55b34-...",
  "organizationId": "629d3780-...",
  "divisionId": "...",      // present when an admin is working in a division
  "url": "..."
}
```

A global (no-division) admin's claim simply omits the field. The `JwtFilter.takeIf { isNotBlank() }` guard means an empty-string `divisionId` is also treated as "no division" rather than written as `""`.

### Where divisionId is set (token-exchange flow only)

The Tyk gateway's `auth0-to-m2m` plugin (in `tyk-custom-plugins`) extracts the external `X-Jamf-Division-Id` request header, validates the admin's ACL via the Pro/School permissions endpoint, then calls the M2M Service (Keycloak custom provider in `m2m-foundry`) with an internal `divisionId` header during a **token-exchange** request. The Keycloak provider applies the header value to the tenant/environment claims of the issued M2M token.

Two consequences:

- **`client_credentials` flow never adds divisionId.** Only token exchange (initiated by Tyk from an external request) triggers the divisionId-injection code path. A direct `curl` to the M2M issuer with `grant_type=client_credentials` will always return claims without `divisionId`, regardless of how the work has progressed — this is by design, not a bug.
- **The `divisionId` header is reject-listed for non-Tyk callers.** `m2m-terraform` configures the `reject-headers` executor on the M2M service so only the Tyk plugin can set the header. Direct callers cannot spoof a division.

### Validation rules enforced by the M2M service

- `divisionId` must be a valid UUID.
- No cross-division token exchange: if the subject token already carries a `divisionId` and the header supplies a different value, the exchange is rejected.
- No organization-level tokens with `divisionId`: rejected if the resulting token would be at organization level.
- Header value wins if provided; otherwise the existing claim value is preserved; otherwise `null`.

### Service-side reading

`JwtFilter` reads `info["divisionId"]?.toString()` from the verified `https://www.jamf.com/tenant` claim and writes it to `exchange.attributes[DIVISION_HEADER]` (the `X-DivisionId` key). `AbstractApiRequest.callerDivision: String?` exposes the value to handlers — `null` means the caller is a global admin. Services that want division-aware filtering compare object `divisionId` to `callerDivision` via the `divisionAllowed` predicate (a global caller sees everything; otherwise exact match).

---

## In-Pod JwtFilter

Both declaration-service and DSS replace the sidecar with an in-process Spring WebFlux filter. The pattern was introduced in declaration-service under DDMR-1088 (`com.jamf.declaration.auth.JwtFilter`) and DSS has since adopted it (`com.jamf.declarationstorage.auth.JwtFilter`). scoping-engine still uses the sidecar.

Both filters share these properties:

- **Multiple issuers, one filter.** `JwtProperties.issuers` is a list of `JwtIssuerProperties`; each entry has its own `issuer`, optional `jwksTemplate`, and `requiredScopes`. The filter looks up the decoder by the inbound JWT's `iss` claim. Unknown issuer → 401 ("Unsupported JWT issuer").
- **Scope check is ANY-match across a set.** `JwtScopeValidator` parses the `scope` claim (space- or comma-separated) and succeeds if any required scope is present. declaration-service's default `requiredScopes` is `{declaration-service-product, blueprint-components-api-product}` (the two Tyk products that route to the same pod, broadened by PR #158).
- **Open endpoints are hardcoded** to `HEAD /api/v1` (connectivity check) and the actuator base path. Less flexible than the sidecar's configurable `open` list. The filter sets an `OPEN_REQUEST_ATTR` exchange attribute so downstream code can tell whether the request was authenticated.
- **Tenant is passed via WebFlux exchange attributes, not request headers.** When `authEnabled` is true, the filter reads `https://www.jamf.com/tenant.tenantId` (and `.environmentId`, `.divisionId`) from the verified JWT and stores them as `exchange.attributes[TENANT_HEADER]` / `exchange.attributes[ENVIRONMENT_HEADER]` / `exchange.attributes[DIVISION_HEADER]`. `AbstractApiRequest` reads from those attributes, throwing 401 if tenant is absent. When `authEnabled` is false (test/local), the same attributes are populated from inbound `X-TenantId` / `X-EnvironmentId` / `X-DivisionId` headers instead. The division attribute is gated by `takeIf { it.isNotBlank() }` so empty-string claim values are normalized to "no division" rather than written as `""`.
- **CSA path.** DSS's filter additionally resolves CSA tokens via `CsaAuthResolver` / `CsaTenantResolver`, which looks up the tenant in the `tenant-authorizer` DynamoDB table. declaration-service is M2M-only.

Practical consequence: port 8080 on DSS no longer trusts an inbound `X-TenantId` header in prod — callers must send a Bearer JWT, just as they would via the gateway path. Port-forwarding the pod and sending only `X-TenantId` returns 401 in any environment where `authEnabled` is true.

---

## CSA (Client-Side Authentication)

CSA is the auth mechanism for end-user Jamf ID sessions. It is Auth0-backed:

1. The user authenticates against Auth0 with their Jamf ID credentials, receiving an opaque token.
2. That opaque token is exchanged at the CSA token service (`/v2/token`) for a signed JWT (access token + refresh token).
3. The JWT is presented as a Bearer token to DDmR APIs.

The `CsaTokenProvider` (in `declaration-storage-client-core`) is used by services that need to call another service as a CSA user. It:

- Accepts Auth0 client credentials and a Jamf ID username/password.
- Fetches and caches the access JWT, refreshing it in a background thread once half the TTL has elapsed.
- Exposes the cached JWT via `Supplier<String>` for use in `DeclarationClientCsaAuth`, which sets both the `Authorization: Bearer <token>` header and a required `x-customer-id` header.

CSA JWKS keys are served from S3 (e.g., `csa-public-key-store-production.s3.amazonaws.com`). The sidecar loads those keys at startup and caches them for 24 hours.

---

## HC Environment Auth Differences

The `hc` (healthcare/stateramp) environment is an isolated AWS deployment with its own M2M infrastructure. The differences are entirely in configuration—the sidecar and application code are the same.

In `platform-shared-values/values/aws/hc/stage/us/us-east-2/scoping-engine/values.yaml`:

```yaml
authProperties:
  m2m:
    jwksInternal: https://us1.stage.platform-hc.jamflabs.io/m2m/realms/platform/protocol/openid-connect/certs
auth:
  env: hc-stage
```

Key differences from commercial stage:

- The M2M JWKS URL points to `platform-hc.jamflabs.io` rather than `platform.jamflabs.io`. This is injected as a ConfigMap override into the sidecar, overriding the baked-in stage TOML.
- `auth.env: hc-stage` sets `MICRONAUT_ENVIRONMENTS=k8s,hc-stage` in the sidecar container. There is no `application-hc-stage.toml` in the sidecar; the env value is used to activate any future environment-specific config without requiring a separate image.
- Pulsar and crypto-keys endpoints also use `platform-hc.jamflabs.io` hostnames instead of `platform.jamflabs.io`.
- The AWS account is `604006981984` rather than the commercial account.

---

## Header Contract Summary

| Header | Source | Required | Behavior if absent |
|---|---|---|---|
| `X-TenantId` | JWT sidecar (M2M path) or tenant-authorizer via HAProxy (CSA path) | Yes | `AbstractApiRequest` throws 401 |
| `X-EnvironmentId` | JWT sidecar (from JWT claim) | No | `tenantEnv` is `null` |
| `X-Auth-Src` | `ddmr-authorizer-tenant` (CSA path only) | No | Informational only, not read by scoping-engine |
| `X-B3-TraceId` / `X-B3-SpanId` | Tracing infrastructure | No | MDC context left unpopulated |
