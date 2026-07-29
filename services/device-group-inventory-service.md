# Device Group Inventory Service

Last reviewed: 2026-04-15; amended 2026-07-29 to consolidate the division-data negative, drop drifting values, and add verification commands. **Nothing here was re-verified against code on the amend date**, so treat every claim as a pointer and run the checks at the end before relying on it.

**Owner:** Data Manager team (`#ask-data-manager`)
**Repo:** `jamf/device-group-inventory-service`

## Summary

A Ktor/Kotlin platform service exposing a RESTful API for querying and managing device group information. It acts primarily as a **proxy to the Jamf Pro API**: most operations are forwarded to Jamf Pro via M2M auth and mapped to a platform-standard representation. It also keeps a local PostgreSQL database (R2DBC + Exposed ORM) holding group metadata.

**Trap: this service did not surface division, site, or location data for groups as of 2026-04-15.** The fields it maps from the Jamf Pro group API did not include them, so a caller cannot get a group's division from here. This matters for division-scoped work (see `docs/auth-and-tenancy.md`), where the absence of a group-to-division source shapes the design. The date is load-bearing: re-derive this before designing around it, because it is exactly the kind of gap that gets closed without the doc being updated. Check with the field grep in the verification section.

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

Paths and their purpose are the durable contract here; the field lists below drift with the upstream Jamf Pro API.

### Filtering and sorting

RSQL filter on `GET /v1/device-groups` supports `name`, `description`, `deviceType`, `groupType`, and `id`. The `id` field supports the `=in=` and `=out=` operators. Sort fields are `name`, `description`, `deviceType`, `groupType`.

### Response shape (list item)

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

A single-group `GET` additionally includes `criteria` (smart group criteria array, nullable).

## Data Model (PostgreSQL)

The `device_group` table is keyed on a **composite primary key of `tenant_id` + `group_id`**, which is the durable design point: rows are tenant-scoped, so there is no global group lookup. It also carries `name`, `description`, and created/updated timestamps. Read the current column set from the migrations rather than from here.

## External Dependencies

- **Jamf Pro API**: the primary data source. CRUD operations proxy to the `v1/groups` endpoints on Jamf Pro via M2M auth (Robocop).
- **API Gateway (Tyk)**: routes external traffic to the service.
- **M2M Auth (Robocop)**: authenticates inbound requests and outbound calls to Jamf Pro.

**Trap: a per-tenant circuit breaker (Resilience4j) sits in front of Jamf Pro.** When it opens for a tenant, requests fail fast without reaching Jamf Pro, so the symptom is immediate errors for one tenant while others are healthy. Do not read that as Jamf Pro being down. The failure-rate threshold is configured in the service, not fixed here.

## Events

**None. This service neither produces nor consumes Pulsar events.** Derived by grepping the source for Pulsar producers and consumers; re-derive with the command below rather than assuming it still holds.

## Key Design Notes

- **Proxy pattern**: most operations forward directly to Jamf Pro and map the response. The local database *appears* to serve as a cache or secondary store rather than the primary source of truth (this was not confirmed against code).
- **Version gating**: some features (such as ID-based filtering) require a minimum Jamf Pro version. `JamfProTenantVersionService` checks the tenant's Pro version before allowing these operations, so a capability can be present in the API and still refuse to work for an older tenant.

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/device-group-inventory-service; git -C $R fetch origin -q

# What landed since this doc was reviewed, and what is still unmerged
git -C $R log --oneline origin/main --since=2026-04-15
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Does it surface division/site/location yet? (the trap above)
git -C $R grep -in 'division\|site\|location' origin/main -- src/main

# Confirm the "no Pulsar events" claim
git -C $R grep -in 'pulsar\|producer\|consumer' origin/main -- src/main

# Real mapped field set and circuit-breaker config
git -C $R grep -n 'groupPlatformId\|groupJamfProId\|membershipCount' origin/main -- src/main
git -C $R grep -n 'failureRateThreshold\|circuitbreaker' origin/main -- src/main/resources
```
