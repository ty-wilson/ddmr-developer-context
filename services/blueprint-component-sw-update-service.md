# Blueprint Component Software Update Service

Last reviewed: 2026-04-07; **substantially corrected 2026-07-29** after re-verifying against `origin/main`. Four claims in the previous version were wrong or missing: the OS-version regex example, the "no Pulsar" claim, the existence of an automatic-enforcement configuration, and the GDMF integration. See the corrections noted inline. Re-verified on the amend date: the version constraint and its DTO, the Pulsar producer, the configuration variants, and the GDMF client. Not re-verified: the endpoint request/response shapes and the DSS call paths, which date from the original review.

**Owner:** Ocean team

## Summary

A thin translation layer in the Blueprints system. It converts a software update configuration into an Apple DDM declaration of type `com.apple.configuration.softwareupdate.enforcement.specific` and persists it in the Declaration Storage Service (DSS). It also exposes a validation endpoint so callers can check a configuration before translating, and a cleanup endpoint to delete previously created declarations. The service owns **no** persistent storage of its own: all state lives in DSS.

## Configuration variants

`SwUpdateConfiguration` is an **interface**, not a single shape, with two implementations discriminated by an `enforcementType` field:

- **`MANUAL`** (`SwUpdateManualConfiguration`): the admin names an exact target. Carries `targetOSVersion`, `targetLocalDateTime`, and optional `detailsURL`.
- **`AUTOMATIC`** (`SwUpdateAutomaticConfiguration`): the deadline is derived from OS release date rather than named. Carries `enforceAfterDays` (constrained to a small range), `deploymentTime` (device-local time of day), an optional `detailsURL`, and an enforcement strategy (`LATEST` versus `SEMANTIC`). Validated by a class-level `@ValidSwUpdateAutomaticConfiguration` constraint, so cross-field rules live there rather than on individual fields.

Read the current field set and bounds from the DTO records rather than from here.

## API Endpoints

Base path: `/v1/components/sw-update`. All endpoints accept and return `application/json`. With M2M auth enabled (production) the caller supplies a Bearer token; in non-M2M environments (local/test) a `tenantId` header is required instead.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/translate` | Translate a configuration into a DDM declaration in DSS; returns the created deployable objects. |
| `POST` | `/validate` | Validate a configuration against the same rules as translate, creating nothing. |
| `POST` | `/cleanup` | Delete DSS declarations previously created by this service. |

`translate` returns deployable objects of the form `{type: "DSS", id: "<uuid>", versionHash: "<serverToken>"}`. **`versionHash` is the DSS `serverToken`** (a hash over payload and type), and callers need it to detect whether a declaration changed, so store it.

`validate` returns `{errors: [{code, path, message}]}`. Error codes are the uppercased simple names of the violated Jakarta constraint annotations (`NOT_NULL`, `PATTERN`, and so on).

## Data Model

**Correction: this service does own a database.** An earlier version of this doc said "no persistent storage, all state lives in DSS". It has a relational datasource with Flyway migrations under `src/main/resources/db/migration/` and a `Version` table behind `VersionRepository`, used to hold the OS version catalogue described below. Declaration payloads still live in DSS; the local database is for versions, not declarations.

The single DDM type created is `com.apple.configuration.softwareupdate.enforcement.specific` in group `CONFIGURATION`, with UpperCamelCase payload keys matching Apple's schema (`TargetOSVersion`, `TargetLocalDateTime`, `DetailsURL`).

## OS version catalogue and the GDMF refresh loop

This is the least obvious part of the service and it is the link between "Apple ships a release" and "Blueprint deployment activity spikes". Verified against `origin/main` on 2026-07-29:

1. A Kubernetes **CronJob** (`<release>-fetch-versions`, schedule from `.Values.cronJobs.updateVersions.schedule`) runs `curl` against the admin port at `/actuator/availableversions`.
2. That is the `@WriteOperation` on the `availableversions` actuator endpoint, which calls `OsVersionService.updateVersions()`.
3. `updateVersions()` reads Apple's **GDMF** feed through `GdmfAdapter` / `GdmfClient`, extracting iOS, macOS, and visionOS versions with their posting and expiration dates.
4. It diffs that set against the `Version` table: obsolete rows deleted, new or changed rows upserted.
5. **If anything changed**, it calls `notifyTranslationChanged()`, which produces a `blueprint-component-translation-changed` event on Pulsar keyed on the component identifier `com.jamf.ddm.sw-updates`. Per `docs/event-layer.md`, blueprint-management-service consumes that topic.

**Trap: an Apple release can therefore set work in motion with no admin involvement.** Nothing in this service is scheduled internally (no `@Scheduled`, no Pulsar listener), so it is not self-driving in isolation, but the CronJob plus the GDMF diff means a new OS version appearing upstream is enough to emit the translation-changed event. What blueprint-management-service does on receipt is in BMS and is **not verified here**; if you are tracing an OS-release-correlated spike, that consumer is the next hop to read. Related: PI-1341 (Done) recorded roughly 950 Blueprint deployment failures during OS release events, caused by DynamoDB throttling in DSS, and noted that the only automatic recovery was the next Apple release.

**Trap: `DetailsURL` is omitted from the payload entirely rather than set to null** when `detailsURL.included` is false or absent. The DSS payload is forwarded verbatim to Apple devices, so a null would be a different (and invalid) declaration than an absent key.

## Dependencies

| Dependency | How used |
|---|---|
| **Declaration Storage Service (DSS)** | Creates declarations and deletes them by ID, via a Spring HTTP interface client (`DeclarationStorageServiceClient`). |
| **Jamf M2M (`jamf-platform-m2m`)** | `M2MAccessTokenProvider` supplies a service-to-service Bearer token before each DSS request. Gated by `jamf.platform.m2m.authentication-enabled`. |
| **Apple GDMF feed** (`GdmfClient` / `GdmfAdapter`) | Read by `OsVersionService` to maintain the local OS version catalogue (versions plus posting and expiration dates, per platform). See the refresh loop below. Whether `AUTOMATIC` deadline resolution reads that catalogue was not traced, so do not assume it. |
| **Apache Pulsar** | Produces `blueprint-component-translation-changed` (see below). |

**Correction: this service is not Pulsar-free.** The previous version of this doc stated "No Pulsar topics produced or consumed." That is wrong. The service depends on `spring-boot-starter-pulsar` and ships a `BlueprintComponentTranslationChangedProducer` plus `MessagingConfiguration`, so it **produces** `blueprint-component-translation-changed`. Note `docs/event-layer.md` attributes that topic's production to blueprint-management-service; this service is an additional producer.

## Traps

**Trap: the `targetOSVersion` regex is stricter than it looks, and the previous worked example here was wrong.** The pattern is `^[1-9]\d?(?:\.(?:\d|[1-9]\d)){1,2}$` (on `SwUpdateManualConfiguration.targetOSVersion`). Verified behaviour:

| Value | Result | Why |
|---|---|---|
| `17.5`, `17.5.1`, `17.5.10`, `26.1` | valid | one or two dot segments, each 1 to 2 digits |
| `17`, `18` | invalid | at least one dot segment is required |
| `17.5.100` | invalid | a segment may not exceed two digits |
| `17.05` | invalid | leading zeros are rejected |
| `1.2.3.4` | invalid | at most two dot segments |

The earlier claim that `17.5.10` is invalid was incorrect; `17.5.100` is the failing case. Check candidate values against the pattern before calling `/translate`.

**Trap: translate always creates, it never upserts.** `currentDeployableObjects` is accepted in the request and ignored. Unless the caller invokes `/cleanup` on the old declaration, every translate leaves an orphaned DSS record behind. This is the mechanism behind fleet-wide re-scope amplification when OS versions bump: a new declaration UUID per bump means an assign-new plus a delete-old cascade.

**Trap: `/validate` always returns `200`.** Validation failures live in the response body, not the status code. Never treat a `200` as "valid" without inspecting the `errors` array.

**Trap: `detailsURL.included` gates whether the URL reaches the payload at all.** With `included` false, `DetailsURL` is omitted regardless of whether `value` is populated. With `included` true, `value` must be a non-empty HTTP or HTTPS URL or validation fails.

**Trap: cleanup does not short-circuit, and a non-200 means partial failure.** The method attempts every ID and aggregates failures via suppressed exceptions, so some deletions may have succeeded. Blindly retrying the whole list is safe because DSS treats deleting an already-deleted declaration as success.

**Swagger UI is served in all environments** at `/v1/components/sw-update/swagger-ui/index.html`, including through the SBOX Tyk gateway. It is the most reliable source for the current request and response shapes.

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/blueprint-component-sw-update-service; git -C $R fetch origin -q

# What landed since review, and what is unmerged
git -C $R log --oneline origin/main --since=2026-04-07
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# The authoritative version constraint (and which DTO carries it)
git -C $R grep -n 'Pattern(regexp' origin/main -- src/main/java

# Configuration variants and their bounds
git -C $R ls-tree -r --name-only origin/main -- src/main/java | grep 'dto/SwUpdate'

# Confirm Pulsar production and the topic name
git -C $R grep -in 'pulsar\|TranslationChanged' origin/main -- src/main/java pom.xml

# Live API shape, and the DSS contract tests
curl -s "https://tyk.sbox.ocean.jamf.build/{namespace}/blueprints/components/sw-update/v3/api-docs" | jq -r '.paths | keys[]'
cd $R && ./mvnw failsafe:integration-test -Dpact.tests.skip=false
```
