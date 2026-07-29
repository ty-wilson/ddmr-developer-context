# DDmR JWT Sidecar

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the sunset status below, the profile TOML file list, the fixed port and JWKS cache declarations in `application.toml`, and the `allow` scope comparison in `JwtProxyFilter.kt`. **Not re-verified and older:** the header-mode semantics, the open-endpoint stripping behavior, and the per-profile JWKS details, which date from 2026-04-07. Commands to re-derive all of it are in "Where to find the data" at the end.

**Owner:** DDmR team

## Read this first: the sidecar is sunset

`origin/main` has received no functional change since **2026-05-01**, when three commits titled "TRIVIAL Decommission" landed via PR #55 (`jamf/sunset`). The README carries a `[!CAUTION]` banner:

> THIS PROJECT IS SUNSET AND WILL NOT RECEIVE ANY FURTHER UPDATES.
> As there is no FIPS-compliant GraalVM available, there is no path forward without a rewrite for FedRAMP.

`catalog-info.yaml` sets `lifecycle: archived`, and the decommission commits also stripped Dependabot config and trimmed the build pipeline. **Do not treat this doc as describing the current auth pattern.** DDmR services are migrating to an in-pod Spring WebFlux `JwtFilter` (declaration-service first, under DDMR-1088; DSS has followed).

**`docs/auth-and-tenancy.md` owns the question of which services still run the sidecar versus an in-pod filter.** That list changes, so read it there and verify against the cluster rather than trusting any list. The mechanics below remain accurate for pods that still carry the container, and remain the reference for what an in-pod filter must reproduce.

---

## What it does

A lightweight Micronaut/Kotlin proxy that runs as a Kubernetes sidecar container in front of a DDmR service. It listens on port **7070**, validates every inbound JWT (CSA or M2M), enforces scope-based authorization, extracts claims into HTTP headers, and forwards the mutated request to the main application container on port **8080**. The app container never handles the original `Authorization` header; it only sees the synthesized claim headers, so it has no JWT-parsing responsibility of its own. The image is built as a GraalVM native image, which is precisely why there is no FIPS path.

---

## Request Flow

1. Request arrives at the pod on port 7070.
2. **Actuator bypass**: paths under `/actuator-jwt/` are served by the sidecar's own health/metrics endpoints; the main app never sees them.
3. **Open endpoint check**: if the request matches a configured open-endpoint rule (method + path prefix), it is proxied to 8080 with all claim headers stripped/blanked. No authentication.
4. **JWT extraction**: the `Bearer` token is parsed from `Authorization` using Nimbus JOSE+JWT. Parse failure or missing token gives 401.
5. **Issuer lookup**: the token's `iss` claim is looked up in a map keyed by JWKS URL. Unknown issuers give 401.
6. **JWKS signature verification**: async and cached. Failure gives 401.
7. **Scope authorization**: the `scope` claim is checked against `^<configured-scope>:(\S+)$` and the captured permission must equal `allow`. The scope prefix is configured separately for CSA and M2M.
8. **Claim-to-header extraction**: JWT claims map to HTTP headers per the `[proxy.headers.*]` config. A missing `REQUIRED` claim gives 401.
9. **Proxy**: the mutated request (scheme `http`, host `localhost`, port 8080) is forwarded and the response returned unchanged.

---

## Header Mapping

Headers are defined under `[proxy.headers.<header-name>]` in the service-specific TOML. Each entry is auth-type-aware: a separate claim and mode for CSA versus M2M tokens.

| Field | Description |
|---|---|
| `csaClaim` / `m2mClaim` | JWT claim name to extract (e.g. `sub`, `tenant_id`). Leave blank to always strip the header for that auth type. |
| `csaClaimField` / `m2mClaimField` | If the claim is a JSON object, the field within it. Leave blank for a top-level string claim. |
| `csaMode` / `m2mMode` | One of `REQUIRED`, `OPTIONAL`, `BOOLEAN`. |

**Header modes:**
- `REQUIRED`: claim must be present, 401 if missing.
- `OPTIONAL`: header is omitted from the proxied request if the claim is absent (a blank value removes the header).
- `BOOLEAN`: sets the header to `"true"` if the claim exists and is non-empty/non-false, otherwise `"false"`.

**Trap: any header named in `proxy.headers` is stripped from the incoming request, including on open endpoints.** This is what stops a client spoofing `X-TenantId` on an unauthenticated path or on a path whose auth type does not populate that claim. An in-pod replacement filter that only *adds* headers and never strips them loses this protection silently.

---

## JWT Validation

Two auth types are supported.

**CSA (Auth0)**, configured under `[csa]`: `jwks` (JWKS endpoint URL) and `scope` (required scope prefix).

**M2M (Keycloak client credentials)**, configured under `[m2m]`:
- `jwksInternal`: JWKS URL for internal M2M tokens (required).
- `jwksExternal`: JWKS URL for external M2M tokens (optional, only created if configured).
- `internalIssuer`: override for the issuer string used as the map key when the JWKS URL differs from the issuer.
- `jwksInternalAlt` / `jwksExternalAlt`: additional JWKS endpoints for migration periods where two issuer URLs are simultaneously valid.
- `scope`: required scope prefix.

The issuer-to-validator map is built at startup. Lookup is a **prefix match**: a registered map key that starts with the token's `iss` claim is accepted (the map key is the prefix, the token `iss` is the value being tested). At startup, `JwtProxyInfo` eagerly fetches all JWKS and logs a warning if any fail; **the sidecar still starts**.

Setting `scope = "*"` for either auth type disables scope checking entirely for that type (`JwtProxyFilter` short-circuits on `props.scope == "*"`).

**Trap: JWKS are cached long enough that identity-provider key rotation is invisible until expiry, and nothing forces a refresh.** The window is one value, `micronaut.caches.jwks.expire-after-write` in `application.toml`; read it there rather than from this page. The consequence is what matters: after a rotation, tokens signed with the new key fail with a signature 401 until the cache ages out. `/actuator-jwt/health` checks live JWKS *reachability* and does **not** force a cache refresh, so a green health check during a rotation is not evidence the keys are current. The only reliable remedies are waiting out the window or restarting the container.

---

## Configuration Profiles

The active profile comes from the Micronaut environment, set at deployment time (e.g. `MICRONAUT_ENVIRONMENTS=stage`, or `k8s,hc-stage`). `application.toml` holds defaults; each environment file overrides JWKS URLs and optionally scope, port, and open-endpoint settings.

Profile TOMLs present on `origin/main` as of 2026-07-29 (`application-<profile>.toml` under `src/main/resources/`):

`stage`, `stage-alt`, `fi`, `hc-stage`, `stable-dev`, `prod-use1`, `prod-euc1`, `prod-apne1`.

The profile *names* are the durable part; every URL, scope, and commented-out block inside them is code state that drifts. **Read each TOML for its specific URLs and for which auth types it actually configures.** Two notes worth carrying:

- `stage-alt` exists specifically to hold `jwksInternalAlt` / `jwksExternalAlt` alongside the primary entries, for infra-migration windows where two issuers are valid at once.
- `fi` and `hc-stage` are the profiles most likely to carry an `internalIssuer` override, because their JWKS host and token `iss` diverge.

Each service using the sidecar supplies its own `application.toml` overlay with service-specific `[proxy.headers.*]`, `[proxy.open]`, and `scope` values, mounted as a ConfigMap. The per-environment JWKS profiles are shared across services. `docs/auth-and-tenancy.md` shows the scoping-engine overlay as a worked example.

---

## Gotchas

**Trap: the listen port is baked into the native image build, and config alone will not move it.** `application.toml` sets `micronaut.server.port = 7070` with an in-file comment that it must match the exposed port in `build.gradle.kts` for native images. Changing only the config produces a container that listens on one port and exposes another, which presents as a pod that passes its build but fails every readiness probe.

**Trap: the scope permission must be exactly `allow`, and read/write scopes are rejected.** The regex captures the permission segment after the colon and compares it to the literal `ALLOW` constant (`"allow"`) in `JwtProxyFilter.kt`. A token scoped `ddmr:read` or `ddmr:write` is rejected. **This is intentional**: the sidecar is an all-or-nothing gate, not a fine-grained authz layer. Do not "fix" it by loosening the comparison; the service behind it does its own authorization.

**Trap: `internalIssuer` / `internalAltIssuer` are the usual cause of a blanket M2M 401.** In some environments the JWKS URL and the token's `iss` claim differ (infra migration). `internalIssuer` sets the map key used for issuer lookup independently of `jwksInternal`. If it is wrong, **every** M2M token from that issuer 401s, which looks like an outage rather than a config error.

**Alt JWKS entries silently do nothing when not configured, which is the desired behavior.** `JwtProxyM2MInternalAltSignature` and `JwtProxyM2MExternalAltSignature` are `@Requires(beanProperty = "jwksInternalAlt")` conditional beans. Outside migration windows their absence is correct, so "the alt bean is not there" is not a symptom.

**Health endpoint path is `/actuator-jwt/`, not `/actuator/`.** This avoids colliding with the main app's own actuator. Kubernetes liveness/readiness probes should target port 7070 at this path.

---

## Where to find the data (verify rather than trust)

```bash
JS=~/Projects/DDmR/ddmr-jwt-sidecar; git -C $JS fetch origin -q
git -C $JS log --oneline origin/main --since=2026-07-29    # expect no output: sunset
git -C $JS for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Confirm the sunset status yourself before building on anything here
git -C $JS show origin/main:README.md | head -12
git -C $JS show origin/main:catalog-info.yaml

# Port, actuator path, and the JWKS cache window, all in one file
git -C $JS show origin/main:src/main/resources/application.toml

# Which profiles exist, then read the one you care about for its actual URLs
git -C $JS ls-tree --name-only origin/main src/main/resources/ | grep '^.*toml$'
git -C $JS show origin/main:src/main/resources/application-<profile>.toml

# The scope comparison and the "*" bypass
git -C $JS grep -n 'ALLOW\|ScopeRegex\|== "\*"' origin/main -- src/main/kotlin
```

Which pods still run the sidecar is a cluster question, not a repo question:

```bash
kubectl --context platformsvc-dev -n ddmr-dev get pods \
  -o custom-columns='POD:.metadata.name,CONTAINERS:.spec.containers[*].name'
```

A container named `auth` alongside the app container means the sidecar is still in the pod. If it is absent, the service has an in-pod `JwtFilter` instead and this doc does not describe its auth path: read `docs/auth-and-tenancy.md`.
