# MDM Schema Ingest Pipeline

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** Goldminers team

## Summary

The MDM Schema Ingest Pipeline is a fully event-driven, serverless system that tracks Apple's public `device-management` GitHub repo, transforms its YAML payload definitions into JSON Schema, and serves the results over an internal HTTPS endpoint. It is owned by the Goldminers team. The Lambda code lives in `mdm-schema-ingest-inbound-adapter` (a TypeScript/Node.js npm-workspaces monorepo); the AWS infrastructure is declared in `mdm-schema-ingest-infrastructure` using Terraform + Terragrunt. Schemas are written to versioned S3 prefixes and served through an internal ALB backed by a path-mapping Lambda. Deployed to three dedicated AWS accounts: dev (`380922964950`), stage (`369976064392`), prod (`984688522419`), all in `us-east-2`.

---

## Pipeline Stages

### 1. Ingest (`ingest/`)

An EventBridge scheduler fires the `mdm-git-ingest` Lambda hourly (prod and dev: `rate(1 hours)`; stage: `rate(30 minutes)`). The Lambda:

1. Checks for an initialized git working tree for `https://github.com/apple/device-management.git` on an EFS volume (mount point `$EFS_MOUNT_POINT`, e.g. `/mnt/lambda/`). If absent, initializes a new git repo with `git init` and adds `origin` as a remote — no initial clone is performed at this step.
2. Runs `git fetch --prune`. Any branch refs returned (new or updated) are merged into the working tree so one version is generated per scan.
3. Copies the merged tree into a versioned snapshot directory named `{repo-name}-{version}` on EFS.
4. Reads the previous version integer from SSM Parameter Store (`/mdm_schema/version/{repo-name}`), creates the parameter if it does not exist (starts at `1`), then increments and saves the new version.
5. On any failure the `origin` remote is removed and re-added with the same URL, which clears all remote-tracking references so that git will re-detect the same commits on the next scan, effectively retrying automatically.

The Lambda requires a `git` binary as a Lambda layer (`git-binary-layer`) extracted from the AWS Lambda Amazon Linux image; `simple-git` (npm) and `js-yaml` are provided via separate Node.js layers.

### 2. Transformation (`transformation/`)

A Parameter Store `Create`/`Update` event fires an EventBridge rule, which invokes the `mdm-schema-transformation` Lambda.

1. Reads the current version from Parameter Store and locates the corresponding EFS snapshot directory.
2. Iterates every `.yaml` file in the versioned EFS snapshot directory recursively (profiles under `mdm/profiles/` have full support; other MDM component types are best-effort).
3. Converts each YAML schema to JSON Schema using `js-yaml` plus custom mapping logic in `schema-transformation.ts` and `subkeys-transformation.ts`.
4. Clones `mdm-jamf-schema` (a private GitHub repo) to Lambda `/tmp` using a GitHub App (app ID `1087011`, installation `58361391`, private key from Parameter Store). For each generated schema, looks up the corresponding file at the same relative path (e.g. `mdm/profiles/com.apple.wifi.managed.json`) and deep-merges any Jamf-specific metadata (sensitive flags, validation rules, business metadata) into the JSON Schema. Clones the jamf schema repo once per invocation; failure to clone aborts the entire transformation.
5. Uploads all enhanced schemas to the JSON Schema S3 bucket under `device-management/{version}/mdm/profiles/{payload-type}.json`.
6. Deletes the versioned EFS snapshot directory.
7. Emits a `custom.transformation-lambda` event to EventBridge to fan out to the three downstream lambdas.

### 3. Post-Transformation (parallel fan-out)

All three lambdas are triggered simultaneously by the `schema-transformed` EventBridge rule.

**Archive (`archive/`)** — Downloads every schema from the JSON Schema S3 bucket and commits them to the `json-schema-mirror` git repo, tagged with the schema version number.

**Java Enhancements (`java-enhancements/`)** — Downloads schemas from the JSON Schema bucket, adds `javaName` and `existingJavaType` properties needed by `jsonschema2pojo` for DTO generation (with a special rename for the conflicting `APNsItem` in the Cellular payload), then uploads to a separate Java schema S3 bucket under the same versioned path structure.

**Translations (`translations/`)** — Downloads schemas from the JSON Schema bucket, generates translation-schema files containing localised titles and descriptions for each key, then pushes results to the `mdm-schema-translations` git repo.

---

## S3 Bucket Layout

Three private S3 buckets with versioning enabled, all in `us-east-2`:

| Bucket variable | Purpose | Key pattern |
|---|---|---|
| `json_bucket_name` | Primary JSON Schema output | `device-management/{version}/mdm/profiles/{payload-type}.json` |
| `translations_bucket_name` | Localised translation schemas + metadata | `device-management/{version}/{locale}/metadata.json`, `device-management/{version}/{locale}/...` |
| `ui_schema_bucket_name` | Hand-authored UI overrides from `mdm-ui-schema` | `device-management/{version}/mdm/profiles/{payload-type}.json` |

The `mdm-ui-schema` repo (`device-management/mdm/profiles/`) holds manually curated UI schema JSON files. Changes there are pushed into the UI Schema S3 bucket independently via a separate GitHub Actions workflow (OIDC role: `ui-schema-repo-role`).

---

## Serving Layer (ALB + Path-Mapping Lambda)

A public-certificate, HTTPS-only internal ALB (`mdm-schema-load-balancer`) sits behind a Route 53 A record. Dev and stage use `api.{env}.mdm-schema.jamf.build`; prod uses per-region subdomains under `mdm-schema.platform.jamfapps.io` (e.g. `use2.mdm-schema.platform.jamfapps.io`). Access is restricted to CloudWAN trust CIDRs and any explicitly configured CIDR blocks; there is no public internet access. The ALB forwards all traffic to the `path-mapping` Lambda (Node.js 24.x), which translates URL paths to S3 `GetObject` calls.

**URL patterns handled by the path-mapping Lambda:**

| Pattern | Bucket | S3 key template |
|---|---|---|
| `GET /v1/metadata/{version}/{payload-type}` | JSON Schema | `device-management/{version}/mdm/profiles/{payload-type}.json` |
| `GET /v1/declarative/{version}/declarationbase` | JSON Schema | `device-management/{version}/declarative/declarations/declarationbase.json` |
| `GET /v1/declarative/{version}/{category}/{...}` | JSON Schema | `device-management/{version}/declarative/declarations/{category}/{...}.json` (allowed categories: `activations`, `assets`, `configurations`, `management`) |
| `GET /v1/ui-schema/{version}/{payload-type}` | UI Schema | `device-management/{version}/mdm/profiles/{payload-type}.json` (returns `{}` on 404) |
| `GET /v1/translations/{version}/{payload-type}/{locale}[/{repo}]` | Translations | Resolved via `{repo}/{version}/{locale}/metadata.json` then the path from the `payloadTypes` map |
| `GET /health` | — | 200 health check |

The Lambda returns CORS headers (`Access-Control-Allow-Origin: *`) on all responses and supports `OPTIONS` pre-flight requests.

---

## How Frontends Consume the API

Frontends call the internal ALB endpoint directly using the environment-specific domain:

- dev: `https://api.dev.mdm-schema.jamf.build/v1/...`
- stage: `https://api.stage.mdm-schema.jamf.build/v1/...`
- prod (per region): `https://use2.mdm-schema.platform.jamfapps.io/v1/...`, `https://euc1.mdm-schema.platform.jamfapps.io/v1/...`, `https://apne1.mdm-schema.platform.jamfapps.io/v1/...`

The `API_VERSION` (currently `v1`) is baked into the path-mapping Lambda as an environment variable and all URL patterns are prefixed with it. Frontends do not call S3 directly.

---

## Known Consumers

- `configuration-profile-service` — uses schemas from the Java schema S3 bucket at build time via `jsonschema2pojo` for DTO generation. The `javaName`/`existingJavaType` properties added by the Java Enhancements Lambda are specifically there to support this.
- `configuration-profile-plist-migrator` — calls the `mdm-schema-internal` endpoint at runtime (via the internal ALB) to look up schema definitions during migration processing.

---

## Gotchas

- **Git binary layer is manually maintained.** The `git-binary-layer` ZIP is not built by the CI pipeline; it must be manually extracted from the AWS Lambda Amazon Linux image following the Dockerfile pattern in the README whenever the git version needs to be bumped.
- **EFS is the ingest/transformation handoff.** The two Lambdas share state through EFS, not through an event payload. If the ingest Lambda succeeds but the transformation Lambda fails before deleting the EFS snapshot, the snapshot will be left behind. The next successful run cleans it up.
- **Transformation is all-or-nothing for Jamf enhancements.** If `mdm-jamf-schema` cannot be cloned (network error, auth failure, invalid JSON in any jamf schema file), the entire transformation invocation fails. Missing jamf schema files for individual payloads are acceptable and silently skipped.
- **Only profiles are fully validated.** Declarative DDM schemas and other MDM payload types are ingested and converted but were not fully validated against the Apple YAML source during initial development.
- **Infrastructure changes to `modules/` do not auto-trigger CI.** Changes to Terraform module files without a corresponding `terragrunt/` change require a manual workflow trigger in `mdm-schema-ingest-infrastructure`.
- **Terraform state lock races.** Simultaneous branch discoveries can cause DynamoDB lock contention on the Terraform state table. The fix is to wait and re-run the failed pipeline.
- **UI schemas are a separate publish path.** The `mdm-ui-schema` repo pushes directly to the UI Schema S3 bucket via its own GitHub Actions workflow. The ingest/transformation pipeline does not touch that bucket.
- **Versioning is per-repo.** The version integer in Parameter Store is namespaced by repository name (`/mdm_schema/version/{repo-name}`), which means E2E tests can run the full pipeline against a different source repo without colliding with the production version counter.
