# Device Declaration Reporting Service

> **The previous version of this page was badly wrong and is the reason this warning is at the top.** It described DRS as a skeleton with one `/test` endpoint, no persistence, no Pulsar wiring, and unauthenticated requests. **Every one of those negatives is false.** They were written against a local checkout that was roughly 4,100 commits behind `origin/main`. If you read that version and concluded Jabberwocky had not built this service, discard that conclusion. The lesson generalises: this repo moves fast, so read it via `git show origin/main:<path>` and never from whatever the working tree happens to be on.

Last reviewed: 2026-07-29. Fully re-derived against `origin/main` on that date: the endpoint list, the three consumed Pulsar topics, the "produces no events" negative, the PostgreSQL data model, M2M auth being **enabled** in prod, the DSS client integration, and the caller list. Nothing on this page is carried forward from the 2026-04-07 review, because effectively all of it was wrong.

**Owner:** Jabberwocky team, `#help-jabberwocky` on Slack. The repo maintains its own `CLAUDE.md`, `README.md`, and a `.claude/skills/` directory; those are authoritative for internals and workflow. Run the commands in **Where to find the data** at the end before acting on any value here.

## Summary

Device Declaration Reporting Service (DRS) is a Spring Boot / Java service (standard MVC, not WebFlux) that answers "what declarations does this device have, and are they in the state we expect". It consumes Apple DDM status reports and device management-state events from Pulsar, persists declaration status to PostgreSQL, and serves paginated, RSQL-filterable REST APIs per device and per declaration. At query time it enriches the device-reported state by comparing it against the expected assignments fetched live from Declaration Storage Service, so a declaration that is assigned but not yet reported shows as pending rather than missing.

It is a production service with its own Terraform infrastructure, per-region prod values (`use2`, `euc1`, `apne1`), a Prometheus rules template, autoscaling, contract verification in CI, and a **separate `drs-dlq` deployable** (its own Helm chart, database migration, and `DlqMessageController`) for inspecting and replaying dead-lettered messages.

---

## API surface (shape, not an inventory)

Reachable through Tyk at listen path **`/ddm/report`** with `strip_listen_path: true`, so the gateway path `/ddm/report/v1/devices/{id}/declarations` hits the service as `/v1/devices/{id}/declarations`. Tenant comes from the JWT via a `@TenantId` parameter annotation on every controller method; there is no tenant path or query parameter.

- **`/v1/devices/...`** (`DeviceReportController`): per-device declaration report, an RSQL-filtered and paginated variant, the device's channel list, and a `POST /v1/devices/declarations` that accepts a status report directly rather than via Pulsar.
- **`/v1/declarations/...`** (`DeclarationReportController`): per-declaration report and an RSQL-filtered device list for that declaration.
- **`/internal/v1/tenants/{tenantId}`** (`TenantDecommissionController`): device count for a tenant, and `DELETE` to decommission it. Writes an audit row; see the trap below.
- Actuator on the management port, including a custom `NativeMemoryEndpoint`.

Regenerate the exact routes with the grep in the verification section. Two things a route list will not tell you:

- **Endpoints carry machine-readable deprecation metadata.** `GET /v1/devices/{deviceId}` is deprecated in favour of `GET /v1/devices/{deviceId}/declarations`, and the OpenAPI annotations declare RFC 9745 `Deprecation`, RFC 8594 `Sunset`, and a `Link: rel="successor-version"` header plus explicit `deprecation-date` / `sunset-date` / `successor-endpoint` extension properties. Read those from the annotations rather than assuming a timeline: at review time the deprecation date was 2026-05-17 and the sunset a year later.
- **RSQL filtering is a real query language on a fixed field allow-list**, not free-form search. The allow-list lives in `RsqlFilterFields` with a per-type parser and strategy under `service/advancedfind/`. A field that is not registered there is rejected, and the repo ships an `add-rsql-filter` skill precisely because adding one touches several files.

---

## Data Model (PostgreSQL)

Flyway migrations in `src/main/resources/db/migration/`, plus a separate schema for the `drs-dlq` module. Do not quote a migration count; `ls` the directory (the highest at review time was V21).

Tables, by role rather than by column:

- `status_report_declaration` and `client_information`: the device-reported state, one row per declaration per device-channel plus the reporting client's metadata.
- `declaration_assignment` and `declaration_information`: expected assignments and resolved declaration metadata, fed by the `declaration-assignment-changed` consumer.
- `tenant_decommission_audit`: one row per decommission operation.
- `shedlock`: distributed lock table so scheduled work (data retention) runs on one pod only.

Regenerate the column-level schema from the migration files rather than reading a table here. Migration history worth knowing about, because it changes query semantics: V13 added cascade deletes to all FKs, V14 replaced unique constraints with composite ones, V17 added optimistic-locking versions, and V20/V21 collapsed the separate reason tables into a single `reasons` JSONB column.

---

## Events

### Consumed

Three topics, each with its own DLQ topic and `max-redeliver-count`, all using subscription name `device-declaration-reporting-service` and `SubscriptionType.Shared`:

| Topic | Consumer class | Purpose |
|---|---|---|
| `persistent://pdd/apple-ddm/statusreport` | `StatusReportConsumer` | Apple DDM status reports from the MDM layer |
| `persistent://pdd/default/device-management-state` | `DeviceManagementStateConsumer` | Device management-state transitions |
| `persistent://pdd/default/declaration-assignment-changed` | `DeclarationAssignmentChangedConsumer` | Expected-assignment changes, produced by DSS |

DLQ topic names are derived as `<topic>-<subscription>-dlq`. The `drs-dlq` deployable reads those topics.

### Produced

**None.** DRS is a pure consumer plus REST reader; there are no `PulsarTemplate` sends or producer beans under `src/main`. Derived 2026-07-29 by grepping `src/main` for producer/send sites (command in the verification section). This is the mirror image of DSS, which is a pure producer, so the DSS to DRS relationship is one-directional over Pulsar and bidirectional only in the sense that DRS also calls DSS over HTTP.

---

## Known Callers

- **`declaration-reporting-mfe`** (micro-frontend-hub) via a generated OpenAPI client under `src/api/declaration-reporting/`, using `drsReportPath: '/ddm/report'`. Loaded by `react-vite-shell` as `DeclarationReportingMfeRemote`.
- **`devices-inventory`** (micro-frontend-hub) via its own generated client (`bin/openapi-ts-generation.sh`, checked by `pnpm run openapi:check`), also against `/ddm/report`.

Both are generated clients, which means a response-shape change in DRS breaks them at codegen-check time rather than at runtime, and both need regenerating together. Derived 2026-07-29 by grepping sibling repos for `declaration-reporting` and `/ddm/report`.

Tyk exposes DRS as both internal (`declaration-reporting-service-{use1,euc1,apne1}`) and **external** (`declaration-reporting-service-external-{us,eu,apne}`) API products, so there may be customer-facing callers outside this list. Check the external access policies in `tyk-gateway-management/prod/api-products/declaration-reporting-service/` before treating the list above as complete.

---

## Dependencies

| Dependency | How it is used |
|---|---|
| **Declaration Storage Service** | `service/dss/DssClientService` via **both** `com.jamf.ddm:declaration-mdm-springboot-starter` and `com.jamf.ddm:declaration-product-springboot-starter`. Fetches declaration items, device assignments, and single declaration definitions at query time. Read versions from `gradle/libs.versions.toml`. |
| **MDM Server (Jamf Pro / School / Elevate)** | Source of the status reports, indirectly via the `statusreport` topic |
| **Apache Pulsar** | The three consumed topics above, plus DLQ topics |
| **PostgreSQL** | Persistence, Flyway-managed |
| **M2M (robocop)** | Inbound request auth and outbound DSS token acquisition (`DssM2MAuthService`) |
| **Resilience4j** | Circuit breaker named `dssClient` guarding every DSS call |

---

## Traps

**Trap: M2M authentication is enabled in production. Do not repeat the old claim that requests are unauthenticated.** The default `application.yaml` sets `jamf.platform.m2m.authentication-enabled: false`, which is what made this look unauthenticated, but `values-prod.yaml` activates the `m2m` Spring profile and `application-m2m.yaml` overrides the flag to `true`. `application-local.yaml`, `application-perf.yaml`, and the test profile keep it `false`. So the effective answer depends entirely on which profile is active, and reading only the base `application.yaml` gives the wrong one. Check the profile list in the relevant `values-*.yaml`, then the matching `application-<profile>.yaml`.

**Trap: `Optional.empty()` from the DSS client does not mean "no assignments".** `DssClientService` deliberately distinguishes three outcomes and the Javadoc says so explicitly: `Optional.of(assignments)` when the device exists, `Optional.of(emptyList)` when DSS returns 404 (device genuinely has no assignments), and `Optional.empty()` only from the Resilience4j fallback when the `dssClient` circuit is open or DSS errored. Treating `empty()` as "no assignments" turns a DSS outage into a report that every device is compliant with nothing. There is a counter, `drs.get_assigned_declarations.fetch_fallback_count`, that fires on exactly this path; watch it before trusting a sudden drop in reported declarations.

**Trap: `DELETE /internal/v1/tenants/{tenantId}` is a real destructive operation behind an innocuous-looking internal route.** It decommissions a tenant's data and writes to `tenant_decommission_audit`. There is a companion device-count endpoint and a `workflow-tenant-decommission.yaml` GitHub workflow, which is the intended entry point. Do not exercise the DELETE by hand to "see what it returns".

**Trap: message traces are deliberately not parented to the producer.** `MessageTraceLinker` starts a **new root trace per consumed message** and attaches an OpenTelemetry span *link* back to the producer, rather than continuing the producer's trace. This is intentional (it stops a high fan-out produce from collapsing hundreds of thousands of messages into one mega-trace), but it means searching Tempo for a producer's trace ID will not show DRS's processing as a child span. Follow the span link instead.

**Trap: robocop and the DSS client library are both log-level-suppressed on the auth-failure path.** `com.jamf.stratus.m2m.robocop` and `com.jamf.ddm.client.product.client.internal.ProductClientExecutor` are pinned to `WARN` in `application.yaml` with comments explaining why (robocop logs M2M failures at ERROR, the client library at JUL SEVERE, and `DssM2MAuthService` already emits one WARN per blocked tenant and caches it). So a tenant whose M2M auth is failing produces roughly one log line, not a stream. Absence of ERROR logs is not evidence that DSS auth is healthy.

---

## Where to find the data (verify rather than trust)

**Read via `origin/main`.** The local checkout of this repo was ~4,100 commits and 10 months behind on 2026-07-29, which is what produced the wrong page this replaced.

```bash
R=~/Projects/DDmR/device-declaration-reporting-service; git -C $R fetch origin -q
git -C $R log --oneline origin/main --since=2026-07-29
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# How far behind is this working tree? Run this BEFORE reading any file from it.
git -C $R log --oneline HEAD..origin/main | wc -l

# Regenerate the route inventory
git -C $R grep -nE '@(Get|Post|Put|Delete|Patch)Mapping' origin/main -- 'src/main/java/**/controller/*.java'

# Effective auth: which profiles a values file activates, then the profile's override
git -C $R grep -n 'authentication-enabled' origin/main -- 'src/main/resources/*'
git -C $R show origin/main:values/values-prod.yaml | grep -A4 profiles

# Consumed topics and subscriptions (and confirm there is still no producer)
git -C $R show origin/main:src/main/resources/application.yaml | sed -n '/pulsar:/,/retry:/p'
git -C $R grep -nE 'PulsarTemplate|\.send\(|producerFor' origin/main -- 'src/main/java/**'

# Schema and RSQL filter allow-list, rather than transcribed columns/fields
git -C $R ls-tree --name-only origin/main src/main/resources/db/migration/
git -C $R show origin/main:src/main/java/com/jamf/drs/service/advancedfind/RsqlFilterFields.java
```

Callers are all generated OpenAPI clients in micro-frontend-hub, so re-derive them there and check the Tyk external policies for customer-facing traffic:

```bash
git -C ~/Projects/DDmR/micro-frontend-hub grep -rln 'ddm/report' origin/main -- 'apps/*/src/**'
ls ~/Projects/DDmR/tyk-gateway-management/prod/api-products/declaration-reporting-service/
```
