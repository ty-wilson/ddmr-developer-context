# Scoping Engine

Last reviewed: 2026-04-07

**Owner:** DDmR team

## Summary

Scoping Engine is the platform service responsible for tracking which devices belong to which scopes, and for pushing the right declarations and VPP app assignments to devices when scope membership changes. A "scope" is a named collection of device groups with an enabled/disabled state; when a device's group membership changes, Scoping Engine recomputes that device's scope set, updates a management-properties declaration in Declaration Storage Service, and fires membership-changed events so downstream services can react. It is the authority on scope-to-group, scope-to-assignment, and device-to-group membership state.

---

## API Endpoints

All routes live under `/api/v1`. Every request must include the `X-TenantId` header (401 if absent). `X-EnvironmentId` is optional and flows through to outbound events.

Authentication uses M2M tokens (`robocop`). The serialization format is JSON with `kotlinx.serialization` — `ignoreUnknownKeys = true` so callers can add fields freely, and `encodeDefaults = true`.

### Scope management

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/scope` | Create a new scope. Body: `{ include: [{type: "groupId", value: "..."}], enabled: true }`. Returns `{ scopeId }`. On creation, immediately syncs devices in any included groups. |
| `PUT` | `/scope/{scopeId}` | Add or remove groups from an existing scope. Body: `{ add: { include: [...] }, remove: { include: [...] } }`. |
| `GET` | `/scope/{scopeId}` | Retrieve a scope's groups and enabled state. 404 if not found. |
| `DELETE` | `/scope/{scopeId}` | Delete a scope and all its data. Triggers device syncs for previously scoped devices if the scope was enabled. |
| `PUT` | `/toggle-scopes` | Bulk enable/disable scopes. Body: `{ enabled: ["id1"], disabled: ["id2"] }`. Dispatches an `ApiRequestEvent` for async processing. |
| `GET` | `/scope/{scopeId}/devices/count` | Count of devices currently in scope. Response: `{ count, countType }` where `countType` is `EXACT` or `LOWER_BOUND` (capped for large/slow scopes). |
| `GET` | `/scope/{scopeId}/devices/publish/{action}` | Re-publish all scope membership events. `action` must be `stream` (NDJSON response) or `events` (fires Pulsar events). Hidden from public API docs — used for backfill/repair. |

### Assignment management

| Method | Path | Description |
|--------|------|-------------|
| `PUT` | `/scope/{scopeId}/assignment` | Apply or remove declaration and VPP app assignments on a scope. Dispatches an `ApiRequestEvent` for async processing. |
| `GET` | `/scope/{scopeId}/assignment?deployableType=` | Retrieve current assignments. Query param `deployableType` accepts `DECLARATIONS`, `APPS`, or both. |
| `DELETE` | `/scope/{scopeId}/assignment?deployableType=` | Remove all assignments of the specified deployable type(s). |

### Group utility

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/group/{groupId}/used` | Returns `200` if the group is referenced by any scope, `204` if not. Used by callers before deleting a group. |

---

## Data Model

Single-table DynamoDB design. All keys are scoped per-tenant using a `|` separator.

| Primary key (`pkey`) | Sort key (`psort`) | What it stores |
|---|---|---|
| `SCOPE#<tenant>\|<scopeId>` | `METADATA` | Scope metadata: enabled flag |
| `SCOPE#<tenant>\|<scopeId>` | `GROUP#<groupId>` | Scope-to-group membership |
| `SCOPE#<tenant>\|<scopeId>` | `DECL#<identifier>` | Declaration assignment (identifier → payload ID + channel type) |
| `SCOPE#<tenant>\|<scopeId>` | `APP#<assetId>` | VPP app assignment (asset ID → token ID + channel type) |
| `MEMBERSHIP#<tenant>\|<deviceId>` | `GROUP#<groupId>` | Device-to-group membership |
| `DEVICE-CHANNEL#<tenant>\|<deviceId>` | `C#<channel>` | Device channel info |
| `DEVICE#<tenant>\|<deviceId>\|<channel>` | `SYNC` | Device sync state |
| `DEVICE#<tenant>\|<deviceId>` | `GROUPS_DECL` | Declaration ID for the device's management-properties declaration |

A GSI named `group_index` on the `group_key` attribute enables looking up all scopes or devices that belong to a group. The `group_key` value for scope-group items encodes the enabled state: `ENABLED-GROUPS#<tenant>` or `DISABLED-GROUPS#<tenant>`. Only enabled-scope groups appear on the enabled index, which allows efficient "which enabled scopes does this device belong to" queries.

---

## Events

### Pulsar namespace layout

- Platform namespace (`pdd/default`): shared with the platform — treat topic names as a contract; changes require broad coordination.
- Scoping Engine namespace (`pdd/scoping-engine`): owned by this service.

### Produced

| Topic | Namespace | When fired |
|-------|-----------|------------|
| `device-sync` | `pdd/scoping-engine` | Triggers a device to re-pull its current assignments. Fired when scope or assignment state changes for a device. |
| `api-request` | `pdd/scoping-engine` | Carries async assignment updates and scope toggle operations queued from HTTP handlers. Consumed by this same service. |
| `device-scope-membership-changed` | `pdd/default` | Fired when a device is added to or removed from a scope. Consumed downstream by services that react to scope membership (e.g., to push configurations). |

### Consumed

| Topic | Namespace | Handler | What it does |
|-------|-----------|---------|--------------|
| `device-group-changed` | `pdd/default` | `DeviceGroupChangedHandler` | Adds, removes, or fully syncs a device's group memberships. Recomputes scope membership, updates the management-properties declaration in Declaration Storage Service, and fires scope-membership-changed events if the device's scopes changed. |
| `device-sync` | `pdd/scoping-engine` | `DeviceSyncHandler` | Processes a queued device sync, pushing current declarations and VPP apps to the device. |
| `api-request` | `pdd/scoping-engine` | `ApiRequestEventHandler` | Applies assignment changes (add/remove declarations and apps) and scope toggle operations written by the HTTP handlers. |
| `device-management-channel-changed` | `pdd/default` | `DeviceChannelChangedHandler` | Handles device channel changes (SYSTEM/USER). |

All consumers use `Key_Shared` subscription type. The `PulsarWatchdog` starts listeners after context refresh with infinite exponential-backoff retry — listeners do not auto-start.

---

## How Other Services Interact

**Callers via HTTP:**
- Services that manage scopes (creation, group assignment, enable/disable) call the scope CRUD and toggle endpoints directly.
- Services that assign declarations or VPP apps to scopes call `PUT /scope/{scopeId}/assignment`. The response returns immediately; actual DB writes happen asynchronously when the service consumes the resulting `api-request` event.
- Services that need to know if a group is in use before deleting it call `GET /group/{groupId}/used`.

**Callers via Pulsar:**
- The platform's device-group service publishes `device-group-changed` events when a device's group membership changes. Scoping Engine is the primary consumer of this event and owns the downstream fan-out.

**Downstream consumers of events Scoping Engine produces:**
- Any service that needs to know a device's current scope membership subscribes to `device-scope-membership-changed` on `pdd/default`.

---

## Key Design Decisions and Gotchas

**Assignment writes are asynchronous.** `PUT /scope/{scopeId}/assignment` dispatches an `ApiRequestEvent` and returns `200` before the DB is updated. Callers that immediately read back via `GET /scope/{scopeId}/assignment` may see stale data.

**Toggle is also asynchronous.** `PUT /toggle-scopes` follows the same pattern — a `200` response means the event was queued, not that scopes are toggled.

**`replace` field on assignment updates.** Pass `replace: ["declarations"]`, `["apps"]`, or `["all"]` to treat the provided `applyIdentifiers`/`applyAssets` as the complete set and remove anything not listed. The older `declarations.removeNotApplied` boolean is deprecated — use `replace` instead.

**`group_key` GSI encodes enabled state.** Scope-group items carry a `group_key` of `ENABLED-GROUPS#<tenant>` or `DISABLED-GROUPS#<tenant>`. When a scope is toggled, all its group items are rewritten to flip the key. Queries for "which enabled scopes contain this device" use the enabled partition — do not assume all group items for a scope are on the same GSI partition.

**Device count is best-effort.** `GET /scope/{scopeId}/devices/count` caps both item count and wall-clock time. If either limit is hit, `countType` will be `LOWER_BOUND` rather than `EXACT`. Do not use this endpoint for billing-grade counts.

**Platform topics are shared contracts.** Topics in `pdd/default` (`device-scope-membership-changed`, `device-management-channel-changed`, `device-group-changed`) are used across the platform. Schema or topic-name changes require coordination outside this service.

**External service dependencies.** Scoping Engine calls Declaration Storage Service (via `declaration-product-springboot-starter`) to create and update management-properties declarations, and calls a VPP app service via `VppAppClient` using M2M auth. Both must be reachable for group-change events to complete successfully.
