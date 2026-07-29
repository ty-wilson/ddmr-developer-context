# Blueprint Components Registry Service

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the controller route prefixes, the absence of any write endpoint, the LaunchDarkly-vs-tenant caching split (**the old page was wrong about this**, see below), the `FragmentType` enum (a member was missing), `DEFAULT_PAGE_SIZE` and the API page-size default, the `sync-fragments` topic name, and the Known Callers list (**two callers were missing**). Not re-verified and older, carried forward from the 2026-04-07 review: the fragment visibility rules, the actuator endpoint semantics, and the `AvailableFragmentsSyncFailedException` suppression behaviour. Treat those as a pointer.

Run the commands in **Where to find the data** at the end before acting on any value here. This repo moves quickly and this page falls behind it, so prefer the greps over the prose.

**Owner:** Ocean team

## Summary

Blueprint Components Registry Service is the source of truth for which blueprint components and fragments exist and which are available to a given tenant. A "component" is a named unit of configuration capability (e.g. Software Updates, Configuration Profiles) that a blueprint can include. A "fragment" is a sub-unit within a component: a concrete, selectable piece of configuration (e.g. a specific configuration profile domain).

It stores component and fragment metadata in PostgreSQL (JPA/Hibernate), evaluates LaunchDarkly feature flags and Jamf product type per tenant at read time to filter what is available, and syncs fragment data on demand from each component's own backing service via Pulsar events and actuator-triggered HTTP fetches. Component definitions (identity, capabilities, web app entry point, supported OS) are seeded from static YAML config at startup by `ComponentsInitializer`; fragment payload details are fetched live from the individual component services.

**This service is the routing table BMS uses**, so an error here propagates into deployment behaviour in another team's service. That is the reason for the verification emphasis below.

---

## API surface

Routes live under two prefixes, both read-only. Every request carries an M2M JWT and tenant identity comes from it via `@TenantId`.

- **`/external/v1/components`**, **`/external/v1/fragments`**, **`/external/v1/available-fragment-types`**: customer-facing. Returns `PagedResponse<...ExternalDto>`.
- **`/internal/v1/components`**, plus **`/internal/v1/components/global/{identifier}`**: platform-internal. Returns `...InternalDto`, and the `global` variant returns the component in its non-tenant-filtered state with **no feature-flag evaluation**, for use when tenant context is unavailable.

Regenerate the route list from the controllers rather than trusting a table (grep in the verification section). Behaviours a route list will not tell you:

**Internal and external DTOs differ, and the difference matters.** `ComponentDescriptionInternalDto` includes `capabilities` and `translator.baseUri`; `ComponentDescriptionExternalDto` does not. A caller that needs to know whether a component supports configuration transformation or on-save validations, or where the component's own service lives, **must** use the internal endpoint. This is why BMS calls internal.

**Fragment listing supports `search` (name/description), a repeatable `type` filter, `Accept-Language` for localized name/description sorting, and standard pagination and sort.** The paging parameter is named `page-size` (not `size`); the default page size is set in `application.yml` under the pageable config (`default-page-size`, `size-parameter`). Read the current value there rather than quoting one.

### Actuator (ops/admin)

Spring Boot Actuator `@WebEndpoint` / `@Endpoint` write operations on the management port, not part of the public API. Four exist at review time: `globalfragments`, `limitedfragments`, `specificfragments`, `synclimitedfragments`. The first two re-sync all components; the latter two require `tenantId` plus `componentIdentifier` and are the manual equivalent of what a `SyncFragmentsEvent` does. `ls src/main/java/**/actuator/` regenerates the list.

---

## Data Model

PostgreSQL, single schema, Flyway migrations in `src/main/resources/db/migration/`.

- **`component`**: one row per component. Keyed on a unique `identifier` string (e.g. `com.jamf.ddm.sw-updates`). Carries the LaunchDarkly `feature_flag` key, the `capabilities` set, icon URLs, `web_application_entry_point` (CDN URL for the micro-frontend remote entry), `web_application_capabilities`, `component_api_base_uri` (where to fetch fragments from), `key_details`, `supported_products`, and an owning `team` used for sync-failure metrics. `standalone` is a deprecated boolean; prefer `capabilities`.
- **`component_localization`**: i18n key/value per locale. **`component_supported_os`**: OS family, minimum version, and an optional per-OS feature flag.
- **`fragment`**: single-table inheritance with a `kind` discriminator. The three kinds are the visibility model, so they are worth stating rather than grepping:

| `kind` | Class | Visibility rule |
|---|---|---|
| `GLOBALLY_AVAILABLE` | `GlobalFragment` | Visible to all tenants |
| `LIMITED` | `LimitedFragment` | Visible only to tenants explicitly enrolled via the `limited_tenant_fragment` join table |
| `SPECIFIC` | `SpecificFragment` | Visible only to the tenant that owns it (`owner_tenant_id`) |

- **`limited_tenant_fragment`**: junction table linking a `LimitedFragment` to a tenant ID string, populated when a `SyncFragmentsEvent` is processed.

Do not transcribe the column inventory or the enum members; they drift. Regenerate both from the migrations and `src/main/java/**/model/` (see the verification section). For calibration on how fast they drift: the previous version of this page listed `FragmentType` as four members and it now has five, having gained `AI_GOVERNANCE`.

---

## Events

Consumes `SyncFragmentsEvent` on topic **`pdd/blueprints/sync-fragments-event`** (configured as `jamf.platform.messaging.consumers.sync-fragments.topic-name`), with a derived DLQ topic and a `max-redeliver-count`. Produces no events.

---

## Known Callers

**Trap: this service's entire job is being discovered, so a short caller list is a warning sign rather than a fact.** Anything holding an M2M token with the right scope can read it, and the previous version of this page listed exactly one caller. Re-derived 2026-07-29 by grepping every sibling repo for `components-registry`:

- **`blueprint-management-service`** (Ocean): calls the **internal** component endpoints before save (validating component configurations against the registry) and at deploy time (resolving which component service to call). Results are Caffeine-cached in BMS. See `blueprint-management-service.md`.
- **`blueprint-deployment-service`** (Ocean): also calls the internal endpoints. Config points at `${jamf.platform.internal-gateway.base-uri}/blueprints/components-registry` in prod and `http://blueprint-components-registry-service:8080/internal` in sbox, and it requests the `blueprint-components-registry-api-product` M2M scope.
- **micro-frontend-hub apps**, via Tyk at `/blueprints/components-registry/v1/...`: the `blueprints` host (which also maintains Pact contracts against provider `blueprint-components-registry-service-external`) and `blueprint-component-passcode-settings` (`useComponentRegistryApi.ts`).

Tyk defines both internal (`blueprint-components-registry-api-internal-*`) and external (`blueprint-components-registry-api-external-*`) API products per region, so external customer traffic is possible and this list is a floor, not a ceiling.

---

## Dependencies

| Dependency | How used |
|---|---|
| **Tenants service** (`tenants-odin`) | `TenantServiceClient` / `TenantServiceAdapter` resolves `organizationId`, `environmentId`, and `productCode`, used to build the LaunchDarkly context and determine supported OS families. Retried on 503/504/timeout. Base URI `${jamf.platform.internal-gateway.base-uri}/tenants/api`. |
| **LaunchDarkly** | `FeatureFlagService` evaluates flags against the tenant's LD context. Components and fragments with a `featureFlag` set are hidden when the flag is off. |
| **Individual component services** | `ComponentServiceClient` calls each component's own API (`/fragments`, `/limited-fragments`, `/available-limited-fragments`, `/specific-fragments`) during sync. The per-component base URI comes from the component row's `component_api_base_uri`, which is seeded from static config; `ComponentClientProperties` holds **only** connect/read timeouts, so do not look for URIs there. |
| **Apache Pulsar** | `SyncFragmentsEvent` consumer |

---

## Traps and design decisions

**Trap: there is no HTTP endpoint to create or update a component at runtime, and that is deliberate.** `ComponentsInitializer` runs at startup and upserts component records from `jamf.blueprint.standaloneComponents` and `jamf.blueprint.fragmentedComponents` in `application.yml`. Adding or changing a component requires a code/config change and a redeploy. **Proof:** `src/main/java/**/service/ComponentsInitializer.java` is the only writer of component identity, and there are zero `@PostMapping` / `@PutMapping` / `@PatchMapping` / `@DeleteMapping` annotations anywhere under `src/main/java/**/rest/` (the grep is in the verification section, and it returning nothing is the point).

**Trap: fragment metadata is a synced copy, so the registry can be silently behind the component service.** If a component service changes its fragments, the registry does not notice. A sync must be triggered by `SyncFragmentsEvent` or an actuator call. **Sync beyond one page is unsupported and throws**: `ComponentServiceClient` fetches with a single `DEFAULT_PAGE_SIZE` request and does not paginate. Read the current constant from `client/ComponentServiceClient.java`; it was 1000 at review time, and a component that exceeds it fails the sync rather than truncating.

**Trap: LaunchDarkly is evaluated on every read with no caching, but the tenant lookup *is* cached.** The previous version of this page claimed every read hits Tenants Service, which is wrong and led to the wrong conclusion about the hot path. `FeatureFlagService` has no `@Cacheable` and calls `ldClient.allFlagsState(...)` per request. `TenantServiceAdapter.getTenant` **is** `@Cacheable("tenants")`, with the spec in `application.yml` under `cache.caffeine.specs` (a bounded size and a write-expiry TTL at review time). So the hot-path dependency is the LD SDK's own local evaluation, and Tenants Service is only hit on cache miss or after the TTL. `ComponentServiceClientProvider` is separately cached as `component-service-clients`. **Proof:** `service/FeatureFlagService.java` (no cache annotation) versus `client/TenantServiceAdapter.java` (`@Cacheable("tenants")`).

**Trap: a `LimitedFragment` does not appear for a tenant just because the component service enabled it.** Visibility requires a processed `SyncFragmentsEvent` for that exact tenant plus component pair (or the equivalent actuator call) to populate `limited_tenant_fragment`. "Enabled upstream but invisible in the UI" is the expected state until the event fires.

**Trap: `AvailableFragmentsSyncFailedException` is partially suppressed.** When `syncLimitedFragments` finds fragment identifiers the component service reports as available but that are not present in the registry, it records the failure and **continues saving the known-good relations**, so the sync half-succeeds. Metric recording for this failure path is disabled in `FragmentsUpdater`, which means the failure is not visible on a dashboard. Read the logs, not the metrics.

**Supported OS filtering is product-driven and can disagree with a fragment's own metadata.** The OS families shown are intersected with the set configured in `ProductCapabilitiesProperties` for the tenant's product. A fragment listing macOS support will not offer macOS to a SCHOOL tenant if SCHOOL is not configured for macOS. `docs/frontend.md` covers the same config from the UI side.

**Base URL in production:** `https://<gateway>/blueprints/components-registry`, with Swagger at `/swagger-ui/index.html` on that base.

---

## Where to find the data (verify rather than trust)

The local checkout is often parked on a ticket branch (it was on `DDMR-1230-mfe-namespace` on 2026-07-29), so read through `origin/main`.

```bash
R=~/Projects/DDmR/blueprint-components-registry-service; git -C $R fetch origin -q
git -C $R log --oneline origin/main --since=2026-07-29
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Routes, and the design negative: this second grep returning nothing IS the finding
git -C $R grep -nE '@RequestMapping|@GetMapping' origin/main -- 'src/main/java/**/rest/*'
git -C $R grep -nE '@(Post|Put|Patch|Delete)Mapping' origin/main -- 'src/main/java/**/rest/*'

# Enum members and DB columns, regenerated instead of transcribed
for e in FragmentType ComponentCapability WebApplicationCapability Product; do
  echo "== $e"; git -C $R show origin/main:src/main/java/com/jamf/blueprint/components/model/$e.java
done
git -C $R ls-tree --name-only origin/main src/main/resources/db/migration/

# Drifting values: sync page limit, API page size, cache specs
git -C $R grep -n 'DEFAULT_PAGE_SIZE' origin/main -- 'src/main/java/**'
git -C $R show origin/main:src/main/resources/application.yml | grep -nE 'default-page-size|size-parameter|caffeine|specs|maximumSize'

# The caching claim: one file has no cache annotation, the other does
git -C $R grep -n '@Cacheable' origin/main -- 'src/main/java/**'
```

Re-derive the caller list across sibling repos (run from `~/Projects/DDmR`), then check Tyk for external exposure:

```bash
grep -rIl --include=*.java --include=*.kt --include=*.yaml --include=*.yml --include=*.ts --include=*.tsx \
  'components-registry' . | grep -v ddmr-developer-context | grep -v node_modules | cut -d/ -f2 | sort -u
grep -rn 'blueprint-components-registry-api-external' tyk-gateway-management/prod/plans/
```
