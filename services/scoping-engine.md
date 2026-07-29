# Scoping Engine

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the route table (`RoutingConfig.kt`), the full Pulsar listener set and topic names (`MessagingTopicProperties.kt`, `PulsarRoutingConfig.kt`), the retry-reschedule path, and the named caller list below. **Not re-verified and older:** the request/response body shapes in the endpoint tables and the `replace` / `removeNotApplied` semantics, which date from 2026-05-18. Commands to re-derive any of this are in "Where to find the data" at the end.

**Owner:** DDmR team

## Summary

Scoping Engine is the platform service responsible for tracking which devices belong to which scopes, and for pushing the right declarations and VPP app assignments to devices when scope membership changes. A "scope" is a named collection of device groups with an enabled/disabled state; when a device's group membership changes, Scoping Engine recomputes that device's scope set, updates a management-properties declaration in Declaration Storage Service, and fires membership-changed events so downstream services can react. It is the authority on scope-to-group, scope-to-assignment, and device-to-group membership state.

---

## API Endpoints

All routes live under `/api/{rev}` with a `version("v1+")` predicate, so `/api/v1/...` is what callers use. Every request must include the `X-TenantId` header (401 if absent). `X-EnvironmentId` is optional and flows through to outbound events.

Authentication uses M2M tokens (`robocop`). The serialization format is JSON with `kotlinx.serialization`: `ignoreUnknownKeys = true` so callers can add fields freely, and `encodeDefaults = true`.

The route table is declared in one place. Regenerate it rather than trusting the tables below (see the final section).

### Scope management

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/scope` | Create a new scope. Body: `{ include: [{type: "groupId", value: "..."}], enabled: true }`. Returns `{ scopeId }`. On creation, immediately syncs devices in any included groups. |
| `PUT` | `/scope/{scopeId}` | Add or remove groups from an existing scope. Body: `{ add: { include: [...] }, remove: { include: [...] } }`. |
| `GET` | `/scope/{scopeId}` | Retrieve a scope's groups and enabled state. 404 if not found. |
| `DELETE` | `/scope/{scopeId}` | Delete a scope and all its data. Triggers device syncs for previously scoped devices if the scope was enabled. |
| `PUT` | `/toggle-scopes` | Bulk enable/disable scopes. Body: `{ enabled: ["id1"], disabled: ["id2"] }`. Dispatches an `ApiRequestEvent` for async processing. |
| `GET` | `/scope/{scopeId}/devices/count` | Count of devices currently in scope. Response: `{ count, countType }` where `countType` is `EXACT` or `LOWER_BOUND`. |
| `GET` | `/scope/{scopeId}/devices/publish/{action}` | Re-publish all scope membership events. `action` must be `stream` (NDJSON response) or `events` (fires Pulsar events). Hidden from public API docs, used for backfill/repair. |

### Assignment management

| Method | Path | Description |
|--------|------|-------------|
| `PUT` | `/scope/{scopeId}/assignment` | Apply or remove declaration and VPP app assignments on a scope. Dispatches an `ApiRequestEvent` for async processing. |
| `GET` | `/scope/{scopeId}/assignment?deployableType=` | Retrieve current assignments. Query param `deployableType` accepts `DECLARATIONS`, `APPS`, or both. |
| `DELETE` | `/scope/{scopeId}/assignment?deployableType=` | Remove all assignments of the specified deployable type(s). |

### Group utility and probes

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/group/{groupId}/used` | Returns `200` if the group is referenced by any scope, `204` if not. Used by callers before deleting a group. |
| `GET` | `/group/{groupId}` | Devices-by-group listing, served by `GroupEndpointTestHandler`. Diagnostic, not a product contract. |
| `HEAD` | `/api/{rev}` | Connectivity check for external monitors. |

---

## Data Model

Single-table DynamoDB design, all keys scoped per-tenant with a `|` separator. **`docs/database.md` is the canonical owner of these key patterns, the GSI list, and the `group_key` encoding**; it is kept current and includes the table-definition file path. The table below is an orientation copy.

| Primary key (`pkey`) | Sort key (`psort`) | What it stores |
|---|---|---|
| `SCOPE#<tenant>\|<scopeId>` | `METADATA` | Scope metadata: enabled flag, `division_id` when division-owned |
| `SCOPE#<tenant>\|<scopeId>` | `GROUP#<groupId>` | Scope-to-group membership |
| `SCOPE#<tenant>\|<scopeId>` | `DECL#<identifier>` | Declaration assignment (identifier → payload ID + channel type) |
| `SCOPE#<tenant>\|<scopeId>` | `APP#<assetId>` | VPP app assignment (asset ID → token ID + channel type) |
| `MEMBERSHIP#<tenant>\|<deviceId>` | `GROUP#<groupId>` | Device-to-group membership |
| `GROUP#<tenant>\|<groupId>` | `DIVISION` | Group→division mapping (`division_id`) |
| `DEVICE-CHANNEL#<tenant>\|<deviceId>` | `C#<channel>` | Device channel info |
| `DEVICE#<tenant>\|<deviceId>\|<channel>` | `SYNC` | Device sync state |
| `DEVICE#<tenant>\|<deviceId>` | `GROUPS_DECL` | Declaration ID for the device's management-properties declaration |

`group_index` (hash key `group_key`, projection ALL) is the only GSI. It answers both "which scopes contain this group" and "which devices belong to this group" from one index.

**Trap: `group_key` puts a scope's group items on *different* GSI partitions depending on scope state.** Scope-group items carry `group_key` of `ENABLED-GROUPS#<tenant>` or `DISABLED-GROUPS#<tenant>` (device-membership items use `GROUP#<tenant>|<groupId>` instead). Toggling a scope rewrites every one of its group items to flip the key, and "which enabled scopes contain this device" reads only the enabled partition. Do not assume all group items for a scope live on one partition, and do not assume a disabled scope's groups are absent from the index. Confirm the constructed value in `DynamoDb.kt` before writing a query.

**Trap: a group with no `DIVISION` row is rejected, not treated as global.** That row is written *only* by the `device-group-division-changed` consumer, never by the REST scope-save path, so a group the producer never announced fails division validation on `POST`/`PUT /scope` with `400 "Invalid group id"`. Full worked explanation in `docs/database.md`.

---

## Events

### Pulsar namespace layout

- Platform namespace (`pdd/default`): shared with the platform, treat topic names as a contract; changes require broad coordination.
- Scoping Engine namespace (`pdd/scoping-engine`): owned by this service.

Topic short names and both namespace defaults are declared in `MessagingTopicProperties.kt`, and every one of them is overridable by config plus a `topicNameSuffix`. Read them there rather than from this page.

### Produced

| Topic | Namespace | When fired |
|-------|-----------|------------|
| `device-sync` | `pdd/scoping-engine` | Triggers a device to re-pull its current assignments. Fired when scope or assignment state changes for a device. The only path that passes a delivery delay. |
| `device-sync-bulk` | `pdd/scoping-engine` | Lower-priority bulk variant of `device-sync`, with its own listener and its own metric names. |
| `api-request` | `pdd/scoping-engine` | Carries async assignment updates and scope toggle operations queued from HTTP handlers. Consumed by this same service. |
| `device-scope-membership-changed` | `pdd/default` | Fired when a device is added to or removed from a scope. |

### Consumed

| Topic | Namespace | Handler | What it does |
|-------|-----------|---------|--------------|
| `device-group-changed` | `pdd/default` | `DeviceGroupChangedHandler` | Adds, removes, or fully syncs a device's group memberships. Recomputes scope membership, updates the management-properties declaration in DSS, and fires scope-membership-changed events if the device's scopes changed. |
| `device-sync` | `pdd/scoping-engine` | `DeviceSyncHandler` | Processes a queued device sync, pushing current declarations and VPP apps to the device. |
| `device-sync-bulk` | `pdd/scoping-engine` | `DeviceSyncHandler` | Same handler, separate listener and subscription, for bulk-priority syncs. |
| `api-request` | `pdd/scoping-engine` | `ApiRequestEventHandler` | Applies assignment changes and scope toggle operations written by the HTTP handlers. |
| `device-management-channel-changed` | `pdd/default` | `DeviceChannelChangedHandler` | Handles device channel changes (SYSTEM/USER). |
| `device-group-division-changed` | `pdd/default` | `DeviceGroupDivisionChangedHandler` | Writes the `GROUP#<tenant>\|<groupId> / DIVISION` row. Sole writer of group→division state. |

All consumers use `Key_Shared` subscription type. `PulsarWatchdog` starts listeners after context refresh with unbounded exponential-backoff retry: listeners do not auto-start, and a pod can be Ready while consuming nothing. `docs/event-layer.md` owns the listener-settings and subscription-naming detail.

---

## How Other Services Interact

### Callers via HTTP

| Caller | Team | What it calls | Where in that repo |
|---|---|---|---|
| `blueprint-deployment-service` | Ocean | `PUT /api/v1/scope/{id}/assignment`, `PUT /api/v1/toggle-scopes` | `client/ScopingEngineClient.java`, `client/ScopingEngineAdapter.java`, called from `service/DeploymentService.java`. `@Retryable` on 5xx and `ResourceAccessException`. |
| Jamf Pro Server (`jss`) | Jamf Pro | `GET /scoping/api/v1/group/{groupId}/used` **only** | `scoping-engine-client/scoping-engine-client-impl/.../ScopingEngineServiceImpl.java`. Path is a Jamf Pro property (`com.jamf.pro.scopingengine.endpoint.group.used`) with that value as default, and the whole call is gated on `isScopingEngineEnabled()`. |
| `scoping` MFE (`@jmf/scoping`) | DDmR | `POST /scoping/api/v1/scope`, `GET /scoping/api/v1/scope/{id}` | `micro-frontend-hub/apps/scoping/src/client/scoping/useScopingApi.ts` |
| `blueprints` MFE (`@jmf/blueprints`) | DDmR | `GET /scoping/api/v1/scope/{id}`, `GET /scoping/api/v1/scope/{id}/devices/count` | `micro-frontend-hub/apps/blueprints/src/client/scoping/useScopingApi.ts` |
| `scope-membership-tool` | DDmR | Backfill/repair jobs | `ddmr-deployments/helm/scope-membership-tool/` |

The two browser callers reach the service through the `/scoping` Tyk route, not directly. Route definitions live in `tyk-gateway-management/*/api-products/scope-eng/`.

**Note there is no service that creates scopes on the backend.** Scope creation (`POST /scope`) comes from the `scoping` MFE and from tests/tools. `blueprint-management-service` only *stores* a `scopeId` on a `BlueprintVersion`; it never calls Scoping Engine. Re-derive with the grep in the final section before assuming a new backend caller exists.

### Callers via Pulsar

- **Jamf Pro Server (`jamf-messaging`)** produces `device-group-changed` and `device-management-channel-changed`. Scoping Engine is the primary consumer of `device-group-changed` and owns the downstream fan-out.
- **Jamf School / division producers** produce `device-group-division-changed`. In `platform-staging` this producer has been observed effectively silent, which is what makes the missing-`DIVISION`-row trap above bite.

### Downstream consumers of events Scoping Engine produces

- `device-scope-membership-changed`: **`blueprint-report-aggregation-service`** (Ocean) is the only consumer, via `messaging/DeviceScopeMembershipChangedListener.java`. `compliance-benchmark-report-service` consumes `device-group-changed` directly from the platform, not this topic.
- `device-sync`, `device-sync-bulk`, `api-request`: Scoping Engine itself, no external consumers.

---

## Key Design Decisions and Gotchas

**Trap: assignment writes and toggles are asynchronous, and the `200` lies.** `PUT /scope/{scopeId}/assignment` and `PUT /toggle-scopes` both dispatch an `ApiRequestEvent` and return `200` before anything is written. A caller that immediately reads back via `GET /scope/{scopeId}/assignment` sees stale data. This is the usual cause of "my assignment did not save" reports that turn out to be a read race.

**Trap: the retry path has no max-retry bound.** A retryable error in `DeviceSyncHandler.processEvent` (typically `M2MTokenAcquisitionException(retryable=true)`) calls `deviceService.syncDeviceWithDelay(...)`, producing a fresh `DeviceSyncEvent` with a Pulsar `deliverAfter` delay. A device-channel with a *permanent* failure, for example orphan DynamoDB state for a tenant no longer in HAL, loops forever until the DynamoDB state is removed or Pulsar's max-redelivery cuts in. The delay is no longer a single constant: it comes from the matched `SyncErrorHandlingResult` variant (`RETRY`, `RETRY_LONG_DELAY`, `RETRY_NEW_AUTH`, each with its own `rescheduleDelay`) in `DeployableObjectSyncAdapter.kt`. Read the current values there.

Because the loop republishes 1:1, it dominates log volume and `pulsar_rate_in` while contributing **zero net backlog growth**. Do not conflate the retry-storm tenant with the backlog-growth tenant. See `docs/observability.md`.

**Device count is best-effort.** `GET /scope/{scopeId}/devices/count` caps both item count and wall-clock time; if either cap is hit, `countType` is `LOWER_BOUND` rather than `EXACT`. `EXACT` means the walk completed, not that the number is authoritative at read time. Never use this endpoint for billing-grade counts.

**`replace` field on assignment updates.** Pass `replace: ["declarations"]`, `["apps"]`, or `["all"]` to treat the provided `applyIdentifiers`/`applyAssets` as the complete set and remove anything not listed. The older `declarations.removeNotApplied` boolean is deprecated: use `replace`.

**Platform topics are shared contracts.** `device-scope-membership-changed`, `device-management-channel-changed`, `device-group-changed`, and `device-group-division-changed` all live in `pdd/default` and are used across the platform. Schema or topic-name changes require coordination outside this service.

**External service dependencies.** Scoping Engine calls Declaration Storage Service (via `declaration-product-springboot-starter`, wrapped by `DeclarationStorageWrapper`) to create and update management-properties declarations, and calls a VPP app service via `VppAppClient` using M2M auth. DSS must be reachable for group-change events to complete. `VppAppClient` is called during device-sync processing (via `AppSyncAdapter`), not during group-change handling.

---

## Where to find the data (verify rather than trust)

```bash
SE=~/Projects/DDmR/scoping-engine; git -C $SE fetch origin -q
git -C $SE log --oneline origin/main --since=2026-07-29
git -C $SE for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Regenerate the route table (the only declaration site)
git -C $SE show origin/main:src/main/kotlin/com/jamf/platform/scoping/config/RoutingConfig.kt

# Every topic short name and namespace default, plus the suffix mechanism
git -C $SE show origin/main:src/main/kotlin/com/jamf/platform/scoping/config/properties/MessagingTopicProperties.kt

# The full listener set (6 as of 2026-07-29) and which handler each binds to
git -C $SE grep -n -A7 '@PulsarListener' origin/main -- src/main/kotlin

# The unbounded-retry site and the current reschedule delays
git -C $SE grep -n 'syncDeviceWithDelay' origin/main -- src/main/kotlin
git -C $SE grep -n -A12 'enum class SyncErrorHandlingResult' origin/main -- src/main/kotlin

# The group_key values actually written, before writing any GSI query
git -C $SE grep -n 'group_index\|GROUP_INDEX_KEY\|ENABLED\|DISABLED' origin/main \
  -- src/main/kotlin/com/jamf/platform/scoping/datasource/
```

Rebuild the caller and consumer lists rather than trusting them. Run these from `~/Projects/DDmR`:

```bash
# Consumers of the one platform topic this service produces
grep -rln 'device-scope-membership-changed' */src/main

# HTTP callers: client classes, then the Tyk routes that front them
grep -rIln --include=*.java --include=*.kt --include=*.ts \
  -e 'ScopingEngineClient' -e 'ScopingEngineService' -e "/scoping/api" . \
  | grep -v '/.claude/worktrees/' | grep -v node_modules
ls tyk-gateway-management/*/api-products/scope-eng/
```

Negative claims here ("only consumer", "no backend caller") were derived from `main`. Check unmerged branches in the candidate repo before repeating them.
