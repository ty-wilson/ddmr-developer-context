# Blueprint Component Declarations Service

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the component-type set (**eleven**, not the eight previously documented), the LaunchDarkly flag keys (**four**, not two), the absence of any Pulsar dependency, the MVC-plus-virtual-threads stack, the DSS and tenant client base-URI properties, and the Tyk listen path. **Not re-verified and older:** the per-component payload field lists, the `TranslateRequest`/`ValidationResult` body shapes, and the `Included`/`OptionallyIncluded` mechanics, which date from 2026-04-07. Commands to re-derive all of it are in "Where to find the data" at the end.

**Owner:** Ocean team

## Summary

Blueprint Component Declarations Service (BCDS) is a stateless translation layer that converts blueprint component configurations into DDM declarations stored in Declaration Storage Service (DSS). When a blueprint is saved or updated, a Blueprints service posts a component's typed configuration to BCDS, which validates it, maps it to the correct Apple DDM declaration types, creates those declarations in DSS, and returns the resulting DSS IDs and version hashes (`serverToken`s). BCDS also provides per-component validation endpoints that return field-level errors without writing anything, and cleanup endpoints that delete previously-created declarations from DSS. The service owns no persistent state of its own; all storage is delegated to DSS.

**It is Spring Boot MVC, not reactive, running on virtual threads** (`spring-boot-starter-webmvc`, `spring.threads.virtual` enabled). That distinction changes how you reason about blocking calls here: a blocking DSS call is fine and does not starve an event loop, unlike in the WebFlux DDmR services. Read the runtime and Boot versions from `pom.xml` rather than from this page.

---

## API Endpoints

All routes sit under `/v1/components/<component-type>`. Authentication is an M2M bearer token in production. In non-M2M environments, a `tenantId` header is required on every request (injected by the API gateway in production). The Tyk listen path is `/blueprints/components/declarations`, targeting `blueprint-component-declarations-service.ocean-prod:8080`.

Each component type exposes the same three operations, implemented by a `*ComponentTranslationController`, `*ComponentValidationController`, and `*ComponentCleanupController` triple sharing `AbstractComponent*Controller` base classes.

**Do not trust a component count: the repo is actively adding components.** The controller package is the declaration site, and the one-line grep in the final section regenerates the list. As of **2026-07-29** there are eleven:

| Component type | Base path |
|---|---|
| Audio Accessory Settings | `/v1/components/audio-accessory-settings` |
| Disk Management | `/v1/components/disk-management` |
| Free-form | `/v1/components/free-form` |
| Math Settings | `/v1/components/math-settings` |
| Passcode | `/v1/components/passcode` |
| Safari Bookmarks | `/v1/components/safari-bookmarks` |
| Safari Extensions | `/v1/components/safari-extensions` |
| Safari Settings | `/v1/components/safari-settings` |
| Service Background Tasks | `/v1/components/service-background-tasks` |
| Service Configuration Files | `/v1/components/service-configuration-files` |
| Software Update Settings | `/v1/components/software-update-settings` |

### `POST /{base}/translate`

Translates a component configuration into DSS declarations, creating them and returning their details.

Request body (`TranslateRequest<T>`):
- `configuration` (required): component-specific configuration object.
- `declarationIdentifierPrefix` (optional, nullable): prefix used to generate stable asset declaration identifiers. Required for components that produce `com.apple.asset.data` declarations (Service Background Tasks, Service Configuration Files).
- `currentDeployableObjects`: list of `{type, id}` pairs for declarations that already exist for this component. Carried for the caller's reference, not acted on by the service.

Response body (`TranslateResponse`):
```json
{
  "deployableObjects": [
    {
      "type": "DSS",
      "id": "<dss-declaration-uuid>",
      "identifier": "<asset-identifier or null>",
      "versionHash": "<serverToken from DSS>",
      "declarationType": "CONFIGURATION | ASSET",
      "channelType": "SYSTEM | USER"
    }
  ]
}
```

One entry per created declaration. Components producing paired asset + configuration declarations return multiple entries. Flag-gated user-channel components can return two entries with different `channelType` values.

Errors communicating with DSS result in `500 Internal Server Error`.

### `POST /{base}/validate`

Validates a configuration without writing anything to DSS. **Always returns `200 OK`, even when validation fails.**

```json
{
  "errors": [
    { "code": "NOT_NULL", "path": "configuration.targetOSVersion", "message": "must not be null" }
  ]
}
```

An empty `errors` array means valid. JSON deserialization errors (wrong type for a field) are caught and returned as validation errors rather than `400`s.

### `POST /{base}/cleanup`

Deletes a set of DSS declarations previously created by a translate call. Request body (`CleanupRequest`) is a `deployableObjects` list of `{type, id}` pairs. Returns `200` with empty body on success; if any deletion fails, all failures are collected and thrown together and the response is `500`.

---

## Component Configurations

Field-level detail drifts with every component addition; read the configuration records in the model package. The durable, non-obvious pieces:

- **Free-form** accepts an arbitrary list of declarations, each with a `kind` (CONFIGURATION or ASSET), an Apple DDM `type` string, and a raw JSON `payload`. All free-form declarations target the `SYSTEM` channel.
- **Disk Management** maps to `com.apple.configuration.diskmanagement.settings` and is a **versioned sealed interface** discriminated by a `version` field. V1 (`externalStorage`/`networkStorage` as bare enums) is `@Deprecated`; V2 uses the `{value, Included}` pair per field. **Jackson defaults to V1 when `version` is absent**, so new callers must send `"version": 2` explicitly.
- **Service Configuration Files** (`com.apple.configuration.services.configuration-files`) and **Service Background Tasks** (`com.apple.configuration.services.background-tasks`) each produce one or more `com.apple.asset.data` declarations plus one configuration declaration. Asset identifiers are generated as `<declarationIdentifierPrefix>asset_<N>`.
- Everything else maps to a single `com.apple.configuration.*` type, with the Math/Safari family additionally gated on feature flags (below).

---

## Dependencies

### Declaration Storage Service (DSS)

The only external service BCDS calls for data operations. Base URI is `jamf.dss.client.base-uri`, defaulting to `${jamf.platform.internal-gateway.base-uri}/dss`. BCDS uses Spring's declarative HTTP client (`@PostExchange`/`@DeleteExchange`), **not** the `declaration-product-springboot-starter` that the DDmR services use:

- `POST /api/v2/declaration`: creates a declaration, returns `{id, serverToken}`
- `DELETE /api/v1/declaration/{declarationId}`: deletes a declaration

DSS errors propagate as `DeclarationStorageClientException` and are mapped to `500` by `RestErrorControllerAdvice`. Connect and read timeouts are configured, not hardcoded; read them from `src/main/resources/application.yml` rather than quoting a number.

`services/declaration-storage-service.md` records why the client split matters: BCDS is in the "own declarative HTTP client" group, so a DSS contract change has to land in two separate client codebases.

### Tenant Service

Tenant metadata lookups via `jamf.tenant.client.base-uri` (`${internal-gateway}/tenants/api`), cached with Spring's `@Cacheable("tenants")`.

### LaunchDarkly

Flags gate user-channel declaration creation. The keys are the durable identifiers and live in one enum, `service/featureflag/FeatureFlag.java`. As of 2026-07-29 there are four:

- `safariExtensionsUserChannelDeclaration`
- `safariBookmarksUserChannelDeclaration`
- `safariSettingsUserChannelDeclaration`
- `mathSettingsUserChannelDeclaration`

Each takes its component from a single `SYSTEM` declaration to a `SYSTEM` **plus** `USER` pair, via `createConfigurationDeclarationSystemAndUserChannelByFeatureFlag` in `DeclarationService`.

**Trap: translate output shape is conditional on flag state, so the same request yields a different number of `deployableObjects` in different environments.** A caller or test that assumes one entry per component breaks when a flag is turned on, and the failure surfaces downstream in assignment wiring rather than here. Always check the flag's current state in LaunchDarkly for the environment you are debugging before treating an extra or missing `USER` entry as a bug.

**LaunchDarkly health is in the actuator readiness probe** (`featureFlag` group), so the service will not report ready if the LD SDK key is missing or the connection fails. A pod stuck un-ready with otherwise healthy dependencies is usually an LD problem.

### No Pulsar integration

BCDS neither produces nor consumes events; it is a synchronous HTTP service only. Derived from `git grep -i pulsar origin/main -- pom.xml src/main` returning nothing on 2026-07-29. Worth re-deriving before repeating: the sibling `blueprint-component-sw-update-service` **does** produce a Pulsar event, and was also documented as event-free until 2026-07-29.

### Known callers

Both are Ocean services and both reach BCDS through the base URI supplied by `blueprint-components-registry-service` (`translator.baseUri` per component), not a hardcoded host:

- **`blueprint-management-service`**: `POST {base}/validate` at blueprint save time. `client/ComponentServiceClient.java`.
- **`blueprint-deployment-service`**: `POST {base}/translate` and `POST {base}/cleanup` at deploy/undeploy time. `client/ComponentServiceClient.java`.

---

## Key Design Decisions and Gotchas

**BCDS is stateless and DSS owns everything.** No database of its own. If a translate call partially succeeds (some DSS creates succeed, then one fails), the **caller** must clean up by calling cleanup with whatever was returned before the failure.

**Trap: `declarationIdentifierPrefix` is required for asset-producing components, and omitting it fails late.** Service Background Tasks and Service Configuration Files generate identifiers like `<prefix>asset_1`. A null prefix produces null identifiers, which means DSS has no stable identifier for the asset and the caller cannot reliably assign it by name. Nothing rejects the request; the problem appears later as an unassignable asset.

**Trap: the `Included` field controls field *presence*, not nullability.** Many configuration models use a `{value, Included}` pair (the `OptionallyIncluded` interface). When `Included` is false (or absent, defaulting to true), the entire field is excluded from the DSS payload via a custom Jackson filter. That is different from setting the value to null, and the two produce different Apple declarations. Do not treat them as interchangeable when constructing requests.

**Trap: `/validate` returning `200` says nothing about validity.** Field validation errors never produce a 4xx. Even hard serialization failures are caught and returned as `200` with a validation error entry. Any caller that branches on HTTP status alone treats every invalid configuration as valid. Inspect the `errors` array.

**Cleanup is not transactional.** Cleanup attempts every listed deletion and collects failures. If some succeed and some fail, the successful deletes are gone and cannot be rolled back. The endpoint returns `500` if any deletion failed but **does not report which ones succeeded**, so a retry of the full list is the only safe recovery.

**Swagger UI is enabled** (`springdoc.api-docs.enabled` and `springdoc.swagger-ui.enabled` are both true in `application.yml`), served under the gateway prefix at `/blueprints/components/declarations/swagger-ui/index.html`. This is the fastest way to get exact request/response shapes per component type, and is more current than any table on this page.

---

## Where to find the data (verify rather than trust)

```bash
BCDS=~/Projects/DDmR/blueprint-component-declarations-service; git -C $BCDS fetch origin -q
git -C $BCDS log --oneline origin/main --since=2026-07-29
git -C $BCDS for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Regenerate the component-type list (the authoritative declaration site)
git -C $BCDS grep -h 'RequestMapping' origin/main -- src/main/java \
  | grep -o '"/v1/components/[^"]*"' | sort -u

# Current LaunchDarkly flag keys, and which component each gates
git -C $BCDS show origin/main:src/main/java/com/jamf/blueprint/component/declarations/service/featureflag/FeatureFlag.java
git -C $BCDS grep -n 'FeatureFlag\.' origin/main -- src/main/java/com/jamf/blueprint/component/declarations/service/DeclarationService.java

# Re-derive the "no Pulsar" negative
git -C $BCDS grep -i -n pulsar origin/main -- pom.xml src/main    # expect no output

# Client base URIs, M2M scopes, timeouts, springdoc, virtual threads: all one file
git -C $BCDS show origin/main:src/main/resources/application.yml
```

Live shapes beat any transcription. Swagger is enabled in every environment:

```
https://<gateway-host>/blueprints/components/declarations/swagger-ui/index.html
```

Before concluding a component type or flag does not exist, check unmerged branches: this repo gains components on ticket branches (`OCEAN-*`) well before they reach `main`.
