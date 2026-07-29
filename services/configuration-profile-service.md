# Configuration Profile Service

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the controller/endpoint inventory (`*Rs.java` plus `ComponentContractRs`), the schema-version source chain (`SupportedSchemaRs` → `PropertyFileMdmSchemaProvider` → `schema.version` / `supported-schema.txt` / `gradle.properties`), the rollback and delete paths (`PlistExceptionHandler`, `ConfigProfileCrudService`), and the Gradle task names. Not re-verified and older (2026-04-07): the Mongo entity field list, the S3/MinIO storage details, the LaunchDarkly plist-URL provider behavior, and the regions.

Two 2026-04-07 claims were wrong and are corrected below: a `/v2/components/ddm-profile/validate-on-save` endpoint (no such mapping exists in the repo's history), and "the rollback handler logs but does not always re-raise" (it always re-raises; the real hazard is different and is now a labelled trap). See `## Where to find the data`.

**Owner:** Goldminers team

## Summary

Configuration Profile Service (CPS) is a Java / Spring Boot microservice with two distinct responsibilities: it manages Apple legacy configuration profiles (stores profile data in MongoDB, serializes them to Apple plist XML, and stores the plist files in S3), and it implements the DDM component contract used by blueprint-management-service to validate, translate, and clean up DDM-style configuration profiles as deployable declarations in Declaration Storage Service (DSS). Runtime and framework versions are declared in `build.gradle` and `gradle.properties`; read them there. All endpoints require M2M bearer auth via Tyk (listen path `/cps`). Deployed in us-east-2, eu-central-1, and ap-northeast-1.

CPS also answers the platform's **"last supported MDM schema version"** question for everyone else. See the trap below; it is the single most misunderstood thing about this service.

---

## API Endpoints

Three controller families plus the DDM component contract. Regenerate the exact surface instead of trusting a table (this is how the `validate-on-save` error above was caught):

```bash
R=~/Projects/DDmR/configuration-profile-service
git -C $R grep -n 'RequestMapping\|GetMapping\|PostMapping\|PutMapping\|DeleteMapping\|@Hidden' \
  origin/main -- 'src/main/java/**/*Rs.java' 'src/main/java/**/ComponentContractRs.java'
```

Or read the live spec: the OpenAPI document is served by the running service, and the `@Hidden` annotation is what keeps the internal endpoints out of it.

| Family | Base path | Notes |
|---|---|---|
| Legacy profile CRUD | `/v1/profile` | Create / read / list / update / soft-delete. `@Hidden`, so absent from the public OpenAPI spec. Used by non-Blueprints flows. |
| Plist | `/v1/plist` | `GET /{uuid}` returns the serialized Apple plist XML (`application/xml`). There is also a `POST /v1/plist` that ingests raw profile XML and returns a `Declaration`. |
| DDM component contract | `/v2/components/ddm-profile` | `validate`, `translate`, `transform-configuration`, `cleanup`, `fragments`. Inherited from the shared `ComponentContractRs<T>` base and consumed by blueprint-management-service. |
| Supporting | `/v1/mdm-schema/version`, `/v1/metadata/profile[/{internalUuid}]` | The schema-version endpoint (see trap). The metadata endpoints are `@Hidden` admin views over the version history. |

`GET /v1/plist/{uuid}` is the URL devices and the Jamf Pro server ultimately fetch to get the actual profile payload. The URL handed to DSS is constructed through the external Tyk gateway, e.g. `https://<EXT_API_GW_URL>/cps/v1/plist/<uuid>`.

---

## Trap: CPS owns "last supported schema version", and it is not the ingested version

`GET /v1/mdm-schema/version` returns `MdmSchemaProvider.getLatestSupported()`. In `PropertyFileMdmSchemaProvider` that resolves in order: the Spring property `schema.version` if set, otherwise the classpath resource `supported-schema.txt` baked in at build time, otherwise the constant `FAILOVER_VERSION`. The build-time value comes from `mdm.schema.version` in `gradle.properties`.

**This endpoint is what the platform means by "the schema version", and Tyk routes it as such.** In `tyk-gateway-management`, both the `mdm-schema` and `mdm-schema-internal` API products define a separate Tyk API with `listen_path: /mdm-schema/v1/version` (and `/mdm-schema-internal/v1/version`) whose `target_url` is **CPS** `/v1/mdm-schema/version`, not the mdm-schema ingest ALB. So:

- The ingest pipeline's SSM "latest ingested" version and CPS's "last supported" version are different numbers and the supported one lags.
- Bumping the ingest version does not change what any MFE requests. That requires a CPS rebuild (`mdm.schema.version`) or a `schema.version` override.
- **Bumping the schema version may add or remove valid `payloadType` values**, because the payload DTOs are code-generated from that version's schemas. A payload type that validated yesterday can become unknown.
- Any service needing to know whether CPS supports a given schema version must call the endpoint. Do not infer it from the ingest pipeline.

See `services/mdm-schema-ingest.md` and `docs/frontend.md` for the other side of this.

---

## Data Model

### MongoDB (`config-profiles` collection, multi-tenant via `MultiTenantMongoDbFactory`)

`ConfigProfileEntity` is the root document:

| Field | Notes |
|-------|-------|
| `uuid` (`@Id`) | The Apple `PayloadUUID`; public identifier |
| `internalUuid` | Stable across recreations; tracks version lineage |
| `declarationId` | DSS declaration ID this profile is linked to (null for non-DDM profiles) |
| `configProfileDto` | Embedded profile data, also the API request/response body |
| `version` | Incremented on each recreation |
| `deleted` | Soft-delete flag |
| `modifiedAt` / `deletedAt` | Audit timestamps |
| `schemaVersion` | The `getLatestSupported()` value at creation time (set in `ProfileCreator`) |

Field-level bean validation (patterns, max lengths, required/min-size) lives on `ConfigProfileDto`. Read the annotations rather than a transcription; they change with the DTO:

```bash
git -C $R grep -n '@Size\|@Pattern\|@NotNull\|@NotEmpty' origin/main \
  -- 'src/main/java/com/jamf/configprofile/input/ConfigProfileDto.java'
```

`payloadContent` is a list of `GeneratedPayload`, and the supported `payloadType` values are **code-generated from the MDM schema JSON files**, so any list written here is stale the moment the schema version moves. Ask the service instead:

```bash
# Payload types CPS currently offers as fragments, for the version it supports
curl -s -H "Authorization: Bearer $TOKEN" "$GW/cps/v2/components/ddm-profile/fragments" \
  | jq -r '.. | .payloadType? // empty' | sort -u
```

### S3 / MinIO (plist storage)

Serialized Apple plist XML files are stored in S3, keyed by profile UUID. Locally, MinIO is used (`docker compose up -d`). The `PlistRepository` interface abstracts the backend; `S3PlistRepository` in production, `InMemoryPlistRepository` in tests.

---

## Dependencies

| Dependency | Purpose |
|-----------|---------|
| **MongoDB / DocumentDB** | Primary store for profile metadata and payload data. Multi-tenant: the database is selected per-request from the M2M token. |
| **AWS S3 / MinIO** | Stores the serialized plist XML files. |
| **Declaration Storage Service (DSS)** | CPS calls DSS `POST /api/v2/declaration` to create `com.apple.configuration.legacy` declarations and `DELETE /api/v1/declaration/{id}` to remove them. The DSS client uses M2M auth (robocop). |
| **blueprint-management-service** | Calls CPS `/v2/components/ddm-profile/*` as part of the Blueprints DDM pipeline. |
| **Tyk gateway (external)** | The plist download URL embedded in declarations points through the external gateway (`EXT_API_GW_URL`). CPS also calls its own internal gateway for M2M token exchange. |
| **Jamf Pro Server** | Used as a fallback `ProfileUrlProvider` when the feature flag directs plist URLs to Jamf Pro rather than the external Tyk gateway. |
| **LaunchDarkly** | `FeatureFlagService` wraps LaunchDarkly to control which plist URL provider is active and to gate fragment availability per payload type (`featureFlag` field on `FragmentDto`). |
| **mdm-schema pipeline (build time)** | Java-enhanced schemas from the Java schema S3 bucket feed `jsonschema2pojo` DTO generation. |

**Config-profiles MFE:** the MFE calls `GET /v2/components/ddm-profile/fragments` to discover which payload types are available and render the matching UI fragments (display names, icons, supported OS versions, i18n strings). It then posts validated and translated profile data through blueprint-management-service, which calls `/validate`, `/translate`, and `/cleanup` on CPS.

---

## Key Design Decisions and Gotchas

**Profiles are never mutated; they are recreated.** `ConfigProfileCrudService.recreate()` finds the existing entity's `internalUuid` via `findTopByInternalUuidAndDeletedIsFalseOrderByVersionDesc`, increments the version, and saves a new document (`ProfileRecreator`). Mongo therefore holds a version history, but only the latest non-deleted entity for a given `uuid` is authoritative. The `@Hidden` `/v1/metadata/profile` endpoints expose that history. If `recreate` finds nothing, it logs a warn and falls through to `create`, so a recreate against a deleted profile silently becomes a create at version 1. Named files: `ConfigProfileCrudService.java`, `ProfileRecreator.java`, `ProfileCreator.java`.

**Trap: rollback is not implemented, and the orphan is the S3 plist, not the Mongo document.** All three methods on `PlistExceptionHandler` (the only `ProfileRollback` implementation) log and then throw `PlistCreationException`, so callers do get a failure. But each carries a literal `// TODO rollback, status?` and none restores prior state. Two concrete consequences:

- `ConfigProfileCrudService.delete(uuid)` soft-deletes the Mongo document **first**, then deletes the plist. If the S3 delete throws, the caller sees a 500 while the Mongo row is already `deleted=true` and the S3 object still exists. Retrying the delete hits `profileDoesntExist` logic against a row that is present-but-deleted.
- `deleteByDeclarationId(declarationId)`, which is the DDM `/cleanup` path, marks Mongo documents deleted and **never touches S3 at all**. Every DDM cleanup leaves its plist object behind by design of the current code, not by failure.

So the S3 bucket accumulates plists for deleted profiles. Do not reason about this from the `ProfileRollback` interface name; read `PlistExceptionHandler.java` and `ConfigProfileCrudService.java`.

**Two profile workflows: legacy CRUD versus DDM/Blueprints.** `/v1/profile` is the non-Blueprints path (its `@Hidden` annotation and the comment in `ConfigProfileCrudExtendedService` confirm this). `/v2/components/ddm-profile` is the Blueprints DDM path. They share storage but go through different service layers, and only the DDM path sets `declarationId`.

**Sensitive fields are split before translation.** `transform-configuration` separates sensitive payload fields (passwords, secrets identified by MDM schema annotations) into a `privateConfiguration` block. `translate` accepts that private configuration back and merges it before writing to DSS. Never log or surface the private configuration block.

**Profile URL is environment-dependent.** The plist URL embedded in a DSS declaration is built from `EXT_API_GW_URL`. A LaunchDarkly flag can redirect it to a Jamf Pro server URL instead. If plist downloads fail in one environment only, check which `ProfileUrlProvider` is active before looking at storage.

**Trap: multi-tenancy is implicit and there is no tenant parameter on any endpoint.** The tenant is resolved from the M2M token on every request and selects the MongoDB database via `MultiTenantMongoDbFactory` (`src/main/java/com/jamf/configprofile/config/data/MultiTenantMongoDbFactory.java`); `M2MTenantPrincipal.fromSecurityContext()` is how service code reads it. Nothing in a request path or body names a tenant, so a request replayed with a different token silently targets a different database, and a UUID that "does not exist" may simply live in another tenant's DB. There is no cross-tenant query path.

---

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/configuration-profile-service; git -C $R fetch origin -q

# What changed since this doc was last reviewed
git -C $R log --oneline origin/main --since=2026-07-29

# Unmerged work, before concluding an endpoint or payload type does not exist
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Regenerate the endpoint surface (catches doc drift; @Hidden marks internal-only)
git -C $R grep -n 'RequestMapping\|GetMapping\|PostMapping\|PutMapping\|DeleteMapping\|@Hidden' \
  origin/main -- 'src/main/java/**/*Rs.java' 'src/main/java/**/ComponentContractRs.java'

# The schema version chain, build-time value first
git -C $R show origin/main:gradle.properties | grep mdm.schema.version
git -C $R show origin/main:src/main/java/com/jamf/configprofile/schema/PropertyFileMdmSchemaProvider.java

# Live: what version CPS says it supports, and which payload types that yields
curl -s -H "Authorization: Bearer $TOKEN" "$GW/cps/v1/mdm-schema/version"
curl -s -H "Authorization: Bearer $TOKEN" "$GW/cps/v2/components/ddm-profile/fragments" \
  | jq -r '.. | .payloadType? // empty' | sort -u

# Rollback / orphan behavior, at the source
git -C $R show origin/main:src/main/java/com/jamf/configprofile/service/PlistExceptionHandler.java
git -C $R grep -n -A8 'void delete(String uuid)\|deleteByDeclarationId' origin/main \
  -- 'src/main/java/com/jamf/configprofile/service/ConfigProfileCrudService.java'
```

Local Gradle tasks (run from a checkout, not from this doc):

```bash
./gradlew fetchSchemaFromGit --schemaEnv <env> --schemaVersion <version>  # pull a schema version
./gradlew generateJsonSchema2Pojo                                          # regenerate payload DTOs
./gradlew verifyContractsWithPact                                          # Pact provider verification
```

CPS is a Pact provider (`pact-provider-name: configuration-profile-service`). Run `verifyContractsWithPact` before changing request or response shapes on any public endpoint; breaking the contract fails the pipeline.
