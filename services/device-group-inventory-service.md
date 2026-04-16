# Device Group Inventory Service

Last reviewed: 2026-04-15

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** Data Manager team (`#ask-data-manager`)
**Repo:** `jamf/device-group-inventory-service`

## Summary

A Ktor/Kotlin platform service that provides a RESTful API for querying and managing device group information. It acts primarily as a **proxy to the Jamf Pro API** — most operations are forwarded to Jamf Pro via M2M auth and mapped to a platform-standard representation. It also has a local PostgreSQL database (R2DBC + Exposed ORM) that caches group metadata (tenantId, groupId, name, description, timestamps).

**Note:** As of 2026-04-15, this service does **not** return division/site/location data for groups. The Jamf Pro group API fields mapped are: `groupPlatformId`, `groupJamfProId`, `groupName`, `groupDescription`, `groupType`, `smart`, `membershipCount`, `criteria`.

---

## API Endpoints

All routes under `/v1/device-groups` require M2M authentication. Exposed via Tyk at `/management/device-groups`.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/device-groups` | List groups (paginated, RSQL filterable, sortable) |
| `GET` | `/v1/device-groups/{id}` | Get a single group by UUID |
| `POST` | `/v1/device-groups` | Create a group (proxied to Jamf Pro) |
| `PATCH` | `/v1/device-groups/{id}` | Update a group (proxied to Jamf Pro) |
| `DELETE` | `/v1/device-groups/{id}` | Delete a group (proxied to Jamf Pro) |
| `GET` | `/v1/device-groups/{id}/members` | List device UUIDs in a group |
| `PATCH` | `/v1/device-groups/{id}/members` | Add/remove devices from a static group |
| `GET` | `/v1/devices/{id}/device-groups` | List groups a device belongs to |

### Filtering & Sorting

RSQL filter on `GET /v1/device-groups` supports fields: `name`, `description`, `deviceType`, `groupType`, `id`. The `id` field supports `=in=` and `=out=` operators. Sort fields: `name`, `description`, `deviceType`, `groupType`.

### Response Schema (list item)

```json
{
  "id": "UUID",
  "name": "string",
  "description": "string",
  "deviceType": "COMPUTER | MOBILE",
  "groupType": "SMART | STATIC",
  "memberCount": 0
}
```

Single-group GET additionally includes `criteria` (smart group criteria array, nullable).

---

## Data Model (PostgreSQL)

### `device_group` table

| Column | Type | Notes |
|--------|------|-------|
| `tenant_id` | UUID | Composite PK with group_id |
| `group_id` | UUID | Composite PK with tenant_id |
| `name` | VARCHAR(255) | |
| `description` | TEXT | |
| `created_time` | DATETIME | Default: current |
| `updated_time` | DATETIME | Default: current |

---

## External Dependencies

- **Jamf Pro API** — primary data source. All CRUD operations proxy to `v1/groups` endpoints on Jamf Pro via M2M auth (Robocop). Per-tenant circuit breaker (Resilience4j, 80% failure threshold).
- **API Gateway (Tyk)** — routes external traffic to the service.
- **M2M Auth (Robocop)** — authenticates inbound requests and outbound calls to Jamf Pro.

---

## Events

None. This service does not produce or consume Pulsar events.

---

## Key Design Notes

- **Proxy pattern**: Most operations forward directly to Jamf Pro and map the response. The local database appears to serve as a cache/secondary store rather than the primary source of truth.
- **Version gating**: Some features (like ID-based filtering) require a minimum Jamf Pro version. `JamfProTenantVersionService` checks the tenant's Pro version before allowing these operations.
- **No division data**: The service does not store or return site/location/division information for groups. This data is not present in the Jamf Pro group API fields that are mapped.
