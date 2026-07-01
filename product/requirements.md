# Miqwad — Business Flow & Requirements

The supply/demand lifecycle and the full MoSCoW requirement set for V1 — what a Saudi car-rental dealer legally and operationally needs to run on the platform.

> 🟡 **Provisional** — the **business-flow detail** (step order, who-does-what, required fields per step, exceptions) is assumed and must be confirmed by **dealer discovery interviews**. Tracked as **P-7..P-12** in [../STATUS.md](../STATUS.md). The requirement *set* and packaging are settled; the operational *choreography* may shift.

---

## Who we serve

Miqwad is a two-sided platform with three actors in every flow.

| Actor | Who | Role |
|---|---|---|
| **Dealer / agency** (supply, our paying customer) | TGA-licensed rental office (مكتب تأجير), 5–500 cars, single- to multi-branch | Owns/controls the fleet; rents to renters; must file every contract via Tajeer, invoice via ZATCA, track via Wasl |
| **Customer** (demand) | Saudi nationals, residents/iqama, GCC visitors, tourists, corporate accounts | Needs a vehicle for a day to months; identity via Absher, contract e-signed in the Tajeer flow |
| **Platform** (Miqwad team) | Operator | Approves dealers, sets commission/SaaS terms, monitors the marketplace, settles payouts, handles escalated disputes |

A distinct sub-type, the **rental broker / agent (وسيط تأجير)**, arranges rentals without owning the fleet. Miqwad itself operates partly in this category, which shapes its own licensing.

---

## The end-to-end business flow

### Dealer lifecycle

1. **Apply & verify** — submit CR, TGA license, VAT/ZATCA registration, Naql/Tajeer account, branches; Miqwad approves.
2. **Configure** — branches, staff & roles, connect Tajeer credentials, ZATCA (CSID), payment account, Wasl/GPS devices.
3. **Build the fleet** — add vehicles with full docs (Istimara, insurance, Operating Card), photos, categories, rate plans.
4. **Operate** — receive bookings (marketplace + walk-in), confirm, register in Tajeer, hand over with photo evidence, track live, take back, invoice via ZATCA, handle deposits and damage.
5. **Maintain** — preventive schedules, work orders, keep the fleet roadworthy and compliant.
6. **Get paid & settle** — collect payment, receive marketplace settlement (gross − commission), reconcile.
7. **Improve** — read analytics (utilization, revenue/vehicle, top cars), adjust pricing and fleet mix.

### Customer lifecycle

Browse (anonymous, dateless in-app Home/Feed) → Discover (search with dates: live availability + all-in price) → Select & quote (itemized) → Verify identity & license (KYC: national ID/iqama + driver's license, before the first booking) → Book (phone OTP) → Pay (rental + deposit, both charged up front) → Sign (Tajeer e-contract via Absher) → Receive (pickup/delivery, review handover photos) → Use (drive, extend, manage in-app) → Return (return inspection diffed against handover; deposit refunded, kept against evidenced damage; ZATCA invoice; Tajeer close) → Reflect (rate — reviews only after `completed` — rebook). A booking may also branch to **cancel / no-show** per the cancellation policy below.

### The booking state machine

`pending → confirmed → active → completed`, with branches to `cancelled` and `no_show`. Every transition carries a hard guard — **payment before confirmed**, **Tajeer registered before active**, **handover inspection before active**, **return inspection + cleared ZATCA invoice before completed**. These invariants are non-negotiable. Full ownership of the state machine and channel model is in [use-cases.md](use-cases.md).

---

## Packaging, channels & migration (V1 product model)

### Three packages

Sold as three separable packages, gated by entitlement (full tier/limit matrix and enforcement owned by [pricing-packaging.md](pricing-packaging.md)):

| Package | Listed to customers? | Runs dealer's own ops? | Billing |
|---|---|---|---|
| **Marketplace** | Yes | Minimum slice to fulfill a Miqwad booking | Commission |
| **Management** | No | Full operating system | SaaS subscription (free/standard/pro), no commission |
| **Both** | Yes | Yes | Discounted subscription + commission |

### Multi-channel bookings

Every booking carries a `channel`. Commission accrues **only** on `marketplace`.

| Channel | Source | Commission | Blocks availability | Full compliance (Tajeer/ZATCA/handover) |
|---|---|---|---|---|
| `marketplace` | Miqwad-sourced | ✅ | ✅ | ✅ |
| `dealer_direct` / `walk_in` | Dealer's own | — | ✅ | ✅ |
| `external_aggregator` | e.g. Telgani (manual entry; API sync is V2) | — | ✅ | ✅ |

This is the wedge: Miqwad is the operating layer **regardless of where the booking came from**. The channel decision is owned by [use-cases.md](use-cases.md).

### Migration

A dealer onboarding from spreadsheets or other software brings their data: **full historical import** of fleet (+docs), customers (+KYC), active bookings, and historical contracts/invoices/maintenance. Imported historical Tajeer contracts and ZATCA invoices are flagged `imported/historical` and are **never re-submitted** to the government. Full design in `engineering/data-migration.md`.

---

## Requirements — Must Have (V1) ✅

Legally mandatory or operationally non-negotiable. Without all of them, a Saudi dealer cannot legally operate.

### Onboarding, fleet & access
- Dealer application capture (CR, TGA license, VAT, Naql/Tajeer ref) + platform approval workflow.
- **Multi-branch from day one** (branch = location with address, geo, hours, staff).
- Staff users with **role-based access** (owner, manager, branch agent, accountant); branch-scoped permissions.
- Vehicle CRUD (make/model/year/trim, plate, VIN, color, transmission, fuel, seats, category, home branch, odometer).
- Compliance docs per vehicle: Istimara, insurance + expiry, **Operating Card** + expiry (Tajeer requires a valid card to issue a contract); vehicle photos.
- Vehicle status lifecycle (available / rented / maintenance / out-of-service / draft) + expiry alerts (insurance, Operating Card, registration).

### Discovery, availability, pricing & booking
- **Discovery feed** — an **anonymous, dateless** in-app **Home screen** (a React Native surface in the customer app, **not** a separate web app): featured/nearby cars, categories, dealers, for inspiration and orientation before a date window is chosen. Precedes search in the customer journey (UC-03).
- **Time-window availability** (available iff no overlapping block) — the rental-specific core, distinct from e-commerce stock.
- **Database-level double-booking prevention** (exclusion constraint).
- Rate plans: daily/weekly/monthly, refundable deposit, mileage cap + extra-km, min/max days.
- Marketplace search (city + date window, all-in price) + **itemized quote** (rental + deposit + VAT + mileage).
- **Two-phase booking**: reserve (pending) → pay → confirm. Walk-in/dealer-direct entry (no commission). Booking view/cancel/extend.
- **Ranking is organic in V1** (availability, price, distance, rating). A reserved `vehicle.promotion_rank` seam is carried from day one so **paid placement can be added in V2 with no rework** (see [pricing-packaging.md](pricing-packaging.md)); V1 never reads it for ordering.

### Saudi regulatory integrations (all mandatory)

| Integration | Requirement |
|---|---|
| **Tajeer** | Register the unified e-contract on confirmation (dealer's own Naql credentials — only the lessor can operate the contract); customer e-signs via Absher; close on return |
| **ZATCA Phase 2** | Signed e-invoice (XML + TLV QR), clear/report, store result; mandatory from day one for new LLCs |
| **Absher** | Renter (and authorized-driver) identity verification, built into the Tajeer signing flow |
| **Wasl** | Register passenger rental vehicles + report movements/trips/sensor data to TGA in real time |

### Identity, handover & trust
- Customer phone-OTP auth.
- **Customer KYC / license capture** — before a customer's **first booking can be confirmed and reach `active`**, they enter identity (**national ID *or* iqama**; GCC-ID/passport also accepted) + **driver's license (number + expiry)**, valid for the full rental duration (a Tajeer requirement), stored **encrypted and KSA-resident (PDPL)**. **V1 is data-entry only**; **Absher verification is wired later** when the API docs arrive — the fields and the hard gate on `active` ship in V1 regardless.
- **Handover inspection**: odometer, fuel, mandatory **geotagged + timestamped photos** (front/rear/sides/interior/odometer), customer e-acknowledgment.
- **Return inspection**: same, diffed against handover. Damage records (area, severity, estimate, linked photo). **Tamper-evidence** (hash + timestamp + geotag).
- **Photo capture is dealer-side.** Inspection photos are captured on the **dealer** app; the **customer app is a photo *viewer* + e-acknowledge only** (it does not capture inspection photos). Location on customer listings is a **static map thumbnail** with a **native-maps deep link** — no interactive in-app map in V1.

### Cancellation, refund & no-show (V1)
Every booking can branch to `cancelled` or `no_show`; the policy is enforced with proportional commission reversal and availability release (flow in [use-cases.md](use-cases.md) UC-17).

> **PROPOSED — finalized after the dealer-discovery interviews.** Both the *numbers* below **and the ownership model** (platform-wide vs per-dealer) are deferred to post-interview sign-off (P-7…P-12); only the *mechanism* (cancel → refund per policy, proportional commission reversal, availability-block release) is settled for V1.
> **Interim to build against:** one **platform-wide** policy for **marketplace** bookings (the figures below); dealers' own terms for **dealer-direct / walk-in**; a **per-dealer override seam reserved for V2** (bounded by platform min/max, shown per listing).
>
> - **Free cancellation up to 24h before pickup** → **full refund** (rental + deposit).
> - **Cancellation within 24h of pickup** → **first rental day forfeited**, **deposit refunded**.
> - **No-show** (customer never arrives, no handover) → **first day + booking fee forfeited**, **deposit refunded**, booking → `no_show`.
> - **Early return** → **unused days are NOT refunded in V1**.
> - **Damage on return** → **deducted from the deposit, capped at the deposit amount**; any remainder refunded (damage beyond the deposit is handled out-of-band, not auto-charged in V1).

### Payments & deposits
- **Moyasar** gateway: Mada, Visa/Mastercard, Apple Pay, STC Pay. Card data never touches Miqwad servers.
- **Refundable deposit**: charged up front at booking, refunded in full on a clean return (kept against evidenced damage). Refund handling.
- Money is always shown **itemized** (rental + refundable deposit + VAT).

### Maintenance, tracking & finance
- Preventive schedules (every N km or N days); **work orders** (open → in-progress → completed, cost, odometer, vendor).
- A work order **auto-blocks** the vehicle for the service window and recomputes next-due on completion; due-soon/overdue dashboard alerts.
- **Wasl compliance registration + reporting stays V1** (register passenger rental vehicles + report movements/trips/sensor data to TGA) — *pending confirmation that all dealers use Wasl*. The operator-facing **live fleet map / telemetry visualization / geofencing** (position, status, speed, ignition, odometer; trip/route history) **moves to V1.5** — the regulatory obligation is V1, the internal live-visibility UI is V1.5.
- ZATCA invoice list with clearance status + QR; **commission only on marketplace bookings** (fairness rule); payout/settlement (gross − commission = net) with reconciliation.

### Dealer dashboard
- The dealer dashboard is a **"Today / Action Inbox"** — a **prioritized action queue** (pending bookings, handovers/returns due today, expiring docs, maintenance due, Tajeer/ZATCA failures), **not** static counts. It surfaces the next decision and deep-links to the action.

### Platform admin
- Dealer approval, commission/SaaS-plan configuration; platform KPIs (active dealers, GMV, commission, top cities).

### Packaging & entitlements
- Three packages (Management / Marketplace / Both) with tiers (free/standard/pro), selectable per dealer.
- **Entitlement enforcement server-side** (module gate, numeric-limit gate, package gate) **and** reflected in frontend navigation.
- Subscription billing + commission settlement; **Miqwad→dealer invoices are ZATCA standard e-invoices**. Free entry tier with enforced limits (vehicles/branches/staff).

### Booking channels
- Channel on every booking; **external-aggregator manual entry** that blocks availability and runs full compliance; commission accrues only on `marketplace`, reversed proportionally on refund.

### Data migration
- **Full historical import stays a V1 MUST, sequenced last** (built after the core operating loop is stable, as the final V1 workstream — not dropped).
- Full historical import (fleet +docs, customers +KYC, active bookings, historical contracts/invoices/maintenance).
- Import validation + bad-row quarantine + idempotent re-runs (CSV/Excel templates).
- Imported historical gov records flagged `imported/historical`, excluded from registration/clearance.

### Cross-cutting (non-functional)
- Multi-tenancy with strict per-dealer isolation (token-derived + RLS); Arabic-first, RTL, bilingual AR/EN.
- PDPL: KSA residency, encryption at rest for PII, access logging. Money as integer halalas. Audit trail on booking/contract/payment/inspection state changes. Mobile apps (iOS + Android) for customers; responsive web for dealers.
- **Distributed tracing** — request/trace correlation across the modular monolith and the external-integration adapters (Tajeer/ZATCA/Wasl/Moyasar) so a booking or saga can be followed end-to-end.
- **App-layer rate limiting** — per-endpoint / per-caller limits (OTP requests, search, money- and external-system-touching endpoints) enforced in the application, not only at the edge.

### Promoted to V1 (scope owned by [backlog](../delivery/backlog.md))
Notifications (push/SMS/email + log) · customer booking history + downloadable receipts/ZATCA invoices · ratings & reviews (only after a booking is `completed`) · delivery & collection scheduling · one-way rentals · dealer reports & analytics (+ PDF/Excel) · branch-level permissions + consolidated reporting · **Saher** traffic-violation passthrough · **dealer self-service Tajeer/ZATCA/Wasl connection wizard** (canonical onboarding path; support-assisted fallback).

---

## Deferred to V1.5 🟠

Right-sized out of V1 for sequencing (not capacity), close behind the launch loop:

- **Operator live fleet map / telemetry visualization / geofencing** (position, status, speed, ignition, odometer; trip/route history). **Wasl compliance registration + reporting remains V1** — only the internal live-visibility UI moves here.

---

## Deferred to V2 (was "Should Have") 🟡

Only four former Should-have items remain V2:

- Promotions / discounts / coupons engine.
- Additional / authorized-driver capture (Tajeer supports it; V1 contract carries the single primary renter only).
- Upsells / add-ons at booking (GPS, child seat, insurance upgrade, additional driver).
- Customer support tooling (in-app chat or ticketing) for the post-launch support hire.

---

## Could Have (V2–V3) 🔵

Loyalty/rewards · corporate accounts · car-as-subscription · dynamic pricing · digital key / keyless access · predictive maintenance · channel/distribution (OTA/aggregator publish) · dealer-branded website booking widget · advanced BI · multi-currency / GCC expansion · insurance management module · photo-based damage-detection assist.

## Won't Have (yet) — out of scope

AI call agent / voice booking · autonomous-vehicle / MaaS · EV-specific fleet tooling · P2P / car-sharing host model · full accounting/ERP suite · non-KSA operations.

---

## The Saudi compliance layer (the moat)

No global rental SaaS handles these out of the box. Each is mandatory, credential-gated with approval lead time, and together they form the barrier a generic competitor (or an aggregator like Telgani) does not cross.

| System | Governs | Why it's hard | Status |
|---|---|---|---|
| **Tajeer** | Unified e-rental-contract (required since 2022) | Government API; needs TGA license + active Naql; only the lessor operates the contract | Must V1 |
| **ZATCA Phase 2** | E-invoicing (signed XML + QR, clearance) | Cryptographic signing, CSID onboarding, ongoing clearance | Must V1 |
| **Wasl** | Real-time fleet **compliance** reporting to TGA | Device integration + continuous reporting | **Must V1** (registration + reporting); *pending confirmation all dealers use Wasl*. Operator live-map UI → **V1.5** |
| **Absher** | Renter identity + e-signature | Integrated into the Tajeer flow | Must V1 |
| **TGA licensing** | The legal right to operate | Path differs for operator vs broker; prerequisite for Tajeer | Prerequisite |

**Sequencing implication:** these credential-gated integrations are the long pole of V1 — the work AI tooling speeds up least (sparse public docs, mandatory approval queues). The business manager must initiate all credential applications in week 0. See [../decisions/adr-log.md](../decisions/adr-log.md) (ADR-014/015/021) and [../STATUS.md](../STATUS.md) (P-1..P-6).
