# Declaration Storage Service

Last reviewed: 2026-04-07

**Owner:** DDmR team

## Summary

Declaration Storage Service (DSS) is the system of record for Apple Declarative Device Management (DDM) declarations and their device assignments. It owns two things: declaration payloads (the JSON content, group, type, and a SHA-256 `serverToken` that Apple devices use to detect changes) and assignment records that map a `(tenant, device, channel)` tuple to a declaration under a named identifier. DSS exposes both a product-facing API (used by Jamf services to manage declarations and assignments) and an MDM-facing API (used by the MDM layer to satisfy Apple DDM check-in requests). When assignments change, DSS publishes a Pulsar event so downstream consumers (e.g., the MDM layer) can trigger device sync. The service is tagged `pii` and `nist` and operates in the `blueprints` system at service tier 2.

---

## API Endpoints

All routes require `X-TenantId` (required, 401 if missing), `X-EnvironmentId` (optional), and `X-TenantEncryption` (optional boolean, controls BYOK encryption for this request) headers.

### v2 — Declaration Administration (`/api/v2/declaration`)

These are the current non-deprecated endpoints. Prefer v2 over v1.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/declaration` | Create a declaration. Returns `{id, serverToken}` with `201 Created`. Accepts `channelReplacement` field (a placeholder string replaced with the actual channel at device fetch time). |
| `GET` | `/api/v2/declaration/{id}` | Retrieve a declaration. Returns `{type, group, payload, serverToken, channelReplacement?}`. `204` if not found, `401` if wrong tenant. |
| `PATCH` | `/api/v2/declaration/{id}` | Update one or more fields. At least one field must be set. Returns `{serverToken}` (updated or unchanged). If payload/type changes, emits assignment-changed events for all currently assigned devices. Set `channelReplacement` to explicit null to remove it. |

### v2 — Assignment Administration (`/api/v2/assignment`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/assignment/device/{device}/{channel}` | Apply/remove/replace assignments for one device. Accepts `applyIdentifiers` (map of identifier → declaration ID), `removeIdentifiers` (set of identifiers), `removeNotApplied` (bool), and `tag`. Returns `200` with error detail for any invalid declarations or tag/tenant mismatches instead of a hard failure. |
| `GET` | `/api/v2/assignment/device/{device}/{channel}` | Stream all assignments for a device as NDJSON. Requires `?tags=<tag>` (repeatable; use blank string for untagged) or `?allTags=true`. |
| `GET` | `/api/v2/assignment/declaration/{id}` | Stream all device assignments for a declaration as NDJSON. Same `tags`/`allTags` query parameter requirement. |
| `DELETE` | `/api/v2/assignment/device/{device}/{channel}` | Delete all assignments for a device matching the given `?tags` (required, blank string for untagged). |

### v1 — MDM API (`/api/v1/device/{device}/{channel}`)

Used by the MDM layer to respond to Apple DDM check-in requests. Not intended to be called by product services.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/device/{device}/{channel}/tokens` | Returns Apple `TokensResponse` (SyncTokens with DeclarationsToken + Timestamp). |
| `GET` | `/api/v1/device/{device}/{channel}/declaration-items` | Returns Apple `DeclarationItemsResponse` with all assigned declarations grouped by type. |
| `GET` | `/api/v1/device/{device}/{channel}/declaration/{group}/{identifier}` | Returns a single declaration payload formatted for Apple's DDM check-in. Applies `channelReplacement` substitution before returning. |

### v1 — Product API (`/api/v1/declaration`, `/api/v1/assignment`)

The v1 product API is fully deprecated (deprecation headers set per endpoint, all deprecated as of 2025). Avoid in new code; use v2 equivalents.

### Connectivity Check

`HEAD /api/v1` — simple reachability probe for external monitors. Returns `200` with no body.

---

## Data Model (DynamoDB)

Single-table design. Table name is configured per environment. Primary key: `pkey` (hash) + `psort` (range).

| pkey pattern | psort pattern | Record type | Notes |
|---|---|---|---|
| `DECL#<uuid>` | `PAYLOAD` | Declaration payload | Stores `group`, `type`, `payload` (prefixed), `payloadToken`, `payloadEdek`, `tenant`, `channelReplacement` |
| `MDM#<tenant>\|<device>\|<channel>` | `A#<identifier>` | Assignment record | Stores `declaration_key` (→ `DECL#<uuid>`), `identifier`, `tenant`, `tag`, `created`, `last_device_access` |
| `MIGRATION` | `#FROM#<tenantId>` | Migration lock | Presence means tenant data is mid-migration; most write endpoints return `503` during this time |

**GSIs:**
- `declaration_index` (hash: `declaration_key`, projection: ALL) — enables querying all assignments for a given declaration without a full scan
- `tenant_index` (hash: `tenant`, projection: KEYS_ONLY) — tenant-level key lookup

**Payload storage format:** Payloads are always stored with a prefix: `json:` for plaintext, `iron:` for IronCore (BYOK) encrypted. Always strip the prefix before using the value. Encrypted payloads have a companion `payloadEdek` attribute (the encrypted data encryption key).

**`payloadToken`:** A SHA-256 hex digest over `(payload + type)`. Apple devices compare this token to detect whether a declaration has changed. It is recomputed on any update that changes `payload` or `type`.

---

## Events

### Produced

| Topic | Schema | Key | When |
|---|---|---|---|
| `persistent://pdd/default/declaration-assignment-changed` | `DeclarationAssignmentChangedEvent` | `tenantId + deviceId` | After any assignment add/remove that succeeds, and after a declaration edit that changes `payloadToken` |

`DeclarationAssignmentChangedEvent` fields: `tenantId`, `environmentId` (nullable), `deviceId`, `channel`.

### Consumed

DSS does not consume any Pulsar topics. It is a pure producer from an eventing standpoint.

---

## Client Libraries

Two client libraries exist for product services to call DSS. Do not construct raw HTTP calls — use one of these.

### `declaration-product-springboot-starter`

**Artifact:** `com.jamf.ddm:declaration-product-springboot-starter`

Spring Boot auto-configuration wrapper. Drop the starter on the classpath and configure via `application.yaml` properties. Provides a pre-wired `DeclarationProductClient` bean. This is the right choice for most Spring Boot services.

### `declaration-product-spring-client`

**Artifact:** `com.jamf.ddm:declaration-product-spring-client`

Raw Spring (non-Boot) reactive client. Build manually via `DeclarationProductClient.builder()` — requires a `WebClient`, auth strategy, and host. Uses a fluent DSL:

```java
client.addDeclaration()
    .withGroup(DeclarationGroup.CONFIGURATION)
    .withType("com.apple.configuration.legacy")
    .withPayload("{...}")
    .execute()  // returns Mono<DeclarationCreatedResult>
```

Available operations: `addDeclaration()`, `getDeclaration()`, `editDeclaration()`, `removeDeclaration()`, `assignDeclaration()`, `assignDeviceDeclarations()`, `removeDeviceAssignments()`, `getDeclarationAssignments()`, `getDeviceAssignments()`.

Auth modes: CSA (`DeclarationClientCsaAuth`), M2M (`DeclarationClientM2mAuth`), or a custom `DeclarationClientAuth` implementation.

Note: The client library wraps v1 endpoints. If you need v2-only features (e.g., `channelReplacement`, v2 assignment tagging queries), you will need to call the v2 API directly until the client is updated.

---

## Key Design Decisions and Gotchas

**Tagging isolates ownership.** Assignments carry an optional `tag` string. Add/remove/replace operations only touch records whose tag matches the caller's tag. Blank/null and empty string are treated as equivalent. Never operate on `allTags=true` data as a product service — those assignments belong to other owners.

**`channelReplacement` is a server-side token substitution.** When set on a declaration, DSS replaces that string in the payload JSON with the real channel identifier at fetch time (`GET .../declaration/{group}/{identifier}`). This lets a single declaration payload be shared across channels. The placeholder must be 4–32 non-whitespace characters.

**BYOK encryption is mid-rollout.** Which tenants get payload encryption is controlled by `privacy.force-tenants` config or the per-request `X-TenantEncryption` header. Internally, DSS always reads the prefix (`json:` / `iron:`) to determine how to decode; callers never see raw prefixed values through the API.

**503 during tenant migration.** While a tenant migration is in progress (indicated by a `MIGRATION / #FROM#<tenantId>` item in DynamoDB), most write endpoints and some reads return `503 Service Unavailable`. Your client should treat 503 as transient and retry with backoff.

**`removeNotApplied` is deprecated on v1.** The v1 `assignDeviceDeclarations` endpoint had a `removeNotApplied` flag. On v2 the same parameter exists but is superseded by explicit tag-scoped replace semantics. Prefer v2.

**NDJSON streaming on assignment reads.** `GET .../assignment/...` endpoints return `application/x-ndjson`, one JSON object per line, not a JSON array. Parse line-by-line.

**`serverToken` vs `identifier`.** These are easily confused. The `serverToken` (also called `payloadToken` internally) is a hash Apple uses to detect payload changes — it belongs to the declaration. The `identifier` is the logical name the declaration is assigned under on a device (e.g., `jamf-managed-identity`) — it belongs to the assignment. Declaration IDs (UUIDs from DSS) are separate from both.

**Assignment modification failures are non-fatal in v2.** `POST /api/v2/assignment/device/{device}/{channel}` returns `200` even when some assignments could not be applied (wrong tag/tenant mismatch). Check the `errors` array in the response body.

**Contract tests via Pact.** DSS maintains consumer-driven contract tests in `contract-test/`. If you are adding a new operation or changing a response shape, verify the Pact contracts still pass before merging.
