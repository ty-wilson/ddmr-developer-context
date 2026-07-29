# Declaration Schema Customization — Design

Status: **analysis complete, two decisions open** (see [Open decisions](#open-decisions))
Date: 2026-07-29
Worked examples: DDMR-1224 (shipped), DDMR-1228 (next)

## Problem

Generative declarations (the schema-driven `com.jamf.ddm-strict` path) render Apple's DDM schema faithfully. That is usually correct and occasionally wrong: sometimes Jamf needs to override what Apple published, add a constraint Apple did not specify, or change how a field is presented.

Today there is **no place to author those opinions as data**. The declarations MFE has no overlay, no ui-schema, and no server-side validation, so any customization has to be wedged into one of two frontend transform files as a hardcode. That is the pattern DDMR-1224 and DDMR-1228 both hit, and it will recur with every new declaration type.

Config profiles solved this years ago with an upstream customization apparatus. This doc measures what they built, establishes what declarations already have for free, and identifies the genuinely missing pieces.

## Current state: how the two pipelines differ

Both MFEs consume the same `mdm-schema` serving layer. They consume it very differently.

| | Config profiles | Declarations |
|---|---|---|
| Schema fetched | `GET /v1/metadata/{scan}/{type}` — fully transformed **JSON Schema** | `GET /v1/declarative/{v}/{cat}/{type}` — **raw Apple DDM** (Apple's dialect, not JSON Schema) |
| Apple-dialect → JSON Schema | Server-side, ingest transformation Lambda | **Client-side**, in the MFE query layer |
| Validation/business overrides | `mdm-jamf-schema` enhancer, deep-merged server-side at ingest | none |
| Presentation overrides | `mdm-ui-schema`, merged in browser as JSFG `uiSchema` | none (no uiSchema sent) |
| Text/label overrides | `mdm-schema-translations` (`jamfTitle`/`jamfDescription`/`jamfEnums`) | same layer, partially wired |
| Server-side validation | generated DTOs w/ bean validation, enforced on `/validate` | **none** (handler is a stub) |

The root cause of the divergence: the transformation Lambda was scoped to profiles only. Its own ADR says *"only profiles have full support for parsing process... only MVP for profiles... others were not validated against YAML 1:1."* So declarations bypassed the whole apparatus and reimplemented a subset of the translation client-side.

## What Goldminers actually built (measured, not assumed)

This matters because "adopt their pipeline" sounds cheap and is not, while "replicate their overlay" sounds expensive and is.

| Layer | Size | Nature |
|---|---|---|
| Transformation Lambda | **~480 LOC** core | Pure Apple-dialect → JSON Schema **mechanics**. Recursive tree-rewriting with correctness-critical invariants (cycle guards on self-referential subkeys, `$defs`/`$ref` hoisting, `supportedOS` inheritance, `ANY` → dictionary). This is the real engineering. |
| `mdm-jamf-schema` enhancer | **3 files, ~30 lines** | The entire body of Jamf schema "customization": `sensitive` / `sensitiveId` flags on password fields. Nothing else. The design doc advertises "validation rules and business metadata"; none was ever authored. |
| `mdm-ui-schema` | 9 files, 5-key vocabulary | `uiType`, `hidden`, `order`, `tags`, `appSearchSupportedOS`. Hand-authored presentation hints. |
| java-enhancements → DTOs | Lambda + build-time codegen | Schema keywords compiled to JSR-380 bean validation, enforced server-side on `/validate`. |

Two findings that cut against adopting their pipeline:

- **The expensive part is mechanics we already have.** Our client-side transform already does the subset declarations need. Adopting the Lambda means rebuilding and re-validating 480 LOC of tricky recursion against declaration YAML (structurally different from profiles, never exercised there) to gain an overlay mechanism whose content is ~30 lines.
- **They do not convert `rangelist` → `enum` either.** That gap exists on both sides, so it is net-new work regardless of path. It is a five-line normalization, not a pipeline rebuild.

## What declarations already have (do not rebuild)

All verified this session against live endpoints and running code:

- **The translations layer works end to end for declarations.** `jamfTitle` / `jamfDescription` / `jamfEnums` are served for declaration types and consumed by the form. Verified by deploying real overrides to dev and seeing them render.
- **JSFG's validation engine is complete.** `json-schema-library@10.2.1` is draft 2020-12 and JSFG passes the schema straight to it, so `enum`, `const`, `pattern`, `if`/`then`/`else`, and `dependentRequired` are **already validated natively**. No engine work is needed for any constraint we are likely to want.
- **Nested `$ref` fields localize correctly.** JSFG unrefs *both* the schema (`unrefSchema`) and the localizations (`unrefLocalizations`), so `$defs`-nested fields receive their labels. Verified empirically: a `jamfEnums`/`jamfTitle` planted on `content-cache`'s `$defs.RangesItem.type` rendered in the array-item form. (An earlier version of this analysis claimed a nested-`$ref` gap and proposed an MFE fix for it. That was wrong; the test disproved it.)
- **JSFG already accepts a `uiSchema`** (`setUiSchema`) and applies widget/hidden/order/tags. Declarations simply never sent one.
- **The shared serving layer** (`/v1/declarative`, `/v1/translations`, `/v1/version`) is already consumed.

## The real gaps

1. **The client transform is a lossy allow-list.** `buildSchemaProperty` rebuilds each Apple field copying only a fixed key set, silently discarding anything else, notably `rangelist` (so enum fields render as free textboxes and validate nothing) and `assettypes`. `convertDeclarationToSchema` likewise drops top-level `if`/`then`/`dependentRequired`. Config profiles avoid this class of bug entirely by never rebuilding.
2. **No `uiSchema` is sent**, so presentation cannot be customized at all.
3. **No server-side field validation.** `/api/v1/strict/components/validate` is a **stub** that always returns `ValidationResponse(emptyList())`. There is no JSON-Schema validator on the classpath, `schema-processor` never descends into Apple's `payloadkeys` (it extracts only type/group/allowed-channels), and the `schemaUrlBase`/`schemaVersion`/`schemaTtl` config is dead (read by no code). This is why an invalid declaration can save and silently fail to deploy.
4. **Per-declaration metadata is hand-maintained** in `declaration-service`'s `application.yaml` (`enabledDeclarationTypes` + `declaration-metadata`: name, icon, `supportedOS`, feature flag). It is authoritative for which platforms a declaration is offered on, independent of Apple's schema.

## Design: category to mechanism

The core principle: **text belongs in the translations layer; constraints belong in the schema layer.** Do not build one overlay that tries to do both. Translations *win over* schema text in the MFE's precedence chain, so a schema-level overlay would be silently ineffective for titles and descriptions.

| Category | Mechanism | Status |
|---|---|---|
| **1. Faithful passthrough** (`rangelist`→`enum`, `assettypes`, `pattern`, top-level conditionals) | Fix the allow-list in the MFE query layer | Table stakes, not customization. Partly shipped (DDMR-1224) |
| **2. Text / label overrides** (title, description, enum labels) | **Existing** `mdm-schema-translations` (`jamfTitle`/`jamfDescription`/`jamfEnums`) | Works today. No new mechanism needed |
| **3. Structural / validation overrides** (min/max, required, conditional-required, platform) | Needs a home. See below | **The actual gap** |
| **4. Presentation** (widget, hide, order, grouping) | JSFG `uiSchema` via `setUiSchema` + authored content | JSFG ready; needs wiring + content |

For category 3, the shape that fits our constraints (DDmR-owned, replicate the technique rather than the plumbing) is a **thin per-declaration overlay**: partial JSON-schema fragments deep-merged onto the fetched Apple schema, the same technique as `mdm-jamf-schema` but ours. The merge is roughly 20 lines. This is where "min is really 300", "this field is required when its parent is present" (DDMR-1228), and "restrict to macOS" live as data.

## The API-first constraint

This is the decision that most shapes the design. If `declaration-service`'s API is a first-class contract that programmatic callers use, then a client-only overlay fails two tests: the **API is unguarded** (exactly DDMR-1228's silent deploy), and **FE and BE validation drift** because they derive constraints independently.

Under API-first, two things become requirements rather than options:

1. The server must perform field-level validation (today: a stub).
2. FE and BE must validate against a **single source of truth**, or they diverge.

The naive way to get parity is to reimplement the transform in Kotlin, which is the expensive trap. **A better shape:** promote the existing TypeScript transform (with the passthrough fix) plus the overlay merge into a **build-time generator** that emits one finished JSON-Schema artifact per declaration type. The MFE renders and validates against that artifact; `declaration-service` bundles the same artifacts and validates payloads with an off-the-shelf Kotlin JSON-Schema validator (e.g. networknt). The backend never reimplements the transform, it just loads a schema and validates. Parity by construction.

Trade-offs, stated honestly:

- **Loses runtime auto-discovery.** The MFE currently discovers types live. Build-time artifacts mean a rebuild to add a type. Mitigating context: the backend *already* requires per-type manual config and pins `schemaVersion`, so a manual per-type cadence exists today, and pinning is what makes parity deterministic.
- **Needs publish plumbing** so `declaration-service`'s build can pull the artifacts (a small DDmR-owned analogue of how `configuration-profile-service` pulls the java-enhancements repo).
- **Needs validator-parity fixtures.** `json-schema-library` and networknt are both draft 2020-12 but must be proven to agree.
- **Needs a contract confirmation** that blueprint orchestration treats a non-empty `errors[]` from `/strict/components/validate` as deploy-blocking. Nothing in-repo proves it does, and there is no pact interaction covering it.

## Sensitive inputs

Corrected finding: **Goldminers' sensitive-data work is in flight, not absent.** `origin/GOLD-753-schema-driven-sensitive-data` (last updated 2026-07-21, unmerged) implements it: the schema `sensitive` flag drives generated `splitSensitive(...)` per payload, a `ConfigurationTransformer` splits a submitted config into a public half (secrets masked with a placeholder) and a private half keyed by `payloadIdentifier`, and it extends a **shared platform contract**, `SensitiveDataComponentContractRs` from `com.jamf.platform:blueprint-component-contract-starter`. Note the actual encryption and storage of the private half lives in the platform caller, not in `configuration-profile-service`.

For declarations, Apple's model changes the risk profile: secrets live in separate **credential asset** declarations that carry a `DataURL` pointer rather than inline secret material, and configuration declarations reference credentials by **asset UUID**. Among the currently-enabled generative types, nothing carries an inline secret (at most an asset-reference UUID, which is not a secret).

However, this is **not** "never." Apple's credential asset types do have genuine inline secret fields (`credential.usernameandpassword.Password`, `credential.scep.Challenge`, `credential.identity.Password`), they are simply not enabled yet, and the already-enabled `app.managed` references exactly those asset types for its Passwords/Identities/Certificates features. Enabling them is one step away.

Recommendation: **do not build server-side secret handling now**, but author the overlay in GM's vocabulary (`sensitive`, `uiType: password`) so masking is a data edit and tagging is forward-compatible with the platform contract. Treat "generative declarations reference secrets only by asset UUID" as an **explicit assumption requiring stakeholder confirmation**, not an established fact.

## Effort

| Work item | Where | Size |
|---|---|---|
| Passthrough fix (`rangelist`→`enum`, carry `const`/`pattern`/conditionals/`assettypes`) | MFE query layer | S |
| Text/label overrides | translations repo (existing) | XS per field |
| Overlay mechanism (loader + deep-merge + apply point) | MFE query layer | S |
| Conditional-required (DDMR-1228: overlay `if`/`then` + carry conditionals + verify field-level error surfacing) | MFE + overlay | M |
| uiSchema wiring + first content | MFE + overlay | S–M |
| **Client-side subtotal** | **DDmR MFE only, no cross-team** | **~2–3 weeks** |
| Build-time artifact generator + publish plumbing | new, DDmR-owned | M–L |
| Server-side validation (validator dep, kotlinx↔jackson bridge, wire the stub, structured errors, pact) | declaration-service | L |
| **API-first parity total** | **DDmR-owned, no Goldminers dependency** | **~4–6 weeks** |
| *Deferred:* server-side secret handling | declaration-service + platform | L, and no in-repo reference to copy |

## Open decisions

1. **API-first timing.** Does the server need field-level validation parity *now* (design the shared-artifact pipeline up front), or is it acceptable to ship the client fixes first and land parity as a fast-follow with the parity target made explicit? This decides whether the build-time generator is phase 1 or phase 2.
2. **Sensitive-input scope.** Can the team commit to "generative declarations reference secrets only by asset UUID, never inline"? If yes, no server secret path is ever needed. If no, budget for it as real greenfield work and coordinate with the platform contract.

## Worked examples

**DDMR-1224** exercised three of the four categories and proved the mechanisms:
- *Interval min/description* → the enforced minimum (300) was correct; Apple's description text was wrong (its "60" is copied from the neighbouring `ManagementReportingInterval`). Fixed as a `jamfDescription` **text override** in the translations layer. Category 2.
- *Parent Selection Policy accepted junk* → Apple marks it `rangelist`; the allow-list dropped it, so it rendered as a free textbox. Fixed by normalizing `rangelist`→`enum` in the query layer, which yields a constrained dropdown *and* real validation. Category 1.
- *Platform over-scoping* → the served schema is correctly macOS-only; the extra platforms came from hand-maintained `declaration-metadata` config in `declaration-service`. Fixed there. Neither a schema nor an overlay problem.

**DDMR-1228** is the category-3 case: `AppConfig.DataAssetReference` is `presence: optional` per Apple, but Jamf wants it non-blank when its parent object is enabled. That is deliberately *overriding Apple with a Jamf opinion*, expressible as overlay `if`/`then` + `required` (which `json-schema-library` already validates), and it is the case that most argues for server-side enforcement since the current failure mode is a silent non-deploy.

## Verification recipes

Prefer re-checking these over trusting the text above.

```bash
# Which schema version is the UI actually asking for? (can lag SSM's latest-ingested)
curl -s -H "X-TenantId: default-tenant" -H "tenantId: default-tenant" \
  https://tyk.sbox.ocean.jamf.build/mdm-schema/v1/version

# Raw Apple schema for a declaration (check rangelist / supportedOS / minimum yourself)
curl -s -H "X-TenantId: default-tenant" -H "tenantId: default-tenant" \
  "https://tyk.sbox.ocean.jamf.build/mdm-schema/v1/declarative/<ver>/configurations/content-cache.settings"

# Served translations, including jamf* overrides
curl -s "https://tyk.sbox.ocean.jamf.build/mdm-schema/v1/translations/<ver>/com.apple.configuration.content-cache.settings/en-US"

# Deploy a translations branch to an env bucket (dev|stage|prod)
gh workflow run s3-upload.yml --repo jamf/mdm-schema-translations -f env=dev -f ref_name=<branch>

# Which declaration types are enabled, and their hand-maintained platform metadata
git -C declaration-service show origin/main:src/main/resources/application.yaml   # strictvalidation:
```

**Always check for in-flight branches before concluding another team has not built something.** The single largest correction in this analysis came from `git for-each-ref --sort=-committerdate refs/remotes/origin` on `configuration-profile-service`, which surfaced the unmerged sensitive-data work that `origin/main` does not show.

## Provenance

Verified empirically: JSFG engine capabilities and `$ref` localization behavior (planted-label test), translations end-to-end for declarations (deployed to dev and rendered), the served content-cache schema (`minimum: 300`, `rangelist`, macOS-only `supportedOS`), the `/strict/components/validate` stub, the absence of a JSON-Schema validator in `declaration-service`, GM's overlay/Lambda/ui-schema sizes, and GM's in-flight sensitive-data branch.

Inferred, not proven: that blueprint orchestration would treat a populated `errors[]` as deploy-blocking; validator parity between `json-schema-library` and a Kotlin validator; effort estimates.

Corrections made during analysis, recorded so they are not re-derived: there is **no** nested-`$ref` localization gap; config profiles **do** reword description prose via `jamfDescription`; `SecretsService` in `declaration-service` is infrastructure for the service's own DSS credential, **not** user-secret handling.
