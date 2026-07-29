# Infrastructure

Last reviewed: 2026-07-29. Re-verified against both repos on that date: the `grunt/<env>/{dynamodb,roles,policies}/<service>/` and `infrastructure/aws/<partition>/<env>/<account>_<id>/<region>/<resource>/` directory shapes, the staging-vs-HC service asymmetry, the `ddmr-integration-*` table set, and where PITR is actually declared (the previous claim that PITR covered staging was wrong; see below). NOT re-verified: the account/region tables, the mdm-schema-ingest accounts, and the tag conventions, which date from 2026-06-24. Identifiers are recorded here because they are expensive to rediscover, not because they were re-confirmed.

## Two Infrastructure Repositories

DDmR infrastructure is split across two Terraform/Terragrunt repositories with different lineages and purposes.

### ddmr-terraform (legacy, Jenkins-based)

`github.com/jamf/ddmr-terraform` manages the commercial AWS accounts (dev, sandbox, staging, production). It predates the Highway-to-Prod rollout and is deployed via a **Jenkins pipeline** that:

- Runs parameterized plan stages for each environment/component combination
- Requires manual approval before apply on the main branch
- Runs inside Kubernetes with dedicated service accounts

This repo owns the canonical table-definition YAML files and IAM policy JSON files that are shared across environments. The `grunt/` directory is organized as `grunt/{environment}/{resource-type}/{service}/`, where resource-type is `dynamodb`, `roles`, or `policies`.

### ddmr-infrastructure (newer, OpenTofu + GitHub Actions + HC)

`github.com/jamf/ddmr-infrastructure` was stood up to manage the **High Compliance (HC)** AWS account. It uses:

- **OpenTofu** (binary: `tofu`) with Terragrunt
- **GitHub Actions** → Highway-to-Prod webhook → Argo for deployment
- A strict directory hierarchy: `infrastructure/aws/{partition}/{env}/{account_name}_{account_id}/{region}/{resource}/terragrunt.hcl`

The `hc/` partition holds real provisioned infrastructure. The `commercial/` partition (`sbox`, `stage`) is scaffold with placeholder account IDs, and may not be present on your branch at all.

**Trap: a plan under `commercial/` can look valid while targeting nothing.** The scaffold carries placeholder account IDs, so `tofu plan` renders a plausible diff against an account that is not the real commercial account. Confirm the account ID in the directory name against the topology table below before you trust any plan or apply there. Enumerate what is genuinely provisioned with `find infrastructure/aws -name terragrunt.hcl`.

Which repo to use: HC resources go in `ddmr-infrastructure`. Commercial-environment changes (dev/sandbox/staging/production) go in `ddmr-terraform`.

---

## AWS Account Topology

| Environment | Account ID   | Region(s)                                    | Repo            |
|-------------|--------------|----------------------------------------------|-----------------|
| dev         | 381491946762 | us-east-2                                    | ddmr-terraform  |
| sandbox     | 183197288009 | us-east-1                                    | ddmr-terraform  |
| staging     | 535838898545 | us-east-1                                    | ddmr-terraform  |
| production  | 613358915025 | us-east-1, eu-central-1, ap-northeast-1      | ddmr-terraform  |
| HC stage    | 604006981984 | us-east-2                                    | ddmr-infrastructure |

Production is the only multi-region deployment in `ddmr-terraform`.

---

## Resources Managed

### DynamoDB Tables

Table schemas are defined as YAML files in `grunt/` (ddmr-terraform) and `infrastructure/aws/module_vars/` (ddmr-infrastructure). All tables use a single-table design with string `pkey`/`psort` (or `pk`) keys. Per-environment table names follow the pattern `ddmr-{env}-{service}`.

**Trap: production drops the env segment.** The prod tables are `ddmr-declaration-storage`, `ddmr-scoping-engine`, and `ddmr-tenant-authorizer` (account 613358915025, us-east-1). A script that interpolates `ddmr-production-...` from the pattern silently targets a nonexistent table.

The `tenant_index` GSI that used to exist on the declaration-storage and scoping-engine tables was removed in DDMR-1035 (2026-04-02) and is now absent from Terraform and every live table. Queries written against it will fail rather than fall back.

| Service                   | Table name (staging example)              | GSIs                                      |
|---------------------------|-------------------------------------------|-------------------------------------------|
| declaration-storage-service | ddmr-staging-declaration-storage        | `declaration_index` (declaration_key)     |
| scoping-engine            | ddmr-staging-scoping-engine               | `group_index` (group_key)                 |
| tenant-authorizer         | ddmr-tenant-authorizer                    | `claimTenantMigration_index`              |

Canonical key patterns and GSI details live in `database.md`.

**Trap: PITR is not uniform across environments.** Point-in-time recovery is set per table in the per-environment `dynamodb/terragrunt.hcl` (and in `module_vars/dynamo-db.hcl` for HC), not globally. It is present in some environments and absent in others, so do not assume you can restore. Check the live table before you need to:

```bash
aws dynamodb describe-continuous-backups --table-name <table>
```

Staging also provisions a parallel `ddmr-integration-*` table set alongside the main tables, so integration test runs write to their own copies rather than to the staging tables. The names are declared in `grunt/staging/dynamodb/terragrunt.hcl`; grep there rather than assuming the set matches the main table list.

### S3 Buckets

S3 resources are minimal. The sandbox environment has a single bucket:

- `ddmr-performance-test-results` — stores results from performance/load tests, AES256 encrypted

State backend buckets (one per environment/region) are managed outside these repos.

### IAM Roles

Each service gets an OIDC-assumable IAM role so its Kubernetes pod can access AWS resources without long-lived credentials. Roles are created using `terraform-aws-modules/terraform-aws-iam//modules/iam-assumable-role-with-oidc`.

**The durable fact is the asymmetry: HC provisions a narrower service set than commercial staging.** Staging carries roles for the tenant-authorizer and tenant-migration workloads on top of the three core services; HC stage does not. So a service that resolves its role fine in staging can fail to assume anything in HC, and "it works in stage" is not evidence for HC. Enumerate both sides rather than trusting a list:

```bash
ls ddmr-terraform/grunt/*/roles/
ls ddmr-infrastructure/infrastructure/aws/hc/*/*/*/roles/
```

The same two paths with `policies/` instead of `roles/` enumerate the policy side, which tracks the role set.

### IAM Policies

Each service has a corresponding policy granting it access to its own DynamoDB table and Secrets Manager secrets. Policy JSON templates live at the top level of `grunt/` as `{service}-serviceaccount-policy.json`.

**Trap: `declaration-service` breaks the policy-file naming pattern.** Its file is `declaration-serviceaccount-policy.json`, with no `-service` before `-serviceaccount`. A generated or globbed filename will miss it.

Typical grants, using scoping-engine as the example:

- DynamoDB: `BatchGetItem`, `BatchWriteItem`, `ConditionCheckItem`, `PutItem`, `DescribeTable`, `DeleteItem`, `GetItem`, `Scan`, `Query`, `UpdateItem`
- Secrets Manager: `GetSecretValue`, scoped to `arn:aws:secretsmanager:*:{account}:secret:ddmr/{env}/scoping/*`

---

## OIDC IAM Trust Pattern

All service IAM roles use EKS OIDC federation as their trust mechanism. The OIDC provider URL corresponds to the EKS cluster in each account/region. Roles are scoped to specific Kubernetes service accounts via `oidc_fully_qualified_subjects`.

The subject pattern is `system:serviceaccount:{k8s-namespace}:{service}-acct`. The namespace is `ddmr-stage` for staging and HC, `ddmr-integration` for integration test runs.

Staging example for scoping-engine:

```
system:serviceaccount:ddmr-stage:scoping-engine-acct
system:serviceaccount:ddmr-stage:device-channel-migration-acct
system:serviceaccount:ddmr-integration:scoping-engine-acct
system:serviceaccount:ddmr-integration:device-channel-migration-acct
```

HC stage is tighter: it has no `ddmr-integration` subjects. A pod running in an integration namespace on the HC side has no trusted subject and will fail `AssumeRoleWithWebIdentity`, which surfaces as a generic credentials error rather than a trust-policy message. Confirm the current subject list with `grep -rn 'oidc_fully_qualified_subjects' -A6` under the relevant `roles/` directory.

---

## mdm-schema-ingest-infrastructure

`github.com/jamf/mdm-schema-ingest-infrastructure` is a **separate Terraform/Terragrunt repo** owned by the Goldminers team (Jira: GOLD) that manages the MDM schema ingestion pipeline. It is not part of the DDmR service infrastructure, but DDmR services consume the schema endpoint it exposes. Pipeline behavior and the served routes are documented in `frontend.md`.

The pipeline is entirely serverless and EventBridge-orchestrated:

1. A scheduled Lambda (`mdm-git-ingest`) clones Apple MDM schema repos using EFS-mounted storage.
2. On schema change, `mdm-schema-transformation` transforms raw schemas to JSON.
3. Downstream Lambdas run in parallel: Java enhancements, archival, and translations.
4. An internal ALB-backed Lambda (`path-mapping`) serves schemas at `{env}.mdm-schema.jamf.build`.

Key resources: EFS share, ALB, S3 buckets (JSON schemas, UI schemas, translations), EventBridge rules, Lambda Layers (git, git-node, yaml-node).

| Environment | Account ID   | Region(s)                             | Domain                          |
|-------------|--------------|---------------------------------------|---------------------------------|
| dev         | 380922964950 | us-east-2                             | dev.mdm-schema.jamf.build       |
| stage       | 369976064392 | us-east-2                             | stage.mdm-schema.jamf.build     |
| prod        | 984688522419 | us-east-2, ap-northeast-1, eu-central-1 | prod.mdm-schema.jamf.build    |

---

## Tag Conventions

The two repos tag differently, and the divergence matters for anything that filters resources by tag (cost reports, inventory queries, Backstage joins). A query written for one repo's resources will not match the other's.

### ddmr-terraform (global-tags.yaml)

```yaml
team-email: engineering.ddm@jamf.com
team: ddmr
deployment-repository: https://github.com/jamf/ddmr-terraform
deployment-software: terraform
domain: capabilities
system: blueprints
```

Environment-specific tag files (`dev-tags.yaml`, `staging-tags.yaml`, etc.) add `environment: {env}`. Per-resource tags add `name` and `component`.

### ddmr-infrastructure (common.hcl)

```hcl
owner                 = "ddmr"
team-email            = "ddmr@jamf.com"
team-slack-channel    = "help-ddmr"
system                = "ddmr"
deployment-repository = "https://github.com/jamf/ddmr-infrastructure.git"
deployment-software   = "terragrunt"
```

Key divergences: `ddmr-infrastructure` uses `owner=ddmr`, `system=ddmr`, and `team-slack-channel=help-ddmr`; `ddmr-terraform` uses `team=ddmr`, `system=blueprints`, and no slack tag.

`ddmr-infrastructure` is the pattern for new provisioning. That is provenance, not a quality judgement: it was introduced with the HC account and built around Highway-to-Prod, whereas `ddmr-terraform` predates Highway-to-Prod and is still the only repo that manages the commercial accounts. Neither is being retired by the other on any schedule recorded here.

---

## Local Development Notes

**ddmr-infrastructure:** Run `./enable-local-mode.sh` before working locally. It switches from role-based to `AWS_PROFILE` auth.

**Trap: run `./enable-local-mode.sh restore` before committing.** Local mode edits tracked files. CI fails if those edits are left in, and the failure points at the auth config rather than at your change.

**ddmr-terraform:** Navigate to the target environment directory and run standard `terragrunt plan/apply`. The Jenkins pipeline handles CI, so local applies should be coordinated carefully.

Both repos use S3 backends with DynamoDB locking for state management.

---

## Where to find the data (verify rather than trust)

```bash
# What is actually provisioned in ddmr-infrastructure (vs scaffold)
find ddmr-infrastructure/infrastructure/aws -name terragrunt.hcl

# The real per-env service sets, both repos (this is the asymmetry)
ls ddmr-terraform/grunt/*/roles/
ls ddmr-infrastructure/infrastructure/aws/hc/*/*/*/roles/

# Where PITR is declared, and where it is not
grep -rn 'point_in_time_recovery_enabled' ddmr-terraform/grunt ddmr-infrastructure/infrastructure

# Table names for an env, including the ddmr-integration-* set
grep -rn 'resource_name\|table_name' ddmr-terraform/grunt/staging/dynamodb/terragrunt.hcl

# Live truth for a table: keys, GSIs, and restore window
aws dynamodb describe-table --table-name ddmr-staging-scoping-engine
aws dynamodb describe-continuous-backups --table-name ddmr-staging-scoping-engine

# Which OIDC subjects a role actually trusts
aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'
```
