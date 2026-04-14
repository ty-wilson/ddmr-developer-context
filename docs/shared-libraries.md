# Shared Libraries

Last reviewed: 2026-04-13

## spring-m2m-authentication

Repository: `spring-m2m-authentication`
Artifact: `com.jamf.platform.m2m:spring-m2m-authentication`
Owner: Ocean team

Spring Boot autoconfiguration library for M2M JWT authentication. **Servlet-only** — gated by `@ConditionalOnWebApplication(type = SERVLET)`, does not activate in WebFlux applications. Provides inbound JWT validation (`SecurityFilterChain` with OAuth2 resource server), outbound M2M token acquisition (OAuth2 client credentials flow), and principal resolution from JWT claims. The `M2MPrincipal` types and `M2MTokenConverter` are framework-agnostic and can be used as dependencies in reactive services.

### Key components
- `M2MPrincipal` (sealed) with four subtypes: `M2MTenantPrincipal`, `M2MEnvironmentPrincipal`, `M2MOrganizationPrincipal`, `M2MRootPrincipal` — selected based on JWT claims (`https://www.jamf.com/tenant`, `/environment`, `/organization`)
- `M2MAccessTokenProvider` — programmatic M2M token acquisition for outbound service-to-service calls
- `@TenantId`, `@EnvironmentId`, `@OrganizationId` — controller parameter annotations that resolve IDs from the current principal
- `DeclarationClientAuthAutoConfiguration` — auto-registers DSS client auth when `declaration-client-api` is on the classpath
- `ApiGateway` enum — resolves `(deployment, region)` to JWKS/issuer URIs for each environment

### Configuration
Properties under `jamf.platform.*`. Authentication can be disabled (`jamf.platform.m2m.authentication-enabled=false`) for local development, falling back to header-based principal resolution from `X-TenantId`/`X-Environment-Id`/`X-Organization-Id` headers. Intended as the in-process replacement for the `ddmr-jwt-sidecar`.

---

## platform-messaging-client-java

Repository: `platform-messaging-client-java`

Java abstraction layer over Apache Pulsar for DDmR services. Split into two published modules.

### Modules

**`platform-messaging-client-java-core`** (`com.jamf.platform.messaging.client:platform-messaging-client-java-core`)
- Core Pulsar interaction: `PlatformPulsarClientBuilder`, `PlatformProducerBuilder`, `PlatformConsumerBuilder`
- Authentication providers: `OAuth2AuthenticationProvider`, `TokenizerAuthenticationProvider`, `AuthenticationDisabledProvider`
- Producer and consumer interceptors: `HeadersProducerInterceptor`, `LoggingProducerInterceptor`, `LoggingConsumerInterceptor`
- Payload encryption via `CryptoKeyServiceClient`
- OpenTelemetry tracing hooks for producers and consumers
- Environment configuration (dev, stage, us-prod, local)
- Dependencies: Apache Pulsar client, OpenTelemetry API, Caffeine, Jackson

**`platform-messaging-client-java-spring`** (`com.jamf.platform.messaging.client:platform-messaging-client-java-spring`)
- Spring Boot integration wrapping the core module
- `@EnablePlatformMessaging` annotation to activate autoconfiguration
- `PulsarAuthenticationSelector` — selects authentication strategy from Spring properties
- `PulsarEnvironmentSelector` — resolves target Pulsar environment from Spring properties
- Depends on `spring-context` and the core module above

### Who uses it
- `scoping-engine`: `platform-messaging-client-java-spring` (Spring Boot Pulsar integration)
- `declaration-storage-service`: `platform-messaging-client-java-spring`

### Versioning
Uses a `VERSION` environment variable in CI; defaults to `1.0.0-SNAPSHOT` locally. Published to `https://artifactory.jamf.build/artifactory/libs-release`.

---

## declaration-storage-client-core

Repository: `declaration-storage-client-core`

Shared auth contract and HTTP plumbing used by both the MDM and Product Declaration Storage Service clients. Intentionally has no external runtime dependencies in its `api` module so framework-specific modules can adopt it freely.

### Modules

**`api`** (`com.jamf.ddm:declaration-client-api`)
- Pure Java (no framework dependencies by design)
- `DeclarationClientAuth<H>` — interface (`Consumer<H>`) that each framework-specific auth implementation satisfies
- `AbstractDeclarationClientAuth<H>` — base class holding a `Supplier<String> tokenProvider`, with `fetchToken()` and `staticSupplier()` helpers
- `M2mTokenFetcher` / `M2mTokenFetcherAdapter` — M2M token fetch utilities
- Versioned independently at `declaration-client-api`; changes here require all downstream modules to republish

**`springboot31`** (`com.jamf.ddm:declaration-client-core`, qualifier `springboot31`)
- Spring Boot 3.1 implementation of the auth contract
- Provides WebFlux-based HTTP client infrastructure and Spring Boot autoconfiguration
- Used as the foundation by both MDM and Product starters

**`spring6`** — Spring 6 (non-Boot) variant

**`springboot4`** — Spring Boot 4 variant

All modules require Java 17. Tests require a `CSA_CREDENTIALS` environment variable (JSON blob or AWS secret name pointing to OAuth/CSA credentials).

---

## declaration-storage-product-client

Repository: `declaration-storage-product-client`

Client for calling the Declaration Storage Service's **Product API** (used by DDmR product-side services that need to read, write, and assign declarations).

### Modules

**`dsl`** — Builder/DSL layer. Java 11, no Spring dependencies. Not published as a standalone artifact; bundled into the other JARs at build time.
- `DeclarationProductClientDsl` — abstract base defining fluent factory methods: `getDeclaration()`, `addDeclaration()`, `assignDeclaration()`, etc.
- Key domain models: `DeclarationDefinition`, `DeclarationAssignment`, `DeviceAssignment`, `DeviceChannel`, `DeclarationGroup` (ASSET, CONFIGURATION, MANAGEMENT, ACTIVATION)
- Tagging system: assignments carry origin tags so multiple sources can manage declarations without interfering

**`client`** (`com.jamf.ddm:declaration-product-spring-client`)
- Spring (non-Boot) reactive client using WebClient (Mono/Flux)
- Implements `ProductClientDslExecutor` with serialization, auth, and retry around HTTP calls
- Depends on `declaration-client-core` (spring6 variant) for auth plumbing

**`starter`** (`com.jamf.ddm:declaration-product-springboot-starter`)
- Spring Boot 3.1 autoconfiguration wrapper around the client
- Properties-driven setup; supports CSA, M2M (Stratus robocop), and custom auth modes
- Depends on `declaration-client-core` (springboot31 variant)

### Authentication modes
- **CSA** (`DeclarationClientCsaAuth`) — Cloud Service Account with token provider
- **M2M** (`DeclarationClientM2mAuth`) — Stratus machine-to-machine via robocop
- **Custom** — implement `DeclarationClientAuth`

### Who uses it
- `scoping-engine`: `declaration-product-springboot-starter` (via `DeclarationAssignments` bean)
- `declaration-service`: `declaration-product-springboot-starter`

---

## declaration-storage-mdm-client

Repository: `declaration-storage-mdm-client`

Client for calling the Declaration Storage Service's **MDM API** (used by MDM-facing services that need to read device-scoped declaration data).

### Modules

**`dsl`** — Builder/DSL layer (same pattern as product client, MDM-API-specific operations). Not published separately.

**`client`** (`com.jamf.ddm:declaration-mdm-spring-client`)
- Spring (non-Boot) reactive client
- Depends on `declaration-client-core` (spring6 variant)

**`starter`** (`com.jamf.ddm:declaration-mdm-springboot-starter`)
- Spring Boot autoconfiguration wrapper
- Depends on `declaration-client-core` (springboot31 variant)

### Who uses it
- `declaration-service-component-tests`: `declaration-mdm-springboot-starter` (component test client)
- `scoping-engine-component-tests`: `declaration-mdm-springboot-starter` (component test client)

---

## ddmr-gradle-coordinate-plugin

Repository: `ddmr-gradle-coordinate-plugin`  
Plugin ID: `com.jamf.ddm.coordinates`  
Plugin portal: `https://rt.jamf.build/gradle-plugins`

Gradle plugin that standardizes artifact naming, versioning, ECR image tagging, and Artifactory publication routing across all DDmR services and libraries.

### Versioning strategies

**`versioning.buildSemantic(major, minor, patch[, qualifier][, uniqueLocal])`**
- MAIN builds: `major.minor.patch` (e.g., `1.2.3`) or `major.minor.patch-qualifier`
- Branch builds: appends `+<short-branch>.<build-num>` (e.g., `1.2.3+DDM-3.8`)
- Local builds: appends `+LOCAL` (or `+LOCAL.<epoch>` if `uniqueLocal = true`)
- Used for libraries where the version must be stable and explicitly bumped

**`versioning.buildGenerated([uniqueLocal])`**
- Produces `<short-branch>.<build-num>` (e.g., `MAIN.8`, `DDM-3.8`)
- Local: `LOCAL` (or `LOCAL.<epoch>`)
- Used for services where every build gets a unique version

**SNAPSHOT support**: setting `INCLUDE_SNAPSHOT_LITERAL` appends `-SNAPSHOT` at the end of the full version string for branch builds only (not MAIN), after any `+branch.buildnum` suffix — e.g., `1.2.3+DDM-3.8-SNAPSHOT`.

### Artifact configuration

`artifactName.set("my-artifact-id")` overrides the JAR/publication artifact ID from the Gradle project name.

`coordinates.getMavenRepoUrl()` returns:
- MAIN branch → `https://artifactory.jamf.build/artifactory/libs-release-local`
- Other branches → `https://artifactory.jamf.build/artifactory/libs-snapshot-local`

`ArtifactExistsTask` — Gradle task that fails if the artifact version already exists in the release repo. Used in CI to prevent publishing over an existing semantic version on MAIN, and to warn on branches before merge.

### Container/ECR helpers

**`containerImage.buildEcrTargetUri("team/service")`** — Returns the appropriate ECR URI:
- MAIN → `359585083818.dkr.ecr.us-east-1.amazonaws.com/jamf/ga`
- Branch → `359585083818.dkr.ecr.us-east-1.amazonaws.com/jamf/test`

**`containerImage.buildTargetTag()`** — Tag format: `<branch>.<date>.<per-day-count>` (e.g., `MAIN.2023-03-28.1`). `IMAGE_RELEASE_CANDIDATE=true` replaces the branch segment with `RC`, producing the same `RC.<date>.<count>` pattern (e.g., `RC.2023-03-28.1`). Also supports `IMAGE_TAG_OVERRIDE`. When `BUILDS_TODAY` is unavailable the count falls back to seconds since midnight rather than a build counter.

**ECR auth helpers**: `containerImage.cacheEcrAuthorization()` / `cachedEcrUsername()` / `cachedEcrPassword()` — fetch and cache ECR tokens via the AWS SDK. No-op for LOCAL builds by default.

**`containerImage.buildPublicEcrUri("vendor/image", "tag")`** — Constructs a pull URI from the Jamf public ECR (`359585083818.dkr.ecr.us-east-1.amazonaws.com/jamf/public`).

### JIRA branch shortening
The plugin trims branch names using configured JIRA markers (default: `["DDM"]`) so that `DDM-123-some-description` becomes `DDM-123` in version strings and image tags.

### Dependency management and publishing
All DDmR libraries follow the same pattern enabled by this plugin:
1. Apply `com.jamf.ddm.coordinates` for versioning and repo routing
2. Use `getMavenRepoUrl()` in the publishing block with `ARTIFACTORY_USR` / `ARTIFACTORY_PSW` credentials supplied by Jenkins
3. Register `checkArtifactExists` to guard against accidental version reuse
4. Use semantic versioning for libraries, generated versioning for services
