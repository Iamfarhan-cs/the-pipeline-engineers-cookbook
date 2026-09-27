# Recipe 03 — Resolve Frontend Telemetry Validation Errors

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 3  
> **Source implementation commit:** 4e28c7dbd — fix: resolve frontend telemetry validation errors  
> **Source repository:** emi-frontend

## 1. What This Recipe Teaches

This recipe hardens the frontend telemetry implementation from Stages 1 and 2 so it passes the project's automated validation and strict TypeScript checks.

By completing it, you should understand how to:
- investigate CI failures without changing working telemetry behavior unnecessarily;
- satisfy repository-wide copyright/header validation;
- normalize optional values before they enter a strict telemetry property contract;
- prevent undefined from entering scalar telemetry properties;
- fix strict TypeScript test assertions safely;
- distinguish functional changes from validation-only changes;
- verify lint and type-check gates before moving to backend transport.

Stage 3 is a **hardening stage**. It does not introduce a new telemetry event or change the privacy model.

## 2. The Problem

Stages 1 and 2 established the telemetry contract and five onboarding journey events. The implementation then encountered project validation failures in copyright/header validation and strict TypeScript validation.

The repository checks involved:
```text
node scripts/check-copyright-headers.mjs
vue-tsc --noEmit
npm run lint
```

The problem was not the telemetry architecture itself. The implementation needed to conform to existing repository governance and type rules. fileciteturn11file0L8-L15

## 3. Why This Stage Exists

Telemetry properties are defined as:
```ts
Record<string, string | number | boolean | null>
```

An optional frontend value can instead have type:
```text
string | undefined
```

That creates a type mismatch because the telemetry contract permits string, number, boolean, and null, but not undefined.

Stage 3 resolves the mismatch without changing the event meaning. It also adds the required Zolvat copyright headers. fileciteturn11file0L8-L15

# 4. Stage 3 Architecture

Stage 3 introduces no new pipeline component:

```text
Vue / React application
        |
        v
trackEvent()
        |
        v
createTelemetryEvent()
        |
        +--> strict scalar types
        +--> redaction
        |
        v
CustomEvent
```

The stage hardens the existing client telemetry module before backend transport is introduced. fileciteturn11file0L70-L72

## 5. Step 1 — Analyze the Validation Failures

Start with the actual CI or local validation commands:

```text
Run validation
      |
      v
Identify failing files
      |
      +--> missing copyright headers
      +--> optional telemetry property
      +--> strict test assertion
      |
      v
Apply minimal fixes
      |
      v
Run validation again
```

The engineering pattern is: **fix the smallest layer responsible for the failure instead of changing unrelated runtime behavior.** The source implementation records lint/type-check analysis as the first step. fileciteturn11file0L19-L24

## 6. Step 2 — Add Required Copyright Headers

Add the standard header:

```ts
// Copyright © 2026 Zolvat Ltd
```

to:

```text
src/telemetry/index.ts
src/telemetry/redact.ts
src/telemetry/index.spec.ts
src/telemetry/redact.spec.ts
src/views/client/ChooseAccountType.spec.ts
```

The repository validation script expects this header and blocks validation when it is absent. fileciteturn11file0L21-L24

Do not disable the validation script. Satisfy the repository rule.

## 7. Step 3 — Normalize Optional Telemetry Properties

The main type issue occurs in:

```text
src/views/client/verification/IndividualVerification.vue
```

Original:

```ts
account_type: accountType.value
```

Corrected:

```ts
account_type: accountType.value ?? null
```

This converts undefined to null while preserving an actual string value. The source implementation records this exact change. fileciteturn11file0L21-L24

## 8. Why null Instead of undefined?

The telemetry contract is:

```ts
type TelemetryProperties = Record<
  string,
  string | number | boolean | null
>;
```

| Runtime value | Allowed? |
|---|---:|
| string | Yes |
| number | Yes |
| boolean | Yes |
| null | Yes |
| undefined | No |

Therefore:

```ts
account_type: accountType.value ?? null
```

means:

```text
accountType.value = "PERSONAL"
        |
        v
account_type = "PERSONAL"
```

or:

```text
accountType.value = undefined
        |
        v
account_type = null
```

The source identifies null coalescing as the design decision used to maintain strict conformance. fileciteturn11file0L37-L40

## 9. Why This Matters for Data Pipelines

A strict frontend type boundary protects the downstream data contract.

Conceptually:

```text
Frontend
   |
   v
undefined
   |
   v
JSON serialization / transport
   |
   v
field may disappear
```

versus:

```text
Frontend
   |
   v
null
   |
   v
JSON
   |
   v
field remains explicitly present
```

Explicit nullability is easier for downstream consumers to reason about than a field that silently disappears.

## 10. Step 4 — Fix Strict Test Assertions

The second TypeScript issue occurs in:

```text
src/views/client/corporate-account-registration/__tests__/CorporateAccountForm.spec.ts
```

Original:

```ts
submitButton.element
```

Corrected:

```ts
submitButton!.element
```

The non-null assertion tells TypeScript that the test has already established that the button exists at that point. The source implementation records this exact correction. fileciteturn11file0L21-L24

This is a test typing correction and does not change application runtime behavior.

## 11. Step 5 — Verify the Validation Gates

After the fixes, run:

```text
vue-tsc --noEmit
npm run lint
```

The Stage 3 source records both commands passing without errors or warnings. fileciteturn11file0L82-L84

Also verify:

```text
node scripts/check-copyright-headers.mjs
```

The source records successful copyright-header validation. fileciteturn11file0L88-L90

The validation flow is:

```text
Copyright validation
        |
        v
TypeScript validation
        |
        v
Lint
        |
        v
Stage 3 complete
```

## 12. Exact Changes

### A. Copyright headers

Added:

```ts
// Copyright © 2026 Zolvat Ltd
```

to:

```text
src/telemetry/index.ts
src/telemetry/redact.ts
src/telemetry/index.spec.ts
src/telemetry/redact.spec.ts
src/views/client/ChooseAccountType.spec.ts
```

### B. Telemetry property normalization

Changed:

```ts
account_type: accountType.value
```

to:

```ts
account_type: accountType.value ?? null
```

in:

```text
src/views/client/verification/IndividualVerification.vue
```

### C. Test strictness

Changed:

```ts
submitButton.element
```

to:

```ts
submitButton!.element
```

in:

```text
src/views/client/corporate-account-registration/__tests__/CorporateAccountForm.spec.ts
```

These exact technical changes are documented in the source Stage 3 record. fileciteturn11file0L29-L33

## 13. What Did Not Change

Stage 3 did **not** change:
- telemetry event names;
- telemetry event timing;
- redaction rules;
- browser CustomEvent architecture;
- database behavior;
- HTTP transport;
- backend ingestion;
- privacy boundaries.

The source explicitly states that there were no privacy-boundary changes. fileciteturn11file0L94-L96

This is a validation-hardening stage, not a feature stage.

## 14. Transaction Safety

Not applicable. The changes are limited to TypeScript typing, repository header compliance, and test assertion strictness. There is no database or business transaction modification. fileciteturn11file0L58-L60

## 15. Idempotency

Not applicable. Stage 3 does not create or transmit new events and does not introduce persistence behavior. fileciteturn11file0L64-L64

## 16. Database Schema

No database changes are introduced. Stage 3 is client-side validation hardening only. fileciteturn11file0L76-L78

## 17. Why Minimal Fixes Matter

A validation fix should not become a feature rewrite.

If the compiler reports:

```text
string | undefined
```

against:

```text
string | number | boolean | null
```

do not weaken the entire telemetry contract by adding undefined.

Instead normalize the value where its meaning is known:

```ts
account_type: accountType.value ?? null
```

Likewise, if the repository requires copyright headers, add them instead of disabling the checker.

The Stage 3 implementation follows this minimal-change approach. fileciteturn11file0L37-L40

## 18. Independent Implementation Exercise

Create a strict telemetry contract in another TypeScript frontend:

```ts
type TelemetryProperties = Record<
  string,
  string | number | boolean | null
>;
```

Create an optional value:

```ts
const selectedPlan: string | undefined = getSelectedPlan();
```

Normalize it without weakening the contract:

```ts
const properties: TelemetryProperties = {
  plan: selectedPlan ?? null,
};
```

Then create a test with an optional DOM element. If your test logic guarantees that the element exists, use an appropriate strict assertion:

```ts
button!.element
```

Finally, add a repository validation rule requiring a standard header and make the CI check fail when the header is absent.

The goal is to learn how to preserve a strict contract while adapting uncertain application values at the boundary.

## 19. Validation Checklist

### Type safety
- [ ] No telemetry property can become undefined.
- [ ] Optional telemetry values use explicit null normalization.
- [ ] TelemetryProperties remains strict.
- [ ] vue-tsc --noEmit passes.

### Repository governance
- [ ] Required copyright headers exist.
- [ ] Copyright validation passes.
- [ ] npm run lint passes.

### Runtime behavior
- [ ] Existing telemetry event names are unchanged.
- [ ] Existing telemetry timing is unchanged.
- [ ] Redaction behavior is unchanged.
- [ ] CustomEvent architecture is unchanged.

### Pipeline boundary
- [ ] No backend changes are introduced.
- [ ] No database schema is introduced.
- [ ] No HTTP transport is introduced.

## 20. Scope Boundaries

### Implemented
- copyright headers;
- optional telemetry property normalization;
- strict test assertion fix;
- lint validation;
- TypeScript validation;
- copyright validation.

### Deferred to Stage 4
- backend ingestion API;
- persistence table in svc.

### Deferred to Stage 5
- frontend HTTP network transport.

These boundaries are explicitly recorded in the Stage 3 implementation. fileciteturn11file0L100-L104

## 21. What You Should Understand Before Recipe 04

Production pipeline work includes more than feature implementation. You must protect the business contract, type contract, repository governance, and CI validation together.

You should be able to explain:
- why undefined is different from null in a strict telemetry contract;
- why optional frontend values should be normalized at the telemetry boundary;
- why repository validation rules should be satisfied rather than disabled;
- why test-only type fixes can be necessary even when runtime behavior is correct;
- why validation hardening should avoid unnecessary runtime changes;
- how frontend contract quality affects downstream pipeline stability.

The Stage 3 result is:

```text
Stage 1 telemetry foundation
          +
Stage 2 journey events
          |
          v
Stage 3 validation hardening
          |
          +--> strict types
          +--> lint compliance
          +--> copyright compliance
          |
          v
Ready for backend ingestion work
```

## Source Traceability

This recipe is derived from the supplied Stage 3 implementation record:

- Source commit: <code>4e28c7dbd</code>
- Subject: <code>fix: resolve frontend telemetry validation errors</code>
- Repository: <code>emi-frontend</code>
- Implementation scope: fileciteturn11file0L3-L15
- Implementation sequence: fileciteturn11file0L19-L24
- Technical changes: fileciteturn11file0L29-L33
- Design decisions: fileciteturn11file0L37-L40
- Validation: fileciteturn11file0L82-L90
- Privacy and scope boundaries: fileciteturn11file0L94-L104