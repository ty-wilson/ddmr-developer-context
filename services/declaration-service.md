# Declaration Service

Last reviewed: 2026-04-07

## Summary

Declaration Service is a Spring Boot (WebFlux/coroutines) microservice that acts as a translation and validation layer between declaration components (e.g., the Component Registry / Blueprint UI) and the Declaration Storage Service (DSS). When a component configuration containing raw declarations is submitted, declaration-service assigns stable identifiers, resolves cross-declaration payload references, stores each declaration in DSS via the `declaration-product-springboot-starter` client, and returns a list of deployable objects back to the caller. It also handles cleanup (deleting DSS records) and exposes a fragments catalog endpoint so UIs can discover which declaration types are enabled and their display metadata.

---

## API Endpoints

All routes are under `/api/v1`. The `X-TenantId` header is required on every endpoint except `GET /strict/components/fragments`.

### Custom declaration flow (lenient validation)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/components/translate` | Translates a list of declarations into DSS records. Validates declarations but proceeds with warnings; returns `deployableObjects[]` with DSS IDs, version hashes, and identifiers. |
| `POST` | `/api/v1/components/validate` | Validates declaration JSON structure and logs issues; always returns `200 { errors: [] }` (validation problems are logged, not rejected). |
| `POST` | `/api/v1/components/cleanup` | Deletes a list of DSS records by ID. Used to clean up after failures or replacements. |

### Strict declaration flow (schema-validated)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/strict/components/translate` | Same as the lenient translate but enforces schema validation before storing. |
| `POST` | `/api/v1/strict/components/validate` | Schema-validated validate; currently returns `200 { errors: [] }` (stub behavior, validation is run server-side but not surfaced to callers yet). |
| `POST` | `/api/v1/strict/components/cleanup` | Same as lenient cleanup. |
| `GET`  | `/api/v1/strict/components/fragments` | Returns the catalog of enabled declaration types with display names, descriptions, icons, supported OS versions, and i18n metadata. Does not require `X-TenantId`. |

### Other

| Method | Path | Description |
|--------|------|-------------|
| `HEAD` | `/api/v1` | Connectivity/accessibility check — used by external monitors to verify the API is reachable. Not a health endpoint. |

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

The `strict` endpoints use the same shape except declarations omit `payloadKey` (the service assigns indices internally).

---

## Dependencies

| Dependency | How it is used |
|------------|----------------|
| Declaration Storage Service (DSS) | All declaration create/delete operations go through the `declaration-product-springboot-starter` client (`DeclarationProductClient`). declaration-service does not own any persistent storage itself. |
| Jamf robocop (M2M) | Fetches per-tenant M2M tokens with scopes `tenant` and `declaration-storage-product`. Credentials are read from AWS Secrets Manager at startup. |
| AWS Secrets Manager | Stores the M2M client ID and secret. Accessed via `SecretsService` using `aws.sdk.kotlin:secretsmanager`. |
| Apple DDM schema (external URL) | The strict validation path fetches JSON schemas from an external URL base (`strictvalidation.schemaUrlBase`) and caches them with a configurable TTL (default 60 minutes, schema version 17). |

---

## Key Design Decisions and Gotchas

**Identifier generation during translate.** Before writing to DSS, the service builds a map of placeholder strings (`$PAYLOAD_0`, `$PAYLOAD_1`, ...) to final identifiers (`<prefix>custom_0`, `<prefix>custom_1`, ...). It then does a deep JSON walk to replace any occurrence of these placeholder strings inside declaration payloads. This allows declarations to cross-reference each other by placeholder before they have real DSS IDs.

**Partial failure rollback.** If any declaration write fails mid-batch, `TranslateHandler.handleError` attempts to delete all successfully-written declarations before re-throwing. Individual delete failures are logged but do not suppress the original error.

**Lenient vs. strict path.** The `/components/*` (lenient) endpoints run schema and business-rule validation but only log issues — they never reject a request on validation grounds alone. The `/strict/components/*` endpoints are intended to enforce validation more strictly, but as of the last review the validate endpoint still returns an empty error list. Do not rely on the validate endpoint to surface errors to end users yet; check the service logs.

**Validation on translate, not just validate.** Both translate endpoints run the full validator internally. The lenient translate logs debug-level; the strict translate path follows stricter handling. Callers should not assume a successful translate means the declaration is spec-compliant.

**Serialization.** Uses `kotlinx.serialization` (not Jackson) for request/response models. The codec is configured with `ignoreUnknownKeys = true` and `decodeEnumsCaseInsensitive = true`. `ChannelType` values (`SYSTEM`, `USER`) are case-insensitive.

**`StrictValidationProperties` is config-driven.** The set of enabled declaration types and their display metadata (names, icons, OS version ranges) live in Helm values under the `strictvalidation` prefix. Adding a new declaration type to the fragments catalog requires a Helm values change, not a code change. Duplicate type entries in config cause a hard startup failure.

**No persistent storage.** declaration-service holds no database of its own. All state lives in DSS. If you need to query or list existing declarations, go to DSS directly.

**Tenant is a UUID string.** `X-TenantId` must be parseable as a `java.util.UUID`. Passing a non-UUID string will cause a 500 at the M2M token fetch, not a 400.

---

## How It Differs from Declaration Storage Service (DSS)

This is a common point of confusion:

| | declaration-service | Declaration Storage Service (DSS) |
|---|---|---|
| **Role** | Translation, validation, identifier assignment, orchestration | Persistent storage and retrieval of declaration records |
| **Owns data?** | No | Yes |
| **Called by** | Component Registry / Blueprint services | declaration-service (and any other service needing raw declaration CRUD) |
| **Calls** | DSS (via `declaration-product-springboot-starter`) | DynamoDB (or equivalent backing store) |
| **Identifier assignment** | Yes — generates `<prefix>custom_N` identifiers | No — accepts identifiers provided by callers |
| **Validation** | Schema and business-rule validation of declaration payloads | Basic structural persistence; minimal semantic validation |
| **Fragment catalog** | Yes (`GET /strict/components/fragments`) | No |

In short: if you need to store or fetch a raw declaration, talk to DSS. If you need to take a component configuration (with cross-referencing placeholders) and turn it into a set of stored, identified declarations in one shot, talk to declaration-service.
