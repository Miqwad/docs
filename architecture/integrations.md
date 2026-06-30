# Miqwad — Integration Adapters

One spec per external system Miqwad talks to: purpose, operations, credential handling, and resilience. Every adapter sits behind a Kotlin interface (the domain depends on the interface, never the wire) and is driven through the outbox wherever it mutates money or contracts.

> ## 🟡 Contract detail is **mostly Provisional** — ZATCA + Moyasar now ✅ in hand
>
> The **adapter approach is Decided**: provider-agnostic ports behind a dedicated circuit breaker per system, credential-gated, fetching secrets from the vault at call time.
>
> **✅ Vendor docs received 2026-06-30 for ZATCA (P-2) and Moyasar (P-5)** — their sections below carry the real contracts. **🟡 Still Provisional (no vendor docs yet):** Tajeer (P-1), Wasl (P-3), Absher (P-4), notifications (P-6) — their request/response shapes are refined on receipt of official vendor docs + credential onboarding. See [STATUS](../STATUS.md) **P-1..P-6**.
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

## 2. ZATCA Phase 2 — e-invoicing (Fatoora) ✅ contract in hand (vendor docs received 2026-06-30)

**Purpose.** Generate a signed UBL 2.1 invoice + TLV QR, then **Clear** (Standard / B2B, synchronous) or **Report** (Simplified / B2C, async) via the **Fatoora API v2**. Serves two directions: `dealer_to_customer` and `platform_to_dealer` (Miqwad billing dealers). Signing uses the **official ZATCA SDK — see [ADR-014](../decisions/adr-log.md#adr-014--zatca-phase-2-wrap-the-official-sdk).**

> ### ✅ Decided: **official ZATCA SDK R4.0.0 embedded IN-PROCESS** — no sidecar, no reimplementation
>
> The SDK is run **in-process inside the JDK 25 modular monolith**. **✅ Verified at scaffold (2026-06-30): the SDK's sign/hash pipeline runs on JDK 25** — the in-process smoke test (Santuario canonicalization + SHA-256 over a real sample invoice) passes, so the Java-21-toolchain fallback is **not** needed. We do **not** reimplement the cryptographic stamping and we do **not** stand up a sidecar.
>
> **⚠️ Packaging caveat (F-7), found at scaffold:** the official R4.0.0 jar is a **fat jar that bundles an entire Spring Framework** (~3,000 `org.springframework` classes) plus Jackson/Saxon/HttpClient, so it **cannot sit on the application classpath** — it shadows Spring 7 (`NoSuchMethodError`). Embed it via an **isolated child-first classloader** (preferred — keeps the vendor crypto jar byte-for-byte, isolates *all* bundled conflicts) or a **relocated/repackaged jar**, in both cases behind the `ZatcaSigner` port. In the walking skeleton the jar is **compile-only** and exercised by an **isolated smoke test**, so it cannot affect the running app. Also: onboard against the **GA R4.x** line (R4.0.0 is beta, shipping `3.0-SNAPSHOT` internal libs).

**Split of work — SDK (offline) vs adapter (online):**

| Half | Where | What |
|---|---|---|
| **Offline** | Embedded SDK, in-process | **CSR generation**, **XAdES sign/stamp** (secp256k1 / SHA-256), **TLV QR**, **invoice hash**, **PIH-chain**, and **local validation** (XSD + EN16931 + ZATCA Schematron) — all before any network call |
| **Online** | The adapter | Calls **Fatoora API v2** (header `accept-version: v2`, HTTP Basic = **PCSID username + secret password**) to **Clear** (Standard/B2B — synchronous; ZATCA co-stamps before the invoice reaches the buyer) or **Report** (Simplified/B2C — async, self-stamped, ≤24h) |

| | |
|---|---|
| **Operations** | `generateCsr(config)` · `sign(invoiceXml) → {hash, qr, signedXml}` (SDK, offline) · `validate(signedXml)` (SDK, offline — XSD + EN16931 + Schematron) · `clear(invoice)` / `report(invoice) → zatcaUuid` (Fatoora v2) · `status(invoiceId)` |
| **CSID onboarding (per EGS)** | CSR from a `.properties` config (VAT number — 15 digits, starts/ends `3`; EGS serial `1-…\|2-…\|3-<UUID>`; invoice-type `1100` = Standard + Simplified; branch address; …) → **Compliance CSID** (issued against an **OTP**) → compliance checks (one sample invoice per document type) → **Production CSID + secret (PCSID)**. **Sandbox** uses the `-nonprod` / `-sim` endpoints |
| **Invoice shape** | **UBL 2.1 `<Invoice>` always** (even for credit/debit notes). Type code in the element (**388** invoice / **383** debit / **381** credit) + a `name` attr whose first 2 digits = **`01` Standard** vs **`02` Simplified**. Carries **`ICV`** (invoice counter), **`PIH`** (prev-invoice-hash, base64 — **the first invoice uses the fixed seed** = base64 SHA-256 of `"0"`), and **`QR`** (base64 TLV, 9 tags incl. **tag 9 = ZATCA CA stamp** for Simplified). The XAdES signature lives in `UBLExtensions` |
| **Credentials** | Per-dealer **CSID/PCSID** (compliance → production) for dealer invoices; **Miqwad's OWN platform EGS + CSID** (separate ZATCA onboarding) for platform→dealer (commission / SaaS) Standard B2B invoices. All in the vault — **never committed** (see §[.gitignore], KMS/Vault only) |
| **Resilience** | Signing/validation is local and fast; clearance timeout ~15s + retry queue; **dedicated breaker**; **idempotent on invoice `uuid`/hash**. **Compensation:** a failed clearance **rolls back booking-confirm exactly like a Tajeer failure** — *money is never held against a rental that doesn't legally exist* (see [sagas-outbox-jobs.md](sagas-outbox-jobs.md)) |
| **Error codes** | Validation reject → `zatca_failed` (details carry the ZATCA/Schematron error) · clearance timeout → invoice `pending_clearance` (retry queue; the car is **not** stranded) |
| **Imported guard** | `source = imported` invoices are never cleared or reported |

**Credential checklist (MBA):** CSID/PCSID onboarding per dealer **+ a separate platform EGS/CSID for Miqwad**; OTP for each Compliance CSID; PIH-chain seeding; Standard-vs-Simplified mapping; sandbox `-nonprod`/`-sim` access.

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

## 5. Moyasar — payments ✅ contract in hand (vendor docs received 2026-06-30)

**Purpose.** Process Mada / Visa / Mastercard / Apple Pay / Samsung Pay / STC Pay. Card data stays in the gateway's hosted SDK and **never touches Miqwad servers** — Miqwad stores only gateway references (PCI minimization). Integrated via REST behind a provider-agnostic adapter — see [ADR-015](../decisions/adr-log.md#adr-015--payments-moyasar). **Moyasar is the only payment provider.** Base URL **`https://api.moyasar.com/v1`** (HTTPS only).

> ### ✅ Decided: refundable deposit is **charged up front, refunded on clean return** — NOT an auth-hold
>
> The refundable deposit is taken as a **real payment** at booking and **refunded in full on a clean return** (kept, or partially refunded, on damage). It is **not** a card authorization-hold: auth windows are far too short for multi-day rentals, and Moyasar does not surface issuer auto-void. So Step 2 below uses **charge → refund**, not authorize → capture/void. (Manual auth/capture/void exist on the API — see the row — but we deliberately do not use them for deposits.)

| | |
|---|---|
| **Auth model** | **HTTP Basic**, API key = username, **empty password**. **Publishable key** (`pk_…`) — client-side, **Create-Payment only**, used by the hosted SDK; **Secret key** (`sk_…`) — backend, all operations. **Test vs live by key prefix** |
| **Operations** | `createPayment(amount, source, callback_url, given_id) → payment` (`POST /payments`) · `getPayment(id)` (`GET /payments/:id`) · `refund(id, amount?)` (full or **optional partial**) · *(manual flow, unused for deposits: `manual:true` → authorized; `capture(amount?)`; `void`)* · webhook `onEvent` |
| **Amount** | **Halalas — 1:1 with our money invariant** ([ADR-005](../decisions/adr-log.md#adr-005--money-as-integer-halalas--append-only-ledger)); pass our stored integer **directly**, no conversion |
| **Sources** | `source` ∈ {`creditcard`, `token`, `applepay`, `samsungpay`, `stcpay`}. **Mada is NOT a separate type** — it rides `creditcard` and is identified in the response by **`source.company = "mada"`**. `callback_url` is **required** for `creditcard`/`token` (3DS redirect) |
| **Credentials** | Moyasar keys per environment in the vault; **publishable `pk_…`** shipped to the apps, **secret `sk_…`** backend-only — **never committed** (KMS/Vault only) |
| **Idempotency (create)** | **`given_id`** (a UUIDv4 we generate → becomes the Moyasar payment id); **retry only on 5xx / network**, never on a 4xx |
| **Hosted SDK** | `moyasar-payment-form` collects card data client-side with the **publishable** key and posts **directly to Moyasar** — PAN never touches Miqwad servers (PCI). The RN app uses it via WebView / mobile SDK |
| **Webhooks** | **No HMAC header.** Verify via the **`secret_token` field in the body** (echoes the `shared_secret` set at registration — **constant-time compare**) + check **`live`** + **re-fetch `GET /payments/:id`** with the secret key. Moyasar retries **up to 6 times** → the handler must be **idempotent keyed on event `id`**. Events: `payment_paid` / `failed` / `authorized` / `captured` / `refunded` / `voided` |
| **Resilience** | Timeout ~20s (gateway round-trip excluded from the booking NFR); retries only on idempotent ops; dedicated breaker |
| **Commission reversal** | **Our ledger's job** — Moyasar refunds have no commission concept. On a refund of a `channel = marketplace` booking, **we emit the compensating commission-reversal ledger entry ourselves** (see [sagas-outbox-jobs.md](sagas-outbox-jobs.md)) |
| **Error codes** | `payment_required` / `payment_failed` |

**Credential checklist (MBA):** Moyasar merchant account (SAMA-supervised); test + live keys (`pk_…` / `sk_…`); webhook endpoint + `shared_secret`. **Test card `4111 1111 1111 1111`; pull Moyasar's official Mada + decline test cards before writing the payment test suite** (not in the provided files).

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
| ZATCA | CSID/PCSID (per dealer) + **platform EGS/CSID** (Miqwad) | MBA | ✅ docs in hand; apply CSIDs week 0 | P-2 |
| Wasl | TGA Wasl credentials + obligation confirmation | MBA | apply week 0 | P-3 |
| Absher | access via the Tajeer flow | MBA | with Tajeer | P-4 |
| Moyasar | merchant account + keys (`pk_…`/`sk_…`) | MBA | ✅ docs in hand; provision account | P-5 |
| Notifications | provider selection (e.g. Unifonic) | MBA | select | P-6 |
| GCP / CNTXT | me-central2 access via CNTXT | MBA + Sr BE | prep | — |

---

**See also:** [sagas-outbox-jobs.md](sagas-outbox-jobs.md) (how money/contract calls are dispatched and compensated) · [STATUS](../STATUS.md) (the provisional register) · [decisions/adr-log.md](../decisions/adr-log.md).
