# Configuration Profile Plist Migrator

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the endpoint set and `filterDefaults` header wiring in `MigratorRs.java`, the MCX retype mechanism in `MCXTypeTransformer.java`, the DynamoDB table name and keys, the outbound base URIs and env vars in `application.yml`, and the absence of any scheduling/async machinery. Also checked `tyk-gateway-management` for the `plist-origin` upstream. Not re-verified and older (2026-04-07): the payload-key normalization and dict-to-configuration rules, the GraalVM reflection-config note, and the in-memory schema cache behavior.

Values and enums here are re-derivable. See `## Where to find the data`.

**Owner:** Goldminers team

## Summary

Configuration Profile Plist Migrator (CPPM) is a Spring Boot service (Java, compiled to a GraalVM native binary) that converts legacy Apple Configuration Profile plists into Jamf Blueprints. It operates as an on-demand HTTP API: callers POST raw plist XML and receive either a transformed JSON configuration or a fully-created Blueprint. A separate scheduled-migration flow is *shaped* like a batch queue (records land in DynamoDB with per-migration status), but see the trap below before designing around it. The service does not own device or scope state and **has no Pulsar integration at all**: there are no messaging dependencies in `build.gradle` and no producer or consumer code.

Reachable through Tyk at listen path `/cppm` (`tyk-gateway-management`, `*/api-products/configuration-profile-plist-migrator/`), fronting `https://cppm.<env>.platform.jamflabs.io/api`.

---

## What It Migrates

Input is Apple Configuration Profile plist XML. Each profile contains a `PayloadContent` array of typed payloads (e.g. `com.apple.wifi.managed`, `com.apple.MCX`).

The transformation pipeline applies these rules in order:

1. **Payload key normalization** converts `Payload`-prefixed keys to camelCase (`PayloadType` → `payloadType`, and so on).
2. **Dict-to-configuration** renames the top-level `dict` key to `configuration` when the value is a map.
3. **MCX type resolution** re-types `com.apple.MCX` payloads to a specific sub-type. See the trap below; this is the rule that fails in the field.
4. **Default-value filtering** (opt-in) drops payload keys whose values match the MDM schema defaults fetched from `mdm-schema-internal`.

The output is the transformed JSON configuration. `POST /api/v1/migrate` additionally wraps that configuration into a `BlueprintRequest` and calls `blueprint-management-api` to create the Blueprint.

**Trap: MCX retyping is driven entirely by `PayloadDisplayName`, and a missing display name silently produces an unusable payload.** The mechanism in `MCXTypeTransformer.transform` is: for each payload whose type is exactly `com.apple.MCX`, look at its `PayloadDisplayName` and rewrite the type to `com.apple.MCX.<schemaName>`. The schema name comes from a small hardcoded map, and anything not in the map is derived by stripping characters matching `[\s:()#.-]|802\.1X` from the display name. The hardcoded mappings are:

| `PayloadDisplayName` | Resulting type suffix |
|---|---|
| `FileVault 2 Options` | `FDEFileVaultOptions` |
| `Mobility` | `MobileAccounts` |
| `Managed Wi-Fi` | `WiFiManagedSettings` |

**If `PayloadDisplayName` is null, `setProperMCXType` returns early and the payload type stays `com.apple.MCX`.** No error, no warning. Blueprint creation then fails downstream on an unknown payload type, which reads as a Blueprints problem rather than a missing field in the source plist. When a migration fails on an MCX payload, check the source plist for a display name first. Note also that the two other display names in the map exist because stripping alone gives the wrong answer for them, so a *new* MCX subtype whose display name does not normalize cleanly needs a code change, not config.

---

## How It Runs

The service exposes a REST API on port 8080. All endpoints require M2M authentication (OAuth2, scope `blueprint-management-api-product`).

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/migrate/transform` | Transforms plist XML into JSON config. No Blueprint created. Body: raw plist XML (`Content-Type: application/xml`). |
| `POST` | `/api/v1/migrate` | Transforms plist XML and creates a Blueprint. Returns `201` with `{ id }`. |
| `GET` | `/api/v1/migrate/metadata` | Lists available source profiles from `plist-origin`. Supports pagination and filtering by `uuid`, `profileName`, `platform`, `payloadType`, `key`. |
| `POST` | `/api/v1/migrate/schedule` | Records a batch migration for a list of `ConfigProfileDataDto`. Returns `202` with a list of migration IDs, grouped by platform. |
| `GET` | `/api/v1/history` | Paginated list of past migrations and their status. |
| `GET` | `/api/v1/history/{migrationId}` | Details for a specific migration, including per-profile status. |

**`filterDefaults` is a request header, not a body field, and it is off by default.** All three write endpoints declare `@RequestHeader(value = "filterDefaults", defaultValue = "false")`. It must be passed explicitly on each call, including `/schedule`, where the natural assumption is that it belongs in the JSON body.

### Migration status lifecycle

Scheduled migrations are written to DynamoDB with the initial status from `MigrationHistoryEntity`. The status values live in one enum:

```bash
git -C ~/Projects/DDmR/configuration-profile-plist-migrator show \
  origin/main:src/main/java/com/jamf/plistmigrator/domain/status/MigrationStatus.java
```

**Trap: nothing in this service advances a scheduled migration past its initial status.** Verified 2026-07-29 by three checks that all come back empty or one-way:

- No `@Scheduled`, `@Async`, `@EnableScheduling`, `TaskExecutor`, `ExecutorService`, or `CompletableFuture` anywhere in `src/main`.
- `MigrationHistoryService` exposes exactly `save`, `findHistory`, and `findHistoryDetails`. There is no update or status-transition method, and `save` is only reachable from `MigrationService.scheduleMigration`.
- No endpoint accepts a migration ID for execution; `/history/{migrationId}` is read-only.

So the terminal statuses in the enum are, in this repo, unreachable. `POST /schedule` is a durable to-do list with no consumer. **What would trigger them is not implemented here and, as far as the DDmR workspace shows, not implemented anywhere**: the `plist-origin` upstream that the scheduled flow reads from has no service repo in the workspace, and its Tyk API definition points at the placeholder host `http://not-real-jamf-domain.com/api/` in **all three** environments (dev, stage, prod). Treat the scheduled-migration path as reserved-but-inert. If you need it, the open questions are who owns `plist-origin` and where the executor is meant to live; do not assume a worker exists somewhere and is merely misconfigured. Recheck with the `plist-origin` grep in the verification section, and check unmerged branches in both repos.

**Trap: an unrecognized platform silently becomes `null` rather than a 400.** `PlatformType` maps only `"computers"` and `"devices"` (case-insensitive). Any other value yields a `null` platform on the migration record, and since `scheduleMigration` groups by `platformType`, a bad value forms its own group with no platform.

---

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `blueprint-management-api` | Creates Blueprints from transformed config. Called via M2M. Base URI: `${API_GW_URL}/blueprints/management`. |
| `plist-origin` | Intended source of profile metadata and raw plist XML for the scheduled-migration flow. Base URI: `${API_GW_URL}/plist-origin/api/v1/config-profile-data`, client in `PlistOriginClientProperties` (`plist-origin.client` config prefix). **Not a DDmR-workspace service and absent from the repo service map**, which is why a reader cannot look it up: it is a Tyk API product (`*/api-products/plist-origin/`) reserved for an upstream owned outside this workspace, currently pointed at a placeholder host in every environment. Its commits in `tyk-gateway-management` carry `JSC-*` ticket IDs, suggesting Jamf Service Cloud / Pro ownership. |
| `mdm-schema-internal` | Payload schemas and default values for the `filterDefaults` rule. `${API_GW_URL}/mdm-schema-internal/v1/metadata/%s/%s` and `.../v1/version`. Responses cached in memory keyed by schema URL. Note that `/v1/version` on that product is a Tyk-level route to configuration-profile-service, not to the ingest pipeline; see `services/mdm-schema-ingest.md`. |
| DynamoDB | Migration history. Table `cppm-migration-history` (`MigrationHistoryEntity.TABLE_NAME`), partition key `tenantId`, sort key `migrationId`. `DynamoDbTableInitializer` creates it at startup if absent. |

Required environment variables: `API_GW_URL`, `INT_API_GW_CLIENT_ID`, `INT_API_GW_CLIENT_SECRET`.

---

## Other gotchas

**MDM schema lookups are cached but can fail hard.** The `filterDefaults` rule calls `mdm-schema-internal` for payload schemas. Calls are cached per URL, but an unreachable schema service or a 404 for an unknown payload type raises `PayloadNotFoundException`. Watch for this when migrating profiles with uncommon payload types, and note it only bites when `filterDefaults: true` is passed.

**GraalVM native binary.** The service is compiled to a native image for fast startup. Reflection configuration lives in `src/main/resources/META-INF/native-image/reflect-config.json`. New types needing runtime reflection (for example new Jackson-deserialized domain classes) must be added there and the binary recompiled, or they fail only in the native build and not in a JVM test run. **Mockito is incompatible with AOT, so `processTestAot` is disabled for tests**; this means the test suite does not exercise the AOT path, and a reflection regression can pass CI tests and fail at runtime.

---

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/configuration-profile-plist-migrator; git -C $R fetch origin -q

# What changed since this doc was last reviewed
git -C $R log --oneline origin/main --since=2026-07-29

# Unmerged work: check before concluding the scheduled-migration executor does not exist
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Is there still no background worker? All three should be empty / read-only.
git -C $R grep -rn '@Scheduled\|@Async\|@EnableScheduling\|TaskExecutor\|ExecutorService' \
  origin/main -- 'src/main/*'
git -C $R grep -n 'public ' origin/main \
  -- 'src/main/java/com/jamf/plistmigrator/service/MigrationHistoryService.java'

# Current status enum (do not transcribe it)
git -C $R show origin/main:src/main/java/com/jamf/plistmigrator/domain/status/MigrationStatus.java

# MCX display-name map and the null-display-name early return
git -C $R show origin/main:src/main/java/com/jamf/plistmigrator/service/transformation/MCXTypeTransformer.java

# Outbound base URIs, env vars, and the filterDefaults default
git -C $R show origin/main:src/main/resources/application.yml
git -C $R grep -n 'filterDefaults' origin/main -- 'src/main/java/com/jamf/plistmigrator/MigratorRs.java'

# Does plist-origin have a real upstream yet? (placeholder host = still inert)
git -C ~/Projects/DDmR/tyk-gateway-management grep -rn 'target_url' master \
  -- '*/api-products/plist-origin/api-definitions*.yaml'
```
