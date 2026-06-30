# Miqwad — Integration Adapters

One spec per external system Miqwad talks to: purpose, operations, credential handling, and resilience. Every adapter sits behind a Kotlin interface (the domain depends on the interface, never the wire) and is driven through the outbox wherever it mutates money or contracts.

> ## 🟡 Contract detail is **Provisional**
>
> The **adapter approach is Decided**: provider-agnostic ports behind a dedicated circuit breaker per system, credential-gated, fetching secrets from the vault at call time. The **request/response shapes are Provisional** — they are refined on receipt of official vendor docs + credential onboarding. See [STATUS](../STATUS.md) **P-1..P-6**.
>
> **Payments are Moyasar only.** Cross-links: [ADR-014](../decisions/adr-log.md#adr-014--zatca-phase-2-wrap-the-official-sdk) (ZATCA SDK) · [ADR-015](../decisions/adr-log.md#adr-015--payments-moyasar) (Moyasar) · [ADR-016](../decisions/adr-log.md#adr-016--api-surface-hand-written-rest-v1-for-the-domain--elide-jsonapi-for-crud) (API surface).

---

## Common adapter contract

Every adapter follows the same shape, so one external outage can never cascade into another:

| Aspect | Rule |
|---|---|
| **Interface** | `interface <System>Client` + an `Http<System>Client` impl + a `Stub<System>Client` for local dev |
| **Resilience** | Per-call timeout, bounded retry + backoff, and a **dedicated circuit breaker per system** (a Tajeer outage must not trip ZATCA). Spring Framework 7 core resilience now; Resilience4j for advanced breakers once Boot-4-ready |
| **Secrets** | Credentials fetched from **Secret Manager** at call time — never logged, never stored in Postgres plaintext |
| **Logging** | Structured request/response logging with sensitive fields redacted, for audit + dispute defence |
| **Audit** | Every call writes a `*_link` audit row |
| **Idempotency** | Money- and contract-mutating calls flow through the outbox (see [sagas-outbox-jobs.md](sagas-outbox-jobs.md)); register/clear are guarded by an existing ref so a retry applies once |
| **Imported guard** | Rows with `source = imported` are records already filed with the government during migration — **never** registered, cleared, reported, or closed again (see [data-migration](../engineering/data-migration.md)) |

Adapters are built against **WireMock** until sandbox access lands, so development is never blocked. Each integration is **Togglz-gated per dealer** as that dealer's credentials provision.

---

## 1. Tajeer — unified e-rental contract (TGA / Naql) 🟡 P-1

**Purpose.** Register, close, and cancel the legally-required rental contract. Absher e-sign is part of this flow.

| | |
|---|---|
| **Operations** | `register(bookingContext) → tajeerContractRef` · `close(contractRef)` · `cancel(contractRef)` · `status(contractRef)` |
| **Credentials** | The **dealer's own Naql account** — only the lessor may operate the contract. Stored per-dealer in the vault; gated per dealer by Togglz until provisioned |
| **Resilience** | Timeout ~10s; 3 retries (idempotent `register` guarded by an existing ref); breaker per dealer-tenant pool |
| **Error codes** | Expired operating card / insurance → `tajeer_failed` (422, reason surfaced to dealer) · invalid credentials → `tajeer_auth_failed` · system down → circuit-open → saga `pending_contract` path |
| **Imported guard** | `source = imported` contracts are never registered or closed |

**Credential checklist (MBA):** TGA license + active Naql account per dealer; sandbox endpoint + test credentials; production credential pathway.

---

## 2. ZATCA Phase 2 — e-invoicing (Fatoora) 🟡 P-2

**Purpose.** Generate signed UBL 2.1 XML + TLV QR, then **clear** (B2C simplified) or **report** (B2B standard) to ZATCA. Serves two directions: `dealer_to_customer` and `platform_to_dealer` (Miqwad billing dealers). Signing **wraps the official ZATCA SDK** — see [ADR-014](../decisions/adr-log.md#adr-014--zatca-phase-2-wrap-the-official-sdk).

| | |
|---|---|
| **Operations** | `sign(invoiceXml) → {hash, qr}` (official ZATCA SDK, offline) · `clear(invoice)` / `report(invoice) → zatcaUuid` · `status(invoiceId)` |
| **Credentials** | Per-dealer **CSID** (compliance / production) for dealer invoices; **Miqwad's own CSID** for platform→dealer invoices. In the vault |
| **Resilience** | Signing is local and fast; clearance timeout ~15s + retry queue; breaker |
| **Error codes** | Validation reject → `zatca_failed` (details carry the ZATCA error) · clearance timeout → invoice `pending_clearance` (retry queue; the car is **not** stranded) |
| **Imported guard** | `source = imported` invoices are never cleared or reported |

**Credential checklist (MBA):** CSID onboarding per dealer + for Miqwad; PIH chain handling; simplified-vs-standard mapping.

---

## 3. Wasl — fleet tracking (→ TGA) 🟡 P-3

**Purpose.** Register passenger rental vehicles and report movement/trips to TGA's Wasl platform.

| | |
|---|---|
| **Operations** | `registerVehicle(vehicle)` · `reportTrip/position(batch)` · `status(vehicle)` |
| **Credentials** | Wasl API credentials (platform or per-dealer per TGA rules) in the vault |
| **Integration** | The telemetry pipeline (§7) feeds Wasl in the same provider-agnostic shape — Wasl is one `telemetry_source`. Reporting is batched via JobRunr |
| **Error codes** | `wasl_failed` — **non-blocking** to core booking (degraded path) |

**Credential checklist (MBA):** confirm exact Wasl reporting obligations for passenger rental fleets with TGA; device/data format; credential pathway.

---

## 4. Absher — renter identity + e-signature 🟡 P-4

**Purpose.** Verify renter identity and capture the e-signature, invoked **within the Tajeer flow**.

| | |
|---|---|
| **Operations** | `verifyIdentity(customer)` · `requestSignature(contractRef) → signedAt` |
| **Error codes** | `absher_failed` / `identity_unverified` — **blocks** confirm |
| **Scope** | V1 = primary renter only; an additional authorized driver is V2 (see [backlog](../delivery/backlog.md)) |

**Credential checklist (MBA):** access route (typically via the Tajeer integration); test identities in sandbox.

---

## 5. Moyasar — payments 🟡 P-5

**Purpose.** Process Mada / Visa / Mastercard / Apple Pay / STC Pay. Card data stays in the gateway's hosted fields/SDK and **never touches Miqwad servers** — Miqwad stores only gateway references (PCI minimization). Integrated via REST behind a provider-agnostic adapter — see [ADR-015](../decisions/adr-log.md#adr-015--payments-moyasar). **Moyasar is the only payment provider.**

| | |
|---|---|
| **Operations** | `createPaymentIntent(amount, breakdown) → clientSecret` · `capture` · `authorizeHold(deposit)` · `void` · `refund(partial \| full)` · webhook `onEvent` |
| **Credentials** | Moyasar API keys per environment (vault); publishable key shipped to the apps |
| **Resilience** | Timeout ~20s (gateway round-trip excluded from the booking NFR); retries only on idempotent ops; breaker |
| **Webhooks** | **Signature-verified and deduped by gateway event id** — processed once |
| **Error codes** | `payment_required` / `payment_failed`; deposit-hold expiry policy (re-auth or capture-and-refund) defined up front |

**Credential checklist (MBA):** Moyasar merchant account (SAMA-supervised); sandbox keys; webhook endpoint + secret.

---

## 6. Notifications — email / SMS / push 🟡 P-6

**Purpose.** Transactional messaging to customers and dealer staff (booking confirmations, handover/return prompts, settlement and clearance updates, operational alerts).

| | |
|---|---|
| **Dispatch** | Sent as JobRunr jobs off the outbox, behind a provider-agnostic notification port |
| **Provider** | **Provisional** — provider not yet selected (e.g. Unifonic for SMS in KSA); send/delivery contracts firm up on selection |
| **Credentials** | Provider API keys in the vault, never logged |

**Provisional (P-6):** provider selection + send/delivery contracts pending — see [STATUS](../STATUS.md).

---

## 7. Telemetry ingest — GPS / OBD / Wasl

**Purpose.** Provider-agnostic vehicle position/odometer intake. This is the single source the Wasl adapter (§3) reads from.

| | |
|---|---|
| **Operation** | `POST /dealer/tracking/ingest` (device/gateway → platform): validate, **dedup by `device + recorded_at`**, persist to the partitioned `telemetry_ping`, update `vehicle.live_geo` + Redis `vehicle_live` |
| **Sources** | `gps_device \| obd \| manual \| wasl` — same shape |
| **Resilience** | Out-of-order / delayed / duplicate pings handled (**last-writer-by-device-time**); an offline device falls back to last-known position + a staleness flag |

---

## Credential application tracker — the long pole (MBA, prep week 0)

The government approval queues are on the **V1 critical path**. Adapters are built against WireMock so dev is never blocked; each integration is Togglz-gated per dealer as it provisions.

| System | Needs | Owner | Status | STATUS |
|---|---|---|---|---|
| Tajeer / Naql | TGA license + Naql account (per dealer) | MBA | apply week 0 | P-1 |
| ZATCA | CSID onboarding (per dealer + Miqwad) | MBA | apply week 0 | P-2 |
| Wasl | TGA Wasl credentials + obligation confirmation | MBA | apply week 0 | P-3 |
| Absher | access via the Tajeer flow | MBA | with Tajeer | P-4 |
| Moyasar | merchant account + sandbox | MBA | apply week 0 | P-5 |
| Notifications | provider selection (e.g. Unifonic) | MBA | select | P-6 |
| GCP / CNTXT | me-central2 access via CNTXT | MBA + Sr BE | prep | — |

---

**See also:** [sagas-outbox-jobs.md](sagas-outbox-jobs.md) (how money/contract calls are dispatched and compensated) · [STATUS](../STATUS.md) (the provisional register) · [decisions/adr-log.md](../decisions/adr-log.md).
