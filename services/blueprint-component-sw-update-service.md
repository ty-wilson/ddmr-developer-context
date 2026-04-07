# Blueprint Component Software Update Service

Last reviewed: 2026-04-07

**Owner:** Ocean team

## Summary

Blueprint Component Software Update Service is a thin translation layer in the Blueprints system. Its job is to convert a software update configuration (target OS version + deadline) into an Apple DDM declaration of type `com.apple.configuration.softwareupdate.enforcement.specific` and persist it in the Declaration Storage Service (DSS). It also provides a validation endpoint that callers can use to check a configuration before committing to translation, and a cleanup endpoint to delete previously created declarations. The service owns no database of its own — all state lives in DSS. It is a Spring Boot 3 / Java 21 service, uses M2M auth for DSS calls, and is part of the `ocean` team's `blueprints` system.

---

## API Endpoints

Base path: `/v1/components/sw-update`

All three endpoints accept and return `application/json`. When M2M auth is enabled (production), the caller must supply a Bearer token. In non-M2M environments (local/test), a `tenantId` header is required instead.

### `POST /translate`

Translates a software update configuration into a DDM declaration stored in DSS. Returns the created deployable object(s).

**Request body:**
```json
{
  "configuration": {
    "targetOSVersion": "17.5.1",
    "targetLocalDateTime": "2025-07-10T10:45:06.809",
    "detailsURL": {
      "value": "https://example.com/update-info",
      "included": true
    }
  },
  "currentDeployableObjects": [
    { "type": "DSS", "id": "<existing-declaration-id>" }
  ]
}
```

- `targetOSVersion`: required. Must match `^[1-9]\d?(?:\.(?:\d|[1-9]\d)){1,2}$` (e.g., `17`, `17.5`, `17.5.1`).
- `targetLocalDateTime`: required. Local device time by which update must occur.
- `detailsURL`: optional. If `included` is `true`, `value` must be a non-empty HTTP/HTTPS URL.
- `currentDeployableObjects`: informational; the service does not currently use it for upsert logic (it always creates a new declaration).

**Response body:**
```json
{
  "deployableObjects": [
    { "type": "DSS", "id": "<declaration-uuid>", "versionHash": "<serverToken>" }
  ]
}
```

`versionHash` is the DSS `serverToken` (SHA-256 over payload+type). Store it — callers need it to detect whether the declaration changed.

### `POST /validate`

Validates a configuration against the same rules as `/translate` without creating anything in DSS. Always returns `200 OK`; an empty `errors` array means the configuration is valid.

**Request body:** `{ "configuration": { ... } }` — same `SwUpdateConfiguration` shape as above.

**Response body:**
```json
{
  "errors": [
    { "code": "NOT_NULL", "path": "configuration.targetOSVersion", "message": "must not be null" }
  ]
}
```

Error codes are snake_case (e.g., `NOT_NULL`, `PATTERN`, `INPUT_MISMATCH`).

### `POST /cleanup`

Deletes DSS declarations previously created by this service. Intended to be called when a blueprint component is removed or replaced.

**Request body:**
```json
{
  "deployableObjects": [
    { "type": "DSS", "id": "<declaration-uuid>" }
  ]
}
```

Returns `200` with no body on success. If any deletion fails, the service throws after attempting all deletions (suppressed exceptions); callers should treat a non-200 as a partial or total failure and may retry.

---

## Data Model

This service owns no persistent storage. All data is created in and deleted from DSS. The single declaration type it creates is:

- **DDM type:** `com.apple.configuration.softwareupdate.enforcement.specific`
- **DDM group:** `CONFIGURATION`
- **Payload fields:** `TargetOSVersion`, `TargetLocalDateTime`, `DetailsURL` (omitted if `detailsURL.included` is false or `detailsURL` is null)

The payload uses UpperCamelCase JSON naming (matching Apple's DDM schema). `DetailsURL` is omitted from the JSON entirely (not set to null) when not included, because the DSS payload is forwarded verbatim to Apple devices.

---

## Dependencies

| Dependency | How used |
|---|---|
| **Declaration Storage Service (DSS)** | Creates declarations via `POST /api/v2/declaration`; deletes via `DELETE /api/v1/declaration/{id}`. Called using a Spring HTTP interface client (`DeclarationStorageServiceClient`). |
| **Jamf M2M (`jamf-platform-m2m`)** | `M2MAccessTokenProvider` is called before each DSS request to obtain a service-to-service Bearer token. Controlled by `jamf.platform.m2m.authentication-enabled` (true by default). |

No Pulsar topics produced or consumed. No DynamoDB. No caching.

---

## Gotchas

**The service always creates a new declaration; it does not upsert.** `currentDeployableObjects` in the translate request is accepted but ignored in the current implementation. Callers are responsible for calling `/cleanup` on old declarations before or after calling `/translate` if they want to avoid orphaned DSS records.

**`/validate` always returns `200`.** Validation errors are in the response body, not reflected in the HTTP status code. Do not treat a `200` from `/validate` as "valid" without inspecting the `errors` array.

**`detailsURL.included` gates whether the URL is written to the DDM payload.** If `included` is `false`, the `DetailsURL` field is omitted from the Apple declaration entirely regardless of whether `value` is populated. If `included` is `true`, `value` must be a non-empty HTTP/HTTPS URL or validation will fail.

**`targetOSVersion` format is strict.** The regex `^[1-9]\d?(?:\.(?:\d|[1-9]\d)){1,2}$` means leading zeros and patch versions like `17.5.10` are invalid (the last segment allows only 1–2 digit numbers). Check against this pattern before calling `/translate`.

**Cleanup failures do not short-circuit.** The cleanup method attempts to delete every provided declaration ID and aggregates failures. A non-200 response means at least one deletion failed, but others may have succeeded. Retry only the IDs that failed if you track them; blindly retrying the full list is safe (DSS returns success on deleting an already-deleted declaration).

**Swagger UI available in all environments.** OpenAPI docs are served at `/v1/components/sw-update/swagger-ui/index.html`. In SBOX it is accessible through the Tyk gateway at `https://tyk.sbox.ocean.jamf.build/{namespace}/blueprints/components/sw-update/swagger-ui/index.html`.

**Contract tests via Pact.** The service has consumer-driven contract tests in `**.pact.**`. If you change the DSS API contract, verify these pass: `./mvnw failsafe:integration-test -Dpact.tests.skip=false`.
