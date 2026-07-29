# Shared Libraries

Last reviewed: 2026-07-29. Re-verified against code on that date: the module lists in `settings.gradle.kts` of `declaration-storage-client-core` / `platform-messaging-client-java` / both DSS clients, which `declaration-client-core` variant each client module actually resolves on `origin/main`, the `M2MPrincipal` selection order, and the Gradle plugin's JIRA-marker default. Not re-verified and older: the "Who uses it" consumer lists (correct as last reviewed 2026-04-15), the auth-mode descriptions, and the Artifactory/ECR routing behaviour.

Every version number that used to appear in this doc has been removed on purpose. Versions here drift within weeks; the module topology and the failure modes do not.

---

## spring-m2m-authentication

Repository: `spring-m2m-authentication`
Artifact: `com.jamf.platform.m2m:spring-m2m-authentication`
Owner: Ocean team

Spring Boot autoconfiguration for M2M JWT authentication. Provides inbound JWT validation (`SecurityFilterChain` with OAuth2 resource server), outbound M2M token acquisition (OAuth2 client credentials), and principal resolution from JWT claims.

**Trap: servlet-only.** The autoconfiguration is gated by `@ConditionalOnWebApplication(type = SERVLET)` and **does not activate in a WebFlux application**. You get no filter chain and no principal, with no error. The `M2MPrincipal` types and `M2MTokenConverter` are framework-agnostic, so a reactive service can depend on those directly, but it must wire its own security.

Java/Spring baselines are declared in `build.gradle.kts` (`java { toolchain { … } }` and the Spring BOM). Read them there.

### Key components

- `M2MPrincipal`: a sealed hierarchy with `M2MTenantPrincipal`, `M2MEnvironmentPrincipal`, `M2MOrganizationPrincipal`, `M2MRootPrincipal`. Each carries an optional `M2MActor` from the `act` claim (actor-token delegation).

  **The selection rule is the non-obvious part.** `M2MTokenConverter.convert()` checks claims in a fixed order and returns on the first one present:

  1. `https://www.jamf.com/tenant` → `M2MTenantPrincipal`
  2. `https://www.jamf.com/environment` → `M2MEnvironmentPrincipal`
  3. `https://www.jamf.com/organization` → `M2MOrganizationPrincipal`
  4. none of the above → `M2MRootPrincipal`

  So a token carrying both tenant and environment claims resolves as **tenant**, and `ROOT` is a *fallback for a claimless token*, not an explicitly-granted elevated type. A misissued token with no scoping claim therefore presents as root rather than failing. Note also that a tenant claim may itself carry optional `environmentId`/`organizationId` sub-claims.

- `M2MPrincipalType` enum (ROOT, TENANT, ENVIRONMENT, ORGANIZATION): used by `M2MAuthorizationManager` for type-safe authorization. Global default via `jamf.platform.m2m.allowed-principal-types`, per-path overrides via `jamf.platform.m2m.restricted-paths[]`. Defaults are `@DefaultValue` annotations on the `M2MProperties` record in `PlatformProperties.java`. Read them there rather than trusting a value quoted in prose.
- `M2MAccessTokenProvider`: programmatic token acquisition for outbound calls.
- `@TenantId`, `@EnvironmentId`, `@OrganizationId`: controller parameter annotations resolving IDs from the current principal.
- `DeclarationClientAuthAutoConfiguration`: auto-registers DSS client auth when `declaration-client-api` is on the classpath.
- `ApiGateway` enum: resolves `(Deployment, Region)` to gateway hostnames; `M2MConfigurationInfo` builds issuer/token/JWKS URIs from those hosts. Supports multiple trusted issuers per gateway (e.g. STAGE_US trusts two hosts).
- `M2MClientAutoConfiguration`: outbound client-credentials flow, configured via `jamf.platform.m2m.client.*` (`clientId`, `clientSecret`, `scope`). Picks the client registration matching the current `M2MPrincipal`'s scope automatically.

### Configuration

Properties live under `jamf.platform.*`. Authentication can be disabled for local development (`jamf.platform.m2m.authentication-enabled=false`), which falls back to header-based principal resolution via `M2MPrincipalHeaderFilter` (reads `X-Tenant-Id` / `X-TenantId` / `tenantId`, `X-Environment-Id`, `X-Organization-Id`, `X-Actor-Subject`). With `jamf.platform.m2m.autoconfigure=true` it uses OIDC discovery instead of constructing URIs from gateway hosts.

---

## platform-messaging-client-java

Repository: `platform-messaging-client-java`

Java abstraction over Apache Pulsar for DDmR services. `settings.gradle.kts` is the module list. Check it, because a FIPS variant was added after this doc was first written and a two-module mental model is now wrong:

```bash
git -C ~/Projects/DDmR/platform-messaging-client-java show origin/main:settings.gradle.kts
```

- **`platform-messaging-client-java-core`** (`com.jamf.platform.messaging.client:platform-messaging-client-java-core`): Pulsar interaction (client/producer/consumer builders), authentication providers (OAuth2, tokenizer, disabled), producer/consumer interceptors, payload encryption via `CryptoKeyServiceClient`, OpenTelemetry hooks, environment configuration (dev, stage, us-prod, local). Depends on the Pulsar client, OpenTelemetry API, Caffeine, Jackson.
- **`platform-messaging-client-java-core-fips`**: FIPS-compatible crypto variant. See `project_platform_messaging_fips_lib` notes and DDMR-1058: `MessageCryptoBcFipsCompatible` claims wire-compatibility with the non-FIPS format, and a thread-safety defect in `decrypt()` caused a production outage (2026-05-08 to 2026-05-11), fixed in the 0.7.2 line. Treat this module as load-bearing and read its git log before adopting a version.
- **`platform-messaging-client-java-spring`** (`…:platform-messaging-client-java-spring`): Spring Boot integration wrapping core. `@EnablePlatformMessaging` activates autoconfiguration; `PulsarAuthenticationSelector` and `PulsarEnvironmentSelector` resolve strategy/environment from Spring properties. Depends on `spring-context` plus core.

Class-level inventories go stale; regenerate from the build instead:

```bash
./gradlew :platform-messaging-client-java-core:dependencies --configuration runtimeClasspath
```

### Who uses it

- `scoping-engine`: `platform-messaging-client-java-spring`
- `declaration-storage-service`: `platform-messaging-client-java-spring`

### Versioning

CI supplies a `VERSION` environment variable; the root `build.gradle.kts` falls back to a hardcoded placeholder when it is unset, so **a locally-built jar does not carry a meaningful version**. Read the fallback from `build.gradle.kts` (`System.getenv("VERSION") ?: …`) rather than trusting a value here. Published to `https://artifactory.jamf.build/artifactory/libs-release`.

---

## declaration-storage-client-core

Repository: `declaration-storage-client-core`

Shared auth contract and HTTP plumbing behind both the MDM and Product DSS clients.

### Module matrix

`settings.gradle.kts` includes exactly four modules:

| Module | Target | Artifact |
|---|---|---|
| `api` | framework-free | `com.jamf.ddm:declaration-client-api` |
| `spring6` | Spring Framework (non-Boot) | `com.jamf.ddm:declaration-client-core`, qualifier `spring6` |
| `springboot31` | Spring Boot 3.x | `com.jamf.ddm:declaration-client-core`, qualifier `springboot31` |
| `springboot4` | Spring Boot 4.x | `com.jamf.ddm:declaration-client-core`, qualifier `springboot4` |

The Spring-flavoured modules provide WebFlux-based HTTP client infrastructure plus autoconfiguration, and are the foundation both the MDM and Product starters build on. Java baseline and the exact Spring/Boot BOM are declared per module in `<module>/build.gradle.kts`. Read them there. The `api` module is deliberately framework-free (see below).

**Trap: the variant you need is a Gradle *version qualifier*, not a separate artifact.** All three Spring modules publish under the same `com.jamf.ddm:declaration-client-core` coordinate and differ only by the suffix on the version string: `3.0.1-spring6`, `1.0.1-springboot4`, and so on. Consequences:

- Picking the wrong qualifier **fails at autoconfiguration time, not compile time.** The API surface is near-identical, so it compiles; the mismatch surfaces as a missing bean, a `NoSuchMethodError`, or silently absent autoconfiguration at startup.
- **Version numbers are not comparable across qualifiers.** Each variant has its own independent semantic version line (they are set by separate `versioning.buildSemantic(...)` calls per module), so a higher number does not mean newer or better, it means a different variant.
- Gradle's conflict resolution sees one coordinate. Pulling two variants transitively resolves to *one* of them, which is how a WebFlux-on-Boot-4 service silently ends up running Boot-3 autoconfiguration.

Check what a client module actually resolves rather than assuming:

```bash
git -C ~/Projects/DDmR/declaration-storage-product-client grep -n 'declaration-client-core' origin/main -- '*/build.gradle.kts'
```

### `api` module notes

- `DeclarationClientAuth<H>`: a `Consumer<H>` interface each framework-specific auth implementation satisfies.
- `AbstractDeclarationClientAuth<H>`: base class holding a `Supplier<String> tokenProvider`, with `fetchToken()` and `staticSupplier()` helpers.
- `M2mTokenFetcher` / `M2mTokenFetcherAdapter`: M2M token fetch utilities.

**It intentionally has no external runtime dependencies**, which is what lets framework-specific modules adopt it freely. Adding one would ripple into every consumer. It is also versioned independently as `declaration-client-api`, and **changes here require all downstream modules to republish**. A bump is never local.

### Tests

Tests require a `CSA_CREDENTIALS` environment variable, either a JSON blob or an AWS secret name pointing at OAuth/CSA credentials (see the repo `README.md`). Without it the test suites in all three Spring modules fail on startup, not with a skip.

---

## declaration-storage-product-client

Repository: `declaration-storage-product-client`

Client for the Declaration Storage Service **Product API** (DDmR product-side services that read, write, and assign declarations).

### Modules

- **`dsl`**: Builder/DSL layer, no Spring dependencies. **Not published standalone**; bundled into the other JARs at build time. Its Java baseline is deliberately lower than the Spring modules' so both can consume it; read `dsl/build.gradle.kts` for the current value.
  - `DeclarationProductClientDsl`: abstract base with fluent factories (`getDeclaration()`, `addDeclaration()`, `assignDeclaration()`, …).
  - Key domain models: `DeclarationDefinition`, `DeclarationAssignment`, `DeviceAssignment`, `DeviceChannel`, `DeclarationGroup` (ASSET, CONFIGURATION, MANAGEMENT, ACTIVATION).
  - Tagging: assignments carry origin tags so multiple sources can manage declarations without clobbering each other. This is the mechanism behind DSS's `(wrong tenant or tag)` warnings. See `docs/observability.md`.
- **`client`** (`com.jamf.ddm:declaration-product-spring-client`): Spring (non-Boot) reactive client on WebClient (Mono/Flux). Implements `ProductClientDslExecutor` with serialization, auth, and retry around HTTP calls. Depends on `declaration-client-core`, **`spring6` qualifier**.
- **`starter`** (`com.jamf.ddm:declaration-product-springboot-starter`): Spring Boot autoconfiguration wrapper. Properties-driven; supports CSA, M2M (Stratus robocop), and custom auth. Depends on `declaration-client-core`, **Boot-flavoured qualifier, and which one has changed over time**, so read the constraint in `starter/build.gradle.kts` rather than trusting a qualifier named here.

### Authentication modes

- **CSA** (`DeclarationClientCsaAuth`): Cloud Service Account with token provider
- **M2M** (`DeclarationClientM2mAuth`): Stratus machine-to-machine via robocop
- **Custom**: implement `DeclarationClientAuth`

### Who uses it

- `scoping-engine`: `declaration-product-springboot-starter` (via the `DeclarationAssignments` bean)
- `declaration-service`: `declaration-product-springboot-starter`

---

## declaration-storage-mdm-client

Repository: `declaration-storage-mdm-client`

Client for the DSS **MDM API** (MDM-facing services reading device-scoped declaration data). Same three-module shape as the product client:

- **`dsl`**: Builder/DSL layer, MDM-API-specific operations. Not published separately.
- **`client`** (`com.jamf.ddm:declaration-mdm-spring-client`): Spring (non-Boot) reactive client; depends on `declaration-client-core`, `spring6` qualifier.
- **`starter`** (`com.jamf.ddm:declaration-mdm-springboot-starter`): Spring Boot autoconfiguration wrapper; depends on the Boot-flavoured `declaration-client-core` qualifier declared in `starter/build.gradle.kts`.

The product and MDM clients track each other closely. When one bumps its core qualifier or Spring BOM, check whether the other has too, because a service on both will resolve a single `declaration-client-core`.

### Who uses it

- `declaration-service-component-tests`: `declaration-mdm-springboot-starter` (component test client)
- `scoping-engine-component-tests`: `declaration-mdm-springboot-starter` (component test client)

---

## ddmr-gradle-coordinate-plugin

Repository: `ddmr-gradle-coordinate-plugin`
Plugin ID: `com.jamf.ddm.coordinates`
Plugin portal: `https://rt.jamf.build/gradle-plugins`

Standardizes artifact naming, versioning, ECR image tagging, and Artifactory publication routing across all DDmR services and libraries. All configurable defaults (env var names, ECR registries, Maven repos, JIRA markers) live in one file: `src/main/kotlin/com/jamf/ddm/gradle/CoordinateExtension.kt`.

### Versioning strategies

**`versioning.buildSemantic(major, minor, patch[, qualifier][, uniqueLocal])`**: for libraries whose version must be stable and explicitly bumped.

- MAIN builds: `major.minor.patch`, or `major.minor.patch-qualifier`
- Branch builds: appends `+<short-branch>.<build-num>`
- Local builds: appends `+LOCAL` (or `+LOCAL.<epoch>` when `uniqueLocal = true`)

**`versioning.buildGenerated([uniqueLocal])`**: for services, where every build gets a unique version.

- Produces `<short-branch>.<build-num>`; local builds produce `LOCAL` (or `LOCAL.<epoch>`)

**SNAPSHOT support**: setting `INCLUDE_SNAPSHOT_LITERAL` appends `-SNAPSHOT` to the end of the full version string, for branch builds only (never MAIN), *after* any `+branch.buildnum` suffix.

Format illustrations (these are **formats, not current tags**, the values are arbitrary): a semantic branch build looks like `1.2.3+DDM-3.8`, a generated one like `MAIN.8` or `DDM-3.8`, and with the snapshot literal, `1.2.3+DDM-3.8-SNAPSHOT`.

### JIRA branch shortening

`getShortBranchName()` in `CoordinateExtension.kt` normalizes the branch from `BRANCH_NAME`:

1. Absent → `LOCAL`
2. Contains `/` → keep only the part after the last `/` (`bugfix/foobar` → `foobar`)
3. Starts with a configured JIRA marker → drop everything after the ticket (`DDM-712-some-ticket` → `DDM-712`)
4. `MAIN` plus the release-candidate marker set → `RC`
5. Uppercase

**Trap: the default JIRA marker list does not match DDmR ticket keys.** Verified 2026-07-29 on `origin/main`: `jiraMarkers` defaults to `listOf("DDM")` and the matcher is `Pattern.compile("^($marker-\\d+).*")`. DDmR tickets are `DDMR-####`, and `DDMR-1265-...` does **not** match `^(DDM-\d+).*`, because the character after `DDM` is `R`, not `-`. No DDmR repo checked (`scoping-engine`, `declaration-service`, `declaration-storage-service`, all three DSS client repos, `platform-messaging-client-java`, `ddmr-jwt-sidecar`, `ddmr-authorizer-tenant`) overrides `jiraMarkers`.

Consequence: step 3 is a no-op for DDMR branches, so the **entire branch slug** ends up in version strings and ECR image tags (`DDMR-1265-ADD-DIVISION-HEADER` instead of `DDMR-1265`). Cosmetic rather than functional (nothing errors), but it makes branch artifacts hard to correlate to tickets, and it is why some branch tags are far longer than the documented format suggests. The fix is a one-line `jiraMarkers.set(listOf("DDM", "DDMR"))` (or a plugin default change); it has not been raised as a ticket as far as this doc knows. Re-check before acting:

```bash
git -C ~/Projects/DDmR/ddmr-gradle-coordinate-plugin grep -n 'jiraMarkers' origin/main
```

### Artifact configuration

`artifactName.set("my-artifact-id")` overrides the JAR/publication artifact ID, which otherwise comes from the Gradle project name. Several DDmR repos rely on this. The Gradle project name and the published artifact ID routinely differ (e.g. `declaration-storage-product-client`'s root project is `library-declaration-product-client`), so never infer a coordinate from a directory name.

`coordinates.getMavenRepoUrl()` routes by branch:

- MAIN → `https://artifactory.jamf.build/artifactory/libs-release-local`
- Other branches → `https://artifactory.jamf.build/artifactory/libs-snapshot-local`

`ArtifactExistsTask`: a Gradle task that **fails if the artifact version already exists in the release repo**. Used in CI to prevent publishing over an existing semantic version on MAIN, and to warn on branches before merge. This is the guard that turns a forgotten version bump into a red build instead of a silently overwritten release.

### Container/ECR helpers

**`containerImage.buildEcrTargetUri("team/service")`**: MAIN → `359585083818.dkr.ecr.us-east-1.amazonaws.com/jamf/ga`; branch → `.../jamf/test`. (Registry hosts are `CoordinateExtension` defaults: `mainEcr`, `branchEcr`, `publicEcr`.)

**`containerImage.buildTargetTag()`**: tag format `<branch>.<date>.<per-day-count>`, illustrated by `MAIN.2023-03-28.1` (a **format example**, not a current tag). `IMAGE_RELEASE_CANDIDATE=true` replaces the branch segment with `RC`, giving `RC.<date>.<count>`. `IMAGE_TAG_OVERRIDE` bypasses generation entirely.

**Trap: when `BUILDS_TODAY` is unavailable, the per-day count falls back to seconds since UTC midnight**, not a build counter. Tags built outside CI therefore look like a plausible build number but are really a time-of-day value. They are neither monotonic across days nor comparable to CI-produced tags, and two local builds in the same second collide.

**ECR auth helpers**: `cacheEcrAuthorization()` / `cachedEcrUsername()` / `cachedEcrPassword()` fetch and cache ECR tokens. `buildPublicEcrUri("vendor/image", "tag")` constructs pull URIs from Jamf public ECR.

### Dependency management and publishing

Every DDmR library follows the same pattern:

1. Apply `com.jamf.ddm.coordinates` for versioning and repo routing
2. Use `getMavenRepoUrl()` in the publishing block, with `ARTIFACTORY_USR` / `ARTIFACTORY_PSW` supplied by CI
3. Register `checkArtifactExists` to guard against accidental version reuse
4. Semantic versioning for libraries, generated versioning for services

---

## Where to find the data (verify rather than trust)

```bash
CORE=~/Projects/DDmR/declaration-storage-client-core
PROD=~/Projects/DDmR/declaration-storage-product-client
MDM=~/Projects/DDmR/declaration-storage-mdm-client
PLUGIN=~/Projects/DDmR/ddmr-gradle-coordinate-plugin

# Which modules exist, per repo (the module matrix above regenerates from these)
for r in $CORE $PROD $MDM ~/Projects/DDmR/platform-messaging-client-java; do
  echo "== $r"; git -C $r show origin/main:settings.gradle.kts | grep include; done

# Which declaration-client-core qualifier each client actually resolves, and its Spring BOM
git -C $PROD grep -n 'declaration-client-core\|spring-boot-dependencies\|spring-framework-bom' origin/main -- '*/build.gradle.kts'
git -C $MDM  grep -n 'declaration-client-core\|spring-boot-dependencies\|spring-framework-bom' origin/main -- '*/build.gradle.kts'

# Per-module Java baseline and independent version line (never compare across qualifiers)
git -C $CORE grep -n 'JavaVersion\|buildSemantic\|artifactName' origin/main -- '*/build.gradle.kts'

# What actually lands on the classpath (replaces any hand-written class/dep inventory).
# Module names differ per repo: springboot4/spring6/api here, :client / :starter in the
# DSS clients, :platform-messaging-client-java-core in the messaging lib.
(cd $CORE && ./gradlew :springboot4:dependencies --configuration runtimeClasspath)

# Plugin defaults: JIRA markers, env var names, ECR/Maven hosts, all in one file
git -C $PLUGIN show origin/main:src/main/kotlin/com/jamf/ddm/gradle/CoordinateExtension.kt
```

A library consumer's declared version is not necessarily the resolved one. When behaviour disagrees with the build file, ask Gradle:

```bash
./gradlew :starter:dependencyInsight --dependency declaration-client-core --configuration runtimeClasspath
```

Before concluding a variant or module "doesn't exist yet", check unmerged work. These repos carry variant migrations on ticket branches for long stretches:

```bash
git -C <repo> fetch origin --quiet
git -C <repo> for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -20
```
