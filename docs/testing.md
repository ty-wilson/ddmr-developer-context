# Testing

Last reviewed: 2026-04-07

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

`configuration-profile-service` and `blueprint-management-service` do not have companion test repos at this time; their component-test configuration lives inside the service repo itself.

## Component Tests

Component tests start the service under test inside a Testcontainer and exercise it over HTTP. They are the primary verification gate before deploying a build.

### Stack

- Kotlin 2.0, JDK 21, Spring Boot 3.3, Gradle
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
5. Injects values into test-class fields via custom annotations: `@ScopingEngineClient`, `@ScopingEngineUri`, `@TestPulsarGroupChangeProducer`, `@TestPulsarChannelChangeProducer`, `@DssCreds`, `@BrokerCreds`, `@M2MEnvironment`, `@CryptoCreds`, `@TestPulsarClient`, `@TestPulsarConsumerBuilder`, etc.

### Test Class Structure

```kotlin
@ExtendWith(ScopingEngineExtension::class)
@Tag("system-test")
@SpringBootTest
class ScopeEndpointTest {

    @ScopingEngineClient
    private lateinit var client: WebTestClient

    @Test
    fun testScopeCreateSingleGroupAndDelete() { ... }
}
```

Auth is conditional: connections to `localhost` or the container use an `X-TenantId` header; connections to remote environments use a Bearer token obtained via M2M (Robocop).

### declaration-storage-service-component-tests Variation

The DSS component tests use a `@SpringBootTest` base class (`DeclarationStorageTest`) instead of a custom extension. It implements `PulsarTestContainerSupport` and uses `@DynamicPropertySource` to wire the containerized Pulsar URL into Spring. Containers are started once per class via `@BeforeAll` with a `STARTED` guard flag.

### Running Component Tests

```bash
# Run all component tests (requires Docker)
./gradlew test

# Run only system-tagged tests
./gradlew systemTest

# Run system tests against stable-dev environment
./gradlew systemTestStableDev

# Run a single test class
./gradlew test --tests "com.jamf.platform.scoping.componenttests.ScopeEndpointTest"
```

## System Tests

System tests are smoke tests executed against live deployed environments (integration, staging, production). They are stored in `<service>-system-tests` repos.

For `declaration-storage-service-system-tests`, tests are Postman collections (`dss-smoke-test.postman_collection.json`) run with environment files for each target (`Integration`, `Staging`, `Production.use1`, `Production.eu`, `Production.apac`). GitHub Actions workflows trigger them automatically after deployments.

For `scoping-engine-system-tests`, the repo is a placeholder (README only); system test coverage is provided by the component tests via the `@Tag("system-test")` mechanism and the `systemTest` Gradle task, which runs against a non-containerized environment.

## Performance Tests

Performance tests use [Artillery](https://www.artillery.io/) YAML configs stored in `<service>-performance-tests/artillery-tests/`.

### Config Structure

Each YAML file targets a dedicated performance environment and follows this pattern:

```yaml
config:
  target: "https://<perf-env-host>/<route-prefix>"
  http:
    timeout: 30
  phases:
    - duration: 30
      arrivalRate: 1
      name: Warm up phase
    - duration: 120
      arrivalRate: 1
      rampTo: <service-specific>
      name: Ramp up load
    - duration: 120
      arrivalRate: <service-specific>
      name: Maintain
  plugins:
    ensure:
      thresholds:
        - http.response_time.p99: <service-specific>
      maxErrorRate: 1
    expect:
      reportFailuresAsErrors: true
```

The `before` block acquires an M2M Bearer token and sets up any prerequisite state (e.g. creating a scope). Credentials are loaded from `config/credentials.yml` (gitignored).

`rampTo`, steady-state `arrivalRate`, and p99 thresholds differ per service — see the service-specific sections below.

### Scoping Engine Tests

Tests are named `<resource>-spike-<METHOD>-<groupCount>.yml`. The `groupCount` suffix (1 or 10) indicates payload size. Scenarios loop a single operation once (`count: 1`) to simulate realistic per-request load. `rampTo` is 200, steady-state `arrivalRate` is 200, and the p99 threshold is 1000 ms.

### Declaration Storage Service Tests

The DSS performance tests include CSV device fixture files (`100devices.csv`, `1000devices.csv`) for data-driven scenarios. Test files do not follow the scoping-engine `<resource>-spike-<METHOD>-<groupCount>.yml` naming pattern; DSS files use their own conventions. `rampTo` is 100. The standard p99 threshold is 1000 ms (59 of 64 test files); 5 files use 500 ms, and some large-payload tests use 3000 ms.

## Contract Tests

Contract tests use [Pact](https://docs.pact.io/) (v4 Pact format, `au.com.dius.pact` Gradle plugin version 4.6.x). They live in a `contract-test/` subproject inside the service repo.

### Pact Broker

The shared Pact Broker is at `https://pactbroker.jamf.build`. Consumer pacts are published there; provider verification results are published back. The broker URL, credentials, and branch selectors are passed as system properties at test time:

```
-DpactBrokerHost=https://pactbroker.jamf.build
-DpactBrokerUsername=...
-DpactBrokerPassword=...
-DproviderVersion=<build-version>
-DpublishPactResults=true
```

Pending pacts are enabled (`pactbroker.enablePending=true`) so new consumer contracts don't break provider builds immediately.

### Consumer Tests

Consumer tests use `@ExtendWith(PactConsumerTestExt::class)` and `@PactTestFor`. Each `@Pact`-annotated method builds a `V4Pact` describing the expected interaction using `PactDslWithProvider`. The test method receives a `MockServer` and exercises the real client code against it:

```kotlin
@ExtendWith(PactConsumerTestExt::class)
@PactTestFor(providerName = "declaration-storage-service", port = "0")
class DSSConsumerPactTest {

    @Pact(consumer = "scoping-engine", provider = "declaration-storage-service")
    fun assignDeclarationToDevice(builder: PactDslWithProvider): V4Pact { ... }

    @Test
    @PactTestFor(pactMethod = "assignDeclarationToDevice")
    fun assignDeclarationToDevice(mockServer: MockServer) { ... }
}
```

### Provider Tests

Provider tests use `@Provider`, `@PactBroker`, and `@ExtendWith(PactVerificationSpringProvider::class)`. They start the real service via `@SpringBootTest(webEnvironment = DEFINED_PORT)`, set up state via `@State`-annotated methods that write directly to DynamoDB, and verify interactions fetched from the broker:

```kotlin
@Provider("scoping-engine")
@PactBroker
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
@ContextConfiguration(classes = [TestSpringContextBase::class])
@ActiveProfiles("contract-test")
@ExtendWith(PactDynamoLifecycleExtension::class)
class ScopingEngineProviderPactTest {

    @State("A scope with one group exists")
    fun scopeWithOneGroupExists() { ... }

    @TestTemplate
    @ExtendWith(PactVerificationSpringProvider::class)
    fun verifyPact(context: PactVerificationContext?, request: ClassicHttpRequest?) { ... }
}
```

`@IgnoreNoPactsToVerify` prevents failures when no pacts exist for a branch. `@AllowOverridePactUrl` allows CI to specify a specific pact file via `-DpactFilterUrl=...`.

### Running Contract Tests

```bash
# Consumer tests only (generate pact files)
./gradlew :contract-test:consumerTest

# Provider verification (fetches pacts from broker)
./gradlew :contract-test:test -DpactBrokerHost=https://pactbroker.jamf.build ...
```

## In-Service Integration Tests (DynamoDB)

Tests that need a real DynamoDB are annotated `@DynamoTest`, which composes `@ExtendWith(DynamoLifecycleExtension::class)`. The extension:

- `beforeEach`: creates a uniquely-named DynamoDB table (class name + epoch millis) including the `group_index` GSI, then updates `AwsProperties.dynamodb.table` on the live Spring context to point at it.
- `afterEach`: deletes the table and resets the property to `"invalid"`.

The DynamoDB local instance runs on port 8000 and must be started separately before running integration tests. Use the `@DynamoTest` tag to filter:

```bash
./gradlew test -PincludeTags=integration   # run only DynamoDB-backed tests
./gradlew test -PexcludeTags=integration   # skip DynamoDB-backed tests
```

## Unit Tests

Unit tests use `TestSpringContextBase` as the `@ContextConfiguration` class. It wires the full Spring WebFlux stack (routing, handlers, AWS config) but replaces infrastructure dependencies with Mockito mocks:

- `PulsarService` — mocked; prevents Pulsar connection attempts
- `PulsarWatchdog` — mocked; prevents listener startup
- `DeclarationStorageWrapper` — mocked; prevents DSS HTTP calls

`WebTestClient` is available as a bean for HTTP-level handler tests. Individual handler and service tests inject the beans they need and stub behavior with Mockito-Kotlin.

## Common Stack Summary

| Tool | Role |
|---|---|
| JUnit 5 | Test runner, lifecycle extensions, tags |
| Kotlin / JVM 21 | Language and runtime |
| Spring Boot Test / WebFlux Test | Context loading, `WebTestClient` |
| Testcontainers | Containerized Pulsar, DynamoDB local, service images |
| Mockito-Kotlin | Mocking in unit tests |
| Pact (au.com.dius) 4.6.x | Consumer-driven contract testing |
| Artillery | Performance/load testing |
| Postman / Newman | System smoke tests (DSS) |
| JaCoCo + SonarQube | Coverage reporting (all repos) |
