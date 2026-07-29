# Blueprint Component Custom Declarations

Last reviewed: 2026-04-07; amended 2026-07-29 to drop drifting values, label traps, and add verification commands. **Nothing here was re-verified against code on the amend date**, so treat it as a pointer and run the checks at the end before relying on any claim.

**Owner:** DDmR team

## Summary

`blueprint-component-custom-declarations` is a lightweight Spring Boot WebFlux microservice that acts as the blueprints system's adapter for user-authored ("custom") declarations. Blueprint Management Service calls it as part of the blueprint lifecycle: when a blueprint containing custom declaration components is deployed, this service translates the raw declaration payloads into persisted DSS records and generates the stable identifiers that the rest of the scoping/assignment pipeline depends on. It has no database of its own; its only persistent side effect is writes to Declaration Storage Service (DSS).

## What It Provides

Three endpoints, all under `/api/v1/component`:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/component/translate` | Persists declaration payloads to DSS and returns the deployable objects and identifiers Blueprint Management Service needs to complete the assignment. |
| `POST` | `/component/validate` | Checks that the incoming declaration configuration is parseable JSON. A parse-only check: if the body deserializes cleanly it returns `200 OK`. |
| `POST` | `/component/cleanup` | Deletes DSS records by ID, called when a blueprint or its custom declaration component is removed. |

All three endpoints require the `X-TenantId` header (returns `401` if missing). There is also `HEAD /api/v1` as a reachability probe for external monitors.

## How Translate Works

`POST /component/translate` is the core operation. The caller provides a list of `Declaration` objects, each carrying a `type`, `kind` (`DeclarationGroup`), `channelType` (`SYSTEM` or `USER`), a `payload` (arbitrary JSON), and a numeric `payloadKey` used to correlate entries.

The service:

1. Builds a stable identifier for each declaration using the pattern `%scustom_%d`: the caller-supplied `declarationIdentifierPrefix` concatenated with the `payloadKey` (for example `acme-blueprint-custom_0`).
2. Scans the payload JSON recursively and replaces any occurrence of the placeholder string `$PAYLOAD_<payloadKey>` with that stable identifier. This lets declarations reference sibling declarations by identifier before DSS IDs are known.
3. Calls DSS (`addDeclaration`) with the modified payload for each declaration, obtaining a DSS-assigned UUID.
4. Returns a list of `PostComponentTranslateResponse` objects, each containing the DSS `DeployableObject` (`{type: "DSS", id: "<uuid>"}`) alongside the stable identifier, declaration type, and channel type.

Blueprint Management Service uses the returned identifiers and deployable objects to wire up scoping-engine assignments.

## Authentication to DSS

The service authenticates to DSS using M2M tokens via `robocop`. Credentials (client ID and secret) are resolved at runtime, either from static config or from AWS Secrets Manager under a configurable secret name. Secrets are cached in process with a short TTL so that rotation is picked up without a restart; read the actual TTL from the service config rather than from here. The M2M token itself is fetched per request using the `tenant` scope and the `declaration-storage-product` scope.

The `robocop` `M2MToken` instance is held in an `AtomicReference` and created lazily, so only one instance exists per JVM process.

## Dependencies

| Dependency | Role |
|---|---|
| `declaration-product-springboot-starter` | DSS client library. Wires `DeclarationProductClient` as a Spring bean, used for `addDeclaration` and `removeDeclaration`. |
| `robocop` (`com.jamf.stratus.m2m`) | M2M token provider for authenticating outbound DSS requests. |
| AWS Secrets Manager | Optional source for M2M client credentials, resolved via the `dss.dss-token-credentials.secret` config property. |

**No Pulsar dependency: this service neither produces nor consumes events.** It is a synchronous HTTP service throughout. Derived by grepping the build files and source for Pulsar usage; re-derive with the command below rather than assuming it still holds.

## Configuration

| Property | Description |
|---|---|
| `declaration.client.host` | Base URL for DSS (for example `https://us.api.dev.platform.jamflabs.io/dss`). |
| `m2m.env` | Robocop environment (`dev`, `prod`, and so on). |
| `dss.dss-token-credentials.secret` | AWS Secrets Manager secret name for DSS M2M credentials. Mutually exclusive with `dss.dss-token-credentials.credentials`. |
| `dss.dss-token-credentials.credentials.client-id` / `.client-secret` | Inline static credentials for local dev. Mutually exclusive with `.secret`. |
| `aws.secrets.region` | AWS region for Secrets Manager. |
| `aws.secrets.profile` | Optional AWS profile name for local credential resolution. |

Exactly one of `secret` or `credentials` must be set under `dss.dss-token-credentials`; startup validation enforces this via `@ValidCredentials`.

## Not to Be Confused With

`declaration-service` follows the same translate/validate/cleanup pattern and is also called by Blueprint Management Service. The distinction: `blueprint-component-custom-declarations` handles user-authored custom declarations (arbitrary payloads composed by end users), whereas `declaration-service` handles DDmR-team-owned declaration types. Both ultimately write to DSS. The Component Registry's `translator.baseUri` for a given component type determines which service BMS routes to.

## Traps

**Trap: validate is a structural check only.** `POST /component/validate` confirms the payload can be deserialized as a list of `Declaration` objects with valid JSON `payload` fields. It does **not** validate payload content against any Apple DDM schema, so a semantically invalid declaration passes validation and fails later. Upstream callers own semantic correctness.

**Trap: the identifier format is contractual.** The `%scustom_%d` pattern (prefix + `custom_` + payloadKey) is baked into the translate response, and both Blueprint Management Service and the scoping engine expect identifiers in that form. Do not change the format without coordinating with consumers.

**Trap: `$PAYLOAD_<n>` placeholders are string-only replacements.** The recursive replacement in `replaceValuesInPayload` substitutes only `JsonPrimitive` string values that match the placeholder exactly. A sibling reference in a non-string position (a JSON number field, say) is silently left unreplaced. Use placeholders only in string-valued JSON fields.

**Trap: no retry or rollback around DSS writes in translate.** If DSS errors partway through the list, the call throws and any DSS records already written in that request are **not** cleaned up automatically. Blueprint Management Service is expected to call cleanup after a partial translate failure.

**Trap: `versionHash` is always empty.** The field in the translate response is hardcoded to `""`. Do not rely on it carrying a meaningful value.

**Trap: the Secrets Manager client is a singleton.** `SecretsServiceBuilder` uses a `compareAndSet` pattern so only one `SecretsManagerClient` is created. Changing the AWS region or profile therefore requires a restart.

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/blueprint-component-custom-declarations; git -C $R fetch origin -q

# What landed since this doc was reviewed, and what is still unmerged
git -C $R log --oneline origin/main --since=2026-04-07
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# Confirm the "no Pulsar" claim
git -C $R grep -in 'pulsar' origin/main -- src/main build.gradle.kts

# Is validate still parse-only, and is the identifier format unchanged?
git -C $R grep -n 'custom_\|replaceValuesInPayload\|versionHash' origin/main -- src/main

# Real secrets-cache TTL and credential validation
git -C $R grep -n 'ttl\|Duration\|ValidCredentials' origin/main -- src/main
```
