# MDM Schema Ingest Pipeline

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the Lambda schedule declarations in `mdm-schema-ingest-infrastructure`, the `API_VERSION` wiring and route table in `path-mapping-lambda.js`, the SSM parameter path and its JSON shape in `ingest/src/version-handler.ts`, and (in `tyk-gateway-management`) that `/v1/version` is a gateway-level route to configuration-profile-service rather than to this pipeline's ALB. Not re-verified and older (2026-04-07): the AWS account IDs, the EFS handoff details, the post-transformation fan-out internals, and the git-binary-layer note.

Values here drift. See `## Where to find the data` at the bottom.

**Owner:** Goldminers team

## Summary

The MDM Schema Ingest Pipeline is a fully event-driven, serverless system that tracks Apple's public `device-management` GitHub repo, transforms its YAML payload definitions into JSON Schema, and serves the results over an internal HTTPS endpoint. The Lambda code lives in `mdm-schema-ingest-inbound-adapter` (a TypeScript/Node.js npm-workspaces monorepo); the AWS infrastructure is declared in `mdm-schema-ingest-infrastructure` using Terraform + Terragrunt. Schemas are written to versioned S3 prefixes and served through an internal ALB backed by a path-mapping Lambda. Deployed to three dedicated AWS accounts: dev (`380922964950`), stage (`369976064392`), prod (`984688522419`), all in `us-east-2`.

---

## Pipeline Stages

### 1. Ingest (`ingest/`)

An EventBridge scheduler fires the `mdm-git-ingest` Lambda on a per-environment cadence. **The schedule is not the same in every environment and has changed at least once**, so read it from Terragrunt rather than from any doc: `modules/event-bridge/vars.tf` declares `ingest_schedule_expression` with a module default, and each `terragrunt/{dev,stage,prod}/global/mdm-ingest/event-bridge/terragrunt.hcl` overrides it. The Lambda:

1. Checks for an initialized git working tree for `https://github.com/apple/device-management.git` on an EFS volume (mount point `$EFS_MOUNT_POINT`, e.g. `/mnt/lambda/`). If absent, initializes a new git repo with `git init` and adds `origin` as a remote. No initial clone happens at this step.
2. Runs `git fetch --prune`. Any branch refs returned (new or updated) are merged into the working tree so one version is generated per scan.
3. Copies the merged tree into a versioned snapshot directory named `{repo-name}-{version}` on EFS.
4. Reads the previous version from SSM Parameter Store at `/mdm_schema/version/{repo-name}`, creates the parameter if absent, then increments and saves. **The parameter value is a JSON `SchemaVersion` object, not a bare integer**: it carries `version`, `date`, `commit`, and `message`, so consumers must parse it (see the `jq -r '.version'` in the verification section).
5. Also emits a CloudWatch metric `SchemaVersion` in namespace `MDM/Schema`, dimensioned by `Repository`. This is the cheapest way to see the ingested version over time without SSM access.
6. On any failure the `origin` remote is removed and re-added with the same URL, which clears all remote-tracking references so git re-detects the same commits on the next scan, effectively retrying automatically.

The Lambda requires a `git` binary as a Lambda layer (`git-binary-layer`) extracted from the AWS Lambda Amazon Linux image; `simple-git` (npm) and `js-yaml` are provided via separate Node.js layers.

### 2. Transformation (`transformation/`)

A Parameter Store `Create`/`Update` event fires an EventBridge rule, which invokes the `mdm-schema-transformation` Lambda.

1. Reads the current version from Parameter Store and locates the corresponding EFS snapshot directory.
2. Iterates every `.yaml` file in that snapshot directory recursively.
3. Converts each YAML schema to JSON Schema using `js-yaml` plus custom mapping logic in `schema-transformation.ts` and `subkeys-transformation.ts`.
4. Clones `mdm-jamf-schema` (a private GitHub repo) to Lambda `/tmp` using a GitHub App (private key from Parameter Store). For each generated schema, looks up the corresponding file at the same relative path (e.g. `mdm/profiles/com.apple.wifi.managed.json`) and deep-merges any Jamf-specific metadata (sensitive flags, validation rules, business metadata) into the JSON Schema. The jamf schema repo is cloned once per invocation; failure to clone aborts the entire transformation.
5. Uploads all enhanced schemas to the JSON Schema S3 bucket under `device-management/{version}/mdm/profiles/{payload-type}.json`.
6. Deletes the versioned EFS snapshot directory.
7. Emits a `custom.transformation-lambda` event to EventBridge to fan out to the three downstream lambdas.

### 3. Post-Transformation (parallel fan-out)

All three lambdas are triggered simultaneously by the `schema-transformed` EventBridge rule.

**Archive (`archive/`)** downloads every schema from the JSON Schema S3 bucket and commits them to the `json-schema-mirror` git repo, tagged with the schema version number.

**Java Enhancements (`java-enhancements/`)** downloads schemas from the JSON Schema bucket, adds `javaName` and `existingJavaType` properties needed by `jsonschema2pojo` for DTO generation (with a special rename for the conflicting `APNsItem` in the Cellular payload), then uploads to a separate Java schema S3 bucket under the same versioned path structure.

**Translations (`translations/`)** downloads schemas from the JSON Schema bucket, generates translation-schema files containing localised titles and descriptions for each key, then pushes results to the `mdm-schema-translations` git repo. Only the `en-US` base is auto-generated; every other locale and all Jamf label overrides are hand-authored in that repo (see `docs/frontend.md`).

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

A public-certificate, HTTPS-only internal ALB (`mdm-schema-load-balancer`) sits behind a Route 53 A record. Dev and stage use `api.{env}.mdm-schema.jamf.build`; prod uses per-region subdomains under `mdm-schema.platform.jamfapps.io` (e.g. `use2.mdm-schema.platform.jamfapps.io`). Access is restricted to CloudWAN trust CIDRs plus any explicitly configured CIDR blocks; there is no public internet access. The ALB forwards all traffic to the `path-mapping` Lambda (Node.js), which translates URL paths to S3 `GetObject` calls.

The route table, the allowed declarative categories, and the 403/404/405 behavior are all one file: `modules/path-mapping-lambda/path-mapping-lambda.js`. Regenerate the current routes instead of trusting a copy:

```bash
git -C ~/Projects/DDmR/mdm-schema-ingest-infrastructure grep -n 'matchPath(path' \
  origin/main -- modules/path-mapping-lambda/path-mapping-lambda.js
```

Shape of the mapping (each route resolves to one bucket + key template):

| Pattern | Bucket |
|---|---|
| `/{API_VERSION}/metadata/{version}/{payload-type}` | JSON Schema |
| `/{API_VERSION}/declarative/{version}/{category}/{...}` | JSON Schema (`declarative/declarations/...`) |
| `/{API_VERSION}/ui-schema/{version}/{payload-type}` | UI Schema (returns `{}` on 404) |
| `/{API_VERSION}/translations/{version}/{payload-type}/{locale}[/{repo}]` | Translations, resolved via `{repo}/{version}/{locale}/metadata.json` |
| `/{API_VERSION}/other/{...}` | JSON Schema |
| `/health` | none, 200 health check |

`API_VERSION` is not hardcoded in the path patterns: it is a Lambda environment variable set from the Terraform variable `api_version` (`modules/path-mapping-lambda/main.tf` and `vars.tf`) and interpolated into every route regex. Read the current value from Terragrunt rather than assuming. Any path that matches no pattern returns **403 `Access to this path is forbidden`**, not 404, which is a common source of "the schema is missing" misdiagnoses. Non-GET, non-OPTIONS methods return 405. CORS headers (`Access-Control-Allow-Origin: *`) are returned on all responses and `OPTIONS` pre-flight is supported.

---

## Trap: there are two version notions and they can disagree

The pipeline owns only one of them.

- **"Latest ingested" version**: the integer inside the SSM parameter `/mdm_schema/version/device-management`. This is the version new schema, UI-schema, and translation content is uploaded *under*.
- **"Last supported" version**: what `GET /v1/version` returns to callers, and what the MFEs actually request.

**`GET /v1/version` is not served by this pipeline at all.** In `tyk-gateway-management`, both the `mdm-schema` and `mdm-schema-internal` API products define a *separate* Tyk API whose `listen_path` is `/mdm-schema/v1/version` (respectively `/mdm-schema-internal/v1/version`) with `target_url` pointing at **configuration-profile-service** `/v1/mdm-schema/version`. CPS answers from `PropertyFileMdmSchemaProvider.getLatestSupported()`, backed by the `schema.version` Spring property or the baked-in `supported-schema.txt` resource (build-time value in CPS `gradle.properties` as `mdm.schema.version`).

Consequences worth internalising:

- The supported version **lags** the ingested one, by design. Freshly-ingested schemas exist at a version no UI is asking for yet.
- Bumping the ingest version does not make the new schemas reachable through the MFEs. CPS has to be rebuilt or reconfigured.
- Someone debugging "the pipeline ingested it but the UI does not see it" will get nowhere reading this repo. The lag is a CPS deploy question.
- The path-mapping Lambda will 403 (not 404) on `/v1/version` if the Tyk-level route is bypassed and the request reaches the ALB directly.

See `docs/frontend.md` for the consumer-side view and `services/configuration-profile-service.md` for the CPS side.

---

## Known Consumers

- `configuration-profile-service` (Goldminers) uses schemas from the Java schema S3 bucket at build time via `jsonschema2pojo` for DTO generation. The `javaName`/`existingJavaType` properties added by the Java Enhancements Lambda exist specifically to support this. CPS also serves the "last supported version" endpoint described above.
- `configuration-profile-plist-migrator` (Goldminers) calls the `mdm-schema-internal` endpoint at runtime (via Tyk to the internal ALB) to look up schema definitions and defaults during migration processing.
- `blueprint-component-configuration-profiles` MFE fetches transformed JSON Schema, UI schema, and translations.
- `blueprint-component-declarations` MFE fetches the raw Apple declarative schemas from `/v1/declarative/...` and transforms them client-side.

---

## Gotchas

- **Git binary layer is manually maintained.** The `git-binary-layer` ZIP is not built by CI; it must be manually extracted from the AWS Lambda Amazon Linux image following the Dockerfile pattern in the README whenever the git version needs a bump.
- **EFS is the ingest/transformation handoff.** The two Lambdas share state through EFS, not through an event payload. If ingest succeeds but transformation fails before deleting the EFS snapshot, the snapshot is left behind. The next successful run cleans it up.
- **Transformation is all-or-nothing for Jamf enhancements.** If `mdm-jamf-schema` cannot be cloned (network error, auth failure, invalid JSON in any jamf schema file), the whole transformation invocation fails. Missing jamf schema files for individual payloads are acceptable and silently skipped.
- **Profiles got the validation attention; other payload types did not.** During initial development, only schemas under `mdm/profiles/` were validated against the Apple YAML source. Declarative DDM schemas and other MDM component types are ingested and converted, but were never validated to the same standard at that time. Treat this as provenance about how the pipeline was built, not as a statement about today's correctness; if a declarative schema looks wrong, compare the served JSON against the Apple YAML directly.
- **Infrastructure changes to `modules/` do not auto-trigger CI.** Changes to Terraform module files without a corresponding `terragrunt/` change require a manual workflow trigger in `mdm-schema-ingest-infrastructure`.
- **Terraform state lock races.** Simultaneous branch discoveries can cause DynamoDB lock contention on the Terraform state table. The fix is to wait and re-run the failed pipeline.
- **UI schemas are a separate publish path.** The `mdm-ui-schema` repo pushes directly to the UI Schema S3 bucket via its own GitHub Actions workflow. The ingest/transformation pipeline does not touch that bucket.
- **Versioning is per-repo.** The version in Parameter Store is namespaced by repository name (`/mdm_schema/version/{repo-name}`), so E2E tests can run the full pipeline against a different source repo without colliding with the production version counter.

---

## Where to find the data (verify rather than trust)

```bash
INF=~/Projects/DDmR/mdm-schema-ingest-infrastructure
ADP=~/Projects/DDmR/mdm-schema-ingest-inbound-adapter
git -C $INF fetch origin -q; git -C $ADP fetch origin -q

# What changed since this doc was last reviewed
git -C $INF log --oneline origin/main --since=2026-07-29
git -C $ADP log --oneline origin/main --since=2026-07-29

# Unmerged work, before concluding something is not implemented
git -C $INF for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Actual per-environment ingest schedule and API_VERSION (never quote these from a doc)
git -C $INF grep -rn 'ingest_schedule_expression\|api_version' origin/main \
  -- modules/ terragrunt/

# Current serving routes, allowed declarative categories, and the 403-not-404 behavior
git -C $INF show origin/main:modules/path-mapping-lambda/path-mapping-lambda.js

# Latest INGESTED version (JSON value: parse it, don't read it raw)
aws ssm get-parameter --name /mdm_schema/version/device-management \
  --query Parameter.Value --output text | jq -r '.version'

# Last SUPPORTED version (answered by configuration-profile-service, not this pipeline)
git -C ~/Projects/DDmR/tyk-gateway-management grep -rn -B2 -A2 '/v1/version' \
  master -- 'dev/api-products/mdm-schema*/api-definitions.yaml'
```
