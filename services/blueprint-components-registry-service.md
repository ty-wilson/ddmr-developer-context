# Blueprint Components Registry Service

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** Ocean team

## Summary

Blueprint Components Registry Service is the source of truth for which blueprint components and fragments exist and which are available to a given tenant. A "component" is a named unit of configuration capability (e.g., Software Updates, Configuration Profiles) that a blueprint can include. A "fragment" is a sub-unit within a component — it represents a concrete, selectable piece of configuration (e.g., a specific configuration profile domain). The service stores component and fragment metadata in a relational database (Postgres via JPA/Hibernate), evaluates LaunchDarkly feature flags and Jamf product type (PRO vs. SCHOOL) per-tenant at read time to filter what is available, and syncs fragment data on demand from each component's own backing service via Pulsar events and actuator-triggered HTTP fetches. Component definitions (identity, capabilities, web app entry point, supported OS) are seeded from static YAML config at startup via `ComponentsInitializer`; fragment payload details are fetched live from the individual component services.

---

## API Endpoints

All routes are under `/external/v1` (customer-facing) or `/internal/v1` (platform-internal). Every request must carry a valid M2M JWT. Tenant identity is extracted from the JWT via `@TenantId`. Pagination uses `page` (zero-based) and `page-size` query params, defaulting to 100 items per page.

### External component endpoints (`/external/v1/components`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/external/v1/components` | List all components visible to the tenant. Results are filtered by feature flags and product type. Returns `PagedResponse<ComponentDescriptionExternalDto>`. |
| `GET` | `/external/v1/components/{identifier}` | Get a single component by identifier. Returns `ComponentDescriptionExternalDto`. 404 if not found or not visible to the tenant. |

### Internal component endpoints (`/internal/v1/components`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/internal/v1/components` | List components with feature flags evaluated for the tenant. Returns `PagedResponse<ComponentDescriptionInternalDto>`, which includes `capabilities` and `translator` fields absent from the external DTO. |
| `GET` | `/internal/v1/components/{identifier}` | Get a single component for the tenant with feature flags applied. |
| `GET` | `/internal/v1/components/global/{identifier}` | Get a component in its global (non-tenant-filtered) state — no feature flag evaluation. Used internally when a tenant context is unavailable. |

### Fragment endpoints (`/external/v1/fragments`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/external/v1/fragments` | List fragments visible to the tenant. Supports `search` (matches name/description), `type` (multi-value: `APP_CATALOG`, `APP_STORE`, `CONFIGURATION`, `LEGACY_PAYLOAD`), `Accept-Language` for localized name/description sorting, and standard pagination + sort. |
| `GET` | `/external/v1/fragments/{identifier}` | Get a single fragment by identifier. 404 if not found or not visible to the tenant. |

### Available fragment types (`/external/v1/available-fragment-types`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/external/v1/available-fragment-types` | Returns the distinct `FragmentType` values present among the tenant's visible fragments. |

### Actuator (ops/admin) endpoints

These are Spring Boot Actuator `@WebEndpoint` / `@Endpoint` operations, exposed on the management port. They are not part of the public API.

| Operation | ID | What it does |
|---|---|---|
| `POST` | `/actuator/globalfragments` | Triggers a full re-sync of global fragments for all components from their backing services. |
| `POST` | `/actuator/limitedfragments` | Triggers a full re-sync of limited fragments for all components. |
| `POST` | `/actuator/specificfragments` | Triggers a re-sync of specific (tenant-owned) fragments for all components. |
| `POST` | `/actuator/synclimitedfragments` | Syncs limited fragment availability for a specific `tenantId` + `componentIdentifier`. Useful for manually re-driving what a `SyncFragmentsEvent` would do. |

---

## Data Model

The service uses a relational database (JPA/Hibernate, single schema).

### `component` table

| Field | Notes |
|---|---|
| `id` | UUID PK |
| `identifier` | Unique string, e.g. `com.jamf.ddm.sw-updates` |
| `name` | Display name |
| `description` | Optional |
| `team` | Owning team (used for sync failure metrics) |
| `feature_flag` | LaunchDarkly flag key; if set, component is hidden unless flag is on for the tenant |
| `type` | `FragmentType` enum: `APP_CATALOG`, `APP_STORE`, `CONFIGURATION`, `LEGACY_PAYLOAD` |
| `standalone` | Deprecated boolean; prefer `capabilities` |
| `capabilities` | Set of `ComponentCapability`: `GLOBAL_FRAGMENTS`, `LIMITED_FRAGMENTS`, `SPECIFIC_FRAGMENTS`, `CONFIGURATION_TRANSFORMATION`, `ON_SAVE_VALIDATIONS` |
| `icon_url_default/light/dark` | Icon URLs |
| `web_application_entry_point` | CDN URL for the micro-frontend remote entry |
| `web_application_capabilities` | Set of `WebApplicationCapability` |
| `component_api_base_uri` | Base URI for the component's own service, used when fetching fragments |

Related tables: `component_localization` (i18n key/value per locale), `component_supported_os` (OS + optional feature flag per OS entry).

### `fragment` table (single-table inheritance)

Fragments share one table with a `kind` discriminator column:

| `kind` | Class | Visibility rule |
|---|---|---|
| `GLOBALLY_AVAILABLE` | `GlobalFragment` | Visible to all tenants |
| `LIMITED` | `LimitedFragment` | Visible only to tenants explicitly enrolled via the `limited_tenant_fragment` join table |
| `SPECIFIC` | `SpecificFragment` | Visible only to the tenant that owns it (`owner_tenant_id`) |

Key fragment fields: `identifier`, `name`, `description`, `feature_flag`, `type`, `key_details`, `supported_products`, `supported_os`, `localization`.

### `limited_tenant_fragment` table

Junction table linking a `LimitedFragment` to a `tenantId` string. Populated when a `SyncFragmentsEvent` is processed; the sync queries the component's own service for which limited fragments are currently available for that tenant.

---

## Dependencies

| Dependency | How used |
|---|---|
| **Tenants service** (`tenants-odin`) | Called via `TenantServiceClient` for every request to look up `organizationId`, `environmentId`, and `productCode`. Result is used to build the LaunchDarkly context and determine which OS families are supported. Retried on 503/504/timeout. |
| **LaunchDarkly** | `FeatureFlagService` evaluates all boolean flags for the tenant's LD context. Components and fragments with a `featureFlag` set are hidden if the flag is off. |
| **Individual component services** | `ComponentServiceClient` calls each component's own API (`/fragments`, `/limited-fragments`, `/available-limited-fragments`, `/specific-fragments`) to pull fragment metadata during sync. The base URI per component is configured in `ComponentClientProperties`. |
| **Apache Pulsar** | Consumes `SyncFragmentsEvent` on `${jamf.platform.messaging.consumers.sync-fragments.topic-name}`. |

---

## Gotchas

**Component definitions come from static config, not the API.** `ComponentsInitializer` runs at startup and upserts component records from `jamf.blueprint.standaloneComponents` and `jamf.blueprint.fragmentedComponents` in `application.yml`. There is no HTTP endpoint to create or update a component at runtime. Adding or changing a component requires a code/config change and redeployment.

**Fragment metadata lives in component backing services.** The registry stores a synced copy of fragment data. If a component service updates its fragments, the registry does not pick up changes automatically — a sync must be triggered via Pulsar (`SyncFragmentsEvent`) or the actuator endpoints. Fragment sync exceeding `DEFAULT_PAGE_SIZE` (1000) is explicitly unsupported and will throw.

**Feature flags and product type are evaluated on every read.** There is no caching layer on the LD evaluation path (beyond whatever the LD SDK does internally). Every `listExternal`, `listInternal`, or `findByIdentifier*` call fetches the tenant record from Tenants Service and evaluates all flags. Under high load, the tenant service becomes a dependency on the hot read path.

**Internal vs. external DTOs differ.** The internal DTO (`ComponentDescriptionInternalDto`) includes `capabilities` and `translator.baseUri`; the external DTO (`ComponentDescriptionExternalDto`) does not. Callers that need to know whether a component supports configuration transformation or on-save validations must use the internal endpoint.

**Limited fragment visibility requires an explicit sync per tenant.** A `LimitedFragment` is only visible to a tenant after a `SyncFragmentsEvent` is processed for that tenant+component pair. Simply enabling the fragment in the component's service does not automatically make it appear — the Pulsar event (or an actuator call) must fire first.

**`AvailableFragmentsSyncFailedException` is partially suppressed.** When `syncLimitedFragments` encounters fragment identifiers reported as available by the component service but not yet present in the registry, it records the failure but continues saving the known-good relations. The TODO comment in `FragmentsUpdater` notes that the metric recording for this failure path is currently disabled pending OCEAN-219.

**Supported OS filtering is product-driven.** The OS families shown for a component or fragment are intersected with the set configured in `ProductCapabilitiesProperties` for the tenant's product (`PRO` or `SCHOOL`). A fragment that lists `macOS` support will not show that OS to a SCHOOL tenant if `SCHOOL` is not configured to support macOS.

**Base URL in production:** `https://<gateway>/blueprints/components-registry`. Swagger is available at `/swagger-ui/index.html` on that base.
