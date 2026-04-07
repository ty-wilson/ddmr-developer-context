# Blueprint Component Declarations Service

Last reviewed: 2026-04-07

**Owner:** Ocean team

## Summary

Blueprint Component Declarations Service (BCDS) is a stateless translation layer that converts blueprint component configurations into DDM declarations stored in Declaration Storage Service (DSS). When a blueprint is saved or updated, the Blueprints service posts a component's typed configuration to BCDS, which validates it, maps it to the correct Apple DDM declaration types, creates those declarations in DSS, and returns the resulting DSS IDs and version hashes (`serverToken`s). BCDS also provides per-component validation endpoints that return field-level errors without writing anything, and cleanup endpoints that delete previously-created declarations from DSS. The service owns no persistent state of its own — all storage is delegated to DSS. It is a Java 21 / Spring Boot 3 MVC service (not reactive) using virtual threads.

---

## API Endpoints

All routes sit under `/v1/components/<component-type>`. Authentication is M2M bearer token in production. In non-M2M environments, a `tenantId` header is required on every request (injected by the API gateway in production).

There are nine component types, each exposing the same three operations:

| Component type | Base path |
|---|---|
| Disk Management | `/v1/components/disk-management` |
| Free-form | `/v1/components/free-form` |
| Math Settings | `/v1/components/math-settings` |
| Passcode | `/v1/components/passcode` |
| Safari Extensions | `/v1/components/safari-extensions` |
| Service Background Tasks | `/v1/components/service-background-tasks` |
| Service Configuration Files | `/v1/components/service-configuration-files` |
| Software Update Settings | `/v1/components/software-update-settings` |

### `POST /{base}/translate`

Translates a component configuration into DSS declarations. Creates declarations in DSS and returns their details.

Request body (`TranslateRequest<T>`):
- `configuration` (required) — component-specific configuration object
- `declarationIdentifierPrefix` (optional, nullable) — string prefix used to generate stable asset declaration identifiers; required for components that produce `com.apple.asset.data` declarations (Service Background Tasks, Service Configuration Files)
- `currentDeployableObjects` — list of `{type, id}` pairs for declarations that already exist for this component; used for reference by the caller but not currently acted on by the service

Response body (`TranslateResponse`):
```json
{
  "deployableObjects": [
    {
      "type": "DSS",
      "id": "<dss-declaration-uuid>",
      "identifier": "<asset-identifier or null>",
      "versionHash": "<serverToken from DSS>",
      "declarationType": "CONFIGURATION | ASSET",
      "channelType": "SYSTEM | USER"
    }
  ]
}
```

Returns one entry per created declaration. Components that produce paired asset + configuration declarations (Service Background Tasks, Service Configuration Files) return multiple entries. Components with feature-flag-enabled user-channel declarations (Math Settings, Safari Extensions) may return two entries with different `channelType` values when those flags are enabled.

Errors in communicating with DSS result in `500 Internal Server Error`.

### `POST /{base}/validate`

Validates a component configuration without writing anything to DSS.

Request body (`ValidationRequest<T>`):
- `configuration` (required) — component-specific configuration object

Response body (`ValidationResult`): Always returns `200 OK`, even when validation fails.
```json
{
  "errors": [
    {
      "code": "NOT_NULL",
      "path": "configuration.targetOSVersion",
      "message": "must not be null"
    }
  ]
}
```

An empty `errors` array means the configuration is valid. JSON deserialization errors (e.g., wrong type for a field) are also caught and returned as validation errors rather than `400` responses.

### `POST /{base}/cleanup`

Deletes a set of DSS declarations that were previously created by a translate call.

Request body (`CleanupRequest`):
- `deployableObjects` — list of `{type, id}` pairs to delete

Returns `200` with empty body on success. If any deletion fails, all failures are collected and thrown together; the response is `500`. Cleanup is best-effort per item — a failure on one does not prevent attempting the others, but the final result is still an error if any failed.

---

## Component Configurations

### Free-form

Accepts an arbitrary list of declarations, each with a `kind` (CONFIGURATION or ASSET), an Apple DDM `type` string, and a raw JSON `payload`. All free-form declarations target the `SYSTEM` channel.

### Passcode

Maps to `com.apple.configuration.passcode.settings`. All fields are optional — null fields are omitted from the DSS payload. Fields include `minimumLength`, `maximumFailedAttempts`, `requirePasscode`, `requireAlphanumericPasscode`, `customRegex`, etc.

### Disk Management

Maps to `com.apple.configuration.diskmanagement.settings`. Supports two versions via a `version` discriminator field in the payload:
- Version 1 (deprecated): `externalStorage` / `networkStorage` as bare enum values
- Version 2 (current): each field is a `{value, Included}` pair supporting partial/optional inclusion. Defaults to v1 if no `version` field is present.

### Math Settings

Maps to `com.apple.configuration.math.settings`. Always creates a `SYSTEM` channel declaration. Creates a second `USER` channel declaration when the `mathSettingsUserChannelDeclaration` LaunchDarkly flag is enabled.

### Safari Extensions

Maps to `com.apple.configuration.safari.extensions.settings`. Same dual-channel behavior as Math Settings — creates a `USER` channel declaration when the `safariExtensionsUserChannelDeclaration` flag is enabled.

### Software Update Settings

Maps to `com.apple.configuration.softwareupdate.settings`. Rich configuration including `automaticActions`, `deferrals`, `rapidSecurityResponse`, and `beta`. Many fields use the `{value, Included}` optional-inclusion pattern.

### Service Configuration Files

Maps to `com.apple.configuration.services.configuration-files`. Accepts a list of service config file entries, each requiring a `dataAssetReference` (URL + optional auth + hash) and a `serviceType`. Each entry produces one `com.apple.asset.data` declaration and one configuration declaration. Asset identifiers are generated as `<declarationIdentifierPrefix>asset_<N>`.

### Service Background Tasks

Maps to `com.apple.configuration.services.background-tasks`. Accepts a list of background task entries, each with optional `executableAssetReference`, a list of `launchdConfigurations` (each with a `fileAssetReference`), `taskType`, and `taskDescription`. Each entry produces asset declarations for every referenced file plus one configuration declaration.

### Passcode / Math / Disk / Software Update / Safari / Free-form cleanup controllers

Each component type also has a `*ComponentCleanupController` and `*ComponentValidationController` — same three-operation pattern throughout.

---

## Dependencies

### Declaration Storage Service (DSS)

The only external service BCDS calls for data operations. Configured via `jamf.dss.client.base-uri`. BCDS uses Spring's declarative HTTP client (`@PostExchange`/`@DeleteExchange`):
- `POST /api/v2/declaration` — creates a declaration, returns `{id, serverToken}`
- `DELETE /api/v1/declaration/{declarationId}` — deletes a declaration

Connection and read timeouts both default to 5 seconds. DSS errors propagate as `DeclarationStorageClientException` and are mapped to `500` by `RestErrorControllerAdvice`.

### Tenant Service

Used to look up tenant metadata. Accessed via `jamf.tenant.client.base-uri`. Tenant lookups are cached with Spring's `@Cacheable("tenants")`.

### LaunchDarkly

Two feature flags gate user-channel declaration creation:
- `safariExtensionsUserChannelDeclaration` — enables `USER` channel for Safari Extensions
- `mathSettingsUserChannelDeclaration` — enables `USER` channel for Math Settings

LaunchDarkly health is included in the actuator readiness probe (`featureFlag` group). The service will not report ready if the LD SDK key is missing or the connection fails.

---

## Key Design Decisions and Gotchas

**BCDS is stateless — DSS owns everything.** BCDS does not have its own database. If a translate call partially succeeds (some DSS creates succeed, then one fails), the caller is responsible for cleaning up by calling the cleanup endpoint with whatever was returned before the failure.

**`declarationIdentifierPrefix` is required for asset-producing components.** Service Background Tasks and Service Configuration Files generate asset declarations with identifiers like `<prefix>asset_1`. If the prefix is null, the identifiers will be null, which means DSS has no stable identifier for the asset and the caller cannot reliably assign it by name.

**The `Included` field controls optional fields, not nullability.** Many configuration models use a `{value, Included}` pattern (the `OptionallyIncluded` interface). When `Included` is false (or absent, defaulting to true), the entire field is excluded from the DSS payload via a custom Jackson filter. This is different from setting the value to null. Do not confuse the two when constructing requests.

**Validation returns 200 even for invalid input.** The validate endpoint never returns a 4xx for field validation errors — it always returns 200 with an `errors` list. Only hard serialization failures (e.g., wrong JSON structure) would cause a different status code, and even those are caught and returned as 200 with a validation error entry.

**DiskManagementSettingsConfiguration is a versioned sealed interface.** When no `version` field is present in the JSON body, Jackson defaults to V1. V1 is marked `@Deprecated`. Callers should always include `"version": 2` in new code.

**Cleanup is not transactional.** The cleanup endpoint attempts to delete all listed DSS declarations, collecting any failures. If some succeed and some fail, the successful deletes are gone and cannot be rolled back. The endpoint returns 500 if any deletion failed, but does not report which ones succeeded.

**Swagger UI is available.** When deployed, Swagger UI is accessible at `/blueprints/components/declarations/swagger-ui/index.html` (relative to the gateway base). Useful for exploring exact request/response shapes for each component type.

**No Pulsar integration.** BCDS does not produce or consume any events. It is a synchronous HTTP service only.
