# Testing

Last reviewed: 2026-07-29. Re-verified against sibling repos on that date: the companion-repo naming convention, the Artillery test-file layout and naming in both performance-test repos, `scoping-engine-system-tests` still being README-only, the `au.com.dius.pact` plugin location in `contract-test/build.gradle.kts`, and where the JVM toolchain is declared. NOT re-verified and older (carried from the 2026-04-20 review): the DSS component-test base-class variation, the Postman/CSA system-test details, and the unit-test `TestSpringContextBase` mock list. This pass also stripped tuning constants (`rampTo`, `arrivalRate`, p99 thresholds, ports, language versions) in favor of pointers.

## Overview

DDmR services follow a layered testing strategy: unit tests and integration tests live inside the service repo, while component tests, system tests, and performance tests live in dedicated companion repos.

## Companion Repo Pattern

Each service has one or more companion test repos following a naming convention:

- `<service>-component-tests` — black-box tests against a containerized service instance
- `<service>-system-tests` — smoke tests run against deployed environments (integration, staging, production)
- `<service>-performance-tests` — Artillery load tests run against a dedicated performance environment

Known companion repos:

| Service | Component | System | Performance |
|---|---|---|---|
| scoping-engine | scoping-engine-component-tests | scoping-engine-system-tests | scoping-engine-performance-tests |
| declaration-storage-service | declaration-storage-service-component-tests | declaration-storage-service-system-tests | declaration-storage-service-performance-tests |
| declaration-service | declaration-service-component-tests | — | — |
| ddmr-authorizer-tenant | ddmr-authorizer-tenant-component-tests | — | — |

The `—` cells are real: those services have no repo of that kind, so there is nothing to look for. Not every service has companion repos at all; when they are absent, component-test configuration lives inside the service repo itself (`src/test` plus an `application-component-test.yml`). Check rather than assume:

```bash
gh repo list jamf --limit 1000 | grep -E '\-(component|system|performance)-tests'
```

## Component Tests

Component tests start the service under test inside a Testcontainer and exercise it over HTTP. They are the primary verification gate before deploying a build.

### Stack

- Kotlin + Spring Boot + Gradle. The Kotlin plugin version and JVM toolchain are declared in each repo's `build.gradle.kts` (`plugins { kotlin("jvm") ... }` and the `jvmToolchain { languageVersion... }` block) — read them there; service and companion repo can diverge.
- JUnit 5 (`@ExtendWith`, `@Tag`, `@SpringBootTest`)
- Testcontainers (`PulsarContainer`, `GenericContainer` for DynamoDB local and the service image, `@Testcontainers`)
- `WebTestClient` for HTTP assertions
- SonarQube + JaCoCo for coverage reporting

### Custom Extension Pattern

Each service's component-test repo defines a custom JUnit `TestInstancePostProcessor` extension (e.g. `ScopingEngineExtension`, `DeclarationServiceExtension`). The extension:

1. Loads test config via Spring's `ConfigDataEnvironmentPostProcessor` against `application-component-test.yml`.
2. Resolves credentials — from config properties directly, or from AWS Secrets Manager using a named AWS profile.
3. Optionally starts Pulsar, DynamoDB local, and the service itself as Testcontainers (controlled by `scoping-engine.host=container`). When not using containers, the extension connects to an already-running instance (dev, stage).
4. Creates Pulsar producers/consumers that tests can use to inject or observe events.
5. Injects values into test-class fields via custom annotations (`@ScopingEngineClient`, `@TestPulsarClient`, `@DssCreds`, `@BrokerCreds`, ...). The set grows; regenerate it rather than reading a list:

```bash
grep -rn 'annotation class' scoping-engine-component-tests/src
```

### Test Class Structure

A component test class carries three load-bearing annotations: `@ExtendWith(<Service>Extension::class)` (wires the containers and injection), `@SpringBootTest`, and — when the test is also valid against a deployed environment — `@Tag("system-test")`. Fields are injected by the custom annotations above rather than by Spring.

```kotlin
@ExtendWith(ScopingEngineExtension::class)
@Tag("system-test")
@SpringBootTest
class ScopeEndpointTest {
    @ScopingEngineClient private lateinit var client: WebTestClient
}
```

**Trap: auth differs by target, silently.** Connections to `localhost` or the container send an `X-TenantId` header; connections to remote environments send a Bearer token obtained via M2M (Robocop). The same test class takes a different auth path depending on where it points, so an auth-shaped failure locally and remotely are unrelated problems. Check which branch the extension took before debugging a 401.

### declaration-storage-service-component-tests Variation

The DSS component tests use a `@SpringBootTest` base class (`DeclarationStorageTest`) instead of a custom extension. It implements `PulsarTestContainerSupport` and uses `@DynamicPropertySource` to wire the containerized Pulsar URL into Spring. Containers are started once per class via `@BeforeAll` with a `STARTED` guard flag.

### Running Component Tests

```bash
./gradlew test                       # all component tests (requires Docker)
./gradlew systemTest                 # only @Tag("system-test") tests
./gradlew systemTestStableDev        # system tests against stable-dev
./gradlew test --tests "com.jamf.platform.scoping.componenttests.ScopeEndpointTest"
```

Confirm the task names exist in the repo you are in (`./gradlew tasks --all | grep -i test`) — they are defined per repo, not inherited.

## System Tests

System tests are smoke tests executed against live deployed environments (integration, staging, production). They are stored in `<service>-system-tests` repos.

For `declaration-storage-service-system-tests`, tests are Postman collections under `system-tests/` — `dss-smoke-test.postman_collection.json` (post-deploy smoke check) and `dss-system-test.postman_collection.json` (broader regression suite) — run with per-target environment files under `environments/` (`Integration`, `Staging`, `Sandbox`, `Performance`, `Production.use1`, `Production.eu`, `Production.apac`). GitHub Actions workflows trigger them automatically after deployments.

Both collections exercise **both auth paths** — M2M tokens fetched from robocop AND CSA tokens fetched via the two-hop Auth0 opaque-token → `stage-csa.services.jamfcloud.com/v2/token` flow. CSA-authenticated requests additionally send the `x-customer-id` header. Credentials for the CSA test account are stored in the Postman environment files (non-prod envs use the shared `ddmr.salesforce.test@gmail.com` account).

**Trap: scoping-engine has no separate system-test suite.** The `scoping-engine-system-tests` repo carries only a README (plus `catalog-info.yaml` / `sonar-project.properties`) — there are no tests in it. System coverage for scoping-engine rides on the component tests via `@Tag("system-test")` and the `systemTest` Gradle task, pointed at a non-containerized environment. Looking for scoping-engine smoke tests in that repo is a dead end.

## Performance Tests

Performance tests use [Artillery](https://www.artillery.io/) YAML configs stored in `<service>-performance-tests/artillery-tests/`.

### Config Shape

Each YAML file targets a dedicated performance environment. Every config follows the same three-phase shape under `config.phases`:

1. **Warm up** — short, low fixed `arrivalRate`
2. **Ramp up load** — `arrivalRate` with `rampTo` climbing to the steady-state figure
3. **Maintain** — steady-state `arrivalRate` held for the bulk of the run

Two `plugins` entries do the pass/fail work:

- `ensure.thresholds` — asserts an `http.response_time.p99` ceiling, plus `maxErrorRate`. This is what makes a run fail rather than merely report.
- `expect` with `reportFailuresAsErrors: true` — turns per-request expectation failures into errors so they count against `maxErrorRate`.

A `before` block acquires an M2M Bearer token and sets up prerequisite state (e.g. creating a scope). Credentials are loaded from `config/credentials.yml` (gitignored) — a fresh clone will fail at the `before` block until that file exists.

Phase durations, `rampTo`, steady-state `arrivalRate`, and p99 thresholds differ per service and per test file. Read them from the YAML; do not carry a figure between files.

### Scoping Engine Tests

Tests are named `<resource>-spike-<METHOD>-<groupCount>.yml`, where the `groupCount` suffix indicates payload size. Scenarios loop a single operation once (`count: 1`) to simulate realistic per-request load.

### Declaration Storage Service Tests

DSS test files do **not** follow the scoping-engine naming pattern — they use their own conventions (assignment add/limit variants keyed by identifier reuse and tag behaviour). They also include CSV device fixture files (e.g. `100devices.csv`) for data-driven scenarios. Some large-payload tests deliberately carry a looser p99 threshold than the rest of the suite, so a "failing" threshold may just be the wrong file's expectation.

```bash
ls declaration-storage-service-performance-tests/artillery-tests/ scoping-engine-performance-tests/artillery-tests/
grep -rn 'rampTo\|arrivalRate\|p99\|duration' <perf-repo>/artillery-tests/*.yml
```

## Contract Tests

Contract tests use [Pact](https://docs.pact.io/) via the `au.com.dius.pact` Gradle plugin. They live in a `contract-test/` subproject inside the service repo; the plugin and its version are declared in `contract-test/build.gradle.kts`.

### Pact Broker

The shared Pact Broker is at `https://pactbroker.jamf.build`. Consumer pacts are published there; provider verification results are published back. The broker URL, credentials, and branch selectors are passed as system properties at test time:

```
-DpactBrokerHost=https://pactbroker.jamf.build
-DpactBrokerUsername=...
-DpactBrokerPassword=...
-DproviderVersion=<build-version>
-DpublishPactResults=true
```

**Trap: pending pacts are enabled (`pactbroker.enablePending=true`), so a broken NEW consumer contract passes the provider build silently.** The intent is that a new consumer can't break an existing provider pipeline, but the consequence is that a genuinely incompatible new contract produces a *green* provider build rather than a failure. Do not read a passing provider verification as proof that all published consumer contracts are satisfied — check the broker for pending/unverified interactions.

### Consumer Tests

Consumer tests use `@ExtendWith(PactConsumerTestExt::class)` plus `@PactTestFor(providerName = ..., port = "0")` on the class. Each `@Pact(consumer = ..., provider = ...)`-annotated method builds a `V4Pact` describing the expected interaction with `PactDslWithProvider`; a matching `@Test` with `@PactTestFor(pactMethod = ...)` receives a `MockServer` and exercises the real client code against it.

### Provider Tests

Provider tests use `@Provider("<service>")`, `@PactBroker`, and `@SpringBootTest(webEnvironment = DEFINED_PORT)` with the `contract-test` Spring profile. `@State`-annotated methods set up preconditions by writing directly to DynamoDB (via a Dynamo lifecycle extension), and a `@TestTemplate` method annotated `@ExtendWith(PactVerificationSpringProvider::class)` runs each interaction fetched from the broker.

Two annotations change failure behavior and are easy to miss: `@IgnoreNoPactsToVerify` prevents failures when no pacts exist for a branch (another way a provider build goes green without verifying anything), and `@AllowOverridePactUrl` lets CI target a specific pact file via `-DpactFilterUrl=...`.

```bash
grep -rn '@Provider\|@PactBroker\|@Pact(\|IgnoreNoPactsToVerify' <service>/contract-test/src
```

### Running Contract Tests

```bash
./gradlew :contract-test:consumerTest    # consumer tests only (generate pact files)
./gradlew :contract-test:test -DpactBrokerHost=https://pactbroker.jamf.build ...
```

## In-Service Integration Tests (DynamoDB)

Tests that need a real DynamoDB are annotated `@DynamoTest`, which composes `@ExtendWith(DynamoLifecycleExtension::class)`. The extension:

- `beforeEach`: creates a uniquely-named DynamoDB table (class name + epoch millis) including the `group_index` GSI, then updates `AwsProperties.dynamodb.table` on the live Spring context to point at it.
- `afterEach`: deletes the table and resets the property to the literal string `"invalid"`.

**Trap: `"invalid"` is a real value left behind in the Spring context.** After any `@DynamoTest` finishes, `AwsProperties.dynamodb.table` holds the literal `"invalid"`, not the original config value and not null. A later test in the same context that touches DynamoDB without `@DynamoTest` will attempt a table literally named `invalid` and fail with a confusing `ResourceNotFoundException` rather than a missing-config error.

A DynamoDB local instance must be running before these tests (port is set in the test config / `DynamoLifecycleExtension` — read it there, and note the containerized component tests start their own separate instance). Filter with tags:

```bash
./gradlew test -PincludeTags=integration   # run only DynamoDB-backed tests
./gradlew test -PexcludeTags=integration   # skip DynamoDB-backed tests
```

## Unit Tests

Unit tests use `TestSpringContextBase` as the `@ContextConfiguration` class. It wires the full Spring WebFlux stack (routing, handlers, AWS config) but replaces infrastructure dependencies with Mockito mocks:

- `PulsarService` — mocked; prevents Pulsar connection attempts
- `PulsarWatchdog` — mocked; prevents listener startup
- `DeclarationStorageWrapper` — mocked; prevents DSS HTTP calls

`WebTestClient` is available as a bean for HTTP-level handler tests. Individual handler and service tests inject the beans they need and stub behavior with Mockito-Kotlin. Because the mock list is what keeps unit tests offline, verify it in the class before assuming a unit test cannot reach the network: `grep -n 'MockBean\|@Bean' <service>/src/test/**/TestSpringContextBase.kt`.

## Common Stack Summary

| Tool | Role |
|---|---|
| JUnit 5 | Test runner, lifecycle extensions, tags |
| Kotlin / JVM (version per repo's `jvmToolchain` block) | Language and runtime |
| Spring Boot Test / WebFlux Test | Context loading, `WebTestClient` |
| Testcontainers | Containerized Pulsar, DynamoDB local, service images |
| Mockito-Kotlin | Mocking in unit tests |
| Pact (`au.com.dius.pact`) | Consumer-driven contract testing |
| Artillery | Performance/load testing |
| Postman / Newman | System smoke tests (DSS) |
| JaCoCo + SonarQube | Coverage reporting (all repos) |

---

## Where to find the data (verify rather than trust)

```bash
# 1. Which companion test repos exist for a service (the table above goes stale).
gh repo list jamf --limit 1000 | grep -E '\-(component|system|performance)-tests'

# 2. Real Artillery load shape and thresholds — never quote these from a doc.
grep -rn -A2 'phases:\|rampTo\|arrivalRate\|p99\|maxErrorRate' \
  scoping-engine-performance-tests/artillery-tests/*.yml

# 3. Which injection annotations the component-test extension provides.
grep -rn 'annotation class' scoping-engine-component-tests/src

# 4. Language/runtime actually in use (service and companion repo can differ).
grep -rn 'kotlin("jvm")\|JavaLanguageVersion' <repo>/build.gradle.kts

# 5. Pact plugin version, broker wiring, and the green-build escape hatches.
grep -rn 'au.com.dius.pact\|enablePending\|IgnoreNoPactsToVerify' <service>/contract-test

# 6. Which Gradle test tasks this repo really defines.
./gradlew tasks --all | grep -i test
```

Before concluding a suite doesn't exist, check unmerged work — a companion repo can be README-only on `main` while tests sit on a branch:

```bash
git -C <test-repo> fetch origin --quiet
git -C <test-repo> for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -20
```
