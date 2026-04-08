# Micro Frontend Hub

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

The micro-frontend-hub is the Nx + pnpm monorepo that hosts every MFE app for the Blueprints/DDmR platform. All product UI lives here. MFEs are built as Module Federation remotes and loaded at runtime by a host shell via the Feature Hub pattern, which injects shared services (auth, routing, tenant, etc.) across framework boundaries. The repo ships three host shells (Angular, React/Vite, Vue) and approximately 50+ remote apps and 30+ shared libraries.

## Monorepo Structure

- **Toolchain**: Nx 22 (package mode) + pnpm 9.12.2 workspaces. Node >= 20 required.
- **Layout**:
  - `apps/` — all MFE applications (remotes + shells). Each is an independent deployable unit.
  - `libs/` — shared libraries published to the internal Artifactory npm registry (`https://artifactory.jamf.build/artifactory/api/npm/npm-local/`).
  - `tools/cdk/` — AWS CDK infrastructure (S3 + CloudFront CDN, SemVer-resolving Lambda).
  - `tools/utils/` — build/deployment helper scripts.
- **Package naming**: `@jmf/` prefix for internal apps/libs (e.g., `@jmf/blueprints`). Shared service libs use the `@jamf/mfe-` prefix (e.g., `@jamf/mfe-auth-service`).
- **Nx task graph**: `build` depends on `^build` (lib deps build first). `dev` depends on `^build`. Nx caches `build`, `test`, and `type-check` outputs. Cache stored in `.nx-cache/`.
- **Versioning**: Driven by Conventional Commits + `nx release`. CI auto-bumps `package.json` versions and creates git tags after merge to `main`. Do not bump versions manually — Nx uses git tags as the version source of truth.

## Key MFE Apps

| App (`@jmf/`) | Framework | Description |
|---|---|---|
| `angular-shell` | Angular 19 | Legacy host shell; uses Webpack module federation + `@angular/elements`. Provides Feature Hub services to remotes. |
| `react-vite-shell` | React 18 + Vite | Primary React host shell. Provides Feature Hub services, Sentry integration, routing. |
| `vue-shell` | Vue 3 + Vite | Vue host shell. Mirrors react-vite-shell feature set using `vue-router`. |
| `blueprints` | React 18 + Vite | Main Blueprints product app. Loads blueprint-component-* remotes dynamically. Owns Pact consumer contracts. |
| `blueprint-component-declarations` | React 18 + Vite | DDM declarations management UI within a blueprint. |
| `blueprint-component-configuration-profiles` | React 18 + Vite | Configuration profiles management UI within a blueprint. Loads json-schema-form-generator as a sub-remote. |
| `blueprint-component-*` (20+ apps) | React 18 + Vite | Individual blueprint component editors (passcode, software updates, Safari, disk management, etc.). |
| `blueprints-component-declarations` | React 18 + Vite | Declarations component variant used in the blueprints context. |
| `scoping` | React 18 + Vite | UI for the Scoping Engine — manages scopes, group assignments, and device targeting. |
| `compliance-benchmarks` | React 18 + Vite | Compliance benchmarks and reporting UI. Uses `ky` HTTP client, Pact tests, visual regression tests. |
| `ascent` | React 18 + Vite | Elevate product UI. Uses CSRF session auth, urql/GraphQL, ag-grid, chart.js, MSW for mocks. |
| `app-switcher` | Vue 3 + Vite | App switcher dropdown in host shells. Vue only MFE; uses `@hey-api` OpenAPI client generation. |
| `platform-authorization` | React 18 + Vite | API Clients management UI in Jamf Account. |
| `json-schema-form-generator` | React 18 + Vite | Dynamic JSON Schema form renderer, loaded as a sub-remote by configuration-profiles. |
| `declaration-reporting-mfe` | React 18 + Vite | DDM declaration reporting and status UI. |
| `scim-integration` | React 18 + Vite | SCIM provisioning configuration UI. Uses `ky`. |
| `data-streams` | React 18 + Vite | Data streaming configuration UI. |
| `mms-token-management` | React 18 + Vite | MDM token management. |
| `configuration-profiles-migrator` | React 18 + Vite | Tooling to migrate legacy configuration profiles. |
| `assistant` | React 18 + Vite | AI assistant MFE. |
| `demo-react-remote` / `demo-vue-remote` / `demo-angular-remote` / `demo-vanilla-remote` | Various | Reference implementations for each supported framework. |

## Shell Integration

All MFEs integrate via the **Feature Hub** library (`@feature-hub/core`, `@feature-hub/dom`, `@feature-hub/react`).

- **Host shells** call `createFeatureHub()` to instantiate services and provide them to the Feature Hub Integrator. The `<feature-app-loader>` web component loads remote apps by URL.
- **Remote apps** declare required/optional services in `feature.ts` (the Module Federation entry point). The host injects services at mount time. Services are accessed via `featureAppManager.getFeatureAppScope()` bindings.
- **Module Federation**: Vite remotes use `@module-federation/vite` (1.7.1). Angular shell uses Webpack. Remote entry is exposed as `remoteEntry.js`. Vite remotes place assets under `/assets/`; Webpack remotes place them at root.
- **CSS scoping**: Vite remotes use `@jmf/vite-css-injection` to attach styles to the `feature-app-container` web component rather than `<head>`.
- **Design system**: All apps consume Jamf Nebula web components (`@jamf/design-system-web-components-next`) and shared tokens (`@jamf/design-system-shared`).

### Feature Hub Services (libs/)

Services are injected by the host and consumed by remotes via Feature Hub bindings:

| Service ID | Lib | Purpose |
|---|---|---|
| `jamf:auth_service` | `auth-service` | Auth0 JWT token + refresh subscription |
| `jamf:routing_service` | `routing-service` | `navigate()` + common link constants |
| `jamf:tenant_service` | `tenant-service` | Tenant/environment metadata |
| `jamf:gateway_base_url_service` | `gateway-base-url-service` | Tyk API gateway base URL |
| `jamf:groups_service` | `groups-service` | Smart Groups access |
| `jamf:user_preferences_service` | `user-preferences-service` | User preference store |
| `jamf:permissions_service` | `permissions-service` | Admin permission flags |
| `jamf:organization_service` | `organization-service` | Organization data store |
| `jamf:feature_hub_provider_service` | `feature-hub-provider-service` | Shares the Feature Hub instance with sub-remotes |
| `jamf:blueprints-service` | `blueprints-service` | Communication channel between Blueprints and its component remotes |
| `jamf:json_schema_form_generator_service` | `json-schema-form-generator-service` | Bridge between JSON Schema Form Generator and Configuration Profiles |
| `jamf:nav_prevention_service` | `nav-prevention-service` | Blocks navigation when a form is dirty |
| `jamf:ai_service` | `ai-service` | Sends prompts to the AI Assistant from host apps |
| `jamf:analytics_service` | `analytics-service` | Pendo/Gainsight analytics platform access |
| `jamf:app_store_and_in_house_apps_service` | `app-store-and-in-house-apps-service` | AppStore + InHouse app list |

## Build and Deploy

**Building:**
```sh
pnpm build          # build only affected packages (Nx affected)
pnpm build:all      # build everything
pnpm nx build @jmf/blueprints  # build a specific app
```

**Environments**: `sbox`, `dev`, `stage`, `prod`. Sandbox and Staging are accessible only via Jamf Trust VPN.

**CDN deployment**: Built assets are uploaded to S3 + CloudFront (`cdn.mfe.jamf.io`). A SemVer-resolving Lambda sits in front, enabling partial version pinning:
- `/my-app/1.2.1` → exact version
- `/my-app/1.2` → latest patch for that minor
- `/my-app/1` → latest minor and patch for that major
- `/my-app/latest` → most recently deployed version
- `/my-app/stable` → manually promoted stable release via GitHub Releases

**CI/CD flow (GitHub Actions)**:
1. PR with `deploy:affected` label → deploys to Sandbox.
2. Merge to `main` → `main_bump_versions.yml` auto-bumps package versions, creates git tags, pushes a bump commit.
3. Bump commit triggers `main_apps_deploy.yml` → deploys to Dev, Staging, then Prod.
4. Opt-in manual prod approval: add `require_manual_prod_approval` tag to `project.json`.
5. Use GitHub Merge Queue (not direct merge) — this is enforced.

**Versioning**: Nx `nx release` with `specifierSource: conventional-commits` and `currentVersionResolver: git-tag`. Never bump `package.json` versions manually.

## Local Development

**Prerequisites:**
```sh
pnpm install
cp .env.example.local .env.local   # set VITE_TYK_GATE_URL, org ID, auth token
```

**Standalone (isolated, no host):**
```sh
pnpm nx dev @jmf/blueprints
# or from the app directory:
pnpm dev
```
Each remote app has a `main.ts` that bootstraps it with `@jmf/mock-services` standing in for Feature Hub services. This is the recommended way to develop in isolation.

**Inside a shell (with host):**
```sh
# Start a shell and auto-build all linked remotes first:
pnpm nx dev:withApps @jmf/react-vite-shell
pnpm nx dev:withApps @jmf/vue-shell
pnpm nx dev:withApps @jmf/angular-shell

# or from the root:
pnpm dev:react-shell
pnpm dev:vue-shell
pnpm dev:angular-shell
```
`implicitDependencies` in each shell's `project.json` defines which remotes are built before the shell starts.

**Serving a built remote for host consumption (preview mode):**
```sh
pnpm nx preview @jmf/blueprint-component-declarations
# Remote becomes available at http://localhost:XXXX
```
Note: Vite cannot serve MFE remotes from dev mode — you must use `preview` (built output) when wiring a remote into a host.

**Formatting / linting:**
```sh
pnpm nx format:check   # check Prettier
pnpm nx format:write   # auto-fix
pnpm nx run-many -t lint
```

**Testing:**
```sh
pnpm test                        # Vitest, affected only
pnpm nx test @jmf/blueprints     # single project
pnpm nx integration-tests @jmf/blueprints  # Playwright E2E
pnpm nx integration-tests-pact @jmf/blueprints  # Pact consumer contracts
```

## Shared Libraries and Utilities

- **`@jamf/mfe-blueprints-components`** — Shared React components for blueprint component MFEs (forms, filters, animations). Not a Feature Hub service — imported directly as an npm dependency.
- **`@jmf/react-remote-wrapper`** — React bootstrapping utilities and `AuthProvider` for standalone dev.
- **`@jmf/react-auth-provider`** — Auth0 provider for React apps in standalone/local dev.
- **`@jmf/mock-services`** — Mock implementations of all Feature Hub services for standalone dev.
- **`@jmf/localization`** — react-i18next bootstrapping; used by all React apps.
- **`@jmf/vite-css-injection`** / **`@jmf/webpack-css-injection`** — Injects scoped CSS into the `feature-app-container` element.
- **`@jamf/mfe-vite-remote-module-loader`** / **`@jamf/mfe-webpack-remote-module-loader`** — Dynamic remote module loading for Feature Hub.
- **`@jmf/viteconfig-react`** / **`@jmf/viteconfig-lib`** — Shared Vite configs for React apps and libs.
- **`@jmf/bundler-utils`** — Shared build helpers (`parseAppName`, `getDistPath`, etc.).
- **`@jmf/tsconfig`** — Shared TypeScript base configs.
- **`@jmf/eslint-config`** — Shared ESLint rules.
- **`@jmf/vite-plugin-nebula-rename`** — Vite plugin to resolve Nebula web component version mismatches when multiple versions exist in the same page.
- **CSRF libs** (`@jamf/mfe-csrf-axios/fetch/ky/urql/hey-api-client-fetch/core`) — CSRF token injection for each supported HTTP client. Required for apps using session-based (cookie) auth (e.g., Elevate/ascent).
- **`@jmf/deployment-shared`** — Shared CDK/deployment utilities.
- **`libs/automation`** — Nx generators for scaffolding new React or Vue remote apps.

## Gotchas

- **Vite remotes cannot be served from `dev` mode for host consumption.** Build and `preview` the remote; only then can it be loaded by URL into a shell.
- **`latest` symlink must exist locally.** When developing against a locally-built remote, run `pnpm nx link-latest @jmf/your-app` if the `latest` symlink is missing under `dist/apps/your-app/`.
- **`VITE_TYK_GATE_URL` is required.** Without it, API calls fail silently. Copy `.env.example.local` to `.env.local` at the root and fill in the gateway URL, org ID, and token before starting any app.
- **Per-app `.env` files.** Each app can also have its own `.env` (copied from `.env.example.sbox` or similar). The root `.env.local` is a fallback global override.
- **Feature Service not registered error.** If you see `"The required Feature Service 'jamf:auth_service' is not registered"`, the host shell's `createFeatureHub()` call is missing that service. Check the shell's service registration, not the remote.
- **CDN CloudFront caching.** Stale deployments can persist. Check `x-cache` and `last-modified` response headers (`curl -I 'https://cdn.mfe.jamf.io/my-app/stable/assets/remoteEntry.js'`). Contact Pandocats for manual CDN invalidation if needed.
- **Sandbox and Staging require Jamf Trust VPN.** Apps deployed there are not publicly accessible.
- **Never manually bump package versions.** CI uses git tags as the source of truth via `nx release`. Manual bumps will desync the versioning system.
- **Pact "Can I Deploy" failures.** If the CI Pact check fails, inspect results at `https://pactbroker.jamf.build/` — copy the verification path from the GitHub Action output and append it to the broker URL.
- **`syncpack` for dependency consistency.** Run `pnpm sync:list` to find cross-package version mismatches and `pnpm sync:fix` to resolve them. Version drift across ~50 apps is a known ongoing maintenance concern.
- **Angular shell uses Webpack; all other apps use Vite.** Do not mix bundler-specific config utilities between them.
- **`@nx/devkit>ejs` override.** The root `package.json` includes pnpm overrides for several transitive vulnerabilities (ejs, esbuild, tar, tmp, fast-xml-parser). These are intentional security pins — do not remove.
