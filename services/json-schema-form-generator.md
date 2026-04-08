# JSON Schema Form Generator

Last reviewed: 2026-04-07

> **Point-in-time snapshot.** Verify critical claims against the actual code before acting on them.

## Summary

JSON Schema Form Generator (JSFG) renders interactive forms driven by a JSON Schema document. Given a schema, an optional `uiSchema` (ordering, disabled states, hidden fields, tags), and a `locales` map, it produces a field-per-property form where non-required properties are gated by a toggle checkbox. Changes are surfaced via a path+value event rather than two-way binding, keeping the form stateless from the caller's perspective. JSFG exists in two distinct forms that share the same conceptual model but are implemented differently and deployed separately.

---

## Two Versions

### 1. Standalone npm library (`json-schema-form-generator`)

- **Repo:** `/Users/tyler.wilson/Projects/DDmR/json-schema-form-generator`
- **Package:** `json-schema-form-generator` v0.x, published to the internal Artifactory npm registry
- **Stack:** Vue 3, Vite, vuelidate (v0.7), Jamf Design System web components
- **Output:** A Vue custom element registered as `<jamf-json-schema-form>`, built via `vite.config.lib.ts` into `lib/jsonSchemaForm.js` + type declarations. Vue is an external peer dep — callers must provide it.
- **Entry point:** `json-schema-form-generator/json-schema-form` → `defineCustomElement(Form)` registered under the tag `jamf-json-schema-form`.

**Component props (`JsonSchema.ce.vue` / inner `JsonSchemaForm.ce.vue`):**

| Prop | Type | Notes |
|---|---|---|
| `schema` | `SchemaObject` | Top-level JSON Schema; `$ref` pointers are resolved internally via `unrefSchema` before rendering |
| `uiSchema` | `UiSchema` | Optional. Controls `order`, `disabled`, nested `properties`, `items` |
| `value` | `SchemaValue` (`Record<string, any>`) | Current form value; caller owns the state |
| `locales` | `Record<string, unknown>` | Localisation strings keyed by property name |

**Emits:** `value` — a new `SchemaValue` with the changed field applied (the outer wrapper merges the path+value detail from the inner component).

**Condition processing:** `if/then` JSON Schema conditions are resolved once at mount time against an empty value via `processSchemaConditions`. Dynamic re-evaluation as values change is not supported in this version.

---

### 2. In-hub MFE (`@jmf/json-schema-form-generator`)

- **Repo:** `/Users/tyler.wilson/Projects/DDmR/micro-frontend-hub/apps/json-schema-form-generator`
- **Package:** `@jmf/json-schema-form-generator` v2.x (private, deployed to CDN, not an npm package)
- **Stack:** React 18 + Vue 3 (for web components), Feature Hub (`@feature-hub/core`, `@feature-hub/dom`, `@feature-hub/react`), nanostores, json-schema-library v10, Tanstack Query, framer-motion, Jamf Design System web components
- **JSON Schema validation:** Uses `json-schema-library` (replaces vuelidate) for schema traversal and validation.
- **Entry point:** `feature.tsx` exports a `FeatureAppDefinition<DomFeatureApp>`. The app attaches to a DOM element via `createRoot`, shows a `SkeletonForm` while stylesheets load, then renders the full React tree.

**Feature Hub service interface (`jamf:json_schema_form_generator_service` v1.0.0):**

Defined in `libs/json-schema-form-generator-service`. All state is held in nanostores atoms. The service is a shared Feature Hub service — one instance is created by the host/integrator and injected into both the JSFG MFE and the consuming MFEs.

Key methods:

| Method | Purpose |
|---|---|
| `setJsonSchema(schema)` / `getJsonSchema()` | Push schema into JSFG; returns a `ReadableAtom<Schema>` |
| `setUiSchema(uiSchema)` / `getUiSchema()` | Push uiSchema |
| `setLocalizations(locales)` / `getLocalizations()` | Push localisations |
| `setInitialPayloadFormValue(value)` / `getInitialPayloadFormValueStore()` | Seed the form with an existing value (e.g. for editing) |
| `getPayloadOutputFormValue()` / `setPayloadOutputFormValue(value)` | JSFG writes the current form output here; callers read it on save |
| `getCanBeSaved()` / `setCanBeSaved(bool)` | JSFG sets this false when there are validation errors |
| `getSaveClicked()` / `setSaveClicked(bool)` / `getSaveClickedStore()` | Caller sets true to trigger validation display; JSFG reads it to decide whether to show errors |
| `setFormState(state)` / `getFormState()` | Passes `FormState` from the blueprints service to JSFG |
| `triggerBackNavigation()` / `onBackNavigationTriggered(cb)` | Caller triggers back-nav; JSFG subscribes and recovers a value snapshot on back |

**Additional features vs. standalone:**
- Search bar (debounced, 200 ms) to filter fields by title
- OS / support / tag filter panel (driven by `uiSchema.tags` + `supportedOS` schema metadata)
- `SupportedOs` badge component showing OS compatibility info from the Jamf schema extension
- Dictionary field support (`additionalProperties` schemas open a drawer sub-page via `DictionaryEdit`)
- Animated page transitions (framer-motion) between root form and dictionary sub-page
- `json-schema-library` used for schema validation (via `compileSchema`); the in-hub MFE still has its own bespoke `unrefSchema` utility for `$ref` resolution, same as the standalone

---

## How Consuming MFEs Use JSFG

Both `blueprint-component-configuration-profiles` and `blueprint-component-declarations` follow the same pattern:

1. **Provide the service** — the host Feature Hub registers `createJsonSchemaFormGeneratorService()` so the same instance is shared between the integrator MFE and the JSFG MFE.
2. **Load the MFE** — a `FeatureAppLoader` component (e.g. `JsfgLoader` / `JsonSchemaFormGeneratorMfeLoader`) points to the CDN bucket `json-schema-form-generator` at version `stable` (prod) or `latest` (non-prod). A unique `formId` (UUID) is passed as both `featureAppId` and config, so multiple JSFG instances on the same page are isolated.
3. **Seed data** — before or after the loader renders, the integrator calls `setJsonSchema`, `setUiSchema`, `setLocalizations`, and `setInitialPayloadFormValue` on the service.
4. **Save** — on the parent form submit, the integrator calls `setSaveClicked(true)` then checks `getCanBeSaved()`. If true, it reads `getPayloadOutputFormValue()` for the final value.

The `blueprint-component-declarations` loader intentionally does **not** append the branch namespace suffix to the CDN bucket name, ensuring it always loads the JSFG built from `main`. The `blueprint-component-configuration-profiles` loader does append the namespace suffix, allowing per-branch JSFG builds.

---

## Gotchas

**Two separate codebases, one concept.** The standalone library and the in-hub MFE are not code-shared. Bug fixes in one do not automatically apply to the other. The standalone uses vuelidate and a Vue custom element; the in-hub MFE uses json-schema-library and React with Feature Hub.

**`$ref` resolution is internal.** Both versions call `unrefSchema` at startup to inline all `$ref` pointers. The raw schema with `$ref`s is what you pass in; JSFG handles resolution. Do not pre-resolve before passing.

**Condition evaluation is static in the standalone.** `if/then` conditions in the standalone version are evaluated once against an empty value at mount. If conditions depend on runtime values, use the in-hub MFE version which uses `json-schema-library` for more complete schema traversal.

**`getCanBeSaved()` is not reactive.** The integrator must read it after calling `setSaveClicked(true)` — there is no callback. The JSFG MFE sets `canBeSaved` synchronously via a nanostores subscription when the error store updates, but the timing depends on React render cycles. The configuration-profiles integrator works around this by gating `getPayloadOutputFormValue()` inside the same event handler immediately after `setSaveClicked`.

**`triggerBackNavigation` resets to snapshot.** When the integrator calls `triggerBackNavigation()`, JSFG calls `recoverValueFromSnapshot()` which rolls the form value back to a previous snapshot. This is used when the user navigates back from the dictionary sub-page without saving.
