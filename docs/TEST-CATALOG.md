# Test Catalog — munit-orders-api

> Generated per the [MUnit Test Documentation Standard v1.0](../MUNIT-TEST-DOCUMENTATION-STANDARD.md).
> Source of truth: `src/test/munit/munit-orders-api-suite.xml`. Coverage from `reports/munit-orders-api-report.html`.
> Regenerable from the suite XML + reports — do not hand-edit prose the XML cannot reproduce.

| Metric | Value |
| --- | --- |
| Suites | 1 |
| Tests (`munit:test`) | 20 |
| Units covered | 5 / 5 |
| Line coverage | 100% application coverage · 16/16 processors (gate 80%) — `reports/munit-orders-api-report.html` |
| Last run | **TODO** — no valid run-summary artifact in repo. The only `target/surefire-reports/*` entry is a **failed** Maven run (`errors="1"`, embedded-container BOM `mule-runtime-impl-no-services-bom:4.9.0` unresolved — environment issue, not a test failure). The 100% coverage report evidences a successful run (Anypoint Studio, MMV 4.9.0, per prior records), but the run/failed/errors/skipped counts are not present in the provided reports. Record from an actual `munit-maven-plugin` run. |
| Last generated | 2026-06-30 |

> [!IMPORTANT]
> Per the standard, a green/coverage claim is valid only after a real run. The **coverage** figure above
> is taken from the actual coverage report; the **run-summary counts** are flagged TODO because the repo
> contains no passing run artifact to source them from.

---

## Suite: `munit-orders-api-suite`

- **Source under test:** `src/main/mule/munit-orders-api.xml`
- **Units (+ archetype):**
  - `orders-api-main` — API entry flow + error handler (union)
  - `validate-order-subflow` — choice / router (raises `ORDERS:INVALID`)
  - `build-order-record-subflow` — transformation (pure DataWeave)
  - `route-order-flow` — orchestration + choice
  - `notify-flow` — integration / connector (one `http:request`)
- **Collaborators mocked:** `http:request` (`doc:name = post-alert`)
- **Tags in use:** `smoke`, `regression`, `error`

> **Payload note (from the suite + source):** `post-alert` (`http:request`) has no `target`, so a mocked
> *return* **overwrites** `payload`. On notify-success paths the upstream record (`orderId`/`amount`/`tier`)
> is replaced by the mock body; only fields re-added by the subsequent `set-payload` survive. A mocked
> *error* does **not** overwrite the payload, so the record survives into the error handler.

| Test ID | Unit | Category | Given (inputs) | Mocks | Then (expected) | Tags |
| --- | --- | --- | --- | --- | --- | --- |
| `validate-order-should-pass-when-amount-positive` | validate-order-subflow | happy / branch (otherwise) | amount = 250 | — | no error · amount == 250 (payload passes through unchanged) | smoke, regression |
| `validate-order-should-raise-invalid-when-amount-zero` | validate-order-subflow | error / boundary (≤0 edge) | amount = 0 | — | raises `ORDERS:INVALID` (asserted via `expectedErrorType`) | error, regression |
| `validate-order-should-raise-invalid-when-amount-negative` | validate-order-subflow | error / input-variant | amount = -5 | — | raises `ORDERS:INVALID` | error, regression |
| `validate-order-should-raise-invalid-when-amount-null` | validate-order-subflow | error / boundary (null, first `or` clause) | amount = null | — | raises `ORDERS:INVALID` | error, regression |
| `build-order-record-should-set-priority-tier-when-amount-over-100` | build-order-record-subflow | happy / branch (>100) | amount = 250 | — | tier == priority · amount == 250 · orderId present (non-null) | smoke, regression |
| `build-order-record-should-set-standard-tier-when-amount-under-100` | build-order-record-subflow | input-variant / branch (≤100) | amount = 50 | — | tier == standard · amount == 50 | regression |
| `build-order-record-should-set-standard-tier-at-boundary-100` | build-order-record-subflow | boundary (=100) | amount = 100 | — | tier == standard (100 is not > 100) | regression |
| `build-order-record-should-set-priority-tier-just-over-boundary-101` | build-order-record-subflow | boundary (just over) | amount = 101 | — | tier == priority | regression |
| `notify-should-call-post-alert-and-return-its-response` | notify-flow | happy / behavioural | amount = 250 (any payload) | post-alert → `{ ok: true }` | response.ok == true (mock body survives) · post-alert called ×1 | smoke, regression |
| `notify-should-return-empty-body-when-alert-returns-empty` | notify-flow | boundary / empty-result | any payload | post-alert → `{}` | response is empty (`sizeOf == 0`) · post-alert called ×1 | regression |
| `notify-should-propagate-when-post-alert-connectivity-fails` | notify-flow | error path | any payload | post-alert → **error** `HTTP:CONNECTIVITY` | raises `HTTP:CONNECTIVITY` (no local handler; asserted via `expectedErrorType`) | error, regression |
| `route-order-should-notify-and-mark-notified-true-when-amount-over-100` | route-order-flow | happy / branch (>100) / behavioural | amount = 250 | post-alert → `{}` (200) | status == accepted · notified == true · post-alert called ×1 *(amount/tier overwritten by mock return)* | smoke, regression |
| `route-order-should-not-notify-and-mark-notified-false-when-amount-100-or-less` | route-order-flow | negative / branch (≤100) | amount = 50 | post-alert (defence) | status == accepted · notified == false · tier == standard · amount == 50 · post-alert called ×0 | smoke, regression |
| `route-order-should-not-notify-at-boundary-100` | route-order-flow | boundary (=100) | amount = 100 | post-alert (defence) | notified == false · tier == standard · post-alert called ×0 | regression |
| `route-order-should-notify-just-over-boundary-101` | route-order-flow | boundary (just over) | amount = 101 | post-alert → `{}` (200) | notified == true · post-alert called ×1 | regression |
| `route-order-should-propagate-when-notification-fails` | route-order-flow | error path | amount = 250 | post-alert → **error** `HTTP:CONNECTIVITY` | raises `HTTP:CONNECTIVITY` (propagated, no local handler) | error, regression |
| `orders-api-main-should-accept-and-notify-large-order` | orders-api-main | happy / branch (>100) / behavioural | amount = 250 | post-alert → `{}` (200) | status == accepted · notified == true · post-alert called ×1 | smoke, regression |
| `orders-api-main-should-accept-small-order-without-notifying` | orders-api-main | happy / negative / branch (≤100) | amount = 50 | post-alert (defence) | status == accepted · notified == false · tier == standard · amount == 50 · post-alert called ×0 | smoke, regression |
| `orders-api-main-should-propagate-invalid-when-amount-not-positive` | orders-api-main | error path | amount = 0 | — | raises `ORDERS:INVALID` (mapped to HTTP 400 by the listener; status code not observable via `flow-ref` — OQ-1) | error, regression |
| `orders-api-main-should-accept-order-with-notified-false-when-notification-fails` | orders-api-main | error path / negative | amount = 250 | post-alert → **error** `HTTP:CONNECTIVITY` | accepted via `on-error-continue` · notified == false · amount == 250 · tier == priority · **no `status` field** (observed — OQ-2) | error, regression |

---

## Coverage gaps

All 5 application units have ≥1 test — **no unit-level coverage gaps**.

| Unit | Archetype | Reason untested | Owner |
| --- | --- | --- | --- |
| — | — | All units covered (5/5); application coverage 100% | — |

### Behavioural gaps (spec behaviours not provable through the current tests)

These are not missing *units* but spec behaviours the suite cannot assert as-built; tracked as TODO.

| Behaviour | Why not asserted | Tracking | Owner |
| --- | --- | --- | --- |
| HTTP **status codes** (201 / 400) on the listener response | MUnit `flow-ref` does not invoke `http:listener`; no response-status mapping wired in the app | TODO — OQ-1 (accepted: assert error type / `httpStatus` var only) | Gonzalo Marcos |
| **400 `Error` body shape** `{ error, message }` | `ORDERS:INVALID` handler emits empty `application/java {}`, not the RAML `Error` type | TODO — OQ-6 (flagged as a bug) | Gonzalo Marcos |
| **`status` field on the notification-failure response** | `on-error-continue` only adds `{ notified: false }`; RAML `OrderAccepted` requires `status` | TODO — OQ-2 (asserted observed behaviour: no `status`) | Gonzalo Marcos |
| **Missing-`amount`-key** input | Not in suite; explicit-`null` test deemed sufficient | TODO — OQ-8 (out of scope by decision) | Gonzalo Marcos |
| `build-order-record-subflow` with null / non-numeric `amount` | DataWeave coercion semantics; validation upstream precludes in production | TODO — OQ-3 (out of scope by decision) | Gonzalo Marcos |

---

## Traceability matrix

Sourced from `src/main/resources/api/orders.raml` (`POST /orders`, types `OrderRequest` / `OrderAccepted` / `Error`).

| Requirement / Spec ref | Flow(s) | Test ID(s) | Status |
| --- | --- | --- | --- |
| `POST /orders` → **201** `OrderAccepted`, `notified=true`, `tier=priority` when `amount > 100` | orders-api-main → route-order-flow → notify-flow / build-order-record-subflow | `orders-api-main-should-accept-and-notify-large-order`; `route-order-should-notify-and-mark-notified-true-when-amount-over-100`; `route-order-should-notify-just-over-boundary-101`; `build-order-record-should-set-priority-tier-when-amount-over-100`; `build-order-record-should-set-priority-tier-just-over-boundary-101` | ✅ (status **code** not asserted — OQ-1) |
| `POST /orders` → **201** `OrderAccepted`, `notified=false`, `tier=standard` when `amount ≤ 100` (no notification) | orders-api-main → route-order-flow / build-order-record-subflow | `orders-api-main-should-accept-small-order-without-notifying`; `route-order-should-not-notify-and-mark-notified-false-when-amount-100-or-less`; `route-order-should-not-notify-at-boundary-100`; `build-order-record-should-set-standard-tier-when-amount-under-100`; `build-order-record-should-set-standard-tier-at-boundary-100` | ✅ |
| **201** with `notified=false` *when the alert call fails* | orders-api-main (`on-error-continue HTTP:CONNECTIVITY`) | `orders-api-main-should-accept-order-with-notified-false-when-notification-fails` | ⚠️ behaviour covered, but response **omits `status`** which RAML `OrderAccepted` requires (OQ-2) |
| **400** `Error` when amount missing or `<= 0` (maps from `ORDERS:INVALID`) | orders-api-main / validate-order-subflow | `orders-api-main-should-propagate-invalid-when-amount-not-positive`; `validate-order-should-raise-invalid-when-amount-zero`; `…-negative`; `…-null` | ⚠️ error **type** covered; **400 code + `Error` body shape** not observable / not produced (OQ-1, OQ-6) |
| `notify-flow` reaches the `post-alert` boundary exactly once on notify | notify-flow | `notify-should-call-post-alert-and-return-its-response`; `notify-should-return-empty-body-when-alert-returns-empty` | ✅ |
| Notification connectivity failure surfaces as `HTTP:CONNECTIVITY` (before the entry-flow handler catches it) | notify-flow / route-order-flow | `notify-should-propagate-when-post-alert-connectivity-fails`; `route-order-should-propagate-when-notification-fails` | ✅ |
| `OrderAccepted.orderId` is a string | build-order-record-subflow | `build-order-record-should-set-priority-tier-when-amount-over-100` (orderId non-null) | ✅ (presence only; type not asserted) |
| `OrderAccepted.tier` enum `[standard, priority]` (priority when `amount > 100`) | build-order-record-subflow | `build-order-record-should-set-priority-tier-when-amount-over-100`; `…-standard-tier-when-amount-under-100`; `…-standard-tier-at-boundary-100`; `…-priority-tier-just-over-boundary-101` | ✅ |
| `OrderRequest.amount` must be `> 0` | validate-order-subflow | the three `validate-order-should-raise-invalid-*` tests | ⚠️ RAML does not mark `amount` `required`; missing-key case untested (OQ-8) |
| `Error` type `{ error, message }` produced on 400 | orders-api-main error handler | — | ❌ gap — handler emits empty `{}`, not `Error` (OQ-6) |

---

## Notes

- **Open Questions OQ-1 … OQ-9** are recorded and resolved in [`docs/TEST-DESIGN.md` §10](./TEST-DESIGN.md#10-open-questions-must-be-resolved-before-phase-2). The ⚠️/❌ rows above trace back to those decisions; none are silent omissions.
- Every `munit:test` carries a full-sentence `description` (Layer 1) — no description-TODOs.
- This catalog is fully regenerable from the suite XML + coverage report; re-run the documentation pass after any suite edit.
