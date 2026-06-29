# Test Catalog — munit-orders-api

| Metric | Value |
| --- | --- |
| Suites | 1 |
| Tests (`munit:test`) | 19 |
| Units covered | 5 / 5 |
| Last run | 19 run · 0 failed · 0 errors · 0 skipped (Anypoint Studio, MMV 4.9.0, 2026-06-29) |
| Line coverage | _read from the Studio MUnit coverage report / `munit-maven-plugin` and record here (gate 80%)_ |
| Last generated | 2026-06-29 |

> Source of truth is `src/test/munit/munit-orders-api-suite.xml`. The full suite was last run green in
> Anypoint Studio. Fill the coverage % from the run's coverage report.

---

## Suite: `munit-orders-api-suite`

- **Source under test:** `src/main/mule/munit-orders-api.xml`
- **Units:** `orders-api-main` (API entry + error handler), `validate-order-subflow` (choice/router),
  `build-order-record-subflow` (transformation), `route-order-flow` (orchestration + choice),
  `notify-flow` (integration/connector)
- **Collaborators mocked:** `http:request` (`post-alert`)

| Test ID | Unit | Category | Given (inputs) | Mocks | Then (expected) | Tags |
| --- | --- | --- | --- | --- | --- | --- |
| `orders-api-main-should-accept-and-notify-large-order` | orders-api-main | happy / branch>100 | amount = 250 | post-alert → 200 | status=accepted · notified=true · post-alert called ×1 (record fields overwritten by HTTP response on notify branch) | smoke, regression |
| `orders-api-main-should-accept-small-order-without-notifying` | orders-api-main | happy / negative / branch≤100 | amount = 100 | post-alert (defence) | status=accepted · notified=false · tier=standard · amount=100 · post-alert called ×0 | smoke, regression |
| `orders-api-main-should-propagate-invalid-order-as-orders-invalid` | orders-api-main | error path | amount = 0 | — | flow raises ORDERS:INVALID (on-error-propagate re-raises) | error, regression |
| `orders-api-main-should-accept-order-when-notification-fails` | orders-api-main | error path / negative | amount = 250 | post-alert → HTTP:CONNECTIVITY | accepted via on-error-continue · notified=false · amount=250 · tier=priority · orderId present · status absent | error, regression |
| `validate-order-should-pass-valid-order` | validate-order-subflow | branch (otherwise) / happy | amount = 250 | — | no error · amount unchanged = 250 | smoke, regression |
| `validate-order-should-raise-invalid-when-amount-missing` | validate-order-subflow | branch (when) / error | `{}` (no amount) | — | flow raises ORDERS:INVALID | error, regression |
| `validate-order-should-raise-invalid-when-amount-zero` | validate-order-subflow | branch (when) / boundary / error | amount = 0 | — | flow raises ORDERS:INVALID | error, regression |
| `validate-order-should-raise-invalid-when-amount-negative` | validate-order-subflow | branch (when) / error | amount = -5 | — | flow raises ORDERS:INVALID | error, regression |
| `build-order-record-should-tier-standard-when-amount-50` | build-order-record-subflow | happy / input-variant | amount = 50 | — | amount=50 · tier=standard · orderId non-null string | smoke, regression |
| `build-order-record-should-tier-standard-when-amount-100` | build-order-record-subflow | boundary | amount = 100 | — | amount=100 · tier=standard · orderId non-null string | smoke, regression |
| `build-order-record-should-tier-priority-when-amount-101` | build-order-record-subflow | boundary | amount = 101 | — | amount=101 · tier=priority · orderId non-null string | smoke, regression |
| `build-order-record-should-tier-priority-when-amount-250` | build-order-record-subflow | happy / input-variant | amount = 250 | — | amount=250 · tier=priority · orderId non-null string | smoke, regression |
| `build-order-record-should-raise-expression-error-when-amount-null` | build-order-record-subflow | boundary / error (A4) | `{}` (no amount) | — | transform raises MULE:EXPRESSION (`Null` vs `Number` comparison in `amount > 100`) | error, regression |
| `build-order-record-should-tier-standard-when-amount-negative` | build-order-record-subflow | boundary (A4) | amount = -5 | — | amount=-5 · tier=standard · orderId non-null string | regression |
| `route-order-should-notify-and-mark-notified-when-amount-over-threshold` | route-order-flow | happy / branch>100 / behavioural | amount = 250 | post-alert → 200 | status=accepted · notified=true · post-alert called ×1 (record overwritten by HTTP response) | smoke, regression |
| `route-order-should-not-notify-when-amount-not-over-threshold` | route-order-flow | negative / branch≤100 | amount = 100 | post-alert (defence) | status=accepted · notified=false · tier=standard · post-alert called ×0 | smoke, regression |
| `route-order-should-propagate-when-notification-fails` | route-order-flow | error path | amount = 250 | post-alert → HTTP:CONNECTIVITY | HTTP:CONNECTIVITY propagated (no local handler) | error, regression |
| `notify-flow-should-call-post-alert-once-on-happy-path` | notify-flow | happy / behavioural | amount = 250 | post-alert → 200 | post-alert called ×1 | smoke, regression |
| `notify-flow-should-propagate-connectivity-error` | notify-flow | error path | amount = 250 | post-alert → HTTP:CONNECTIVITY | HTTP:CONNECTIVITY propagated | error, regression |

> **Implementation note — parameterization.** The design proposed two parameterized tests
> (`validate-order-should-raise-invalid-when-amount-not-positive`, 3 rows; and
> `build-order-record-should-map-amount-and-tier`, 6 rows). MUnit 3.6 declares
> `<munit:parameterizations>` under `<munit:config>` (suite-scoped — the whole suite re-runs per
> parameterization), which cannot host two independent data sets plus the non-parameterized tests
> in one "suite per flow file". They are therefore implemented as the explicit per-case tests listed
> above (`…-amount-missing/-zero/-negative` and `…-tier-*-when-amount-*`), preserving identical coverage.

---

## Coverage gaps

| Unit | Archetype | Reason untested | Owner |
| --- | --- | --- | --- |
| _none_ | — | All 5 units in `munit-orders-api.xml` have at least one test. | — |

**Behavioural caveats (not gaps — flagged for the team):**

- **HTTP status codes (201/400) are not asserted.** Per Open Question A1, the app never wires a
  listener response status: success sets no `201`, and the `httpStatus=400` variable is unused. Unit
  tests via `flow-ref` cannot observe a listener status code regardless.
- **`ORDERS:INVALID` re-raises (A2).** `orders-api-main` propagates the error rather than returning a
  400 body; the test asserts the propagated error type only.
- **Connectivity-recovery payload omits `status` (A3).** `notify-flow` fails before the
  `status=accepted` set-payload, so the recovered record has no `status` field — this is asserted as
  current behaviour, not as a desired contract (it diverges from RAML `OrderAccepted.status`).
- **Null amount makes `build-order-record-subflow` throw (A4, verified by run).** `if (payload.amount > 100)`
  compares `Null` to `Number`, which DataWeave (Mule 4.9) rejects with `MULE:EXPRESSION`. The subflow
  does **not** default to `tier=standard` on a null amount. Unreachable in production (validation runs
  first), but the isolated test documents the actual error. The Phase 1 design assumed `null > 100 → false`;
  that assumption was wrong and the test was corrected to expect `MULE:EXPRESSION`.

---

## Traceability matrix (RAML `orders.raml` → tests)

| Requirement / Spec ref | Flow(s) | Test ID(s) | Status |
| --- | --- | --- | --- |
| `POST /orders` → 201 accepted (success) | orders-api-main, route-order-flow | `orders-api-main-should-accept-and-notify-large-order`, `orders-api-main-should-accept-small-order-without-notifying`, `route-order-should-notify-and-mark-notified-when-amount-over-threshold` | ✅ payload proven (201 status code not wired — see A1) |
| `POST /orders` → 400 invalid (amount missing / ≤ 0, maps from ORDERS:INVALID) | orders-api-main, validate-order-subflow | `orders-api-main-should-propagate-invalid-order-as-orders-invalid`, `validate-order-should-raise-invalid-when-amount-missing`, `validate-order-should-raise-invalid-when-amount-zero`, `validate-order-should-raise-invalid-when-amount-negative` | ⚠️ error type proven; HTTP 400 mapping not wired (A1/A2) |
| `tier = priority` when amount > 100 | build-order-record-subflow | `build-order-record-should-tier-priority-when-amount-101`, `build-order-record-should-tier-priority-when-amount-250` | ✅ |
| `tier = standard` when amount ≤ 100 | build-order-record-subflow | `build-order-record-should-tier-standard-when-amount-50`, `build-order-record-should-tier-standard-when-amount-100`, `build-order-record-should-tier-standard-when-amount-negative` | ✅ |
| `notified = false` when the alert call fails | orders-api-main | `orders-api-main-should-accept-order-when-notification-fails` | ✅ (status field absent — see A3) |
| `notified` true/false by amount branch | route-order-flow | `route-order-should-notify-and-mark-notified-when-amount-over-threshold`, `route-order-should-not-notify-when-amount-not-over-threshold` | ✅ |
| `OrderRequest.additionalProperties: false` (reject extra fields) | — | — | ❌ not enforced in the app; not tested (A4) |

---

## Mandatory-coverage rules (Testing Standard §7.2)

| Rule | Satisfied by |
| --- | --- |
| Every error handler has ≥ 1 test | `orders-api-main-should-propagate-invalid-order-as-orders-invalid` (ORDERS:INVALID) + `orders-api-main-should-accept-order-when-notification-fails` (HTTP:CONNECTIVITY) |
| Every choice branch (incl. default) is hit | validate when/otherwise; route when/otherwise; build-order tier priority/standard rows |
| Every flow with external calls has ≥ 1 mock + ≥ 1 verify-call | notify-flow, route-order-flow, orders-api-main all mock `post-alert` and verify (×1 / ×0) |
| Every public API entry flow: happy + ≥ 1 error→status mapping | `orders-api-main` happy + ORDERS:INVALID mapping (status caveat A1/A2) |
