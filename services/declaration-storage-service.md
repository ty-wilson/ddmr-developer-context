# Declaration Storage Service

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the "no Pulsar consumers" negative (see the trap below for the derivation), the `last_device_access` attribute still living on the assignment row, and the Known Callers list (re-derived by grepping every sibling repo for the client artifacts). Not re-verified and older, carried forward from the 2026-06-24 review: the endpoint tables, the `channelReplacement` length rule, the observed channel-string values, and the Helm ingress section. Treat those as a pointer.

Run the commands in **Where to find the data** at the end before acting on any value here.

**Owner:** DDmR team

## Summary

Declaration Storage Service (DSS) is a **Spring Boot WebFlux (Kotlin coroutines)** service and the system of record for Apple Declarative Device Management (DDM) declarations and their device assignments. It owns two things: declaration payloads (the JSON content, group, type, and a SHA-256 `serverToken` that Apple devices use to detect changes) and assignment records that map a `(tenant, device, channel)` tuple to a declaration under a named identifier. DSS exposes both a product-facing API (used by Jamf services to manage declarations and assignments) and an MDM-facing API (used by the MDM layer to satisfy Apple DDM check-in requests). When assignments change, DSS publishes a Pulsar event so downstream consumers can trigger device sync. The service is tagged `pii` and `nist` and operates in the `blueprints` system at service tier 2.

---

## API surface (shape, not an inventory)

Four route families, all requiring `X-TenantId` (401 if missing) plus optional `X-EnvironmentId` and `X-TenantEncryption` (boolean, controls BYOK encryption for this request):

- **`/api/v2/declaration`**: declaration administration (create / get / patch). Current, preferred.
- **`/api/v2/assignment`**: assignment administration, device-scoped and declaration-scoped. Current, preferred.
- **`/api/v1/device/{device}/{channel}/...`**: the MDM API (`tokens`, `declaration-items`, `declaration/{group}/{identifier}`). Used by the MDM layer to answer Apple DDM check-ins; not for product services.
- **`/api/v1/declaration`, `/api/v1/assignment`**: fully deprecated product API (deprecation headers set per endpoint). Use the v2 equivalents.
- **`HEAD /api/v1`**: unauthenticated reachability probe for external monitors, `200` with no body.

Do not transcribe the routes; regenerate them (see the verification section). Behaviours worth knowing that a route list will not tell you:

- **`POST /api/v2/assignment/device/{device}/{channel}`** returns `200` even when some assignments failed (wrong tag or tenant). Read the `errors` array in the body. See the trap below.
- **Assignment reads stream NDJSON** (`application/x-ndjson`), one JSON object per line, not a JSON array. They also *require* a tag selector: `?tags=<tag>` (repeatable, blank string means untagged) or `?allTags=true`.
- **`PATCH /api/v2/declaration/{id}`** emits assignment-changed events for every currently assigned device when `payload`, `type`, or `channelReplacement` changes. A "small metadata edit" can be a fleet-wide fan-out.

---

## Data Model (DynamoDB)

Single-table design. Table name is configured per environment. Primary key: `pkey` (hash) + `psort` (range). `docs/database.md` is authoritative on key names and which GSIs currently exist.

| pkey pattern | psort pattern | Record type | Notes |
|---|---|---|---|
| `DECL#<uuid>` | `PAYLOAD` | Declaration payload | `group`, `type`, `payload` (prefixed), `payloadToken`, `payloadEdek`, `tenant`, `channelReplacement` |
| `MDM#<tenant>\|<device>\|<channel>` | `A#<identifier>` | Assignment record | `declaration_key` (→ `DECL#<uuid>`), `identifier`, `tenant`, `tag`, `created`, `last_device_access` |
| `MIGRATION` | `#FROM#<tenantId>` | Migration lock | Presence means tenant data is mid-migration; most write endpoints return `503` during this time |

**GSIs:** `declaration_index` (hash: `declaration_key`, projection: ALL), which lets DSS query all assignments for a given declaration without a scan. There is no `tenant_index` (removed under DDMR-1035).

**Trap: payloads are always stored with a type prefix, and the prefix is load-bearing.** `json:` means plaintext, `iron:` means IronCore (BYOK) encrypted. Always strip the prefix before using the value; an encrypted payload also carries a companion `payloadEdek` attribute (the encrypted data encryption key). Code that reads the attribute raw and treats it as JSON will fail only for BYOK tenants, so it passes every non-BYOK test.

**Trap: channel strings are not a `SYSTEM`/`USER` enum, and there is no literal `USER` value.** The `<channel>` segment is whatever string the caller (typically scoping-engine) wrote. Values observed in stage tables as of 2026-06-24 include `SYSTEM` (legacy generic device channel), `computer` (macOS), `device` and `mobile_device` (iOS/iPadOS), `watch` (Apple Watch), and **per-user UUIDs** for user channels. Never assume the user-channel string, discover it (query in the verification section).

**`payloadToken`** is a SHA-256 hex digest over `(payload + type)`. Apple devices compare it to detect whether a declaration changed. Recomputed on any update that changes `payload`, `type`, or `channelReplacement`.

---

## Events

### Produced

| Topic | Schema | Key | When |
|---|---|---|---|
| `persistent://pdd/default/declaration-assignment-changed` | `DeclarationAssignmentChangedEvent` | `tenantId + deviceId` | After any assignment add/remove that succeeds, and after a declaration edit that changes `payloadToken` |

`DeclarationAssignmentChangedEvent` fields: `tenantId`, `environmentId` (nullable), `deviceId`, `channel`.

### Consumed

**None. DSS is a pure producer from an eventing standpoint.**

**Trap: this is the most load-bearing negative in this directory, and it is only true of merged `main`.** Downstream teams design around "DSS will never react to an event", so re-derive it rather than quoting it. Two independent derivations, both clean as of 2026-07-29 on `origin/main`:

1. No listener annotations or consumer construction anywhere under `src/main` (`PulsarListener`, `MessageListener`, `newConsumer`, `subscriptionName`).
2. `config/properties/MessagingProperties.kt` declares a nested `Producer` class and **no consumer counterpart**, so there is no config surface a consumer could be wired from. Producer setup lives entirely in `config/EventConfig.kt`.

Two recent branches change assignment-write and `last_device_access` semantics without changing the eventing direction, but they are the kind of work that would: **DDMR-1256** ("do not write if no different", merged into `main` 2026-07-29 as PR #306) suppresses redundant assignment writes, which also suppresses the events those writes would have produced. **`ddmr-1249-move-last-device-access`** (unmerged as of 2026-07-29) rewrites `service/DynamoDbService.kt` to move `last_device_access` off its current row. Check both before reasoning about event volume or about where the attribute lives.

---

## Known Callers

Services that write to or read from DSS at runtime, grouped by how they reach it. Re-derived 2026-07-29 by grepping every sibling repo for the client artifact names and client class names (command in the verification section).

**Via the `declaration-product-springboot-starter`:**
- `scoping-engine`: `DeclarationStorageWrapper` manages management-properties declarations for devices
- `declaration-service`: stores and deletes all declarations it translates
- `blueprint-component-custom-declarations`: user-authored custom declarations
- `device-declaration-reporting-service` (Jabberwocky): query-time enrichment. **Also the only caller outside the MDM layer that uses `declaration-mdm-springboot-starter`**, for the `declaration-items` path. See `device-declaration-reporting-service.md`.

**Via their own declarative HTTP client (not the starter):**
- `blueprint-component-declarations-service`: `DeclarationStorageServiceClient` + `DeclarationStorageAdapter` in `client/`, for the translate/cleanup lifecycle
- `blueprint-component-sw-update-service`: same pattern as above
- `blueprint-deployment-service` (Ocean): `DeclarationStorageServiceClient` + `DeclarationStorageAdapter` in `client/`, requests the `declaration-storage-product` M2M scope
- `configuration-profile-service`: `DeclarationStorageServiceClient` under `declaration/dss/`, called from `DeclarationLegacyProfileConfigurationService`

The split matters when reasoning about response handling (retry, error mapping): the two groups have different client codebases. Changes that affect the HTTP contract need to land in both.

---

## Client Libraries

Do not construct raw HTTP calls; use one of these. Artifact coordinates:

- **`com.jamf.ddm:declaration-product-springboot-starter`**: Spring Boot auto-configuration. Drop it on the classpath, configure via `application.yaml`, get a pre-wired `DeclarationProductClient` bean. Right choice for most Spring Boot services.
- **`com.jamf.ddm:declaration-product-spring-client`**: raw Spring (non-Boot) reactive client. Build via `DeclarationProductClient.builder()` with a `WebClient`, auth strategy, and host. Auth modes: `DeclarationClientCsaAuth`, `DeclarationClientM2mAuth`, or a custom `DeclarationClientAuth`.
- **`com.jamf.ddm:declaration-mdm-springboot-starter`**: the MDM-side counterpart. Used by the MDM layer and by DRS.

See `docs/shared-libraries.md`, and regenerate the operation list from the client repos rather than trusting a copy here.

**Trap: the client DSL is not uniformly v2, and it cannot set `channelReplacement`.** Most operations target v2, but `removeDeclaration()` calls `DELETE /api/v1/declaration/{id}` and `assignDeclaration()` calls `POST /api/v1/assignment/declaration/{id}`. `channelReplacement` is not exposed through the DSL at all, so setting or updating it means calling the v2 API directly.

---

## Key Design Decisions and Gotchas

**Trap: `401 "Tenant mismatch"` from DSS is not a credential failure.** A declaration is looked up by UUID alone, then `DeclarationApiHelper.validateDeclarationAppropriate` compares the request's tenant to the declaration's stored `tenant` attribute. If they differ → **`401 "Tenant mismatch"`** (not 403/404). If the UUID is not found at all → `204` (GET) / `400` (assign). So a 401 means "this token's tenant does not own this declaration." This is the signature of **orphaned-tenant** problems: declarations were stamped with one tenant (e.g. an authorizer-*generated* tenant) and the instance now presents a *different* (real platform) tenant, commonly after that instance's DSS traffic started flowing through the Tyk gateway, which surfaces the real M2M tenant. The fix is to realign the data (migrate the declarations' `tenant`, or have Jamf Pro regenerate its system declarations under the current tenant), not auth config. Full triage in `docs/dss-401-tenant-mismatch-playbook.md`; see `ddmr-authorizer-tenant.md` for how generated vs real (`platformTenant`) tenants arise.

**Trap: assignment modification failures are non-fatal in v2 and easy to miss.** `POST /api/v2/assignment/device/{device}/{channel}` returns `200` even when some assignments could not be applied because of a tag or tenant mismatch. A client that only checks the status code will report success on a partial failure. Check the `errors` array. The related WARN log text is `(wrong tenant or tag)` and is emitted from several sites with different verbs; see the grep note in `docs/observability.md` before searching for it.

**Trap: `tag` is the ownership boundary, and `allTags=true` crosses it.** Add/remove/replace operations only touch records whose tag matches the caller's tag; blank/null and empty string are equivalent. Never operate on `allTags=true` data as a product service, because those assignments belong to other owners.

**`channelReplacement` is a server-side token substitution.** When set on a declaration, DSS replaces that string in the payload JSON with the real channel identifier at fetch time (`GET .../declaration/{group}/{identifier}`), which lets one payload be shared across channels. The placeholder has a length constraint on non-whitespace characters; read the current bound from the validation annotation rather than quoting one.

**BYOK encryption is opt-in per tenant.** Controlled by `privacy.force-tenants` config or the per-request `X-TenantEncryption` header. Internally DSS always reads the payload prefix to decide how to decode; callers never see prefixed values through the API.

**503 during tenant migration.** While a `MIGRATION / #FROM#<tenantId>` item exists, most write endpoints and some reads return `503`. Treat 503 as transient and retry with backoff.

**`serverToken` vs `identifier`.** Easily confused. `serverToken` (internally `payloadToken`) is a hash Apple uses to detect payload changes and belongs to the *declaration*. `identifier` is the logical name the declaration is assigned under on a device (e.g. `jamf-managed-identity`) and belongs to the *assignment*. Declaration IDs (DSS UUIDs) are a third, separate thing.

**Contract tests via Pact** live in `contract-test/`. Adding an operation or changing a response shape means verifying the contracts still pass before merging.

---

## Ingress and Auth

DSS has migrated off the JWT sidecar and validates JWTs in-pod via `com.jamf.declarationstorage.auth.JwtFilter` (the pattern declaration-service adopted under DDMR-1088). The filter handles both M2M and CSA tokens: M2M tenant comes from the `https://www.jamf.com/tenant.tenantId` JWT claim, CSA tenant is resolved via `CsaTenantResolver` against the `tenant-authorizer` table.

Helm templates in `helm/declaration-storage-service/templates/` still define a dual-ingress pattern (gated by `ingress.legacyEnabled: true`, the default):

- **`{release}-authorized`**: HAProxy ingress on `/api` (Prefix) → port 8080. Originally used `haproxy-ingress.github.io/auth-url: svc://tenant-authorizer-svc:8080/authorize` to pre-inject `X-TenantId`; with the in-pod filter that pre-injection is no longer the authoritative path.
- **`{release}-open`**: unauthenticated, routes `/api/v1` (Exact) for the connectivity check.
- **`{release}-fake-gateway`** (sandbox only): routes when `auth.asIngress` is set, for environments without Tyk.

**Trap: port 8080 is not a "trust the inbound `X-TenantId` header" path.** The Kubernetes Service exposes 8080 (`http`) unconditionally and 7070 (`http-authed`) conditionally, which makes 8080 look unauthenticated. In any environment where `authEnabled=true` (prod, stage, hc-stage) port 8080 also requires a Bearer JWT, so port-forwarding for diagnostics needs a real M2M token. The only exception is local/test with `authEnabled=false`, where the filter reads `X-TenantId` straight from headers.

---

## Where to find the data (verify rather than trust)

**Read via `origin/main`, not the working tree.** This checkout is frequently parked on an old ticket branch (it was on `DDMR-1028-add-fedramp-tag`, ~3 months behind, on 2026-07-29), so a plain `Read` of a file here can be badly stale.

```bash
R=~/Projects/DDmR/declaration-storage-service; git -C $R fetch origin -q
git -C $R log --oneline origin/main --since=2026-07-29
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Regenerate the route inventory instead of reading the tables above
git -C $R grep -nE '"(/api/v[12][^"]*)"' origin/main -- 'src/main/kotlin/**/handlers/*' 'src/main/kotlin/**/router/*'

# Re-derive the "no Pulsar consumers" negative (both should be empty / producer-only)
git -C $R grep -nE 'PulsarListener|MessageListener|newConsumer|subscriptionName' origin/main -- 'src/main/**'
git -C $R show origin/main:src/main/kotlin/com/jamf/declarationstorage/config/properties/MessagingProperties.kt

# Tenant-mismatch and tag-mismatch code paths, by symbol not line number
git -C $R grep -n 'Tenant mismatch\|validateDeclarationAppropriate\|wrong tenant or tag' origin/main -- 'src/main/**'
```

Re-derive the Known Callers list across sibling repos (run from `~/Projects/DDmR`):

```bash
grep -rIl --include=*.java --include=*.kt --include=*.gradle --include=*.toml \
  -E 'declaration-(product|mdm)-(springboot-starter|spring-client)|DeclarationProductClient|DeclarationMdmClient|DeclarationStorageServiceClient' . \
  | grep -v ddmr-developer-context | cut -d/ -f2 | sort -u
```

Discover the real channel strings for a device rather than guessing (`SYSTEM`/`computer`/`device`/`mobile_device`/`watch`/user UUIDs all occur; there is no `USER`):

```bash
aws dynamodb query --profile stable_dev --region us-east-2 \
  --table-name ddmr-dev-declaration-storage \
  --key-condition-expression 'begins_with(pkey, :p)' \
  --expression-attribute-values '{":p":{"S":"MDM#<tenant>|<device>"}}' \
  --projection-expression 'pkey,psort' --max-items 25
```
