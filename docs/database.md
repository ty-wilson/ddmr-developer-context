# Database

Last reviewed: 2026-07-29 (Tyler Wilson). Re-verified against `declaration-storage-service` code on that date: the DSS assignment `tag` semantics, `declaration_index` not being tenant-scoped, and declaration IDs being random UUIDs. **Not re-verified and older:** the scoping-engine key patterns, the tenant-authorizer table, and the cross-service isolation notes. Treat those as pointers and confirm before relying on them.

## Overview

DDmR uses DynamoDB with a single-table design. Each service owns its own table; services do not read each other's tables. Cross-service data flows happen over HTTP or Pulsar events.

Table definitions live in `ddmr-terraform/grunt/*-table-definition.yaml`. The primary key across all DDmR tables is `pkey` (hash) and `psort` (range), both strings, except the tenant-authorizer table which uses `pk` with no range key.

EU region note: DDmR DynamoDB tables are in `eu-central-1`, not `eu-west-1`.

## Tables

### scoping-engine

**Table definition:** `ddmr-terraform/grunt/scoping-engine-table-definition.yaml`

Primary key: `pkey` (hash) / `psort` (range)

GSIs:
- `group_index`: hash key `group_key`, projection `ALL` (the **only** GSI on this table)

There is **no `tenant_index`**. It was removed in DDMR-1035 (2026-04-02) and is absent from both the Terraform definition and every live table. To find all of a tenant's rows you must `Scan` with a filter (e.g. `contains(pkey, "<tenant>")`); there is no tenant-partitioned index. When you already know a group, prefer a direct primary-key `Query` on `pkey = GROUP#<tenant>|<groupId>` (or `group_index` by `group_key`) over a scan.

Key patterns:

| pkey | psort | Description |
|------|-------|-------------|
| `SCOPE#<tenant>\|<scopeId>` | `METADATA` | Scope metadata (enabled flag, `division_id` when scope is division-owned) |
| `SCOPE#<tenant>\|<scopeId>` | `GROUP#<groupId>` | Scope-to-group membership. Sets `group_key` for group_index. |
| `SCOPE#<tenant>\|<scopeId>` | `DECL#<identifier>` | Declaration assigned to a scope |
| `SCOPE#<tenant>\|<scopeId>` | `APP#<assetId>` | VPP app assigned to a scope |
| `MEMBERSHIP#<tenant>\|<deviceId>` | `GROUP#<groupId>` | Device-to-group membership. Sets `group_key` for group_index. |
| `GROUP#<tenant>\|<groupId>` | `DIVISION` | Group→division mapping (`division_id`). Absence of the row = group has no division ("global"). |
| `DEVICE-CHANNEL#<tenant>\|<deviceId>` | `C#<channel>` | Device channel info (type, deprecated, remove flags) |
| `DEVICE#<tenant>\|<deviceId>\|<channel>` | `SYNC` | Device sync state |
| `DEVICE#<tenant>\|<deviceId>` | `GROUPS_DECL` | Management properties declaration for a device |

The `group_key` attribute encodes both tenant and scope state. For scope-group items it is `ENABLED-GROUPS#<tenant>` or `DISABLED-GROUPS#<tenant>`. For device-membership items it is `GROUP#<tenant>|<groupId>`. This allows the `group_index` GSI to answer two different queries from a single index:
- Find all scopes that contain a given group (filter on scope key prefix)
- Find all devices that belong to a given group (filter on membership key prefix)

The `GROUP#<tenant>|<groupId> / DIVISION` row is written only by the `DeviceGroupDivisionChangedHandler` consuming the `device-group-division-changed` Pulsar event (never by the REST scope-save path), and is read by the divisions validation on `POST`/`PUT /scope`: a division-scoped caller can only scope a group whose stored `division_id` matches the caller's division, so a group with no `DIVISION` row is rejected with 400 "Invalid group id". See [event-layer.md] and the scoping-engine service doc.

### declaration-storage-service

**Table definition:** `ddmr-terraform/grunt/declaration-storage-table-definition.yaml`

Primary key: `pkey` (hash) / `psort` (range)

GSIs:
- `declaration_index`: hash key `declaration_key`, projection `ALL` (the **only** GSI on this table)

There is **no `tenant_index`**. It was removed in DDMR-1035 (2026-04-02) from both Terraform and all live tables (staging, integration, and prod). The `ddmr-tenant-migration-job` queries `tenant_index` (see `DynamoDbService.kt`), so it **cannot function** as written: the index it depends on no longer exists. That is a structural consequence of DDMR-1035, not a deployment state.

**Trap: the GitOps config does not reflect reality here.** `ddmr-deployments` still declares this CronJob as enabled, so reading intent from that repo will tell you the job runs. Check the cluster instead: `kubectl get cronjob -A | grep tenant-migration`.

Key patterns:

| pkey | psort | Description |
|------|-------|-------------|
| `DECL#<id>` | `PAYLOAD` | Declaration payload (type, JSON/encrypted content, payload token) |
| `MDM#<tenant>\|<deviceId>\|<channel>` | `A#<identifier>` | Declaration assignment for a device+channel+identifier. Sets `declaration_key = DECL#<id>`. |
| `MIGRATION` | `#FROM#<tenantId>#TO#<uuid>` | In-progress tenant migration marker |

The `declaration_index` GSI is used to find all assignments for a given declaration (e.g., when deleting a declaration, all its assignments are removed first). Because projection is `ALL`, the full assignment item is available from the index without a second read. Note the GSI is keyed only on `declaration_key` (`DECL#<id>`), so it is **not** tenant-scoped; callers that read it must filter by the row's `tenant` attribute themselves.

An assignment is uniquely identified by `MDM#<tenant>|<device>|<channel>` + `A#<identifier>`. **`tag` is a non-key attribute, not part of the key.** Null is persisted as the empty string `""`, and null/empty are treated as equal. Because tag is not in the key, a given `(tenant, device, channel, identifier)` slot holds exactly one assignment with exactly one tag value; you cannot have two assignments for the same slot differing only by tag. Add/remove are guarded by a conditional write requiring both the tenant and tag to match the existing row, so a write under a different tag is rejected (surfacing as a `wrong tenant or tag` / `tag mismatch` result) rather than clobbering another owner's assignment. This is the storage-level mechanism behind the tag-based ownership isolation and the `401 "Tenant mismatch"` behavior described in `services/declaration-storage-service.md`. Declaration IDs are random UUIDs (`UUID.randomUUID()` in `createDeclaration`), not content hashes, so a `DECL#<id>` belongs to exactly one declaration owned by one tenant.

Declaration payloads are prefixed `json:` (plaintext) or `iron:` (IronCore encrypted). Encrypted items also store an `edek` (encrypted data encryption key) alongside the payload.

### tenant-authorizer

**Table definition:** `ddmr-terraform/grunt/tenant-authorizer-table-definition.yaml`

Primary key: `pk` (hash only, no range key)

GSIs:
- `claimTenantMigration_index`: hash key `claimTenantMigration`, projection `ALL`

This table does not follow the `pkey`/`psort` convention. It uses `pk` as the sole primary key attribute.

## Cross-service isolation

Services own their own tables and do not cross-read. For example:

- Scoping engine stores declaration identifiers (`DECL#<id>`) as foreign references in scope items, but calls declaration-storage-service over HTTP (via `DeclarationStorageWrapper`) for declaration creation, payload editing, and device assignment, rather than querying declaration-storage's table directly.
- Declaration-storage-service stores assignment records keyed by device+channel but has no awareness of scoping engine's `SCOPE#` or `MEMBERSHIP#` items.

Communication between services happens via HTTP (synchronous) or Pulsar events (asynchronous). There is no shared DynamoDB table between any two DDmR services.

## Terraform source

Table definitions are in `ddmr-terraform/grunt/`:
- `scoping-engine-table-definition.yaml`
- `declaration-storage-table-definition.yaml`
- `tenant-authorizer-table-definition.yaml`

IAM policy documents for DynamoDB access are also in the same directory:
- `scoping-engine-serviceaccount-policy.json`
- `declaration-storage-serviceaccount-policy.json`
- `declaration-serviceaccount-policy.json`
- `tenant-authorizer-serviceaccount-policy.json`
- `tenant-migration-serviceaccount-policy.json`
