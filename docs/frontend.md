# Frontend

Last reviewed: 2026-04-07

## micro-frontend-hub monorepo

All Jamf shared UI is maintained in a single Nx (package mode) + pnpm 9 workspace at `jamf/micro-frontend-hub`. Node.js v20 is required.

```
micro-frontend-hub/
├── apps/        # MFE remote applications and demo shells
├── libs/        # Shared libraries and Feature Hub services
├── tools/
│   ├── cdk/     # AWS CDK deployment infrastructure
│   └── utils/   # Build/deployment scripts
```

All packages use the `@jmf/` prefix (e.g., `@jmf/blueprints`, `@jmf/compliance-benchmarks`). Commit messages must follow Conventional Commits; CI handles version bumps automatically — never bump `package.json` versions manually.

### Tooling

- **Nx**: Task runner and affected-build detection
- **pnpm workspaces**: Dependency management; cross-package references use `workspace:*`
- **Vite** (most apps) + **Webpack** (legacy apps): Bundlers; Module Federation is supported for both via `@module-federation/vite` and custom webpack config
- **Vitest**: Unit tests; **Playwright**: integration/E2E tests
- **Prettier + ESLint** (`@jmf/eslint-config`): Formatting and linting

### Release and deployment

Apps are versioned with SemVer driven by Conventional Commits. After a PR merges, CI creates a `chore: bump versions` commit, tags it, then the deploy workflow releases to Dev, Stage, and Prod automatically. The MFE CDN supports semver-range URL resolution (e.g., `/blueprints/1` resolves to the latest `1.x.x`). `latest` and `stable` aliases are also supported.

---

## MFE apps

Notable apps in `apps/`:

| App | Package | Framework | Purpose |
|-----|---------|-----------|---------|
| `blueprints` | `@jmf/blueprints` | React + Vite | Main Blueprints host that mounts blueprint-component MFEs |
| `blueprint-component-configuration-profiles` | `@jmf/blueprint-component-configuration-profiles` | React + Vite | Config Profiles step inside a Blueprint |
| `blueprint-component-declarations` | `@jmf/blueprint-component-declarations` | React + Vite | Declarations (DDM) step inside a Blueprint |
| `blueprints-component-declarations` | `@jmf/blueprints-component-declarations` | React | Alternate declarations entry point |
| `json-schema-form-generator` | `@jmf/json-schema-form-generator` | React + Vue 3, json-schema-library | Renders forms from JSON Schema; loaded dynamically by other MFEs. See note below about the standalone repo. |
| `app-switcher` | `@jmf/app-switcher` | Vue 3 + Vite | Top-level app navigation widget |
| `compliance-benchmarks` | `@jmf/compliance-benchmarks` | React + Vite | Compliance benchmarks UI |
| `scoping` | `@jmf/scoping` | React + Vite | Device scoping / group assignment UI |
| `platform-authorization` | `@jmf/platform-authorization` | React + Vite | API client management in Jamf Account |
| `ascent` / `assistant` / `data-streams` | — | — | Additional DDmR-owned production apps |
| `declaration-reporting-mfe` / `mms-token-management` / `scim-integration` | — | — | Additional DDmR-owned production apps |
| `angular-shell` / `vue-shell` / `react-vite-shell` | — | Angular / Vue / React | Local development host shells |
| `demo-{react,vue,vanilla}-remote` | — | Various | Reference implementations for each framework |

There are also many smaller `blueprint-component-*` apps (e.g., `blueprint-component-ddm-passcode`, `blueprint-component-sw-update`) representing individual Blueprint step MFEs.

---

## Shell integration: Feature Hub + Module Federation

### Feature Hub pattern

Every MFE remote exposes a `feature.ts` / `feature.tsx` entry point that exports a `FeatureAppDefinition` (from `@feature-hub/core`). The definition declares:

- `dependencies.featureServices` — Feature Hub services the MFE requires (e.g., `jamf:auth_service`, `jamf:routing_service`)
- `ownFeatureServiceDefinitions` — services the MFE provides to child remotes it loads (e.g., `blueprints` provides `jamf:blueprints-service`)
- `attachTo(element: HTMLElement)` — mounts the React/Vue/Angular app into the provided DOM element

The host shell calls `createFeatureHub(...)` to initialize all services, then uses the `<feature-app-loader>` web component (from `@feature-hub/dom`) to load and mount remotes by URL.

### Loading remotes

Remote bundles are served from CloudFront CDN at `cdn.mfe.jamf.io/<app-name>/<semver>/assets/remoteEntry.js`. A SemVer-resolving Lambda fronts S3 so that minor/major version aliases resolve to the latest matching patch. The host fetches the `remoteEntry.js` URL at runtime; no compile-time knowledge of the remote version is needed.

### CSS isolation

Remotes inject their own stylesheets into the `feature-app-container` web component's shadow DOM using the `@jmf/vite-css-injection` or `@jmf/webpack-css-injection` lib. This prevents style bleed between MFEs.

### Services provided to remotes

All services are discovered by string ID at runtime. Key services from `libs/`:

| Service ID | Library | Purpose |
|------------|---------|---------|
| `jamf:auth_service` | `@jamf/mfe-auth-service` | Auth0 token and identity; backed by nanostores |
| `jamf:routing_service` | `@jamf/mfe-routing-service` | Navigation and deep-link helpers |
| `jamf:tenant_service` | `@jamf/mfe-tenant-service` | Tenant and environment info |
| `jamf:user_preferences_service` | `@jamf/mfe-user-preferences-service` | User locale and preferences |
| `jamf:gateway_base_url_service` | `@jamf/mfe-gateway-base-url-service` | Tyk API Gateway base URL |
| `jamf:blueprints_service` | `@jamf/mfe-blueprints-service` | Blueprint form state; shared between Blueprints host and component MFEs |
| `jamf:feature_hub_provider_service` | `@jamf/mfe-feature-hub-provider-service` | Passes Feature Hub instance to nested remotes |
| `jamf:json_schema_form_generator_service` | `@jamf/mfe-json-schema-form-generator-service` | Mediates schema and form values between JSFG MFE and its caller |
| `jamf:permissions_service` | `@jamf/mfe-permissions-service` | Admin permission lookup |
| `jamf:analytics_service` | `@jamf/mfe-analytics-service` | Pendo/Gainsight analytics |

---

## MDM Schema pipeline

The pipeline ingests Apple's public MDM schema YAML, transforms it to JSON Schema, and serves it over an internal HTTPS endpoint. It is owned by the Goldminers team (Jira project: GOLD) and lives across two repos:

- **`mdm-schema-ingest-inbound-adapter`**: Node.js/TypeScript Lambda code (npm workspaces, ESM, esbuild)
- **`mdm-schema-ingest-infrastructure`**: Terraform/Terragrunt IaC

### Pipeline flow

```
EventBridge scheduler (hourly)
  → ingest Lambda
      clones Apple MDM repo to EFS, detects new commits/branches,
      creates versioned snapshots, updates SSM Parameter Store
  → SSM Parameter Store version change (EventBridge rule)
  → transformation Lambda
      reads YAML from EFS, converts to JSON Schema,
      merges Jamf-specific metadata, uploads to S3
  → schema-transformed EventBridge event (three parallel Lambdas):
      ├── java-enhancements Lambda  (adds javaName, existingJavaType)
      ├── archive Lambda            (commits schemas to archive git repo)
      └── translations Lambda      (generates translation schemas)
```

### Storage (three S3 buckets)

| Bucket | Contents |
|--------|----------|
| JSON schemas | Transformed JSON Schema per payload type and DDM declaration type |
| UI schemas | UI-layer hints (field order, widget overrides) |
| Translations | Locale strings per payload type |

### Serving layer

An internal ALB at `{env}.mdm-schema.jamf.build` routes to a path-mapping Lambda (inline Node.js). Routes:

- `GET /v1/metadata/{id}/{payloadType}` — JSON schema for a config profile payload
- `GET /v1/ui-schema/{schemaVersion}/{payloadType}` — UI schema
- `GET /v1/translations/{schemaVersion}/{payloadType}/{locale}` — translations
- `GET /v1/declarative/{schemaVersion}/{category}/{shortType}` — raw Apple DDM declaration schema

Environments: `dev.mdm-schema.jamf.build`, `stage.mdm-schema.jamf.build`, `prod.mdm-schema.jamf.build`.

---

## Config Profiles vs Declarations: schema handling

Both MFEs use the mdm-schema serving layer but consume it differently.

### `blueprint-component-configuration-profiles`

Fetches a fully-transformed **JSON Schema** from `GET /v1/metadata/{scanId}/{payloadType}`. The schema is passed directly to the `json-schema-form-generator-service` via `setJsonSchema()`, and the JSFG MFE renders the form. UI-schema hints and translations are fetched in parallel from their respective endpoints and also forwarded to JSFG.

### `blueprint-component-declarations`

Fetches the **raw Apple DDM declaration** from `GET /v1/declarative/{schemaVersion}/{category}/{shortType}`. The raw response uses Apple's own schema vocabulary (`presence`, `supportedOS` with `introduced`/`removed`/`allowedEnrollments`, `declarationtype`, etc.) and is **not** a JSON Schema. The MFE performs a client-side transform via `convertDeclarationToSchema()` before passing the result to the same `json-schema-form-generator-service`. This transform:

1. Converts `supportedOS` from Apple's internal format to JSFG's `SupportedOses` shape
2. Maps `presence: required/optional` to mark required fields
3. Propagates declaration-level `supportedOS` to individual properties (so JSFG's OS filter works correctly)
4. Emits a standard `required[]` array for `json-schema-library` validation

---

## json-schema-form-generator (`@jmf/json-schema-form-generator`)

There are two distinct versions of JSFG:

- **`apps/json-schema-form-generator`** (in `micro-frontend-hub`): The MFE remote — React + Vue 3, `json-schema-library`, `@jamf/mfe-json-schema-form-generator-service`. Loaded at runtime by other MFEs via `FeatureAppLoader`.
- **Standalone repo** (`jamf/json-schema-form-generator`): Vue 3-only, `vuelidate`. No React, no `json-schema-library`. A separate project; do not conflate with the MFE above.

The in-hub MFE uses nanostores for reactive form state. Communication with the parent happens exclusively through `jamf:json_schema_form_generator_service`:
- `setJsonSchema(schema)` — pushes a new schema into the form
- `setPayloadFormValue(value)` / `setInitialPayloadFormValue(value)` — pre-populate or reset form values
- `getPayloadOutputFormValue()` — retrieve the current form output on save
- `getCanBeSaved()` / `setSaveClicked(true)` — trigger validation before submission

---

## Blueprint backend services

Three Spring Boot (Java 21) services back the Blueprints feature. All use M2M OAuth2 authentication and are deployed on the OCEAN Kubernetes platform.

### blueprint-management-service

Manages blueprint lifecycle: create, edit, version, deploy, undeploy. Key points:

- **Persistence**: PostgreSQL with Flyway migrations; soft-deletes; versioned via `BlueprintVersion` entities
- **Deployments**: Async via Pulsar — deploy/undeploy return 202, actual work triggered by `pdd/blueprints/blueprint-deployment-task` (consumed by `blueprint-deployment-service`, which is owned by the Ocean team, not DDmR)
- **Multi-tenancy**: Every query filters by `tenantId` from M2M JWT; `@TenantId` annotation on controllers
- **JSON Merge Patch** (RFC 7386) for partial blueprint updates, with separate SAVE vs DEPLOY validation profiles
- **External dependencies**: calls component registry service to resolve components; calls tenant service for org info

### blueprint-components-registry-service

Manages the catalog of available Blueprint components — the list of component types that can be added to a Blueprint (e.g., Configuration Profiles, Declarations, App Managed). Exposes a REST API at `/blueprints/components-registry`. No CLAUDE.md was found; the README covers local run and SBOX deployment only.

### blueprint-component-declarations-service

Manages declaration assignment data for Blueprint components of type "declarations". Persists the declaration payloads authored in the `blueprint-component-declarations` MFE. REST API at `/blueprints/components/declarations`.
