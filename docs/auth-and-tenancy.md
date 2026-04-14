# Auth And Tenancy

Last reviewed: 2026-04-13

## Overview

DDmR services use two ingress mechanisms, and traffic does not always pass through the sidecar:

- **Tyk API Gateway (M2M path):** All M2M service-to-service calls route through Tyk, which forwards to the `ddmr-jwt-sidecar` on port 7070. The sidecar validates the M2M JWT and injects HTTP headers (primarily `X-TenantId`) before proxying to the application on port 8080.
- **HAProxy Ingress (CSA/legacy path):** Some services (notably DSS) define a separate HAProxy-based Kubernetes ingress that routes directly to the application on port 8080, **bypassing the sidecar**. Authentication is handled by the `ddmr-authorizer-tenant` via HAProxy's `auth-url` annotation — the authorizer validates the CSA JWT and returns `X-TenantId`, which HAProxy injects into the forwarded request.

Services read `X-TenantId` regardless of which path delivered it. The `spring-m2m-authentication` library handles M2M JWT validation for services migrating away from the sidecar.

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

The sidecar is conditionally included in the Helm deployment template. In `scoping-engine/helm/scoping-engine/templates/deployment.yaml`, if `.Values.auth` is set a second container named `auth` is added to the pod. The image is taken from `auth.repo`/`auth.tag` in `values.yaml`. The MICRONAUT_ENVIRONMENTS variable selects the environment-specific TOML config inside the sidecar image.

Current image tag in scoping-engine `values.yaml`: `MAIN.2026-03-20.62054` from ECR `359585083818.dkr.ecr.us-east-1.amazonaws.com/jamf/ga/ddm/jwt`.

### Auth Type Support

`JwtProxySignatures` maps JWKS URLs to token type (CSA or M2M). The sidecar supports up to four JWKS endpoints simultaneously:

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

---

## Service-Side Header Extraction (`AbstractApiRequest`)

`AbstractApiRequest` is defined in `ApiRequests.kt`. Every HTTP handler in scoping-engine extends it, and it extracts the two tenant headers in its constructor:

- `X-TenantId` (constant `TENANT_HEADER`): required. If absent or blank, throws `ResponseStatusException(401, "No tenant identifier")`.
- `X-EnvironmentId` (constant `ENVIRONMENT_HEADER`): optional. Returns `null` if absent or blank.

These values are available as `tenant` and `tenantEnv` properties on any request object. Neither header is documented in the OpenAPI spec as a direct client concern because they are injected by the sidecar/authorizer, not set by API callers.

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
