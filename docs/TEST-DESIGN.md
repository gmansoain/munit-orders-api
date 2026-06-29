# Test Design Document — munit-orders-api

> **Phase 1 deliverable (Testing Standard §8.2/§8.3, Documentation Standard §5.4).**
> Plan only — no suite XML, no app changes. Read, validate, answer the Open Questions, then approve for Phase 2.
> **Author:** MUnit test-generation agent · **Date:** 2026-06-29 · **Status:** awaiting validation

---

## 1. Summary

| Field | Value |
| --- | --- |
| Application | `munit-orders-api` |
| Target flow file | `src/main/mule/munit-orders-api.xml` |
| API spec | `src/main/resources/api/orders.raml` (`POST /orders`) |
| Mule runtime | 4.9.11 → suite `minMuleVersion="4.9.0"` |
| Global configs | `HTTP_Listener_config`, `HTTP_Request_config` (declared in the target file; none external supplied) |
| Example payloads | None supplied → inline `set-event` payloads |
| Existing suites | None → greenfield |
| Units found | 5 (2 flows, 1 entry flow with error handler, 2 sub-flows) |
| Proposed tests | **13 logical tests** across 5 units (2 of which are parameterized: 3 + 4 rows ⇒ 18 executed cases) |
| Coverage intent | ≥ 80% line gate + all §7.2 mandatory rules (every error handler, every choice branch, every external-call flow has mock+verify, entry flow happy + error-mapping) |
| Suite plan | One suite per flow file → `munit-orders-api-suite.xml` covering all 5 units |

---

## 2. Unit inventory

| Unit | Archetype (§3) | Source |
| --- | --- | --- |
| `orders-api-main` | API listener / entry flow **+** Error handler / on-error scope | `munit-orders-api.xml` L18–48 |
| `validate-order-subflow` | Choice / router flow (raises an error) | `munit-orders-api.xml` L51–62 |
| `build-order-record-subflow` | Transformation sub-flow (pure DataWeave) | `munit-orders-api.xml` L65–78 |
| `route-order-flow` | Orchestration flow **+** Choice / router flow | `munit-orders-api.xml` L81–96 |
| `notify-flow` | Integration / connector flow (one HTTP call) | `munit-orders-api.xml` L99–102 |

> Per §3 Note, multi-archetype units take the **union** of their mandatory categories.

---

## 3. `orders-api-main` — entry flow + error handler

### Contract sheet

| Aspect | Detail |
| --- | --- |
| **Inputs** | HTTP `POST /orders` JSON body `{ amount }` (source replaced by `set-event` in tests) |
| **Pipeline** | `validate-order-subflow` → `route-order-flow` → `ee:transform` (201 body = identity passthrough of `payload`) |
| **Outputs (happy)** | `payload` = record from `route-order-flow` (`orderId`, `amount`, `tier`, `status`, `notified`) re-serialized as JSON |
| **Owned logic (leave real)** | the two referenced sub/flows are internal Mule logic; the `ee:transform` identity passthrough |
| **Boundary collaborators (mock)** | the external HTTP call reached transitively via `route-order-flow` → `notify-flow` → `http:request` (`doc:name="post-alert"`) |
| **Branches** | none directly; routing happens inside `route-order-flow` |
| **Errors handled** | `on-error-propagate type="ORDERS:INVALID"` → set `payload = {}` (application/java), set var `httpStatus = 400`, then **re-raise**; `on-error-continue type="HTTP:CONNECTIVITY"` → `payload ++ { notified: false }`, flow completes successfully |
| **Errors raised** | re-raises `ORDERS:INVALID` (via on-error-propagate) |

> ⚠️ **Observed:** the `httpStatus=400` variable and the "201 body" comment are **not wired** to any listener response / `http:response` status-code builder (the `http:listener-config` has no response block). See Open Questions Q1/Q2.

### Proposed tests

| Test ID | Category | Trigger (§4) | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `orders-api-main-should-accept-and-notify-large-order` | happy / entry | successful outcome branch (amount > 100) | `{ amount: 250 }` | `post-alert` → 200 success | `payload.status == "accepted"`, `payload.notified == true`, `payload.tier == "priority"`, `payload.amount == 250`, `payload.orderId` is non-null string; **verify** `post-alert` `times="1"` | smoke, regression | No |
| `orders-api-main-should-accept-small-order-without-notifying` | happy / input-variant | amount ≤ 100 outcome branch | `{ amount: 100 }` | `post-alert` (defence) | `payload.status == "accepted"`, `payload.notified == false`, `payload.tier == "standard"`; **verify** `post-alert` `times="0"` | smoke, regression | No |
| `orders-api-main-should-propagate-invalid-order-as-orders-invalid` | error path / mapping | handler catches `ORDERS:INVALID` (then re-raises) | `{ amount: 0 }` | none | flow fails with `ORDERS:INVALID` (on-error-propagate re-raises); `post-alert` not called | error, regression | No |
| `orders-api-main-should-accept-order-when-notification-fails` | error path / negative | handler catches `HTTP:CONNECTIVITY` (on-error-continue) | `{ amount: 250 }` | `post-alert` → `HTTP:CONNECTIVITY` error | flow completes successfully; `payload.notified == false`; `payload.amount == 250`, `payload.tier == "priority"`, `payload.orderId` non-null (record fields preserved); **note** `payload.status` is **absent** (see Q3) | error, regression | No |

---

## 4. `validate-order-subflow` — choice / router (raises)

### Contract sheet

| Aspect | Detail |
| --- | --- |
| **Inputs** | `payload.amount` |
| **Outputs** | valid → payload unchanged (logger only); invalid → raises error |
| **Owned logic (leave real)** | the `choice` and the boolean expression `payload.amount == null or payload.amount <= 0` |
| **Boundary collaborators (mock)** | none |
| **Branches** | `when` (amount is null **or** ≤ 0) → `raise-error ORDERS:INVALID`; `otherwise` → logger "Order is valid" |
| **Errors raised** | `ORDERS:INVALID` ("amount must be a positive number") |

### Proposed tests

| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `validate-order-should-pass-valid-order` | branch (otherwise) / happy | positive amount → valid route | `{ amount: 250 }` | none | no error raised; `payload` unchanged (`payload.amount == 250`) | smoke, regression | No |
| `validate-order-should-raise-invalid-when-amount-not-positive` | branch (when) / boundary / error | `amount` null, 0, or negative | rows: `null`, `0`, `-5` | none | flow fails with `ORDERS:INVALID` | error, regression | **Yes** (3 rows: missing-amount / zero / negative) |

---

## 5. `build-order-record-subflow` — transformation (pure DataWeave)

### Contract sheet

| Aspect | Detail |
| --- | --- |
| **Inputs** | `payload.amount` |
| **Outputs** | `{ orderId: uuid(), amount: payload.amount, tier: if (amount > 100) "priority" else "standard" }` |
| **Owned logic (leave real)** | the entire DataWeave transform |
| **Boundary collaborators (mock)** | none |
| **Branches (DW conditional)** | `amount > 100` → `tier = "priority"`; else → `tier = "standard"` |
| **Errors** | none |
| **Non-determinism** | `orderId = uuid()` → assert via matcher (non-null string), never `assert-equals` (§6.6, Q5) |

### Proposed tests

| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `build-order-record-should-map-amount-and-tier` | happy / input-variant / boundary | tier conditional has two classes + the `> 100` boundary | rows: `50→standard`, `100→standard` (boundary, not `>100`), `101→priority` (just-over), `250→priority` | none | `payload.amount` == input amount; `payload.tier` == expected tier; `payload.orderId` is a non-null string (matcher) | smoke, regression | **Yes** (4 rows) |

---

## 6. `route-order-flow` — orchestration + choice

### Contract sheet

| Aspect | Detail |
| --- | --- |
| **Inputs** | `payload.amount` (record fields produced by `build-order-record-subflow`) |
| **Pipeline** | `build-order-record-subflow` → `choice` |
| **Outputs** | `when` (amount > 100): logger → `notify-flow` → `payload ++ { status: "accepted", notified: true }`; `otherwise`: logger → `payload ++ { status: "accepted", notified: false }` |
| **Owned logic (leave real)** | `build-order-record-subflow` (internal transform), the `choice`, the `set-payload` shaping |
| **Boundary collaborators (mock)** | `notify-flow` → `http:request` (`doc:name="post-alert"`) |
| **Branches** | `when amount > 100`; `otherwise` (≤ 100) |
| **Errors** | none caught here; `HTTP:CONNECTIVITY` from `notify-flow` **propagates** (no local handler) |

### Proposed tests

| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `route-order-should-notify-and-mark-notified-when-amount-over-threshold` | happy / branch (>100) / behavioural | `when` branch + collaborator must be called | `{ amount: 250 }` | `post-alert` → 200 | `payload.status == "accepted"`, `payload.notified == true`, `payload.tier == "priority"`; **verify** `post-alert` `times="1"` | smoke, regression | No |
| `route-order-should-not-notify-when-amount-not-over-threshold` | negative / branch (≤100) | `otherwise` branch + "no action" on notify | `{ amount: 100 }` (boundary) | `post-alert` (defence) | `payload.status == "accepted"`, `payload.notified == false`, `payload.tier == "standard"`; **verify** `post-alert` `times="0"` | smoke, regression | No |
| `route-order-should-propagate-when-notification-fails` | error path | collaborator raises, no local handler | `{ amount: 250 }` | `post-alert` → `HTTP:CONNECTIVITY` | flow fails with `HTTP:CONNECTIVITY` (propagated) | error, regression | No |

---

## 7. `notify-flow` — integration / connector

### Contract sheet

| Aspect | Detail |
| --- | --- |
| **Inputs** | inbound `payload` (passed through; the response replaces payload) |
| **Outputs** | the HTTP response payload (app does nothing further with it) |
| **Owned logic (leave real)** | none |
| **Boundary collaborators (mock)** | `http:request` `method=POST path=/post-alert` (`doc:name="post-alert"`) |
| **Branches** | none |
| **Errors** | `http:request` may raise `HTTP:CONNECTIVITY` (and other HTTP errors) — only `HTTP:CONNECTIVITY` is handled upstream |

### Proposed tests

| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `notify-flow-should-call-post-alert-once-on-happy-path` | happy / behavioural | single connector op + verify cardinality | `{ amount: 250 }` | `post-alert` → 200 | **verify** `post-alert` `times="1"` | smoke, regression | No |
| `notify-flow-should-propagate-connectivity-error` | error path | connectivity failure | `{ amount: 250 }` | `post-alert` → `HTTP:CONNECTIVITY` | flow fails with `HTTP:CONNECTIVITY` | error, regression | No |

---

## 8. Coverage & mandatory-rules check (§7.2)

| Mandatory rule | Satisfied by |
| --- | --- |
| Every **error handler** has ≥ 1 test | `orders-api-main` handler → `…should-propagate-invalid-order-as-orders-invalid` (ORDERS:INVALID) + `…should-accept-order-when-notification-fails` (HTTP:CONNECTIVITY) |
| Every **choice branch** (incl. default) is hit | validate-order: `…pass-valid-order` (otherwise) + `…raise-invalid…` (when); route-order: `…notify…over-threshold` (when) + `…not-notify…` (otherwise); build-order tier conditional: priority + standard rows |
| Every flow with **external calls** has ≥ 1 mock **and** ≥ 1 verify-call | `notify-flow` (mock + verify ×1), `route-order-flow` (mock + verify ×1/×0), `orders-api-main` (mock + verify ×1/×0) |
| Every **public API entry flow** has happy + ≥ 1 error→status mapping | `orders-api-main`: happy (`…accept-and-notify-large-order`) + error mapping (`…propagate-invalid-order-as-orders-invalid`, with Q1/Q2 caveat on status wiring) |
| **80% line gate** | All 5 units exercised on happy + branch + error paths; expected ≥ 80%. Final % confirmed by `munit-maven-plugin` in Phase 2. |

**Potential gaps / caveats:** HTTP status codes (201/400) cannot be asserted as wired today (Q1/Q2); the connectivity-recovery payload omits `status` (Q3). No unit is left without a test.

---

## 9. Traceability to the API spec (`orders.raml`)

| Spec ref (`POST /orders`) | Flow(s) | Proposed Test ID(s) | Status |
| --- | --- | --- | --- |
| `201` Order accepted (success body `OrderAccepted`) | orders-api-main, route-order-flow | `orders-api-main-should-accept-and-notify-large-order`, `orders-api-main-should-accept-small-order-without-notifying`, `route-order-should-notify-and-mark-notified-when-amount-over-threshold` | ✅ (status code itself: see Q1) |
| `400` Invalid order — amount missing or ≤ 0 (maps from `ORDERS:INVALID`) | orders-api-main, validate-order-subflow | `orders-api-main-should-propagate-invalid-order-as-orders-invalid`, `validate-order-should-raise-invalid-when-amount-not-positive` | ⚠️ error type proven; HTTP 400 mapping not wired (Q1/Q2) |
| `tier = priority` when amount > 100 | build-order-record-subflow, route-order-flow | `build-order-record-should-map-amount-and-tier` (101/250 rows), `route-order-should-notify-and-mark-notified-when-amount-over-threshold` | ✅ |
| `tier = standard` when amount ≤ 100 | build-order-record-subflow | `build-order-record-should-map-amount-and-tier` (50/100 rows) | ✅ |
| `notified = false` when the alert call fails | orders-api-main | `orders-api-main-should-accept-order-when-notification-fails` | ✅ (but `status` absent — Q3) |
| `notified` true/false by amount branch | route-order-flow | `route-order-should-notify…`, `route-order-should-not-notify…` | ✅ |
| `OrderRequest.additionalProperties: false` (reject extra fields) | — | — | ❌ not enforced anywhere in the app (Q4) |

---

## 10. Open Questions (please decide before Phase 2)

- [ ] **Q1 — HTTP status codes are not wired.** The success path never sets `201`, and `httpStatus=400` is stored in a variable that no listener response/`http:response` status-code builder consumes. Should Phase 2 (a) assert only payload/vars and not status codes, or (b) treat the missing status wiring as a defect to flag? (Unit tests via `flow-ref` cannot observe a listener status code regardless.)
**A1** - Assert only payload/vars
- [ ] **Q2 — 400 path re-raises.** `on-error-propagate ORDERS:INVALID` sets `payload={}` and `httpStatus=400` but then **re-raises**, so `orders-api-main` ultimately throws `ORDERS:INVALID` rather than returning a 400 response. Confirm the test should assert "flow fails with `ORDERS:INVALID`" (we cannot reliably assert the interim `payload={}` / `httpStatus=400` once the error propagates). Acceptable?
**A2** - Yes
- [ ] **Q3 — Connectivity-recovery payload omits `status`.** Because `notify-flow` fails *before* the "Notified=True" `set-payload`, the `on-error-continue` result is `record ++ { notified: false }` with **no `status` field** — which does not satisfy RAML `OrderAccepted` (which requires `status`). Should the test assert `status` is absent (document current behaviour), or is this a bug you want flagged rather than locked in by a test?
**A3** - The test should assert status is absent
- [ ] **Q4 — Out-of-scope inputs for `build-order-record-subflow`.** In production this sub-flow only runs after validation, so `amount` is always present and > 0. Do you want isolated tests for `null`/negative amount on this sub-flow (the transform does not guard them: `null` → `tier=standard`, `amount=null`), or restrict its tests to validated inputs only? Also: `OrderRequest.additionalProperties:false` from the RAML is **not** enforced anywhere in the app — confirm we do not test it (no behaviour to test).
**A4** - I want isolated tests for null/negative amount. Do not test `OrderRequest.additionalProperties:false
- [ ] **Q5 — `orderId` assertion strategy.** `orderId = uuid()` is non-deterministic. Confirm asserting it as a **non-null string** (Hamcrest matcher) rather than an exact value is the intended approach (per §6.6).
**A5** Yes, assert it as non-null string
- [ ] **Q6 — Fixtures / payloads.** No example payloads or golden files were supplied. Confirm inline `set-event` JSON payloads are acceptable (vs. extracting fixtures under `src/test/resources/`).
  **A6** - set-event JSON payloads are acceptable, yes
- [ ] **Q7 — `notify-flow` happy assertion.** The app does nothing with the HTTP response. Confirm the happy-path test should only **verify the call cardinality** (`times="1"`) and not assert on a response body shape we don't supply.
**A7** est should only **verify the call cardinality** (`times="1"`)
