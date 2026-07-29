# Declaration Service

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the route table in `RoutingConfig.kt`, the stub `handleStrictValidate`, the `requiredScopes` ANY-match default in `JwtProperties.kt`, and that `schemaUrlBase`/`schemaTtl`/`schemaVersion` have no consumer outside their properties class. Not re-verified and older (2026-04-28): the translate request/response shapes, the identifier-placeholder walk, and the partial-failure rollback behavior.

Every value and negative below is re-derivable. See `## Where to find the data` at the bottom before acting on anything here.

**Owner:** DDmR team

## Summary

Declaration Service is a Spring Boot (WebFlux/coroutines) microservice that acts as a translation and validation layer between declaration components (e.g. the Component Registry / Blueprint UI) and the Declaration Storage Service (DSS). When a component configuration containing raw declarations is submitted, declaration-service assigns stable identifiers, resolves cross-declaration payload references, stores each declaration in DSS via the `declaration-product-springboot-starter` client, and returns a list of deployable objects back to the caller. It also handles cleanup (deleting DSS records) and exposes a fragments catalog endpoint so UIs can discover which declaration types are enabled and their display metadata.

---

## API Endpoints

All routes are under `/api/v1`. The `X-TenantId` header is required on every endpoint except `GET /strict/components/fragments`.

The full surface is declared in one place, `RoutingConfig.kt`'s `apiRouting` bean. Read it rather than trusting the table:

```bash
git -C ~/Projects/DDmR/declaration-service show \
  origin/main:src/main/kotlin/com/jamf/declaration/config/RoutingConfig.kt | sed -n '/coRouter/,/^    }/p'
```

Two parallel families plus a connectivity check:

| Family | Paths | Notes |
|---|---|---|
| Lenient (custom declarations) | `POST /components/{translate,validate,cleanup}` | Runs validation but never rejects on validation grounds. `translate` logs at `debug`, `validate` logs at `warn`, both return success. |
| Strict (schema-validated) | `POST /strict/components/{translate,validate,cleanup}`, `GET /strict/components/fragments` | `translate` validates internally via `createDeclarationsStrict`. `fragments` returns the enabled declaration types with display names, descriptions, icons, supported OS versions, and i18n metadata, and does not require `X-TenantId`. |
| Connectivity | `HEAD /api/v1` | Reachability probe for monitors outside the cluster. **Not** a health endpoint; health is deliberately internal-only. |

**Trap: `POST /api/v1/strict/components/validate` is a stub. It returns `200 {"errors": []}` without running any validation at all.** A caller that treats a 200 from this endpoint as "the declaration is spec-compliant" will ship broken declarations. `handleStrictValidate` in `ComponentHandler.kt` logs the request and immediately responds with an empty `ValidationResponse`; there is no validator call in its body. The lenient `POST /components/validate` does invoke the validator but only logs the results at `warn` and still returns an empty error list, so *neither* validate endpoint surfaces errors to the caller. To see validation problems, read the service logs, or use `strict/components/translate`, which does enforce. Re-check with:

```bash
git -C ~/Projects/DDmR/declaration-service grep -n -A6 'fun handleStrictValidate' \
  origin/main -- 'src/main/*'
```

### Translate request/response shape

Request body for `POST /components/translate`:
```json
{
  "configuration": {
    "declarations": [
      {
        "type": "com.apple.configuration.some.Type",
        "channelType": "SYSTEM",
        "kind": "CONFIGURATION",
        "payload": { ... },
        "payloadKey": 0
      }
    ]
  },
  "declarationIdentifierPrefix": "<prefix-string>",
  "currentDeployableObjects": []
}
```

Response:
```json
{
  "deployableObjects": [
    {
      "type": "DSS",
      "id": "<dss-uuid>",
      "versionHash": "<token>",
      "identifier": "<prefix>custom_0",
      "declarationType": "CONFIGURATION",
      "channelType": "SYSTEM"
    }
  ]
}
```

The `strict` endpoints use the same shape except declarations omit `payloadKey` (the service assigns indices internally). Authoritative shapes live in `src/main/kotlin/com/jamf/declaration/rest/` (`ApiRequests.kt` and siblings).

---

## Dependencies

| Dependency | How it is used |
|------------|----------------|
| Declaration Storage Service (DSS) | All declaration create/delete operations go through the `declaration-product-springboot-starter` client (`DeclarationProductClient`). declaration-service does not own any persistent storage itself. |
| Jamf robocop (M2M) | Fetches per-tenant M2M tokens with scopes `tenant` and `declaration-storage-product`. Credentials are read from AWS Secrets Manager at startup. |
| AWS Secrets Manager | Stores the M2M client ID and secret. Accessed via `SecretsService` using `aws.sdk.kotlin:secretsmanager`. |
| Apple DDM schema (classpath) | The validation path loads schemas from `classpath:validation-schemas.json`, bundled at build time, and caches them in memory for the process lifetime. |

**Trap: three config properties look like a remote-schema-fetch feature and are wired to nothing.** `strictvalidation.schemaUrlBase`, `strictvalidation.schemaTtl`, and `strictvalidation.schemaVersion` are declared in `StrictValidationProperties.kt` and referenced nowhere else in the repo. Setting `schemaUrlBase` in Helm values will not make the service fetch schemas over HTTP, and bumping `schemaVersion` will not change which schema is validated against. Verified 2026-07-29; re-check with the grep in the verification section, which should return only `StrictValidationProperties.kt`.

---

## Key Design Decisions and Gotchas

**Identifier generation during translate.** Before writing to DSS, the service builds a map of placeholder strings (`$PAYLOAD_0`, `$PAYLOAD_1`, ...) to final identifiers (`<prefix>custom_0`, `<prefix>custom_1`, ...). It then does a deep JSON walk to replace any occurrence of these placeholder strings inside declaration payloads. This allows declarations to cross-reference each other by placeholder before they have real DSS IDs.

**Partial failure rollback.** If any declaration write fails mid-batch, `TranslateHandler.handleError` attempts to delete all successfully-written declarations before re-throwing. Individual delete failures are logged but do not suppress the original error.

**Validation runs on translate, not only on validate.** Both translate endpoints run the validator internally. The lenient translate logs at debug and proceeds; the strict translate enforces. A successful lenient translate is not evidence that the declaration is spec-compliant.

**Serialization.** Uses `kotlinx.serialization` (not Jackson) for request/response models. The codec is configured in `RoutingConfig.configureHttpMessageCodecs` with `ignoreUnknownKeys = true` and `decodeEnumsCaseInsensitive = true`, so `ChannelType` values (`SYSTEM`, `USER`) are case-insensitive. `encodeDefaults` is not set, so it defaults to `false`.

**`StrictValidationProperties` is config-driven.** The set of enabled declaration types and their display metadata (names, icons, OS version ranges) live in Helm values under the `strictvalidation` prefix. Adding a new declaration type to the fragments catalog is a Helm values change, not a code change. Duplicate type entries in config cause a hard startup failure.

**No persistent storage.** declaration-service holds no database of its own. All state lives in DSS. To query or list existing declarations, go to DSS directly.

**Trap: `X-TenantId` must parse as a `java.util.UUID`, and a non-UUID value fails late.** The failure surfaces as a 500 at the M2M token fetch, not a 400 at the edge, so a malformed tenant header looks like a service outage rather than a bad request.

**Two inbound Tyk routes, two different scopes.** Callers can reach the same pod via `/declaration-service/...` (scope `declaration-service-product`) or `/blueprints/components/declaration-service/...` (scope `blueprint-components-api-product`, the unified blueprint-components bundle). The scopes are not interchangeable on a given URL because Tyk's SecurityPolicy is path-bound. The in-pod `JwtFilter` (`src/main/kotlin/com/jamf/declaration/auth/JwtFilter.kt`) also enforces scope via `JwtScopeValidator`, but with ANY-match semantics across the configured `requiredScopes` (default `{declaration-service-product, blueprint-components-api-product}`, set in `JwtProperties.kt`), so any request that survives Tyk on either route also satisfies the pod check. The `declaration-service-component-tests` repo fetches `blueprint-components-api-product` because stable-dev traffic flows through the blueprint-components path. See `docs/api-layer.md` for the full mapping.

**In-pod JWT validation, not the sidecar.** declaration-service authenticates in-process via `JwtFilter` rather than via `ddmr-jwt-sidecar`. The in-pod filter is the sole pod-side gatekeeper and accepts both inbound scopes, per DDMR-1088 / PR #158 (`Allow multiple ANY-match scopes for JWT verification`). **Which DDmR services still run the sidecar changes over time and is owned by `docs/auth-and-tenancy.md`; read it there rather than inferring from this file.** Note that `services/declaration-storage-service.md` records DSS as having migrated too, so any statement that declaration-service is the only migrated service is stale.

---

## Not to Be Confused With

`blueprint-component-custom-declarations` follows a similar translate/validate/cleanup pattern and is also called by Blueprint Management Service during deployment. The distinction is scope of responsibility: declaration-service handles DDmR-team-owned declaration types (structured configurations owned and versioned by the DDmR team), while `blueprint-component-custom-declarations` handles user-authored custom declarations, meaning arbitrary declaration payloads that end users compose themselves. If you are unsure which service a component type routes to, check the Component Registry's `translator.baseUri` for that component.

---

## How It Differs from Declaration Storage Service (DSS)

This is a common point of confusion:

| | declaration-service | Declaration Storage Service (DSS) |
|---|---|---|
| **Role** | Translation, validation, identifier assignment, orchestration | Persistent storage and retrieval of declaration records |
| **Owns data?** | No | Yes |
| **Called by** | Component Registry / Blueprint services | declaration-service (and any other service needing raw declaration CRUD) |
| **Calls** | DSS (via `declaration-product-springboot-starter`) | DynamoDB |
| **Identifier assignment** | Yes, generates `<prefix>custom_N` identifiers | No, accepts identifiers provided by callers |
| **Validation** | Schema and business-rule validation of declaration payloads | Structural persistence, minimal semantic validation |
| **Fragment catalog** | Yes (`GET /strict/components/fragments`) | No |

In short: to store or fetch a raw declaration, talk to DSS. To take a component configuration with cross-referencing placeholders and turn it into a set of stored, identified declarations in one shot, talk to declaration-service.

---

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/declaration-service; git -C $R fetch origin -q

# What changed since this doc was last reviewed
git -C $R log --oneline origin/main --since=2026-07-29

# Unmerged work: check before concluding something is not implemented
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# The full route surface, in one bean
git -C $R show origin/main:src/main/kotlin/com/jamf/declaration/config/RoutingConfig.kt

# Is strict validate still a stub? (a body with no declarationValidator call means yes)
git -C $R grep -n -A6 'fun handleStrictValidate' origin/main -- 'src/main/*'

# Dead config check: if this returns ONLY StrictValidationProperties.kt, the props are still unwired
git -C $R grep -rn 'schemaUrlBase\|schemaTtl\|schemaVersion' origin/main

# Scope set the in-pod filter accepts (ANY-match)
git -C $R grep -n 'requiredScopes' origin/main -- 'src/main/*'
```
