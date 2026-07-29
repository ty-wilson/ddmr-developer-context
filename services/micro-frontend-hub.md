# Micro Frontend Hub

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the bundler outlier claim (now corrected, see below), where `pnpm` overrides live (they moved out of `package.json`), the JSFG loader namespace behaviour (**reversed** by PI-1337, see `json-schema-form-generator.md`), and the Feature Hub service-ID list (two new IDs the old table omitted). Not re-verified and older: the CDN/SemVer resolution rules, the local-dev command list, and the CI/CD flow. Treat those as a pointer.

**This repo maintains its own `CLAUDE.md` at its root, and that is the authoritative guide.** Read it, not this file, for structure, prerequisites, commands, and conventions. What is below is only the cross-repo context plus the traps that cost real debugging time. Run the commands in **Where to find the data** at the end before acting on any value here.

The micro-frontend-hub is the Nx (package mode) + pnpm monorepo that hosts every MFE app for the Blueprints/DDmR platform. All product UI lives here. MFEs are built as Module Federation remotes and loaded at runtime by a host shell via the Feature Hub pattern, which injects shared services (auth, routing, tenant, and so on) across framework boundaries.

## Monorepo shape

```
apps/    # MFE remotes + host shells (Angular, React/Vite, Vue). Each an independent deployable.
libs/    # Shared libraries and Feature Hub services, published to internal Artifactory
tools/cdk/    # AWS CDK infra (S3 + CloudFront CDN, SemVer-resolving Lambda)
tools/utils/  # Build/deployment helper scripts
```

- Required Node/pnpm versions are declared in the repo (`.nvmrc`, `package.json` `engines`/`packageManager`). Read them there rather than trusting a version quoted here.
- **Package naming:** `@jmf/` prefix for internal apps/libs (e.g. `@jmf/blueprints`). Shared service libs use `@jamf/mfe-` (e.g. `@jamf/mfe-auth-service`).
- Artifactory npm registry: `https://artifactory.jamf.build/artifactory/api/npm/npm-local/`.
- **Nx task graph:** `build` and `dev` both depend on `^build`, so lib deps build first. Nx caches `build`, `test`, and `type-check`; cache in `.nx-cache/`.

Do not read an app inventory or a library list out of this file. `ls apps/` and `ls libs/` are one command away and both directories change weekly (see the verification section). The parts worth knowing:

- `blueprints` is the host that mounts the `blueprint-component-*` remotes. There are many of those, one per Blueprint step.
- `angular-shell` / `react-vite-shell` / `vue-shell` are host shells used for local development against a real Feature Hub.
- `json-schema-form-generator` is a remote loaded *by* other remotes, not by the shell. It is also the name of a separate sibling repo; see the trap in `docs/frontend.md` before reasoning about it.
- `demo-{react,vue,angular,vanilla}-remote` are reference implementations, not products.

## Shell integration

All MFEs integrate via **Feature Hub** (`@feature-hub/core`, `@feature-hub/dom`, `@feature-hub/react`).

- **Host shells** call `createFeatureHub()` to instantiate services and provide them to the Feature Hub Integrator. The `<feature-app-loader>` web component loads remotes by URL.
- **Remote apps** declare required/optional services in `feature.ts`/`feature.tsx` (the Module Federation entry point). The host injects services at mount time; remotes access them through `featureAppManager.getFeatureAppScope()` bindings.
- **Module Federation:** remote entry is exposed as `remoteEntry.js`. Vite remotes place assets under `/assets/`; Webpack remotes place them at root.
- **CSS scoping:** remotes attach styles to the `feature-app-container` web component (shadow DOM) via `@jmf/vite-css-injection` / `@jmf/webpack-css-injection` rather than `<head>`.
- **Design system:** Jamf Nebula web components (`@jamf/design-system-web-components-next`) plus shared tokens (`@jamf/design-system-shared`).

Feature Hub services are discovered by **string ID at runtime**, which means a typo is a runtime failure rather than a build failure. IDs follow `jamf:<name>_service` (mostly underscored, but `jamf:blueprints-service` and `jamf:oidc-sso-service` are hyphenated, so do not assume the separator). Regenerate the current ID list with the grep in the verification section rather than reading a table here; the set grows, and IDs added after this review include `jamf:division_service` and `jamf:oidc-sso-service`.

## Build, deploy, and CDN

**Environments:** `sbox`, `dev`, `stage`, `prod`. Sandbox and Staging are reachable only over Jamf Trust VPN.

**CDN:** built assets go to S3 + CloudFront at `cdn.mfe.jamf.io`, fronted by a SemVer-resolving Lambda, so a partial version is a valid URL:

- `/my-app/1.2.1` exact version
- `/my-app/1.2` latest patch for that minor
- `/my-app/1` latest minor and patch for that major
- `/my-app/latest` most recently deployed version
- `/my-app/stable` manually promoted release via GitHub Releases

**CI/CD shape** (workflow filenames drift; read `.github/workflows/` for the current ones):

1. A PR carrying the `deploy:affected` label deploys to Sandbox.
2. Merge to `main` triggers a version-bump workflow that bumps `package.json` versions, tags, and pushes a bump commit.
3. That bump commit, not the merge, triggers the deploy workflow: Dev, then Staging, then Prod.
4. Prod approval can be made manual per app by adding a `require_manual_prod_approval` tag to its `project.json`.
5. Merges go through the GitHub Merge Queue; direct merge is not available.

**Versioning:** Nx `nx release` with `specifierSource: conventional-commits` and `currentVersionResolver: git-tag`.

## Traps

**Trap: never bump `package.json` versions by hand.** `nx release` resolves the current version from **git tags**, so a manual bump desyncs the version source of truth from the tags and the next automated bump computes the wrong number. Let CI do it.

**Trap: a Vite remote cannot be served from `dev` mode for host consumption.** `dev` does not emit a loadable `remoteEntry.js`, so a shell pointed at a dev server gets a remote that never mounts, usually with no clear error. Build the remote and `nx preview` it, then wire that URL into the host. Observed against `@module-federation/vite` 1.7.1 (the version pinned in `apps/blueprint-component-declarations/package.json` as of 2026-07-29); re-check if that major moves, since dev-mode federation support is exactly the kind of thing it would add.

**Trap: the `latest` symlink must exist locally.** When a host loads a locally-built remote and gets a 404, the symlink under `dist/apps/<your-app>/` is usually missing. `pnpm nx link-latest @jmf/your-app` creates it.

**Trap: `VITE_TYK_GATE_URL` missing makes API calls fail silently.** Copy `.env.example.local` to `.env.local` at the root and fill in gateway URL, org ID, and token before starting any app. Each app can also carry its own `.env` (from `.env.example.sbox` or similar); the root `.env.local` is the fallback global override, so a stale per-app `.env` wins over a correct root one.

**Trap: `"The required Feature Service 'jamf:auth_service' is not registered"` points at the host, not the remote.** The remote declared the dependency correctly; the shell's `createFeatureHub()` call is missing that service. Fix the shell's service registration.

**Trap: CloudFront serves stale deployments.** Check `x-cache` and `last-modified` on the remote entry (`curl -I 'https://cdn.mfe.jamf.io/my-app/stable/assets/remoteEntry.js'`) before concluding a deploy did not land. Manual CDN invalidation is a Pandocats request.

**Trap: cross-package dependency versions drift, and nothing fails loudly.** Two apps resolving different versions of the same Nebula or React package on one page is a runtime-only failure. `pnpm sync:list` reports mismatches and `pnpm sync:fix` resolves them (syncpack). `@jmf/vite-plugin-nebula-rename` exists specifically to survive multiple Nebula versions coexisting in one page, which is a symptom of this drift rather than a fix for it. Deliberately no app count here: the number is `ls apps/ | wc -l` and it moves.

**Confirm your app's bundler in its own config before reusing build utilities.** The Angular shell is the Webpack outlier, but it is *not* the only one: `demo-angular-remote` is also Webpack, so "everything except angular-shell is Vite" is wrong and there is nothing stopping a new Angular remote from being added. Check for a `webpack.config.js` in the app directory (grep in the verification section) rather than inferring from framework.

**Trap: `pnpm` overrides are not in `package.json` any more.** They live under `overrides:` in **`pnpm-workspace.yaml`**, alongside `blockExoticSubdeps` and `minimumReleaseAge`. Several are deliberate transitive-vulnerability pins (ejs, esbuild, tar, tmp, ws, glob at the time of review); do not remove one because a lockfile suggests it is redundant. Read the live block from `pnpm-workspace.yaml` rather than any list here, because the set changes with each advisory.

**Pact "Can I Deploy" failures:** the GitHub Action output contains a verification path. Append it to `https://pactbroker.jamf.build/` to see which contract broke.

**Sandbox and Staging require Jamf Trust VPN.** Apps deployed there are not publicly reachable, so a connection failure from outside the VPN is not a deploy failure.

## Where to find the data (verify rather than trust)

Read the repo's own `CLAUDE.md` first. Everything below regenerates content that used to be transcribed into this file.

```bash
R=~/Projects/DDmR/micro-frontend-hub; git -C $R fetch origin -q
git -C $R log --oneline origin/main --since=2026-07-29
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# App and library inventory, plus how many there actually are right now
git -C $R ls-tree --name-only origin/main apps/ libs/

# Which apps are NOT Vite (the Webpack outliers). Do not assume this is only angular-shell.
git -C $R ls-tree -r --name-only origin/main | grep -E 'apps/[^/]+/webpack'

# Current Feature Hub service IDs (regenerates the service table)
git -C $R grep -oh "'jamf:[a-z_-]*'" origin/main -- 'libs/*/src/**' | sort -u

# Security pins and workspace-level dependency policy (NOT in package.json)
git -C $R show origin/main:pnpm-workspace.yaml

# Node/pnpm versions, as declared rather than as remembered
git -C $R show origin/main:package.json | python3 -c 'import json,sys;d=json.load(sys.stdin);print(d["engines"],d["packageManager"])'
git -C $R show origin/main:.nvmrc
```

Before concluding an app, lib, or service ID does not exist, check unmerged work. This repo has many concurrent branches and also carries in-repo worktrees under `.claude/worktrees/`, which a plain `grep -r` will hit and misattribute as `main`:

```bash
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -30
```
