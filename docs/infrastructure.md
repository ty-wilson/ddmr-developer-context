# Infrastructure

Last reviewed: 2026-04-07

## Two Infrastructure Repositories

DDmR infrastructure is split across two Terraform/Terragrunt repositories with different lineages and purposes.

### ddmr-terraform (legacy, Jenkins-based)
`github.com/jamf/ddmr-terraform` manages the commercial AWS accounts (dev, sandbox, staging, production). It predates the Highway-to-Prod rollout and is deployed via a **Jenkins pipeline** that:
- Runs parameterized plan stages for each environment/component combination
- Requires manual approval before apply on the main branch
- Runs inside Kubernetes with dedicated service accounts

This repo owns the canonical table-definition YAML files and IAM policy JSON files that are shared across environments. The `grunt/` directory is organized as `grunt/{environment}/{resource-type}/{service}/`.

### ddmr-infrastructure (newer, OpenTofu + GitHub Actions + HC)
`github.com/jamf/ddmr-infrastructure` was introduced to manage the **High Compliance (HC)** AWS account and serves as the template for new provisioning going forward. It uses:
- **OpenTofu 1.9.0** (binary: `tofu`) with Terragrunt 0.80.4
- **GitHub Actions** → Highway-to-Prod webhook → Argo for deployment
- A strict directory hierarchy: `infrastructure/aws/{partition}/{env}/{account_name}_{account_id}/{region}/{resource}/terragrunt.hcl`

Currently `commercial/` contains two scaffold environments — `sbox` and `stage` — both with placeholder account IDs (`012345678901`); only the `hc/` partition has real provisioned infrastructure.

When working on HC resources, use this repo. For commercial-environment changes (dev/sandbox/staging/production) use `ddmr-terraform`.

---

## AWS Account Topology

| Environment | Account ID   | Region(s)                                    | Repo            |
|-------------|--------------|----------------------------------------------|-----------------|
| dev         | 381491946762 | us-east-2                                    | ddmr-terraform  |
| sandbox     | 183197288009 | us-east-1                                    | ddmr-terraform  |
| staging     | 535838898545 | us-east-1                                    | ddmr-terraform  |
| production  | 613358915025 | us-east-1, eu-central-1, ap-northeast-1      | ddmr-terraform  |
| HC stage    | 604006981984 | us-east-2                                    | ddmr-infrastructure |

Production is the only multi-region deployment for ddmr-terraform. HC stage is currently the only provisioned environment in ddmr-infrastructure.

---

## Resources Managed

### DynamoDB Tables

Table schemas are defined as YAML files in `grunt/` (ddmr-terraform) and `infrastructure/aws/module_vars/` (ddmr-infrastructure). All tables use a single-table design with string `pkey`/`psort` (or `pk`) keys. Per-environment table names follow the pattern `ddmr-{env}-{service}`.

| Service                   | Table name (staging example)              | GSIs                                      |
|---------------------------|-------------------------------------------|-------------------------------------------|
| declaration-storage-service | ddmr-staging-declaration-storage        | `declaration_index` (declaration_key), `tenant_index` (tenant) |
| scoping-engine            | ddmr-staging-scoping-engine               | `group_index` (group_key), `tenant_index` (tenant) |
| tenant-authorizer         | ddmr-tenant-authorizer                    | `claimTenantMigration_index`              |

PITR (point-in-time recovery) is enabled on staging and HC stage tables.

The staging environment also provisions a `ddmr-integration-*` set of tables alongside the main tables to support integration test runs. These include at minimum: `ddmr-integration-declaration-storage`, `ddmr-integration-scoping-engine`, and `ddmr-integration-tenant-authorizer`.

### S3 Buckets

S3 resources are minimal. The sandbox environment has a single bucket:
- `ddmr-performance-test-results` — stores results from performance/load tests, AES256 encrypted

State backend buckets (one per environment/region) are managed outside these repos.

### IAM Roles

Each service gets an OIDC-assumable IAM role so its Kubernetes pod can access AWS resources without long-lived credentials. Roles are created using `terraform-aws-modules/terraform-aws-iam//modules/iam-assumable-role-with-oidc` (v5.20.0).

Services with IAM roles in staging (ddmr-terraform):
- `declaration-service`
- `declaration-storage-service`
- `scoping-engine`
- `tenant-authorizer`
- `tenant-migration`

Services with IAM roles in HC stage (ddmr-infrastructure):
- `declaration-service`
- `declaration-storage-service`
- `scoping-engine`

### IAM Policies

Each service has a corresponding policy granting it access to its own DynamoDB table and Secrets Manager secrets. Policy JSON templates live in `grunt/` as `{service}-serviceaccount-policy.json`. Note that `declaration-service` is an exception to this naming pattern — its file is `declaration-serviceaccount-policy.json` (no `-service` before `-serviceaccount`). Example permissions for scoping-engine:
- DynamoDB: `BatchGetItem`, `BatchWriteItem`, `ConditionCheckItem`, `PutItem`, `DescribeTable`, `DeleteItem`, `GetItem`, `Scan`, `Query`, `UpdateItem`
- Secrets Manager: `GetSecretValue` (scoped to `arn:aws:secretsmanager:*:{account}:secret:ddmr/{env}/scoping/*`)

---

## OIDC IAM Trust Pattern

All service IAM roles use EKS OIDC federation as their trust mechanism. The OIDC provider URL corresponds to the EKS cluster in each account/region. Roles are scoped to specific Kubernetes service accounts via `oidc_fully_qualified_subjects`.

Staging example for scoping-engine:
```
system:serviceaccount:ddmr-stage:scoping-engine-acct
system:serviceaccount:ddmr-stage:device-channel-migration-acct
system:serviceaccount:ddmr-integration:scoping-engine-acct
system:serviceaccount:ddmr-integration:device-channel-migration-acct
```

HC stage example (tighter — no integration namespace):
```
system:serviceaccount:ddmr-stage:scoping-engine-acct
system:serviceaccount:ddmr-stage:device-channel-migration-acct
```

The pattern is `system:serviceaccount:{k8s-namespace}:{service}-acct`. The namespace is `ddmr-stage` for staging/HC, `ddmr-integration` for integration test runs.

---

## mdm-schema-ingest-infrastructure

`github.com/jamf/mdm-schema-ingest-infrastructure` is a **separate Terraform/Terragrunt repo** owned by the Goldminers team (Jira: GOLD) that manages the MDM schema ingestion pipeline. It is not part of the DDmR service infrastructure but DDmR services consume the schema endpoint it exposes.

The pipeline is entirely serverless and EventBridge-orchestrated:
1. A scheduled Lambda (`mdm-git-ingest`) clones Apple MDM schema repos using EFS-mounted storage.
2. On schema change, `mdm-schema-transformation` transforms raw schemas to JSON.
3. Three downstream Lambdas run in parallel: Java enhancements, archival, and translations.
4. An internal ALB-backed Lambda (`path-mapping`) serves schemas at `{env}.mdm-schema.jamf.build`.

Key resources: EFS share, ALB, S3 buckets (JSON schemas, UI schemas, translations), EventBridge rules, Lambda Layers (git 2.40.1, git-node 3.30.0, yaml-node 4.1.1).

| Environment | Account ID   | Region(s)                             | Domain                          |
|-------------|--------------|---------------------------------------|---------------------------------|
| dev         | 380922964950 | us-east-2                             | dev.mdm-schema.jamf.build       |
| stage       | 369976064392 | us-east-2                             | stage.mdm-schema.jamf.build     |
| prod        | 984688522419 | us-east-2, ap-northeast-1, eu-central-1 | prod.mdm-schema.jamf.build    |

---

## Tag Conventions

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

The key difference: `ddmr-infrastructure` uses `owner=ddmr` and `team-slack-channel=help-ddmr`; `ddmr-terraform` uses `team=ddmr` and no slack tag. New resources should follow the `ddmr-infrastructure` pattern as it is the current standard.

---

## Local Development Notes

**ddmr-infrastructure:** Run `./enable-local-mode.sh` before working locally (switches from role-based to `AWS_PROFILE` auth). Run `./enable-local-mode.sh restore` before committing — CI will fail if local-mode changes are left in.

**ddmr-terraform:** Navigate to the target environment directory and run standard `terragrunt plan/apply` commands. The Jenkins pipeline handles CI, so local applies should be coordinated carefully.

Both repos use S3 backends with DynamoDB locking for state management.
