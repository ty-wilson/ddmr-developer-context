# Configuration Profile Plist Migrator

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** Goldminers team

## Summary

Configuration Profile Plist Migrator is a Spring Boot service (written in Java, compiled to a GraalVM native binary) that converts legacy Apple Configuration Profile plists into Jamf Blueprints. It operates as an on-demand HTTP API: callers POST raw plist XML and receive either a transformed JSON configuration or a fully-created Blueprint. A separate scheduled-migration flow allows callers to queue a batch of profiles (sourced from the `plist-origin` service) for async processing, with per-migration status tracked in DynamoDB. The service is designed as a migration utility — it does not own device or scope state and does not produce Pulsar events.

---

## What It Migrates

Input is Apple Configuration Profile plist XML. Each profile contains a `PayloadContent` array of typed payloads (e.g. `com.apple.wifi.managed`, `com.apple.MCX`).

The transformation pipeline applies these rules in order:

1. **Payload key normalization** — converts `Payload`-prefixed keys to camelCase (`PayloadType` → `payloadType`, etc.).
2. **Dict-to-configuration** — renames the top-level `dict` key to `configuration` when the value is a map.
3. **MCX type resolution** — payloads with type `com.apple.MCX` are re-typed to a specific sub-type based on their `PayloadDisplayName` (e.g. `"FileVault 2 Options"` → `com.apple.MCX.FDEFileVaultOptions`). Unrecognized display names are normalized by stripping spaces and special characters.
4. **Default-value filtering** (opt-in via `filterDefaults: true` header) — drops payload keys whose values match the MDM schema defaults fetched from `mdm-schema-internal`.

The output is the transformed JSON configuration. The `POST /api/v1/migrate` endpoint additionally wraps this configuration into a `BlueprintRequest` and calls `blueprint-management-api` to create the Blueprint.

---

## How It Runs

The service exposes a REST API on port 8080. All endpoints require M2M authentication (OAuth2, scope `blueprint-management-api-product`). There is no Pulsar integration.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/migrate/transform` | Transforms plist XML → JSON config. No Blueprint created. Request body: raw plist XML (`Content-Type: application/xml`). |
| `POST` | `/api/v1/migrate` | Transforms plist XML and creates a Blueprint. Returns `201` with `{ id }`. |
| `GET` | `/api/v1/migrate/metadata` | Lists available source profiles from `plist-origin`. Supports pagination and filtering by `uuid`, `profileName`, `platform`, `payloadType`, `key`. |
| `POST` | `/api/v1/migrate/schedule` | Schedules a batch migration for a list of `ConfigProfileDataDto` objects. Returns `202` with a list of migration IDs. Migrations are grouped by platform (`computers` / `devices`). |
| `GET` | `/api/v1/history` | Paginated list of past migrations and their status. |
| `GET` | `/api/v1/history/{migrationId}` | Details for a specific migration, including per-profile status. |

The `filterDefaults` request header (default `false`) controls whether the default-filtering transformation rule is applied.

### Migration Status Lifecycle

Scheduled migrations (via `/schedule`) are written to DynamoDB with status `SCHEDULED`. Status transitions: `SCHEDULED` → `COMPLETED`, `WARNING`, or `FAILED`.

---

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `blueprint-management-api` | Creates Blueprints from transformed config. Called via M2M. Base URI: `${API_GW_URL}/blueprints/management`. |
| `plist-origin` | Source of profile metadata and raw plist XML for the scheduled-migration flow. Called via M2M. Base URI: `${API_GW_URL}/plist-origin/api/v1/config-profile-data`. |
| `mdm-schema-internal` | Provides payload schemas and default values used by the `filterDefaults` rule. Responses are cached in-memory by MDM schema URL. |
| DynamoDB | Stores migration history. Table: `cppm-migration-history`. Partition key: `tenantId`, sort key: `migrationId`. |

Required environment variables: `API_GW_URL`, `INT_API_GW_CLIENT_ID`, `INT_API_GW_CLIENT_SECRET`.

---

## Gotchas

**`filterDefaults` is off by default and per-request.** The `filterDefaults: true` header must be passed explicitly on each `/transform` or `/migrate` call. The `/schedule` endpoint accepts it as part of the request body payload and applies it during processing.

**MCX type resolution depends on `PayloadDisplayName`.** For `com.apple.MCX` payloads, the transformer rewrites the type using the display name. Known mappings (`"FileVault 2 Options"`, `"Mobility"`, `"Managed Wi-Fi"`) are hardcoded. Any other display name is used as-is after stripping whitespace and special characters — if the display name is missing, the type is left as `com.apple.MCX`, which will likely fail Blueprint creation.

**MDM schema lookups are cached but can fail.** The `filterDefaults` rule calls `mdm-schema-internal` to fetch payload schemas. These calls are cached per URL, but if the schema service is unreachable or returns a 404 for an unknown payload type, the request throws `PayloadNotFoundException`. Monitor for this when migrating profiles with uncommon payload types.

**`/schedule` does not execute the migration synchronously.** It records the migration as `SCHEDULED` in DynamoDB and returns immediately. The actual transformation and Blueprint creation must be triggered separately — there is no background worker in this service that picks up scheduled records automatically.

**GraalVM native binary.** The service is compiled to a native image for fast startup. Reflection configuration lives in `src/main/resources/META-INF/native-image/reflect-config.json`. If new types need reflection at runtime (e.g., new Jackson-deserialized domain classes), the reflect config must be updated and the binary recompiled. Mockito is incompatible with AOT — `processTestAot` is disabled for tests.

**Platform is `computers` or `devices`.** `PlatformType` maps `"computers"` and `"devices"` (case-insensitive). Passing any other value results in a `null` platform on the migration record.
