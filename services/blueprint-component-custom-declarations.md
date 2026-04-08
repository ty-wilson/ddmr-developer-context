# Blueprint Component Custom Declarations

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

**Owner:** DDmR team

## Summary

`blueprint-component-custom-declarations` is a lightweight Spring Boot WebFlux microservice that acts as the blueprints system's adapter for user-authored ("custom") declarations. Blueprint Management Service calls this service as part of the blueprint lifecycle — when a blueprint containing custom declaration components is deployed, this service translates the raw declaration payloads into persisted DSS records and generates the stable identifiers that the rest of the scoping/assignment pipeline depends on. It has no database of its own; its only persistent side-effect is writes to Declaration Storage Service (DSS).

---

## What It Provides

Three endpoints, all under `/api/v1/component`:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/component/translate` | Persists declaration payloads to DSS and returns the deployable objects and identifiers Blueprint Management Service needs to complete the assignment. |
| `POST` | `/component/validate` | Validates that the incoming declaration configuration is parseable JSON. Currently a parse-only check — if the body deserializes cleanly, it returns `200 OK`. |
| `POST` | `/component/cleanup` | Deletes DSS records by ID, called when a blueprint or its custom declaration component is removed. |

All three endpoints require the `X-TenantId` header (returns `401` if missing).

There is also `HEAD /api/v1` as a reachability probe for external monitors.

---

## How Translate Works

`POST /component/translate` is the core operation. The caller provides a list of `Declaration` objects, each carrying a `type`, `kind` (`DeclarationGroup`), `channelType` (`SYSTEM` or `USER`), a `payload` (arbitrary JSON), and a numeric `payloadKey` used to correlate entries.

The service:

1. Builds a stable identifier for each declaration using the pattern `%scustom_%d` — the caller-supplied `declarationIdentifierPrefix` concatenated with the `payloadKey` (e.g., `acme-blueprint-custom_0`).
2. Scans the payload JSON recursively and replaces any occurrence of the placeholder string `$PAYLOAD_<payloadKey>` with that stable identifier. This allows declarations to reference sibling declarations by identifier before DSS IDs are known.
3. Calls DSS (`addDeclaration`) with the modified payload for each declaration, obtaining a DSS-assigned UUID.
4. Returns a list of `PostComponentTranslateResponse` objects, each containing the DSS `DeployableObject` (`{type: "DSS", id: "<uuid>"}`) alongside the stable identifier, declaration type, and channel type.

Blueprint Management Service uses the returned identifiers and deployable objects to wire up scoping-engine assignments.

---

## Authentication to DSS

The service authenticates to DSS using M2M tokens via `robocop`. Credentials (client ID and secret) are resolved at runtime, either from static config or from AWS Secrets Manager (under a configurable secret name). Secrets are cached in-process with a 10-minute TTL to respond to rotation. The M2M token itself is fetched per-request using the `tenant` scope and `declaration-storage-product` scope.

The `robocop` `M2MToken` instance is held in an `AtomicReference` and created lazily; only one instance is ever created per JVM process.

---

## Dependencies

| Dependency | Role |
|---|---|
| `declaration-product-springboot-starter` | DSS client library — wires `DeclarationProductClient` as a Spring bean. Used for `addDeclaration` and `removeDeclaration` calls. |
| `robocop` (`com.jamf.stratus.m2m`) | M2M token provider for authenticating outbound DSS requests. |
| AWS Secrets Manager | Optional source for M2M client credentials; resolved via `dss.dss-token-credentials.secret` config property. |

There is no Pulsar dependency — this service neither produces nor consumes events. It is a synchronous HTTP service throughout.

---

## Configuration

| Property | Description |
|---|---|
| `declaration.client.host` | Base URL for DSS (e.g., `https://us.api.dev.platform.jamflabs.io/dss`). |
| `m2m.env` | Robocop environment (`dev`, `prod`, etc.). |
| `dss.dss-token-credentials.secret` | AWS Secrets Manager secret name for DSS M2M credentials. Mutually exclusive with `dss.dss-token-credentials.credentials`. |
| `dss.dss-token-credentials.credentials.client-id` / `.client-secret` | Inline static credentials (for local dev). Mutually exclusive with `.secret`. |
| `aws.secrets.region` | AWS region for Secrets Manager. |
| `aws.secrets.profile` | Optional AWS profile name for local credential resolution. |

Exactly one of `secret` or `credentials` must be set under `dss.dss-token-credentials`; startup validation enforces this via `@ValidCredentials`.

---

## Not to Be Confused With

`declaration-service` follows the same translate/validate/cleanup pattern and is also called by Blueprint Management Service. The distinction: `blueprint-component-custom-declarations` handles user-authored custom declarations (arbitrary payloads composed by end users), whereas `declaration-service` handles DDmR-team-owned declaration types. Both ultimately write to Declaration Storage Service (DSS). The Component Registry's `translator.baseUri` for a given component type determines which service BMS routes to.

---

## Gotchas

**Validate is a structural check only.** `POST /component/validate` confirms the payload can be deserialized as a list of `Declaration` objects with valid JSON `payload` fields. It does not validate the payload content against any Apple DDM schema. Upstream callers are responsible for semantic correctness.

**Identifier format is contractual.** The `%scustom_%d` identifier pattern (prefix + `custom_` + payloadKey) is baked into the translate response. Blueprint Management Service and the scoping engine both expect identifiers in this form. Do not change the format without coordinating with consumers.

**`$PAYLOAD_<n>` placeholders are string-only replacements.** The recursive replacement in `replaceValuesInPayload` only substitutes `JsonPrimitive` string values that exactly match the placeholder. If a declaration references a sibling via `$PAYLOAD_0` in a non-string position (e.g., a JSON number field), it will not be replaced. Callers should only use placeholders in string-valued JSON fields.

**No retry or error handling around DSS writes in translate.** If DSS returns an error mid-list during translate, the call throws and the response is an error. Any DSS records already written in that request are not cleaned up automatically. Blueprint Management Service is expected to call cleanup if translate fails partway through.

**`versionHash` is always empty.** The `versionHash` field in the translate response is hardcoded to `""` with a TODO noting it should be populated with a DSS `serverToken` once the service migrates to the v2 DSS API. Do not rely on this field having a meaningful value.

**Secrets Manager client is a singleton.** `SecretsServiceBuilder` uses a `compareAndSet` pattern to ensure only one `SecretsManagerClient` is created. If the AWS region or profile needs to change at runtime, a restart is required.
