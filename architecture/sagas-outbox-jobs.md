# Miqwad — Sagas, Outbox & Jobs

The two money-and-government sagas (booking-confirm, return-settle), the transactional outbox that drives them, JobRunr as the dispatcher, idempotency, reconciliation, and first-class compensation. Cross-link: [ADR-008](../decisions/adr-log.md#adr-008--sagas-via-persisted-state-machine--transactional-outbox-dispatched-by-jobrunr).

> ✅ **Decided.** Each flow is a **persisted state machine + transactional outbox**; the **outbox table is the source of truth**; **JobRunr** is the single dispatcher; compensation actions are **first-class**.
>
> **The rule that governs everything below:** *money is never held against a rental that doesn't legally exist — if Tajeer registration fails after payment, auto-void/refund.*

---

## 1. Mechanism — transactional outbox + JobRunr

- A state change **and** its intent to call an external system are written in **one DB transaction**: the domain row(s) plus an `outbox` row (`event_type`, `payload`, `status = pending`, `next_attempt_at`). This guarantees the DB commit and the external call never diverge.
- **JobRunr** runs a recurring dispatcher that claims pending `outbox` rows (`FOR UPDATE SKIP LOCKED`), performs the external call through the relevant adapter, and:
  - on **success** → marks `dispatched`;
  - on **failure** → increments `attempts`, sets `next_attempt_at` with backoff;
  - on **exhaustion** → routes to the compensation / escalation path.
- **Saga state** lives on the aggregate (`booking.status` + `rental_contract.status` + `invoice.clearance_status` + `payment.status`), with a lightweight `booking_saga_state` when a step sequence needs explicit tracking. Transitions are guarded.
- **Why not synchronous orchestration:** with money + government contracts in the loop, durable, inspectable, retryable state beats in-memory calls that vanish on a crash.

JobRunr is also the single job engine for reconciliation, telemetry processing, partition maintenance, notifications, payout/commission aggregation, deposit-hold expiry, and import batches.

---

## 2. Idempotency (non-negotiable)

| Layer | Guarantee |
|---|---|
| **Endpoints** | Every money/external endpoint accepts an `Idempotency-Key` header; `IdempotencyService` stores `(key → response)` and replays the original result on repeat |
| **Outbox handlers** | Idempotent by design — keyed on aggregate + step, so a retried job, a duplicated webhook, or a re-dispatched row applies **once** (e.g. "register contract for booking X" checks whether a `tajeer_contract_ref` already exists before calling Tajeer) |
| **Gateway webhooks** | Signature-verified **and** deduped by gateway event id |

---

## 3. Booking-confirm saga

**Goal:** customer pays → booking confirmed → Tajeer contract registered, with **money never held against a rental that doesn't legally exist**.

| Step | Action | On failure → compensation |
|---|---|---|
| 1 | Reserve: `booking = pending` + write `availability_block` (exclusion constraint) | Window taken → `409 vehicle_unavailable`; nothing else created |
| 2 | Payment: authorize rental + **deposit hold** (Moyasar) | Auth fails → stays `pending`, **release block**, tell customer; no contract attempted |
| 3 | Outbox `register_contract` dispatched by JobRunr → Tajeer register | — |
| 4a | Tajeer OK → `contract = registered`, `booking = confirmed`, ledger holds recorded | — |
| 4b | Tajeer **fails** (e.g. expired operating card) | `booking = confirmation_failed`; **auto-void/refund** the authorization within the window; release block; notify customer + dealer with the Tajeer reason; dealer fixes and retries. **Money is never kept.** |
| 4c | Tajeer **down** (circuit open) | `booking = pending_contract`; payment **authorized but not captured** (hold only); retry queue; if unresolved within the hold window → void + refund + ask to retry; dealer sees a "Tajeer unavailable" banner |
| 5 | Edge: Tajeer registered but local confirm write lost | Reconciliation detects the orphan `tajeer_contract_ref` and either completes the local booking or closes the Tajeer contract — **no double registration** |

Commission is computed at confirm **only if `channel = marketplace`**.

---

## 4. Return & settlement saga

**Goal:** return inspection → ZATCA invoice → deposit settle → Tajeer close, **without stranding the car** on an external delay.

| Step | Action | On failure → behaviour |
|---|---|---|
| 1 | Return inspection (photos, odometer, fuel, damages); `vehicle = available` **immediately** | A ZATCA/Tajeer delay must **not** strand the car — release regardless |
| 2 | Outbox `clear_invoice` → ZATCA clear/report | Fails/times out → invoice `pending_clearance` queue, retried; customer gets the compliant invoice once cleared |
| 3 | Deposit settle: clean → **release/refund** in full promptly; damaged → **partial capture** up to evidenced damage, remainder released | Refund fails → `settlement_pending`, auto-retry, then a manual finance task + alert (**never silently dropped**) |
| 4 | Ledger entries: deposit, VAT, commission; commission **reversed** proportionally on any refund | — |
| 5 | Outbox `close_contract` → Tajeer close | Fails → retry queue; the operational return is not blocked; close-reconciliation ensures eventual closure |
| 6 | `booking = completed` | Guard: requires return inspection + invoice `cleared` / `reported` |

---

## 5. Reconciliation jobs (JobRunr, scheduled)

| Job | Cadence | Purpose |
|---|---|---|
| **Payments ↔ gateway** | daily | Mismatches alert finance |
| **Contracts ↔ Tajeer** | scheduled | Detect orphaned / again-needed registrations and contract closes |
| **Invoices ↔ ZATCA** | scheduled | Detect uncleared/failed invoices; re-drive the clearance queue |
| **Outbox sweeper** | scheduled | Re-queue rows stuck `pending` past threshold; alert on poison rows (max attempts) |

All reconciliation is **idempotent** and **skips `source = imported` rows** (§6).

---

## 6. Imported-document guard (see [data-migration](../engineering/data-migration.md))

- Rows with `rental_contract.source = imported` or `invoice.source = imported` are records of documents **already filed with the government** during migration.
- The booking-confirm and return-settle sagas, the outbox dispatcher, and **every reconciliation job** must **exclude** these rows — never register, clear, report, or close them again.
- Enforced **in code** (a `source != imported` predicate on every gov-driving query) **and** asserted by a dedicated test: import a historical contract + invoice, run all sagas/reconcilers, assert **zero external calls** for them.

---

## 7. Failure-handling principles (carry into every adapter)

- **Timeouts + bounded retries + a circuit breaker per external system**; a degraded dependency **fails fast into the queued path**, not a hung user request.
- Every external call is logged with request/response (sensitive fields redacted) for audit + dispute defence.
- **Compensation actions — void authorization, release block, refund, contract close — are first-class operations**, unit-tested in isolation and exercised by the saga e2e tests, **including the payment-OK / Tajeer-fail branch**.

---

**See also:** [integrations.md](integrations.md) (the adapters these sagas call) · [decisions/adr-log.md](../decisions/adr-log.md) (ADR-008) · [STATUS](../STATUS.md).
