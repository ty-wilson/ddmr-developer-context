# JSON Schema Form Generator

Last reviewed: 2026-07-29. Re-verified against `origin/main` on that date: the CDN loader namespace behaviour in both consuming MFEs (the previous claim was **correct when written and has since been reversed**, see the trap below), and the standalone repo's commit recency. Not re-verified and older: the service-method list, the standalone component props and condition handling, and the feature comparison. Treat those as a pointer.

Run the commands in **Where to find the data** at the end before acting on any claim here.

## Summary

JSON Schema Form Generator (JSFG) renders interactive forms driven by a JSON Schema document. Given a schema, an optional `uiSchema` (ordering, disabled states, hidden fields, tags), and a localizations map, it produces a field-per-property form where non-required properties are gated by a toggle checkbox. Changes surface via a path+value event rather than two-way binding, keeping the form stateless from the caller's perspective.

**Two separate codebases share the name "json-schema-form-generator", and only one renders Blueprints forms.** That distinction is documented as a labelled trap in `docs/frontend.md`; read it there rather than a second copy here. In short: the in-hub MFE at `micro-frontend-hub/apps/json-schema-form-generator` is what Blueprints component apps load at runtime; the standalone sibling repo `json-schema-form-generator` is a different project that Blueprints does not load. The two are not code-shared, so a fix in one does not reach the other.

---

## The two implementations

### In-hub MFE (`@jmf/json-schema-form-generator`)

- **Path:** `micro-frontend-hub/apps/json-schema-form-generator` (private, deployed to CDN, not published to npm).
- **Stack:** React plus Vue (for web components), Feature Hub (`@feature-hub/core`, `@feature-hub/dom`, `@feature-hub/react`), nanostores for form state, `json-schema-library` for schema traversal and validation, Tanstack Query, framer-motion, Jamf Design System web components. Read exact versions from the app's `package.json`.
- **Entry point:** `feature.tsx` exports a `FeatureAppDefinition<DomFeatureApp>`. It attaches to a DOM element via `createRoot`, shows a `SkeletonForm` while stylesheets load, then renders the React tree.
- **This is the one that renders declarations and config profiles.**

### Standalone npm library (`json-schema-form-generator`)

- **Path:** the sibling repo `json-schema-form-generator` (same parent directory as this docs repo).
- **Stack:** Vue, Vite, vuelidate, Jamf Design System web components. Built by `vite.config.lib.ts` into `lib/jsonSchemaForm.js` plus type declarations; Vue is an external peer dependency the caller must provide.
- **Output:** a Vue custom element registered as `<jamf-json-schema-form>` via `defineCustomElement(Form)`.
- **Props** (on `JsonSchema.ce.vue` / inner `JsonSchemaForm.ce.vue`): `schema`, `uiSchema`, `value`, `locales`. Regenerate the current list from the component rather than trusting this one (grep in the verification section). **Emits** `value`, a new form value with the changed field applied; the outer wrapper merges the path+value detail from the inner component.

**Trap: the standalone repo is effectively dormant, so "JSFG does X" sourced from it is probably not what production does.** As of 2026-07-29 the newest commit touching code is from 2024-05 (the only newer commit is a `catalog-info` edit). Check `git log` before quoting it as current behaviour, and never as behaviour of the Blueprints form.

---

## Feature Hub service (`jamf:json_schema_form_generator_service`)

Defined in `micro-frontend-hub/libs/json-schema-form-generator-service`. All state is held in nanostores atoms. It is a **shared** Feature Hub service: one instance is created by the host/integrator and injected into both the JSFG MFE and the consuming MFE, which is the entire communication channel between them (there are no props and no direct imports across the boundary).

Method groups, rather than a signature table that drifts:

- **Inputs to the form:** `setJsonSchema` / `setUiSchema` / `setLocalizations`, plus `setInitialPayloadFormValue` to seed an existing value for editing. Each has a `get*` returning a `ReadableAtom`.
- **Output from the form:** `getPayloadOutputFormValue` (JSFG writes, caller reads on save).
- **Validation handshake:** `setSaveClicked(true)` from the caller, then `getCanBeSaved()`. See the trap below.
- **Blueprints integration:** `setFormState` / `getFormState` passes `FormState` through from the blueprints service.
- **Navigation:** `triggerBackNavigation()` / `onBackNavigationTriggered(cb)`.

Regenerate the exact list from the lib's type definitions (see the verification section).

Features the in-hub MFE has that the standalone does not: a debounced field-title search bar, an OS/support/tag filter panel driven by `uiSchema.tags` plus `supportedOS` schema metadata, a `SupportedOs` badge, dictionary-field support (an `additionalProperties` schema opens a drawer sub-page via `DictionaryEdit`), and animated transitions between the root form and the dictionary sub-page. Both versions still use their own bespoke `unrefSchema` utility for `$ref` resolution.

---

## How consuming MFEs use JSFG

`blueprint-component-configuration-profiles` and `blueprint-component-declarations` follow the same four steps:

1. **Provide the service.** The host Feature Hub registers `createJsonSchemaFormGeneratorService()` so one instance is shared between the integrator MFE and the JSFG MFE.
2. **Load the MFE.** A `FeatureAppLoader` points at the JSFG CDN bucket. A unique `formId` (UUID) is passed as both `featureAppId` and config, so multiple JSFG instances on one page stay isolated.
3. **Seed data.** The integrator calls `setJsonSchema`, `setUiSchema`, `setLocalizations`, `setInitialPayloadFormValue`.
4. **Save.** On parent form submit, call `setSaveClicked(true)`, then check `getCanBeSaved()`, then read `getPayloadOutputFormValue()`.

The two loader files are worth diffing directly, because they are the only place this wiring differs:

- `micro-frontend-hub/apps/blueprint-component-configuration-profiles/src/components/JsfgLoader/JsfgLoader.tsx`
- `micro-frontend-hub/apps/blueprint-component-declarations/src/components/JsonSchemaFormGenerator/JsonSchemaFormGeneratorMfeLoader.tsx`

---

## Traps

**Trap: the two loaders no longer disagree about the branch namespace suffix, and this reversed recently.** Both `constructRemoteAppUrl` implementations now build the bucket name as `${deploymentPrefix}${appName}${namespaceSuffix}` from `VITE_NAMESPACE_SUFFIX_LC`, so a branch build of either component app loads the JSFG built from *that same branch namespace* (the suffix is empty for main/prod, so those still load JSFG from `main`). Previously the declarations loader deliberately omitted the suffix, with a comment saying JSFG is "an external shared MFE always deployed from main", which meant per-branch JSFG testing worked for config-profiles but silently did not for declarations. That was changed by **PI-1337 (mfe-hub PR #3072, merged 2026-07-06)**. Consequences: older tickets and any doc predating July 2026 describe the old asymmetry, and if a branch namespace has no JSFG build the declarations loader will now 404 where it used to fall back to `main`. Diff the two files above before trusting either behaviour.

The remaining difference between the loaders is the version alias: the declarations loader requests `stable` when `VITE_DEPLOYMENT_PREFIX === 'prod'` and `latest` otherwise; the config-profiles loader always requests `latest`. So prod declarations and prod config-profiles can be running different JSFG builds.

**Trap: `getCanBeSaved()` is not reactive, so reading it too early returns a stale answer.** There is no callback. JSFG sets `canBeSaved` via a nanostores subscription when the error store updates, but the timing depends on React render cycles. The configuration-profiles integrator works around this by gating `getPayloadOutputFormValue()` inside the same event handler immediately after `setSaveClicked(true)`. Copy that ordering rather than awaiting anything.

**Trap: do not pre-resolve `$ref` before passing a schema in.** Both versions call `unrefSchema` at startup to inline `$ref` pointers, and the declarations path also resolves refs in the localizations (`unrefLocalizations`). Pass the raw schema. Pre-resolving is not just redundant, it changes what the localization lookup keys match.

**Trap: `if`/`then` conditions in the standalone are evaluated once, at mount, against an empty value.** Dynamic re-evaluation as values change is not supported there. If conditions depend on runtime values, this is the in-hub MFE's territory (`json-schema-library` does fuller traversal). Note separately that the declarations path drops top-level conditionals entirely during its schema transform; that is a different bug with the same symptom, and it is documented in `docs/frontend.md`.

**`triggerBackNavigation` rolls the form value back.** It calls `recoverValueFromSnapshot()`, restoring a previous snapshot. This is what makes "back out of the dictionary sub-page without saving" discard edits, and it will also discard edits made in the root form since the snapshot was taken.

---

## Where to find the data (verify rather than trust)

```bash
R=~/Projects/DDmR/micro-frontend-hub; git -C $R fetch origin -q
git -C $R log --oneline origin/main --since=2026-07-29
git -C $R for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(refname:short)' refs/remotes/origin | head -15

# The loader asymmetry: diff the two files rather than trusting either doc
CP=apps/blueprint-component-configuration-profiles/src/components/JsfgLoader/JsfgLoader.tsx
DE=apps/blueprint-component-declarations/src/components/JsonSchemaFormGenerator/JsonSchemaFormGeneratorMfeLoader.tsx
diff <(git -C $R show origin/main:$CP) <(git -C $R show origin/main:$DE)
git -C $R log --oneline -5 origin/main -- $DE   # provenance for any behaviour change

# Regenerate the service surface instead of reading the method groups above
git -C $R grep -n 'set[A-Z]\|get[A-Z]' origin/main -- 'libs/json-schema-form-generator-service/src/**/*.ts' | grep -v spec

# Is the standalone repo still dormant? (newest code commit, not the newest commit)
S=~/Projects/DDmR/json-schema-form-generator; git -C $S fetch origin -q
git -C $S log --oneline -5 origin/main --format='%ad %s' --date=short
git -C $S grep -n 'props\|defineProps' origin/main -- 'src/**/JsonSchema*.ce.vue'
```

Before concluding the in-hub MFE does not support something, check unmerged work, and beware that `grep -r` over this repo also hits in-repo worktrees under `.claude/worktrees/` which are not `main`.
