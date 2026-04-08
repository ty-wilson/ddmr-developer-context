# Configuration Profile Service

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** Goldminers team

## Summary

Configuration Profile Service (CPS) is a Java 21 / Spring Boot 3 microservice owned by the Gold Miners team that has two distinct responsibilities: it manages Apple legacy configuration profiles (stores profile data in MongoDB, serializes them to Apple plist XML format, and stores the plist files in S3) and it implements the DDM (Device Declaration Management) component contract used by blueprint-management-service to validate, translate, and clean up DDM-style configuration profiles as deployable declarations in Declaration Storage Service (DSS). All endpoints require M2M bearer auth via Tyk gateway. The service is deployed in us-east-2, eu-central-1, and ap-northeast-1.

---

## API Endpoints

### Legacy Profile CRUD (`/v1/profile`) — hidden from public OpenAPI
These endpoints are not exposed in the public OpenAPI spec (`@Hidden`). They are used internally.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/profile` | Create a new config profile; returns the profile UUID |
| `GET` | `/v1/profile/{uuid}` | Read a single profile by UUID |
| `GET` | `/v1/profile` | List all non-deleted profiles (returns general data only) |
| `PUT` | `/v1/profile/{uuid}` | Update a profile; UUID in path must match body |
| `DELETE` | `/v1/profile/{uuid}` | Soft-delete a profile and delete its plist from S3 |

### Plist Retrieval (`/v1/plist`)
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/plist/{uuid}` | Returns the serialized Apple plist XML for a profile (Content-Type: application/xml) |

This URL is what devices and Jamf Pro server ultimately fetch to get the actual profile payload. The URL handed to DSS is constructed via the external Tyk API gateway (e.g., `https://<EXT_API_GW_URL>/cps/v1/plist/<uuid>`).

### DDM Component Contract (`/v2/components/ddm-profile`)
These endpoints implement the standard `ComponentContractRs` interface consumed by blueprint-management-service.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v2/components/ddm-profile/validate` | Validate a DDM profile DTO; returns structured validation errors |
| `POST` | `/v2/components/ddm-profile/validate-on-save` | Relaxed validation (allows null required fields) used during draft saves |
| `POST` | `/v2/components/ddm-profile/translate` | Translate a DDM profile into a DSS deployable object; creates or updates a declaration |
| `POST` | `/v2/components/ddm-profile/transform-configuration` | Split configuration into public and private (sensitive) parts |
| `POST` | `/v2/components/ddm-profile/cleanup` | Delete DSS declarations (and associated profiles) by declaration ID |
| `GET`  | `/v2/components/ddm-profile/fragments` | List supported payload type fragments (used by the config-profiles MFE to populate the UI) |

### Supporting Endpoints
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/mdm-schema/version` | Returns the MDM schema version CPS currently supports |
| `GET` | `/v1/metadata/profile` | Paginated metadata for all profiles (hidden, admin use) |
| `GET` | `/v1/metadata/profile/{internalUuid}` | Paginated version history for a given internalUuid (hidden, admin use) |

---

## Data Model

### MongoDB (`config-profiles` collection, multi-tenant via `MultiTenantMongoDbFactory`)

`ConfigProfileEntity` is the root document:

| Field | Type | Notes |
|-------|------|-------|
| `uuid` | String (`@Id`) | The Apple `PayloadUUID`; public identifier |
| `internalUuid` | String | Stable across recreations; tracks version lineage |
| `declarationId` | String | DSS declaration ID this profile is linked to (null for non-DDM profiles) |
| `configProfileDto` | `ConfigProfileDto` | Embedded profile data (see below) |
| `version` | int | Incremented on each recreation |
| `deleted` | boolean | Soft-delete flag |
| `modifiedAt` / `deletedAt` | Instant | Audit timestamps |
| `schemaVersion` | String | MDM schema version the profile was created under |

`ConfigProfileDto` (embedded, also used as the API request/response body):

| Field | Validation |
|-------|------------|
| `payloadUUID` | UUID regex pattern |
| `payloadIdentifier` | max 200 chars |
| `payloadDisplayName` | required, max 200 chars |
| `payloadContent` | required, min 1 item; list of `GeneratedPayload` (generated from MDM JSON schemas) |

Supported `payloadType` values in `GeneratedPayload` include: `com.apple.dock`, `com.apple.screensaver`, `com.apple.applicationaccess`, `com.apple.MCX.FileVault2`, `com.apple.MCX.FDEFileVaultOptions`, `com.apple.font`, and others. The full list is code-generated from MDM schema JSON files.

### S3 / MinIO (plist storage)

Serialized Apple plist XML files are stored in S3, keyed by profile UUID. Locally, MinIO is used (`docker compose up -d`). The `PlistRepository` interface abstracts the storage backend; `S3PlistRepository` is used in production, `InMemoryPlistRepository` in tests.

---

## Dependencies

| Dependency | Purpose |
|-----------|---------|
| **MongoDB / DocumentDB** | Primary store for profile metadata and payload data. Multi-tenant: database is selected per-request from the M2M token. |
| **AWS S3 / MinIO** | Stores the serialized plist XML files. |
| **Declaration Storage Service (DSS)** | CPS calls DSS's `POST /api/v2/declaration` to create `com.apple.configuration.legacy` declarations and `DELETE /api/v1/declaration/{id}` to remove them. The DSS client uses M2M auth (robocop). |
| **blueprint-management-service** | Calls CPS's `/v2/components/ddm-profile/*` endpoints as part of the Blueprints DDM pipeline. |
| **Tyk gateway (external)** | The plist download URL embedded in declarations points through the external Tyk gateway (`EXT_API_GW_URL`). CPS also calls its own internal gateway for M2M token exchange. |
| **Jamf Pro Server** | Used as a fallback `ProfileUrlProvider` when the feature flag directs plist URLs to Jamf Pro rather than the external Tyk gateway. |
| **LaunchDarkly** | `FeatureFlagService` wraps LaunchDarkly to control which plist URL provider is active and to gate fragment availability per payload type. |

---

## How It Relates to the MDM Schema Pipeline and the Config-Profiles MFE

**MDM Schema pipeline:** CPS generates its payload DTOs (`GeneratedPayload` subclasses) from Apple MDM JSON schemas using `./gradlew generateJsonSchema2Pojo`. To pick up a new schema version, run `./gradlew fetchSchemaFromGit --schemaEnv <env> --schemaVersion <version>` then regenerate. The current supported version is surfaced at `GET /v1/mdm-schema/version`. Services that need to know whether CPS supports a given schema version should call that endpoint. Bumping the schema version may add or remove valid `payloadType` values.

**Config-profiles MFE:** The MFE calls `GET /v2/components/ddm-profile/fragments` to discover which payload types are available and render the appropriate UI fragments (including display names, icons, supported OS versions, and i18n strings). Fragment availability can be gated per feature flag via the `featureFlag` field on `FragmentDto`. The MFE then posts validated and translated profile data through blueprint-management-service, which in turn calls the `/validate`, `/translate`, and `/cleanup` endpoints on CPS.

---

## Key Design Decisions and Gotchas

- **Profiles are never mutated; they are recreated.** `recreate()` finds the existing entity's `internalUuid`, increments the version, and saves a new document. This means there is a version history in Mongo, but only the latest non-deleted entity for a given `uuid` is authoritative. The admin metadata endpoints expose this history.

- **Two profile workflows: legacy CRUD vs. DDM/Blueprints.** The `/v1/profile` CRUD path is used by non-Blueprints flows (the `@Hidden` annotation and the comment in `ConfigProfileCrudExtendedService` confirm this). The `/v2/components/ddm-profile` path is the Blueprints DDM path. They share the same underlying storage but go through different service layers.

- **Soft deletes.** Profiles are marked `deleted=true` rather than removed from MongoDB. The S3 plist is hard-deleted on delete. Watch for orphaned Mongo documents if S3 deletion fails (the rollback handler logs but does not always re-raise).

- **Sensitive fields are split before translation.** `transform-configuration` separates sensitive payload fields (e.g., passwords, secrets identified by MDM schema annotations) into a `privateConfiguration` block. The `translate` endpoint accepts this private configuration back and merges it before writing to DSS. Never log or surface the private configuration block.

- **Profile URL is environment-dependent.** The plist URL embedded in a DSS declaration is built from `EXT_API_GW_URL`. A LaunchDarkly flag can redirect this to a Jamf Pro server URL instead. If plist downloads are failing in an environment, check which URL provider is active.

- **Plist generation failures trigger rollback.** If S3 write fails after a Mongo save, `ProfileRollback` attempts to restore the previous state. This is a best-effort in-process rollback, not a distributed transaction.

- **Multi-tenancy is implicit.** The tenant is resolved from the M2M token on every request and selects the correct MongoDB database via `MultiTenantMongoDbFactory`. There is no explicit tenant parameter on any API endpoint.

- **Pact contract tests.** CPS is a Pact provider (`pact-provider-name: configuration-profile-service`). Run `./gradlew verifyContractsWithPact` before changing request/response shapes on any public endpoint. Breaking the Pact contract will fail the pipeline.
