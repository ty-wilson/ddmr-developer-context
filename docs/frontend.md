# Frontend

Last reviewed: 2026-07-29. Re-verified against code/live endpoints on that date: the MDM schema + translations pipeline, the CP-vs-declarations consumption split, and JSFG's localization handling. The monorepo/tooling and MFE-app sections were not re-verified and are older; treat them as a pointer only.

## micro-frontend-hub monorepo

All Jamf shared UI is maintained in a single Nx (package mode) + pnpm workspace at `jamf/micro-frontend-hub`. Required Node and pnpm versions are declared in the repo (`.nvmrc`, `package.json` `engines`/`packageManager`) and in its own `CLAUDE.md` under Prerequisites. Read them there rather than trusting a version quoted here.

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
  → schema-transformed EventBridge event (fans out to parallel Lambdas):
      ├── java-enhancements Lambda  (adds javaName, existingJavaType)
      ├── archive Lambda            (commits schemas to archive git repo)
      └── translations Lambda      (generates translation schemas)
```

### Storage (S3 buckets)

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

**Two version notions, and they can differ.** The ingest keeps a "latest ingested" version in SSM (`/mdm_schema/version/device-management`) — that's the version new content is uploaded *under*. `GET /v1/version` returns the "last **supported** version", which is what the MFEs request and can *lag* the ingested one. So freshly-ingested schema/translations can exist at a version the UI isn't asking for yet. (`sbox` local dev reads the `dev`-tier buckets.)

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

## Translations & localization

Field labels/descriptions come from a separate translations layer, served at `GET /v1/translations/{version}/{type}/{locale}` and authored in the **`jamf/mdm-schema-translations`** repo (Goldminers). The ingest **translations Lambda** auto-generates only the `en-US` base (`title`/`description`) from the schema; every other locale, plus the Jamf overrides, are **hand-authored/merged** into that repo and preserved across schema versions. The repo's `s3-upload` GitHub Action deploys a branch's translations to a per-env bucket (`dev`/`stage`/`prod`) on push to specific branches or via `workflow_dispatch`.

Override fields (all optional, keyed per property):
- `jamfTitle` / `jamfDescription` — override a field's title/description; win over the Apple base text.
- `jamfEnums` — map raw `enum`/`rangelist` values to human-readable dropdown labels. Partial maps are fine; unmapped values fall back to the raw value.

**Consumption / `$ref` handling:** JSFG resolves `$ref`s in **both** the schema (`unrefSchema`) and the localizations (`unrefLocalizations`), so fields behind a `$defs`/`$ref` (e.g. array-item types) get localized too. `jamfEnums` reach the Select widget via `setLocalizations(...)`.

**CP vs declarations divergence.** Two mechanisms deliver localization, and they are easy to conflate:

- `jamfTitle` / `jamfDescription` are folded into the *schema* by the declarations query layer (`buildSchemaProperty`), so they apply without any extra wiring.
- `jamfEnums` are consumed by JSFG's Select widget and only arrive via `setLocalizations(...)`. A component that never calls it gets raw enum values in its dropdowns no matter what the translations contain. Config-profiles forwards schema + ui-schema + localizations; declarations did not, which is why declaration enum dropdowns showed raw values (addressed in DDMR-1224).

**Trap: `buildSchemaProperty` is a lossy allow-list.** The declarations MFE rebuilds each Apple field by copying a fixed set of keywords, silently discarding anything not on the list (notably `rangelist` and `assettypes`; top-level conditionals are likewise dropped by `convertDeclarationToSchema`). A dropped `rangelist` renders as a free text input that validates nothing. Config-profiles is immune because it never rebuilds the schema. When a declaration field renders or validates unexpectedly, compare the served schema against what reaches JSFG before assuming the schema is at fault.

---

## json-schema-form-generator (`@jmf/json-schema-form-generator`)

**Trap: two separate JSFG codebases exist, and only one renders Blueprints forms.** Confirm which you are reading before drawing conclusions about form behavior.

- **`apps/json-schema-form-generator`** (inside `micro-frontend-hub`) is the MFE remote that Blueprints component apps actually load at runtime via `FeatureAppLoader`. It validates using `json-schema-library`. **This is the one that renders declarations and config profiles.**
- **`jamf/json-schema-form-generator`** (standalone sibling repo) is a separate project with its own stack and its own validation approach. It is *not* what the Blueprints MFEs load. Check its commit recency before assuming it is current.

Because they share a name, a grep across sibling repos will hit both. Reasoning about the wrong one produces confident but wrong conclusions about validation and widget behavior.

The in-hub MFE uses nanostores for reactive form state. Communication with the parent happens exclusively through `jamf:json_schema_form_generator_service`:
- `setJsonSchema(schema)` — pushes a new schema into the form
- `setPayloadFormValue(value)` / `setInitialPayloadFormValue(value)` — pre-populate or reset form values
- `getPayloadOutputFormValue()` — retrieve the current form output on save
- `getCanBeSaved()` / `setSaveClicked(true)` — trigger validation before submission
- `setUiSchema(uiSchema)` / `setLocalizations(localizations)` — forward UI hints (widget/hide/order) and localization data (`jamfTitle`/`jamfDescription`/`jamfEnums`) for the form to apply

---

## Blueprint backend services

Spring Boot services back the Blueprints feature. All use M2M OAuth2 authentication and are deployed on the OCEAN Kubernetes platform. Language/runtime versions are declared in each repo's build files. The service map in `CLAUDE.md` is the authoritative inventory; the ones below are the main Blueprints backends.

### blueprint-management-service

Manages blueprint lifecycle: create, edit, version, deploy, undeploy. Key points:

- **Persistence**: PostgreSQL with Flyway migrations; soft-deletes; versioned via `BlueprintVersion` entities
- **Deployments**: Async via Pulsar — deploy/undeploy return 202, actual work triggered by `pdd/blueprints/blueprint-deployment-task` (consumed by `blueprint-deployment-service`, which is owned by the Ocean team, not DDmR)
- **Multi-tenancy**: Every query filters by `tenantId` from M2M JWT; `@TenantId` annotation on controllers
- **JSON Merge Patch** (RFC 7386) for partial blueprint updates, with separate SAVE vs DEPLOY validation profiles
- **External dependencies**: calls component registry service to resolve components; calls tenant service for org info

### blueprint-components-registry-service

Manages the catalog of available Blueprint components — the list of component types that can be added to a Blueprint (e.g., Configuration Profiles, Declarations, App Managed). Exposes a REST API at `/blueprints/components-registry`.

It also decides **which components a product is offered at all**, via a per-product supported-OS capability list in its Spring config (`jamf.product.capabilities.supported-operating-systems`, keyed `pro` / `school`). This is a *product-wide* list and is distinct from an individual declaration's own `supportedOS`, so the two can disagree. When a UI offers a component on a platform Apple doesn't support, check both this config and the owning component service's per-item metadata.

### blueprint-component-declarations-service

Manages declaration assignment data for Blueprint components of type "declarations". Persists the declaration payloads authored in the `blueprint-component-declarations` MFE. REST API at `/blueprints/components/declarations`.

---

## Where to find the data (verify rather than trust)

The schema/translations claims above are all cheaply checkable. Prefer these over the prose. `sbox` reads the `dev`-tier buckets; substitute a real version for `<ver>`.

```bash
GW=https://tyk.sbox.ocean.jamf.build/mdm-schema

# Which version will the MFEs actually request? (the "last supported" one)
curl -s -H "X-TenantId: default-tenant" -H "tenantId: default-tenant" "$GW/v1/version"

# Raw Apple declaration schema: check rangelist / supportedOS / minimum yourself
curl -s -H "X-TenantId: default-tenant" -H "tenantId: default-tenant" \
  "$GW/v1/declarative/<ver>/configurations/<shortType>"

# Transformed JSON Schema for a config-profile payload
curl -s -H "X-TenantId: default-tenant" -H "tenantId: default-tenant" \
  "$GW/v1/metadata/<ver>/<payloadType>"

# Served translations, including jamfTitle / jamfDescription / jamfEnums
curl -s "$GW/v1/translations/<ver>/<fullDeclarationType>/en-US"

# Deploy a translations branch to an env bucket (dev|stage|prod)
gh workflow run s3-upload.yml --repo jamf/mdm-schema-translations \
  -f env=dev -f ref_name=<branch>
```

If a declaration renders or validates unexpectedly, diff the **served** schema against what the MFE hands JSFG. The allow-list trap above means the two can legitimately differ.

Before concluding another team has not implemented something, check unmerged work:

```bash
git -C <repo> fetch origin --quiet
git -C <repo> for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -20
```
