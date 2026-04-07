# Database

Last reviewed: 2026-04-07

## Overview

DDmR uses DynamoDB with a single-table design. Each service owns its own table; services do not read each other's tables. Cross-service data flows happen over HTTP or Pulsar events.

Table definitions live in `ddmr-terraform/grunt/*-table-definition.yaml`. The primary key across all DDmR tables is `pkey` (hash) and `psort` (range), both strings — except the tenant-authorizer table which uses `pk` with no range key.

EU region note: DDmR DynamoDB tables are in `eu-central-1`, not `eu-west-1`.

## Tables

### scoping-engine

**Table definition:** `ddmr-terraform/grunt/scoping-engine-table-definition.yaml`

Primary key: `pkey` (hash) / `psort` (range)

GSIs:
- `group_index` — hash key `group_key`, projection `ALL`
- `tenant_index` — hash key `tenant`, projection `KEYS_ONLY`

Key patterns:

| pkey | psort | Description |
|------|-------|-------------|
| `SCOPE#<tenant>\|<scopeId>` | `METADATA` | Scope metadata (enabled flag, etc.) |
| `SCOPE#<tenant>\|<scopeId>` | `GROUP#<groupId>` | Scope-to-group membership. Sets `group_key` for group_index. |
| `SCOPE#<tenant>\|<scopeId>` | `DECL#<identifier>` | Declaration assigned to a scope |
| `SCOPE#<tenant>\|<scopeId>` | `APP#<assetId>` | VPP app assigned to a scope |
| `MEMBERSHIP#<tenant>\|<deviceId>` | `GROUP#<groupId>` | Device-to-group membership. Sets `group_key` for group_index. |
| `DEVICE-CHANNEL#<tenant>\|<deviceId>` | `C#<channel>` | Device channel info (type, deprecated, remove flags) |
| `DEVICE#<tenant>\|<deviceId>\|<channel>` | `SYNC` | Device sync state |
| `DEVICE#<tenant>\|<deviceId>` | `GROUPS_DECL` | Management properties declaration for a device |

The `group_key` attribute encodes both tenant and scope state. For scope-group items it is `ENABLED-GROUPS#<tenant>` or `DISABLED-GROUPS#<tenant>`. For device-membership items it is `GROUP#<tenant>|<groupId>`. This allows the `group_index` GSI to answer two different queries from a single index:
- Find all scopes that contain a given group (filter on scope key prefix)
- Find all devices that belong to a given group (filter on membership key prefix)

The `tenant_index` is `KEYS_ONLY` and is used for tenant-scoped administrative queries. It does not project non-key attributes.

### declaration-storage-service

**Table definition:** `ddmr-terraform/grunt/declaration-storage-table-definition.yaml`

Primary key: `pkey` (hash) / `psort` (range)

GSIs:
- `declaration_index` — hash key `declaration_key`, projection `ALL`
- `tenant_index` — hash key `tenant`, projection `KEYS_ONLY`

Key patterns:

| pkey | psort | Description |
|------|-------|-------------|
| `DECL#<id>` | `PAYLOAD` | Declaration payload (type, JSON/encrypted content, payload token) |
| `MDM#<tenant>\|<deviceId>\|<channel>` | `A#<identifier>` | Declaration assignment for a device+channel+identifier. Sets `declaration_key = DECL#<id>`. |
| `MIGRATION` | `#FROM#<tenantId>` | In-progress tenant migration marker |

The `declaration_index` GSI is used to find all assignments for a given declaration (e.g., when deleting a declaration, all its assignments are removed first). Because projection is `ALL`, the full assignment item is available from the index without a second read.

Declaration payloads are prefixed `json:` (plaintext) or `iron:` (IronCore encrypted). Encrypted items also store an `edek` (encrypted data encryption key) alongside the payload.

### tenant-authorizer

**Table definition:** `ddmr-terraform/grunt/tenant-authorizer-table-definition.yaml`

Primary key: `pk` (hash only, no range key)

GSIs:
- `claimTenantMigration_index` — hash key `claimTenantMigration`, projection `ALL`

This table does not follow the `pkey`/`psort` convention — it uses `pk` as the sole primary key attribute.

## Cross-service isolation

Services own their own tables and do not cross-read. For example:

- Scoping engine stores declaration identifiers (`DECL#<id>`) as foreign references in scope items, but it calls declaration-storage-service over HTTP (via `DeclarationStorageWrapper`) to resolve payloads rather than querying declaration-storage's table directly.
- Declaration-storage-service stores assignment records keyed by device+channel but has no awareness of scoping engine's `SCOPE#` or `MEMBERSHIP#` items.

Communication between services happens via HTTP (synchronous) or Pulsar events (asynchronous). There is no shared DynamoDB table between any two DDmR services.

## Terraform source

Table definitions are in `ddmr-terraform/grunt/`:
- `scoping-engine-table-definition.yaml`
- `declaration-storage-table-definition.yaml`
- `tenant-authorizer-table-definition.yaml`

IAM policy documents for each service's DynamoDB access are in files named `<service>-serviceaccount-policy.json` in the same directory.
