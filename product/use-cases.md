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
| `confirmed → active` | **Tajeer contract `registered`** first, **and** **handover inspection** complete (with mandatory geotagged + timestamped photos) |
| `active → completed` | **Return inspection** done **and** **ZATCA invoice cleared** first |
| → `cancelled` / `no_show` | Branch states; trigger compensation (release block, void/refund) |

**Rule:** money is never held against a rental that doesn't legally exist — if Tajeer registration fails after payment, the booking auto-voids/refunds.

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

### UC-01 — Dealer onboarding & branch setup
**Actor:** Owner · **Trigger:** invited/approved by Miqwad.
1. Sign in → `POST /dealer/auth/login`; `GET /dealer/me` loads context (shows whether Tajeer + ZATCA are connected).
2. Create branches (multi-branch from day one) → `POST /dealer/branches`.
3. Invite staff and assign roles (manager, branch agent, accountant).
4. **Success:** dealership `active`, ≥1 branch, staff invited, dashboard reachable. *(The business manager's pre-work provisions TGA license, Naql/Tajeer, and ZATCA CSID, so onboarding flips those flags to connected.)*

### UC-02 — Add a vehicle to the fleet
**Actor:** Manager · **Trigger:** new car to list.
1. `POST /dealer/vehicles` (branch, make/model/year, plate, category, transmission, seats, operating-card + insurance).
2. Photos → `POST /dealer/vehicles/{id}/images` (presigned) ×N.
3. Pricing → `PUT /dealer/vehicles/{id}/rate-plan` (daily/weekly/monthly, deposit, mileage cap + extra-km).
4. Optional preventive schedules → `POST /dealer/vehicles/{id}/maintenance-schedules`.
5. Vehicle flips `draft → available`; appears in marketplace search.

### UC-03 — Customer searches & books
**Actor:** Customer · **Trigger:** needs a car.
1. Phone sign-in → `POST /customer/auth/request-otp` → `.../verify-otp`.
2. City/location + dates → `GET /customer/search`. Results show **only vehicles free for the whole window**, all-in daily price + distance.
3. Open a car → `GET /customer/listings/{id}?pickup_at&return_at` returns a **fully itemized quote** (rental + deposit + VAT + mileage). No hidden fees.
4. "Book" → `POST /customer/bookings` creates a **pending** booking + a Moyasar `payment_intent` (rental + deposit, both charged up front).
5. Pay via Mada/Apple Pay, then `POST /customer/bookings/{id}/confirm-payment`.
6. **Success:** booking `confirmed`. If the window was taken first, step 4 returns `409 vehicle_unavailable` and the app re-searches.

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
1. Dashboard flags **due_soon / overdue** (nightly job from odometer + dates) → `GET /dealer/dashboard`.
2. Work order → `POST /dealer/work-orders`. **Auto-writes `availability_block (reason=maintenance)`** so the car can't be booked; sets `vehicle.status = maintenance`.
3. Progress → `PATCH /dealer/work-orders/{id}` (`in_progress → completed`, cost + odometer).
4. On completion the schedule recomputes `next_due_*`, the block releases, car returns to `available`.
5. **Success:** preventive maintenance never missed; a car in the shop is never accidentally rented.

### UC-08 — Live fleet tracking
**Actor:** Manager · **Trigger:** wants real-time visibility.
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
**Actor:** Customer · **Trigger:** booking completed.
1. `POST /customer/bookings/{id}/review` (rating 1–5 + comment).
2. **Success:** review published; affects listing visibility and ranking.

### UC-16 — Saher traffic-violation passthrough
**Actor:** Accountant / Manager · **Trigger:** a Saher fine during a rental window.
1. Record → `POST /dealer/violations` (ref, amount, occurred_at, vehicle/booking).
2. Attribute → `POST /dealer/violations/{id}/attribute` → the renter on record.
3. **Success:** the fine is attributed and visible; charged per policy.

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

The demo walks **UC-03 → UC-04 → UC-05 → UC-06** to show the full loop. See [requirements.md](requirements.md) for the requirement set and [pricing-packaging.md](pricing-packaging.md) for packaging/entitlements.
