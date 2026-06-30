# Test Design Document — munit-orders-api

> **Phase 1 deliverable** (Testing Standard §8.2 / §8.3, Documentation Standard §5.4).
> **Status:** proposed — awaiting human validation. No suite XML written; no app changes made.
> **Author:** MUnit test-generation agent · **Date:** 2026-06-29

---

## 1. Summary

| Field | Value |
| --- | --- |
| Application | munit-orders-api |
| Target flow file | `src/main/mule/munit-orders-api.xml` |
| Mule runtime | 4.9.11 (`minMuleVersion="4.9.0"`) |
| API spec | `src/main/resources/api/orders.raml` (`POST /orders`) |
| Global configs | None supplied (HTTP listener + HTTP request configs are inline in the target file) |
| Example payloads | None supplied (inputs derived from the RAML `OrderRequest` type) |
| Existing suites | None |
| Units found | 5 (2 flows + 2 sub-flows + 1 integration flow) **+ 1 flow-level error handler** |
| Proposed tests | **19** (across one suite — see §6 for the one-suite-per-file consequence) |
| Coverage intent | 100% of flows/branches reachable offline; comfortably above the 80% gate |
| Suite plan | **One** suite `munit-orders-api-suite.xml` (all units live in one production file → one suite, §6.2) |

**Payload-flow headline (read before the tables):** the app's `http:request` (doc:name `post-alert`) has **no `target`**, so a mocked *return* **overwrites `payload`**. On every path where the notification call succeeds, the order record built upstream (`orderId`, `amount`, `tier`) is **gone** by the assertion point — only the fields re-added by the *subsequent* `set-payload` survive. On the mocked-*error* notification path the failing call does **not** overwrite `payload`, so the record **does** survive into the error handler. Assertions below are derived from the payload as it actually exists on each path (Testing Standard §4 WARNING).

---

## 2. Unit inventory

| Unit | Archetype (§3) | Source |
| --- | --- | --- |
| `orders-api-main` | API listener / entry flow **+** Error handler (union, §3 NOTE) | `munit-orders-api.xml:18` |
| `validate-order-subflow` | Choice / router flow (also *raises* an error) | `munit-orders-api.xml:51` |
| `build-order-record-subflow` | Transformation sub-flow (pure DataWeave) | `munit-orders-api.xml:65` |
| `route-order-flow` | Orchestration flow **+** Choice / router (union) | `munit-orders-api.xml:81` |
| `notify-flow` | Integration / connector flow (one `http:request`) | `munit-orders-api.xml:99` |
| *(error handler inside `orders-api-main`)* | Error handler / on-error scope (tested **through** `orders-api-main` — an error handler is not independently `flow-ref`-able) | `munit-orders-api.xml:30` |

---

## 3. `validate-order-subflow` — Contract sheet & proposed tests

### Contract sheet
- **Inputs:** `payload.amount` (number, from JSON body).
- **Logic (owned, leave real):** `choice` — `when payload.amount == null or payload.amount <= 0` → `raise-error ORDERS:INVALID`; `otherwise` → `logger`, payload unchanged.
- **Outputs:**
  - *valid path:* `payload` unchanged (the logger does not mutate it).
  - *invalid path:* error `ORDERS:INVALID`, description `"amount must be a positive number"`; no payload produced (flow aborts).
- **Boundary collaborators (mock):** none — pure in-memory logic.
- **Branches:** `when` (invalid) · `otherwise` (valid).
- **Errors raised:** `ORDERS:INVALID`. Errors caught: none.
- **Payload flow:** no external call → `payload` survives on the valid path; on the invalid path the flow aborts at `raise-error` so `<munit:validation>` does not run (assert via `expectedErrorType`).
- **Determinism note:** `amount == null` is the **first** `or` operand, so for an explicitly-null amount the expression short-circuits to `true` **before** evaluating `null <= 0` — that path is well-defined and safe to assert. (Missing-key and non-numeric inputs are *not* asserted — see Open Questions.)

### Proposed tests
| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `validate-order-should-pass-when-amount-positive` | happy / branch=otherwise | successful (valid) outcome branch | `{ amount: 250 }` | — | no error raised; `payload.amount == 250` (payload passes through unchanged) | smoke, regression | No |
| `validate-order-should-raise-invalid-when-amount-zero` | error / boundary (`<= 0` edge) | unit raises an error type; boundary of `<= 0` | `{ amount: 0 }` | — | `expectedErrorType = ORDERS:INVALID`; empty validation block | error, regression | No |
| `validate-order-should-raise-invalid-when-amount-negative` | error / input-variant | invalid equivalence class (negative) | `{ amount: -5 }` | — | `expectedErrorType = ORDERS:INVALID`; empty validation block | error, regression | No |
| `validate-order-should-raise-invalid-when-amount-null` | error / boundary (null) | first `or` clause (`== null`) fires | `{ amount: null }` | — | `expectedErrorType = ORDERS:INVALID`; empty validation block | error, regression | No |

---

## 4. `build-order-record-subflow` — Contract sheet & proposed tests

### Contract sheet
- **Inputs:** `payload.amount`.
- **Logic (owned, leave real):** one `ee:transform` → `{ orderId: uuid(), amount: payload.amount, tier: if (payload.amount > 100) "priority" else "standard" }`.
- **Outputs:** `payload = { orderId, amount, tier }` (application/json).
- **Boundary collaborators (mock):** none.
- **Branches:** the DataWeave conditional `amount > 100` → `priority` / `standard`.
- **Errors:** none raised or caught.
- **Payload flow:** no external call → output survives to the assertion point.
- **Determinism note (§6.6):** `orderId` is `uuid()` — **non-deterministic**. Assert it with a matcher (`notNullValue` / is-a-`String`), never an exact value. Numeric comparisons `amount > 100` for in-range numbers are well-defined. Null/missing/non-numeric amount are **not** asserted here — see Open Questions OQ-3.

### Proposed tests
| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `build-order-record-should-set-priority-tier-when-amount-over-100` | happy / branch (>100) | success outcome + conditional branch | `{ amount: 250 }` | — | `tier == "priority"`; `amount == 250`; `orderId` is a non-null String | smoke, regression | No |
| `build-order-record-should-set-standard-tier-when-amount-under-100` | input-variant / branch (≤100) | other conditional branch | `{ amount: 50 }` | — | `tier == "standard"`; `amount == 50` | regression | No |
| `build-order-record-should-set-standard-tier-at-boundary-100` | boundary (=100) | limit of the `> 100` condition (100 is **not** `> 100`) | `{ amount: 100 }` | — | `tier == "standard"` | regression | No |
| `build-order-record-should-set-priority-tier-just-over-boundary-101` | boundary (just over) | just-over-max | `{ amount: 101 }` | — | `tier == "priority"` | regression | No |

---

## 5. `notify-flow` — Contract sheet & proposed tests

### Contract sheet
- **Inputs:** inbound `payload` (content irrelevant — not read; the call posts to `/post-alert`).
- **Logic:** single `http:request` (method `POST`, path `/post-alert`, doc:name `post-alert`, config `HTTP_Request_config`).
- **Outputs:** `payload` = the HTTP response body (overwrites inbound payload).
- **Boundary collaborators (mock):** `http:request` (`doc:name = post-alert`) — **the** boundary; always mocked.
- **Branches:** none.
- **Errors raised:** `HTTP:CONNECTIVITY` (and other HTTP errors) on call failure. Caught: none (no handler in this flow → propagates).
- **Payload flow:** mocked *return* overwrites `payload`; mocked *error* aborts the flow (no payload at validation; assert via `expectedErrorType`).

### Proposed tests
| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `notify-should-call-post-alert-and-return-its-response` | happy + behavioural | success branch + collaborator must be called | any payload (e.g. `{ amount: 250 }`) | `post-alert` → 200, body `{ ok: true }` | `payload.ok == true` (mock body survives); **verify** `http:request` (post-alert) `times="1"` | smoke, regression | No |
| `notify-should-return-empty-body-when-alert-returns-empty` | boundary / empty-result | empty-result mandatory case (§3 integration) | any payload | `post-alert` → 200, body `{}` | `payload == {}` (empty object); **verify** `times="1"` | regression | No |
| `notify-should-propagate-when-post-alert-connectivity-fails` | error path | mocked collaborator failure | any payload | `post-alert` → **error** `HTTP:CONNECTIVITY` | `expectedErrorType = HTTP:CONNECTIVITY`; empty validation block | error, regression | No |

---

## 6. `route-order-flow` — Contract sheet & proposed tests

### Contract sheet
- **Inputs:** `payload.amount`.
- **Logic / collaborators:**
  - `flow-ref build-order-record-subflow` — pure in-memory transform, **leave real** (does not cross the boundary). After it: `payload = { orderId, amount, tier }`.
  - `choice` — `when payload.amount > 100` → `logger`, `flow-ref notify-flow`, `set-payload (payload ++ { status: "accepted", notified: true })`; `otherwise` → `logger`, `set-payload (payload ++ { status: "accepted", notified: false })`.
  - `notify-flow` reaches the boundary via `http:request` (`post-alert`) → **mock `http:request`** (by doc:name).
- **Branches:** `when` (>100, notifies) · `otherwise` (≤100, no notify).
- **Errors:** none raised/caught here. A `HTTP:CONNECTIVITY` from `post-alert` **propagates out** of this flow (its only handler lives in `orders-api-main`).
- **Payload flow — critical:**
  - **>100 (notify) success path:** `build` → `{orderId,amount,tier:priority}` → `notify-flow`'s mocked `http:request` **overwrites** `payload` with the mock body → `set-payload` appends `{status, notified:true}`. **Result = mockBody ++ {status:"accepted", notified:true}.** `orderId/amount/tier` are **gone** unless the mock body re-supplies them. → assert **only** `status` and `notified`.
  - **≤100 (otherwise) path:** no external call → `build` output survives → `set-payload` appends `{status, notified:false}`. **Result = {orderId, amount, tier, status:"accepted", notified:false}.** → `amount`, `tier`, `status`, `notified`, `orderId` all assertable.
  - **>100 notify-fails path:** mocked *error* does not overwrite payload; flow aborts at the failing call → `expectedErrorType`.

### Proposed tests
| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `route-order-should-notify-and-mark-notified-true-when-amount-over-100` | happy / branch (>100) + behavioural | success branch + collaborator must be called | `{ amount: 250 }` | `post-alert` → 200, body `{}` | `payload.status == "accepted"`; `payload.notified == true`; **verify** `post-alert` `times="1"`. *(amount/tier not asserted — overwritten by mock return.)* | smoke, regression | No |
| `route-order-should-not-notify-and-mark-notified-false-when-amount-100-or-less` | negative / branch (≤100) | "no action" branch | `{ amount: 50 }` | `post-alert` (defence) | `status == "accepted"`; `notified == false`; `tier == "standard"`; `amount == 50`; **verify** `post-alert` `times="0"` | smoke, regression | No |
| `route-order-should-not-notify-at-boundary-100` | boundary (=100) | limit of `> 100` (100 → otherwise) | `{ amount: 100 }` | `post-alert` (defence) | `notified == false`; `tier == "standard"`; **verify** `post-alert` `times="0"` | regression | No |
| `route-order-should-notify-just-over-boundary-101` | boundary (just over) | just-over-max → notify branch | `{ amount: 101 }` | `post-alert` → 200, body `{}` | `notified == true`; **verify** `post-alert` `times="1"` | regression | No |
| `route-order-should-propagate-when-notification-fails` | error path | mocked collaborator failure propagates (no local handler) | `{ amount: 250 }` | `post-alert` → **error** `HTTP:CONNECTIVITY` | `expectedErrorType = HTTP:CONNECTIVITY`; empty validation block | error, regression | No |

---

## 7. `orders-api-main` — Contract sheet & proposed tests

### Contract sheet
- **Inputs:** HTTP `POST /orders` JSON body → `payload = { amount }`. *(Tested via `flow-ref`; the `http:listener` source is **not** invoked by MUnit — see OQ-1.)*
- **Logic / collaborators:**
  - `flow-ref validate-order-subflow` — in-memory, **leave real** (may raise `ORDERS:INVALID`).
  - `flow-ref route-order-flow` — orchestration, **leave real**; its only boundary is `http:request` (`post-alert`) → **mock `http:request`**.
  - `ee:transform` "201 body" — `output application/json --- payload` (pass-through to JSON).
  - **Error handler:** `on-error-propagate ORDERS:INVALID` → transform to empty `application/java {}`, `set-variable httpStatus=400`, then **re-propagates**; `on-error-continue HTTP:CONNECTIVITY` → `set-payload (payload ++ { notified: false })`, then **continues** (flow returns the handler's payload).
- **Branches:** routing branches live in the delegated flows; here the split is success vs. each handled error type.
- **Errors:** raises/propagates `ORDERS:INVALID`; catches & continues on `HTTP:CONNECTIVITY`.
- **Payload flow:**
  - **Happy >100:** `validate` (pass) → `route` (notify path; mock overwrites payload) → `ee:transform` → `payload = mockBody ++ {status:"accepted", notified:true}`. Assert **status, notified** only.
  - **Happy ≤100:** no external call → `payload = {orderId, amount, tier:"standard", status:"accepted", notified:false}` → `ee:transform`. Assert amount/tier/status/notified.
  - **`ORDERS:INVALID` (amount ≤0/null):** `validate` raises → `on-error-propagate` runs (sets empty body + httpStatus=400) then **re-raises** → flow ends in error. `<munit:validation>` does **not** run. Assert `expectedErrorType = ORDERS:INVALID` only. (httpStatus=400 / empty body are **not** observable in this test — OQ-7.)
  - **`HTTP:CONNECTIVITY` (notify fails):** mocked error in `post-alert` propagates up; `on-error-continue` catches; `payload` at the catch point = the record built **before** the failed call = `{orderId, amount, tier}` (not overwritten — call failed) → `set-payload` appends `{notified:false}` → flow continues and returns it. The `ee:transform` "201 body" **does not run** (it sits after the failure point in the main body). **Result = {orderId, amount, tier, notified:false}** — note **no `status` field** on this path (OQ-2).

### Proposed tests
| Test ID | Category | Trigger | Given (input) | Mocks | Then (assert + verify) | Tags | Param? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `orders-api-main-should-accept-and-notify-large-order` | happy / branch (>100) + behavioural | API entry happy path + downstream call | `{ amount: 250 }` | `post-alert` → 200, body `{}` | `status == "accepted"`; `notified == true`; **verify** `post-alert` `times="1"` | smoke, regression | No |
| `orders-api-main-should-accept-small-order-without-notifying` | happy / branch (≤100) + negative | API entry happy, no-notify branch | `{ amount: 50 }` | `post-alert` (defence) | `status == "accepted"`; `notified == false`; `tier == "standard"`; `amount == 50`; **verify** `post-alert` `times="0"` | smoke, regression | No |
| `orders-api-main-should-propagate-invalid-when-amount-not-positive` | error path (mapped to 400) | entry-flow error → status mapping | `{ amount: 0 }` | `post-alert` (defence, expected `times="0"` — but validation skipped) | `expectedErrorType = ORDERS:INVALID`; empty validation block | error, regression | No |
| `orders-api-main-should-accept-order-with-notified-false-when-notification-fails` | error path (caught/continued) | `on-error-continue HTTP:CONNECTIVITY` | `{ amount: 250 }` | `post-alert` → **error** `HTTP:CONNECTIVITY` | `notified == false`; `amount == 250`; `tier == "priority"` (record survives — failed call did not overwrite); **no `status` field present** (OQ-2) | error, regression | No |

---

## 8. Coverage & mandatory-rules check (§7.2)

| Mandatory rule | Satisfied by | Status |
| --- | --- | --- |
| Every **error handler** has ≥1 test | `orders-api-main` handler exercised by `…-propagate-invalid…` (propagate branch) and `…-notified-false-when-notification-fails` (continue branch) | ✅ |
| Every **choice branch** (incl. default) hit | validate: `…pass…` (otherwise) + `…raise-invalid…` (when); route: `…notify…` (when) + `…not-notify…` (otherwise); build tier conditional: priority + standard tests | ✅ |
| Every flow with **external calls** has ≥1 **mock** + ≥1 **verify-call** | `notify-flow` (mock + verify ×1); `route-order-flow` (mock + verify ×1/×0); `orders-api-main` (mock + verify ×1/×0) | ✅ |
| **Public API entry flow**: happy + ≥1 error→status mapping | `orders-api-main` happy (×2) + `…propagate-invalid…` (ORDERS:INVALID → 400 mapping) | ⚠️ partial — see OQ-1/OQ-7: the actual HTTP **status code** is set on the `httpStatus` var / listener and is **not observable** through a `flow-ref` test |
| Coverage gate 80% | All 5 flows and every reachable branch are exercised | ✅ (to be confirmed by the Phase-2 run) |

**No coverage gaps** at the unit level: all 5 units + the error handler have proposed tests.

---

## 9. Traceability to the API spec (`orders.raml`)

| Spec behaviour (RAML) | Flow(s) | Proposed Test ID(s) | Status |
| --- | --- | --- | --- |
| `POST /orders` → **201** `OrderAccepted`, `notified=true`, `tier=priority` when `amount > 100` | orders-api-main → route-order-flow → notify-flow | `orders-api-main-should-accept-and-notify-large-order`; `route-order-should-notify-and-mark-notified-true-when-amount-over-100`; `build-order-record-should-set-priority-tier-when-amount-over-100` | ✅ (status **code** not asserted — OQ-1) |
| `POST /orders` → **201** `OrderAccepted`, `notified=false`, `tier=standard` when `amount ≤ 100` (no notification) | orders-api-main → route-order-flow | `orders-api-main-should-accept-small-order-without-notifying`; `route-order-should-not-notify-and-mark-notified-false-when-amount-100-or-less`; `build-order-record-should-set-standard-tier-when-amount-under-100` | ✅ |
| **201** with `notified=false` *when the alert call fails* | orders-api-main (on-error-continue) | `orders-api-main-should-accept-order-with-notified-false-when-notification-fails` | ⚠️ behaviour covered, but response **omits `status`** which RAML requires (OQ-2) |
| **400** `Error` when amount missing or `<= 0` (maps from `ORDERS:INVALID`) | orders-api-main / validate-order-subflow | `orders-api-main-should-propagate-invalid-when-amount-not-positive`; `validate-order-should-raise-invalid-when-amount-zero` / `…-negative` / `…-null` | ⚠️ error type covered; **400 code + `Error` body shape** not observable/not produced (OQ-1, OQ-6, OQ-7) |
| `OrderAccepted.orderId` is a string | build-order-record-subflow | `build-order-record-should-set-priority-tier-when-amount-over-100` (orderId is non-null String) | ✅ |
| `OrderRequest.amount` is required `> 0` (`additionalProperties: false`) | validate-order-subflow | invalid tests above | ⚠️ RAML does not actually mark `amount` *required*; missing-key behaviour untested (OQ-8) |

---

## 10. Open Questions (must be resolved before Phase 2)

> Per the hard rules, nothing below is guessed. Each item changes one or more proposed tests.

- [ ] **OQ-1 — HTTP status codes are not wired / not observable.** The 201 success and 400 error responses are referenced only by comments and a `set-variable httpStatus="400"`; I see **no `http:listener` response block** that maps `httpStatus` (or a success status) onto the HTTP response, and MUnit `flow-ref` tests do **not** invoke the listener. **How should the "error → mapped HTTP status" mandatory category be satisfied?** (Options: accept that we assert the *error type* / the `httpStatus` var only; add listener-response config to the app — out of scope for me; or treat HTTP status as an integration-test concern.)
**A1** - accept that we assert the error type / the httpStatus var only
- [ ] **OQ-2 — Connectivity-failure response omits `status`.** On the `HTTP:CONNECTIVITY` path the handler only adds `{ notified: false }`, so the returned object has **no `status: "accepted"`** field, which the RAML `OrderAccepted` type requires. Is this intended, or should the handler also set `status`? (I will assert the **observed** behaviour — `notified=false`, no `status` — unless you say otherwise.)
**A2** - assert the **observed** behaviour — `notified=false`, no `status`
- [ ] **OQ-3 — `build-order-record-subflow` with null / missing / non-numeric `amount`.** The transformation archetype mandates empty/null boundary cases, but `tier: if (payload.amount > 100)` with `amount = null` (or a missing key, or a string) hits **DataWeave comparison/coercion semantics I will not guess** (`null > 100`, missing-key selection, `"abc" > 100`). In production `validate-order-subflow` runs first and guarantees a positive number, so these inputs are arguably unreachable. **Do you want these boundary tests added (and if so, I must verify the runtime result first), or are they out of scope because validation upstream precludes them?**
**A3** - Out of scope
- [ ] **OQ-4 — "positive integer" vs. positive number.** `validate-order-subflow`'s `doc:name` says *"positive integer"*, but the logic (`amount <= 0`) accepts positive **decimals** (e.g. `0.5` is valid) and the RAML type is `number`. Is the integer wording just a label, or should non-integers be rejected? (Affects whether a `amount = 0.5` test asserts *valid*.)
**A4** - accept positive decimals
- [ ] **OQ-5 — Mock granularity for `orders-api-main`.** I propose mocking only the real boundary (`http:request` / `post-alert`) and leaving `validate-order-subflow` and `route-order-flow` **real** (they are in-memory). The §3 table also lists "the orchestration sub-flows" as typical mocks. **Confirm boundary-only mocking** (keeps the entry-flow test meaningful and deterministic) vs. mocking the downstream `flow-ref`s.
**A5** - Mock only real boundary (`http:request` / `post-alert`)
- [ ] **OQ-6 — 400 body shape.** The `ORDERS:INVALID` handler emits an **empty `application/java {}`**, not the RAML `Error { error, message }` shape. Intended? (Currently unobservable anyway due to OQ-7; noted for spec alignment.)
**A6** - Not intended, flag it as a bug
- [ ] **OQ-7 — `on-error-propagate` re-raises.** Because the `ORDERS:INVALID` handler uses `on-error-propagate`, the flow ends in error and `<munit:validation>` does **not** run — so I **cannot** assert `httpStatus == 400` or the empty body in that test; only `expectedErrorType = ORDERS:INVALID`. Acceptable, or do you want the handler restructured (app change — out of scope) to make the mapping assertable?
**A7** - Acceptable
- [ ] **OQ-8 — `amount` missing entirely.** RAML sets `additionalProperties: false` but does **not** mark `amount` as `required`. When the key is absent, `payload.amount` selects to `null`, which `validate` treats as invalid via its first `or` clause. **Do you want an explicit "missing amount key" test** (I'd verify the selector result at run time in Phase 2 before asserting), or is the explicit-`null` test sufficient?
**A8** - the explicit-`null` test sufficient
- [ ] **OQ-9 — Parameterization.** Under "one suite per production flow file" (§6.2), **all 19 tests live in one suite**, which mixes data-driven and non-data-driven cases. Since `<munit:parameterizations>` is **suite-scoped** (§6.5), the validate invalid-input trio (`zero`/`negative`/`null`) — otherwise a ≥3 data-only parameterization candidate — must stay **explicit per-case**. Confirm you accept explicit tests over splitting into a separate parameterized suite.
**A9** - Confirmed

---

## 11. Next step

Read and validate this document, then answer the Open Questions. On approval I will run **Phase 2** (Testing Standard §8.4): emit `src/test/munit/munit-orders-api-suite.xml` implementing exactly the approved tests, run it, reconcile every failure, and generate the catalog + traceability matrix.
