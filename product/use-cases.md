# Miqwad — Use Cases & Flows (V1)

The end-to-end journeys the UI visualizes — actor, trigger, path, the API calls behind each step, and the success state. This doc **owns** the booking state machine and the multi-channel booking decision.

> 🟡 **Provisional** — step order, who-does-what, and required fields per step are assumed and must be confirmed by **dealer discovery interviews** (P-7, P-8, P-11 in [../STATUS.md](../STATUS.md)). The flows and guards are settled; the choreography may be refined.

Three actor groups: **Customer** (renter), **Dealer staff** (owner/manager/agent/accountant), **Platform** (Miqwad team). Money shown as SAR for readability.

---

## The booking state machine (the heart of the system)

`pending → confirmed → active → completed`, with branches to `cancelled` and `no_show`. Each transition has a **hard guard** — these invariants are non-negotiable and modeled explicitly:

| Transition | Hard guard |
|---|---|
| `pending → confirmed` | **Payment charged up front** (rental + deposit) first |
| `confirmed → active` | **Customer KYC complete** (identity + driver's license captured), **Tajeer contract `registered`**, **and** **handover inspection** complete (with mandatory geotagged + timestamped photos) |
| `active → completed` | **Return inspection** done **and** **ZATCA invoice cleared** first |
| → `cancelled` / `no_show` | Branch states; trigger compensation (release block, refund) |

**Rule:** money is never held against a rental that doesn't legally exist — if Tajeer registration fails after payment, the booking auto-refunds (the deposit is charged up front, never held — ADR-015).

---

## Booking channels (decision owned here)

Every booking carries a `channel`. **Commission accrues only on `channel = marketplace`.**

| Channel | Origin | Miqwad commission | Blocks availability | Runs full compliance |
|---|---|---|---|---|
| `marketplace` | Miqwad-sourced | ✅ (~10%) | ✅ | ✅ Tajeer / ZATCA / handover |
| `dealer_direct` | Dealer's own direct sale | — | ✅ | ✅ |
| `walk_in` | Customer at the branch | — | ✅ | ✅ |
| `external_aggregator` | e.g. Telgani — entered manually (API sync is V2) | — | ✅ | ✅ |

All channels write the same `availability_block` (exclusion constraint), so there is **no cross-channel double-booking**, and all run the same Tajeer/ZATCA/handover path. The non-marketplace channels simply accrue no commission — the wedge: Miqwad is the operating layer regardless of where the booking came from.

---

## Core flows

### UC-01 — Dealer onboarding, self-service connection & branch setup
**Actor:** Owner · **Trigger:** approved by Miqwad.
1. Sign in → `POST /dealer/auth/login`; `GET /dealer/me` loads context (shows whether Tajeer / ZATCA / Wasl are connected — initially **not connected**).
2. Create branches (multi-branch from day one) → `POST /dealer/branches`.
3. Invite staff and assign roles (manager, branch agent, accountant).
4. **Self-service connection wizard (canonical path).** The dealer connects their **own** government integrations themselves — a guided wizard walks them through entering/authorizing **Tajeer (Naql)**, **ZATCA (CSID onboarding)**, and **Wasl** credentials → `POST /dealer/connections/{tajeer|zatca|wasl}`. Each connection is validated and flips its flag to `connected`; the wizard shows what's still outstanding. **Support-assisted fallback:** if the dealer gets stuck on a step, Miqwad ops can assist or complete that connection on their behalf — the wizard is the default, hands-on help is the exception, not the rule.
5. **Success:** dealership `active`, ≥1 branch, staff invited, dashboard reachable, and the dealer is guided toward a fully-connected state. Marketplace listing and Tajeer/ZATCA-bound operations stay flag-gated until the matching connection is live.

### UC-02 — Add a vehicle to the fleet
**Actor:** Manager · **Trigger:** new car to list.
1. `POST /dealer/vehicles` (branch, make/model/year, plate, category, transmission, seats, operating-card + insurance).
2. Photos → `POST /dealer/vehicles/{id}/images` (presigned) ×N.
3. Pricing → `PUT /dealer/vehicles/{id}/rate-plan` (daily/weekly/monthly, deposit, mileage cap + extra-km).
4. Optional preventive schedules → `POST /dealer/vehicles/{id}/maintenance-schedules`.
5. Vehicle flips `draft → available`; appears in marketplace search.

### UC-03 — Customer discovers, searches & books
**Actor:** Customer · **Trigger:** opens the app / needs a car.
Journey: **Open app → Home/Feed (anonymous) → Search (anonymous, with dates) → Listing/quote (anonymous) → Sign in (phone-OTP, just-in-time at "Book") → KYC (first booking only) → Book → Pay → Confirm.** Browsing needs **no account**; onboarding is deferred to the moment the customer commits to a booking.
1. **Open the app → Home/Feed. No login.** Land straight on an **in-app Home screen** (React Native, **not** a web app): an **anonymous, dateless** browse of featured/nearby cars, categories, and dealers via `GET /customer/feed` — an **anonymous endpoint** (no bearer token). The goal is inspiration and orientation before committing to a window.
2. **Search — with dates (still anonymous).** From the feed or the search bar the customer enters city/location + a date window → `GET /customer/search` (anonymous). Results show **only vehicles free for the whole window**, all-in daily price + distance.
3. Open a car → `GET /customer/listings/{id}?pickup_at&return_at` (anonymous) returns a **fully itemized quote** (rental + deposit + VAT + mileage). No hidden fees. The listing shows a **static map thumbnail** (deep-links to native maps for directions) — no interactive in-app map.
4. **Sign in — just-in-time.** The customer taps **"Book."** *Only now* does onboarding kick in: if not already signed in, a phone-OTP gate (`POST /customer/auth/request-otp` → `.../verify-otp`) creates or authenticates the account. Everything above needed no account — this is exactly where anonymous browsing meets onboarding.
5. **Verify identity & license (first booking only).** If the customer has not yet completed KYC, a **"Verify identity & license"** screen collects **national ID *or* iqama** + **driver's license number + expiry** → `POST /customer/me/documents`. Data-entry now; **Absher verification is wired later** when the API docs arrive. This is a **hard prerequisite for reaching `active`** — a booking cannot be handed over without it. Returning verified customers skip this step.
6. **Book** → `POST /customer/bookings` creates a **pending** booking + a Moyasar `payment_intent` (rental + deposit, both charged up front).
7. Pay via Mada/Apple Pay, then `POST /customer/bookings/{id}/confirm-payment`.
8. **Success:** booking `confirmed`. If the window was taken first, step 6 returns `409 vehicle_unavailable` and the app re-searches.

### UC-04 — Dealer receives & confirms booking (Tajeer)
**Actor:** Branch agent · **Trigger:** marketplace booking arrives.
1. New booking appears (real-time) → `GET /dealer/bookings?status=pending`.
2. Review customer + dates → `POST /dealer/bookings/{id}/confirm`.
3. Platform **registers the e-contract in Tajeer** (dealer's Naql credentials); `rental_contract.status` `submitted → registered`; customer e-signs (Absher code in the Tajeer flow).
4. An `availability_block (reason=booking)` already holds the car.
5. **Success:** booking `confirmed`, contract `registered`. Tajeer rejection (e.g. expired card) → `422 tajeer_failed` with the reason; the agent fixes it.

### UC-05 — Vehicle handover (pickup)
**Actor:** Branch agent + Customer · **Trigger:** customer arrives.
1. "Start handover" → `POST /dealer/bookings/{id}/inspections` (`type=handover`, odometer, fuel).
2. Photos (front/rear/left/right/interior/odometer) → `POST /dealer/inspections/{id}/photos`, each **geotagged + timestamped + hashed** at capture.
3. Pre-existing damage → `POST /dealer/inspections/{id}/damages`.
4. Customer reviews photos in-app and acknowledges (e-sign).
5. Booking → `active`, `vehicle.status = rented`.
6. **Success:** car is out with tamper-evident condition evidence — the structural fix for damage disputes.

### UC-06 — Return, damage check & e-invoice (ZATCA)
**Actor:** Branch agent + Customer · **Trigger:** customer returns the car.
1. "Start return" → `POST /dealer/bookings/{id}/inspections` (`type=return`, final odometer, fuel).
2. Return photos captured the same way; app diffs against handover.
3. New damage → `POST /dealer/inspections/{id}/damages` (severity + estimate + photo). Extra-km computed from odometer delta.
4. Platform generates the **ZATCA Phase 2 e-invoice** (signed XML + TLV QR), clears/reports it; deposit **refunded** (kept against evidenced damage) — atomic, from platform escrow.
5. `rental_contract` closed in Tajeer; booking → `completed`.
6. **Success:** customer gets a compliant invoice + prompt deposit refund; dealer's books are ZATCA-compliant automatically.

### UC-07 — Maintenance scheduling & tracking
**Actor:** Manager · **Trigger:** schedule comes due, or a repair is needed.
1. Dashboard flags **due_soon / overdue** (nightly job from odometer + dates) → `GET /dealer/dashboard`. The dashboard is a **"Today / Action Inbox"** — a **prioritized action queue** (pending bookings to confirm, handovers/returns due today, expiring docs, maintenance due, Tajeer/ZATCA failures to resolve), **not** a wall of static counts. It surfaces what needs a decision *now* and links straight to the action.
2. Work order → `POST /dealer/work-orders`. **Auto-writes `availability_block (reason=maintenance)`** so the car can't be booked; sets `vehicle.status = maintenance`.
3. Progress → `PATCH /dealer/work-orders/{id}` (`in_progress → completed`, cost + odometer).
4. On completion the schedule recomputes `next_due_*`, the block releases, car returns to `available`.
5. **Success:** preventive maintenance never missed; a car in the shop is never accidentally rented.

### UC-08 — Live fleet tracking *(V1.5 — see [requirements.md](requirements.md))*
**Actor:** Manager · **Trigger:** wants real-time visibility.
> The operator **live-map / telemetry / geofencing** experience is scheduled for **V1.5**. **Wasl compliance registration + reporting stays in V1** (mandatory to TGA); what moves is the internal live-visibility UI, not the regulatory obligation.
1. `GET /dealer/tracking/live` renders every vehicle on a map (status, speed, ignition, odometer).
2. GPS/OBD gateways post → `POST /dealer/tracking/ingest`; current position cached, history retained.
3. Per car → `GET /dealer/tracking/vehicles/{id}/history` shows trips over a range.
4. **Success:** the manager sees where every car is, which are idle, and can spot a rented car leaving an allowed area.

### UC-09 — Multi-branch operations
**Actor:** Owner / multi-branch manager.
1. Switch branch context (or "all branches") via `branch_id` on every list endpoint.
2. Vehicles have a home branch; bookings carry pickup/return branch (one-way capable).
3. Branch agents scoped to their branch (role enforcement); owners/managers see everything.
4. **Success:** branches in Khobar, Dammam, Jubail run from one account, with per-branch and consolidated views.

### UC-10 — Settlement / payout to dealer
**Actor:** Accountant · **Trigger:** settlement period closes.
1. Marketplace bookings accrue commission (`channel=marketplace`); other channels accrue none.
2. At close the platform aggregates captured payments into a `payout` (gross − commission = net) → `GET /dealer/payouts`.
3. **Success:** the dealer sees exactly what they earned, what Miqwad took, and when they're paid.

### UC-11 — Platform approves a new dealership
**Actor:** Miqwad ops · **Trigger:** a dealership applies.
1. Review applicants → `GET /admin/dealerships?status=pending`.
2. Approve → `POST /admin/dealerships/{id}/approve`.
3. Commercial terms → `PUT /admin/dealerships/{id}/commission` (rate bps, SaaS plan, per-vehicle/branch fees).
4. **Package + tier** → `PUT /admin/dealerships/{id}/subscription` (Management / Marketplace / Both; free/standard/pro) → computes entitlement.
5. **Success:** dealership goes `active`, can onboard (UC-01) with its package gating the UI; KPIs update at `GET /admin/overview`.

---

## Channel, migration & post-rental flows

### UC-12 — Dealer logs an external-aggregator booking (e.g. Telgani)
**Actor:** Branch agent · **Trigger:** a booking arrives via an aggregator.
1. "New booking" → `POST /dealer/bookings` with `channel=external_aggregator`, `channel_source="Telgani"`, customer, vehicle, dates.
2. System **blocks availability** (exclusion constraint prevents a clash with a Miqwad booking) and runs the normal **Tajeer + ZATCA + handover** path. **No Miqwad commission.**
3. **Success:** the car is managed in Miqwad regardless of origin. Walk-in/dealer-direct use the same flow minus the external source.

### UC-13 — Dealer migrates from their current system
**Actor:** Owner/admin · **Trigger:** onboarding off spreadsheets/other software.
1. Templates → `GET /dealer/imports/templates/{entity}`; fill vehicles, customers, active bookings, **historical** contracts/invoices/maintenance.
2. Upload → `POST /dealer/imports` (signed URL) → validation; review quarantined rows → `GET /dealer/imports/{id}`; fix + re-upload.
3. Commit → `POST /dealer/imports/{id}/commit` (idempotent). Historical Tajeer contracts and ZATCA invoices flagged `imported` and **never re-submitted**.
4. **Success:** the dealer goes live with full fleet, open bookings (availability correct), and complete history. Cutover runbook in `engineering/data-migration.md`.

### UC-14 — Delivery / collection
**Actor:** Branch agent · **Trigger:** customer wants the car delivered.
1. `POST /dealer/bookings/{id}/deliveries` (address, geo, window, fee).
2. `scheduled → en_route → completed` → `PATCH /dealer/deliveries/{id}`; assign staff.
3. **Success:** door-step delivery scheduled and tracked (entitlement-gated to Standard/Pro).

### UC-15 — Customer reviews after the rental
**Actor:** Customer · **Trigger:** booking reaches `completed`.
1. Reviews are **allowed only after a booking is `completed`** — the return inspection is done and the ZATCA invoice cleared. A review endpoint called on any earlier state is rejected. `POST /customer/bookings/{id}/review` (rating 1–5 + comment).
2. **Success:** review published; affects listing visibility and **organic** ranking (paid placement is a V2 add-on — see [pricing-packaging.md](pricing-packaging.md)).

### UC-16 — Saher traffic-violation passthrough
**Actor:** Accountant / Manager · **Trigger:** a Saher fine during a rental window.
1. Record → `POST /dealer/violations` (ref, amount, occurred_at, vehicle/booking).
2. Attribute → `POST /dealer/violations/{id}/attribute` → the renter on record.
3. **Success:** the fine is attributed and visible; charged per policy.

### UC-17 — Customer cancellation, refund & no-show
**Actor:** Customer (cancel) / System (no-show) · **Trigger:** customer cancels a booking, or fails to show for pickup.
1. **Cancel** → `POST /customer/bookings/{id}/cancel` (allowed while `pending` or `confirmed`, before handover). The system computes the refund from the **cancellation/refund policy** owned by [requirements.md](requirements.md) — the amount depends on how far ahead of pickup the cancellation lands.
2. **Compensation runs** — the `availability_block` is released, the deposit charge is **refunded** (full or partial, for the refundable portion), and any accrued marketplace commission is **reversed proportionally** (ledger records the reversal). If a Tajeer contract was already `registered`, it is closed/cancelled; money is never held against a rental that no longer legally exists.
3. **No-show** — if the customer never arrives and no handover starts, the branch (or a timeout job) marks the booking → `no_show`; the policy's no-show charge applies (first day + booking fee forfeited, deposit refunded — see [requirements.md](requirements.md)) and the block releases.
4. **Success:** booking → `cancelled` or `no_show`; the customer sees the itemized refund breakdown; the car is freed for other bookings. **Exact refund numbers are the PROPOSED policy in [requirements.md](requirements.md) (pending confirmation).**

---

## Flow dependencies (happy path)

```
UC-11 approve ─▶ UC-01 onboard ─▶ UC-02 add vehicle ──┐
                                                       ▼
UC-03 customer books ─▶ UC-04 confirm + Tajeer ─▶ UC-05 handover
        ▲                                              │
        │                                              ▼
   (live inventory)                          UC-06 return + ZATCA + refund
                                                       │
UC-07 maintenance  ◀─ keeps cars serviceable ─────────┘
UC-08 tracking · UC-09 multi-branch · UC-10 payout  (run continuously)
```

The demo walks **UC-03 → UC-04 → UC-05 → UC-06** to show the full loop (UC-03 now opens on the discovery feed and captures KYC before the first booking; UC-17 covers cancellation/refund/no-show off the happy path). See [requirements.md](requirements.md) for the requirement set and [pricing-packaging.md](pricing-packaging.md) for packaging/entitlements.
