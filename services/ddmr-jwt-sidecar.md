# DDmR JWT Sidecar

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** DDmR team

The DDmR JWT sidecar is a lightweight Micronaut/Kotlin proxy that runs as a Kubernetes sidecar container in front of any DDmR service. It listens on port 7070, validates every inbound JWT (CSA or M2M), enforces scope-based authorization, extracts claims into HTTP headers, and forwards the mutated request to the main application container on port 8080. The app container never sees the original `Authorization` header logic — it only sees the synthesized claim headers, so it has no JWT-parsing responsibilities of its own.

---

## Request Flow

1. Request arrives at the pod on port 7070.
2. **Actuator bypass** — paths under `/actuator-jwt/` are forwarded to the sidecar's own health/metrics endpoints; the main app never sees them.
3. **Open endpoint check** — if the request matches a configured open-endpoint rule (method + path-prefix), it is proxied to port 8080 with all claim headers stripped/blanked. No authentication.
4. **JWT extraction** — the `Bearer` token is parsed from the `Authorization` header using Nimbus JOSE+JWT. A parse failure or missing token results in 401.
5. **Issuer lookup** — the token's `iss` claim is looked up in a map keyed by JWKS URL. Unknown issuers return 401.
6. **JWKS signature verification** — async, cached (24 h). Failure returns 401.
7. **Scope authorization** — the `scope` claim is checked against a regex `^<configured-scope>:(\S+)$`. The captured permission must equal `allow`. The scope prefix is configured separately for CSA vs M2M.
8. **Claim-to-header extraction** — JWT claims are mapped to HTTP headers per the `[proxy.headers.*]` config (see below). A missing REQUIRED claim returns 401.
9. **Proxy** — the mutated request (scheme=http, host=localhost, port=8080) is forwarded. The response is returned unchanged.

---

## Header Mapping

Headers are defined under `[proxy.headers.<header-name>]` in the service-specific TOML. Each entry is auth-type-aware: a separate claim and mode is configured for CSA tokens vs M2M tokens.

| Field | Description |
|---|---|
| `csaClaim` / `m2mClaim` | JWT claim name to extract (e.g. `sub`, `tenant_id`). Leave blank to always strip the header for that auth type. |
| `csaClaimField` / `m2mClaimField` | If the claim is a JSON object, the field within it to extract. Leave blank for a top-level string claim. |
| `csaMode` / `m2mMode` | One of `REQUIRED`, `OPTIONAL`, or `BOOLEAN`. |

**Header modes:**
- `REQUIRED` — claim must be present; 401 if missing.
- `OPTIONAL` — header is omitted from the proxied request if the claim is absent (blank value causes the header to be removed).
- `BOOLEAN` — sets the header to `"true"` if the claim exists and is non-empty/non-false, otherwise `"false"`.

Headers not extracted from the JWT (because a claim is blank for that auth type) are actively stripped from the incoming request before proxying. This prevents clients from spoofing claim headers on unauthenticated or mismatched-auth-type paths.

---

## JWT Validation

Two auth types are supported:

**CSA (Customer Service Auth / Auth0)** — configured under `[csa]`:
- `jwks`: URL of the JWKS endpoint.
- `scope`: required scope prefix string.

**M2M (Machine-to-Machine / Keycloak client credentials)** — configured under `[m2m]`:
- `jwksInternal`: JWKS URL for internal M2M tokens (required).
- `jwksExternal`: JWKS URL for external M2M tokens (optional — only created if configured).
- `internalIssuer`: Override for the issuer string used as the map key when the JWKS URL differs from the issuer (used in fi/stage during infra migrations).
- `jwksInternalAlt` / `jwksExternalAlt`: Additional JWKS endpoints for migration periods where two issuer URLs are simultaneously valid.
- `scope`: required scope prefix string.

The issuer-to-validator map is built at startup. Lookup is a prefix match: a token whose `iss` starts with any registered key is accepted. JWKS are fetched asynchronously and cached for 24 hours (Micronaut `jwks` cache). At startup, `JwtProxyInfo` eagerly fetches all JWKS and logs a warning if any fail — the sidecar still starts.

Setting `scope = "*"` for either auth type disables scope checking entirely for that type.

---

## Configuration Profiles

The active profile is determined by the Micronaut environment, set at deployment time (e.g. `MICRONAUT_ENVIRONMENTS=stage`). `application.toml` holds defaults; environment files override JWKS URLs and optionally scope/port/open-endpoint settings.

| Profile | Notes |
|---|---|
| `stage` | Standard staging (jamflabs.io) JWKS for both CSA and M2M internal+external. |
| `fi` | FI staging — uses tyk-gateway JWKS URL with a separate `internalIssuer` override; external M2M is commented out. |
| `stage-alt` | Adds `jwksInternalAlt` / `jwksExternalAlt` for the nebula stage URLs alongside the primary ones. Used during infra migration windows. |
| `stable-dev` | dev.platform.jamflabs.io internal JWKS only; no external. |
| `prod-use1` | US production (us.int.apigw.jamf.com), both internal and external M2M. |
| `prod-euc1` | EU production (eu.int.apigw.jamf.com), both internal and external M2M. |
| `prod-apne1` | AP-NE production; see that TOML for specific URLs. |

Each service that uses this sidecar supplies its own `application.toml` overlay with its service-specific `[proxy.headers.*]`, `[proxy.open]`, and `scope` values. The per-environment JWKS profiles are shared across services.

---

## Gotchas

- **Port is fixed in the native image build** — `application.toml` sets port 7070 and a comment notes this must match the exposed port in `build.gradle.kts` for native images. Changing just the config without updating the build will break the native container.
- **Claim headers are stripped on open endpoints** — even for unauthenticated paths, any header that is in the `proxy.headers` map is blanked before forwarding. Clients cannot sneak claim headers through on open paths.
- **JWKS cache is 24 hours** — key rotation at the identity provider will not be reflected until the cache expires. The health indicator at `/actuator-jwt/health` checks live JWKS reachability but does not force a cache refresh.
- **`internalIssuer` / `internalAltIssuer` overrides** — in some environments the JWKS URL and the `iss` claim in the token differ (infra migration). The `internalIssuer` property sets the map key used for issuer lookup independently of `jwksInternal`. If this is wrong, all M2M tokens from that issuer will return 401.
- **Scope must be `allow`** — the regex extracts the permission segment after the colon (e.g. `ddmr:allow`) and compares it to the literal string `"allow"`. Tokens with scopes like `ddmr:read` or `ddmr:write` will be rejected. This is intentional — the sidecar is an all-or-nothing gate, not a fine-grained authz layer.
- **Health endpoint path** — the sidecar's actuator is at `/actuator-jwt/` (not `/actuator/`). This avoids colliding with the main app's own actuator. Kubernetes liveness/readiness probes should target port 7070 at this path.
- **Alt JWKS entries are only activated when configured** — `JwtProxyM2MInternalAltSignature` and `JwtProxyM2MExternalAltSignature` are `@Requires(beanProperty = "jwksInternalAlt")` conditional beans. They silently do nothing when not set, which is the desired behavior outside migration windows.
