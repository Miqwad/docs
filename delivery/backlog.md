# Miqwad — Build Backlog (V1)

The prioritized, sprint-mapped backlog that bridges design → code: epics → stories → acceptance criteria, sequenced across the six 2-week sprints (W1–W12).

> Companion docs: [project-plan.md](project-plan.md) (sprints, milestones, risks), [roles.md](roles.md). Owners: **SB** senior backend · **MB** mid backend · **FE** frontend (sole; owns UI on an off-the-shelf kit) · **MBA** business/PM.

---

## V1 vs V2 scope (this doc owns the re-scope decision)

✅ **Decided.** Everything in the epics below is **in V1**. V1 is the complete platform — the rental operating system, the customer marketplace (incl. the **customer discovery feed + KYC/license capture**), all four government integrations (Tajeer, ZATCA, Wasl, Absher), packaging/entitlements, multi-channel bookings, an informal **launch landing site**, and full historical migration. Nothing listed here is cut.

> **Time is not the V1 constraint** (AI-assisted small team). Full scope stays; the sprint map **sequences/rebalances** it rather than trimming for capacity. **Deposits are charged up front (no auth-hold)** everywhere payments/deposits appear. **Resolved:** **O-1** — dealer/admin portals on React (Vite) SPA + public/SEO landing site on **Next.js** (ADR-022); **O-2** — self-hosted **Keycloak** (ADR-021).

**Deferred to V1.5** — built next, not in V1:

| 🔵 Deferred to V1.5 |
|---|
| **Live fleet tracking** — operator live-map, telemetry-driven real-time positions, geofencing. *(Wasl **compliance reporting** stays in V1 — see H3.)* |
| A fuller customer **web app** (the V1 landing site is marketing-only, not the app) |

**Deferred to V2** — explicitly out of V1 scope:

| 🔵 Deferred to V2 |
|---|
| Microservice extraction from the modular monolith (only if scale demands) |
| The three additional V2 feature items from the requirements (Part 4) — not represented in any epic below |

Every story traces to a Must-have requirement and to an acceptance test. Nothing in V1 scope is unrepresented; the V1.5/V2 items are deliberately absent.

## Sprint map

Vertical slices; integrate continuously.

| Sprint | Theme | Milestone |
|---|---|---|
| **S1** | tenancy + entitlements + onboarding + fleet | 🔵 M1 walking skeleton |
| **S2** | availability + search (incl. anonymous/dateless) + active-data import | |
| **S3** | booking loop + channels + KYC + reservation hold | 🔵 M2 booking e2e |
| **S4** | handover/return + ZATCA + deposits (charged up front) + cancellation/refund | |
| **S5** | marketplace + discovery feed + Wasl reporting + fleet ops + finance + billing | 🔵 M3 ops + monetization |
| **S6** | breadth + admin controls + landing site + hardening + historical migration (last) | 🔵 M4 full V1 pilot-ready |

> Acceptance criteria are written so a test can assert them. **"Done"** = code + tests + green CI + demoed.

---

## EPIC A — Platform foundation & tenancy · *Prep + S1, SB*

| Story | Acceptance criteria |
|---|---|
| **A1 Scaffold** — modular-monolith skeleton (Spring Modulith), Flyway baseline, CI/CD, GCP staging, **distributed tracing (OpenTelemetry/Micrometer) from day one** | schema migrates on Testcontainers; CI green; one module boots; a request emits an end-to-end trace |
| **A2 Tenancy** — `@TenantId` + RLS + tenant GUC filter | cross-tenant-leak test passes (repo + native query return zero) |
| **A2b RLS auto-enforcement interceptor + ArchUnit rule** *(SB)* — interceptor sets the tenant GUC per request; ArchUnit forbids repos/native queries that bypass the tenant filter | the ArchUnit rule fails the build on a non-tenant-scoped query; interceptor sets GUC on every request |
| **A3 Auth** — **Keycloak** (staff/platform, *O-2 resolved*), customer OTP, JWT → roles + `dealership_id` | role/tenant claims enforced; OTP rate-limited |
| **A4 Outbox + JobRunr** — outbox table + dispatcher + dashboard | an event written in a tx is dispatched once; retry/backoff on failure |
| **A5 OpenAPI contract-coverage CI gate** *(SB)* — CI asserts every IA route has an OpenAPI operation | the gate fails when a documented IA route has no matching OpenAPI operation |

## EPIC B — Packaging, entitlements & billing · *S1 core, S5 billing — SB*

| Story | Acceptance criteria |
|---|---|
| **B1 Plan/subscription/entitlement model** + resolver + cache | changing a subscription recomputes entitlement |
| **B1b Admin plan / entitlement / subscription endpoints** *(SB)* — platform admin CRUD for plans, entitlements, and a dealer's subscription | admin can create a plan + assign/change a dealer subscription; change recomputes entitlement |
| **B2 Entitlement guard** — module/limit/package gates | the three entitlement tests pass |
| **B3 Package-driven onboarding** *(MB/FE)* — admin assigns package; dealer UI gated | Management-only dealer has no marketplace nav; server blocks it too |
| **B4 Subscription billing + dunning** *(S5)* — monthly invoice (platform → dealer ZATCA), past-due degradation | a platform ZATCA invoice is produced; dunning job runs |
| **B5 Free-tier limit enforcement** *(S6)* | adding the 6th vehicle on Free → `402 entitlement_limit_reached{upgrade_to}` |

## EPIC C — Dealership onboarding & fleet · *S1, MB + FE*

| Story | Acceptance criteria |
|---|---|
| **C1 Dealership apply/approve** *(SB/MB)* | pending → active; admin approval screen |
| **C2 Branches + staff/roles** — branch-scoped authz, PostGIS geo | branch_agent can't act on another branch |
| **C3 Vehicle CRUD + docs + images** *(Elide + REST signed-URL upload)* | image upload via GCS signed URL (API not in byte path); expired operating-card blocks bookings |
| **C4 Rate plans + categories + expiry alerts** *(MB)* | nightly expiry-alert job flags due docs |

## EPIC D — Availability & marketplace search · *S2, SB + MB + FE*

| Story | Acceptance criteria |
|---|---|
| **D1 Availability blocks + exclusion constraint** *(SB)* | the no-double-booking race test passes |
| **D2 Redis availability cache** *(SB)* | search reads cache; rebuildable from blocks |
| **D2b Hot-path indexes** *(SB)* — `branch.city`, `rate_plan(vehicle_id)`, `vehicle(category)`, `booking(customer_id)` (Flyway) | the search + customer-bookings queries hit indexes (no seq-scan in `EXPLAIN`) |
| **D3 Marketplace search + listing + quote** *(MB/FE)* — PostGIS radius/nearest, filters, itemized quote, **anonymous / dateless search** (browse without login or dates) | p95 ≤ 800ms on the test fleet; only `marketplace_listing`-entitled dealers appear; dateless query returns candidates |
| **D3b Customer discovery feed + ranking hook** *(MB/FE)* — feed of listings with a **promotion-weight ranking hook that is organic in V1** (weight input exists but is neutral) | feed renders ranked results; promotion weight is wired but organic-only in V1 |
| **D4 Customer app shell + search/feed/listing UI** *(FE)* | RN app boots with RTL + gated nav; search returns live inventory; discovery feed renders |

## EPIC E — Migration / import · *S2 active data (fleet/customers/bookings), S6 history last — MB + FE + MBA*

> **Active data first, full historical last.** Fleet/customers/live-bookings import early (S2); the compliance-sensitive historical import is **sequenced last (S6)**.

| Story | Acceptance criteria |
|---|---|
| **E1 Import module + staging + validation/quarantine** *(MB)* | bad rows quarantined, batch not failed; idempotent re-commit |
| **E2 Templates (vehicle/customer/booking) + import wizard UI** *(FE)* — active data first | dealer bulk-loads fleet; counts verified |
| **E3 Historical import (contracts/invoices/maintenance)** *(MB, S6 — last)* with **imported-doc guard** | imported gov docs flagged; the imported-doc-guard test passes (zero gov calls) |
| **E4 Cutover dry-run** *(MBA, ~W8)* | a real dealer's data imports on staging cleanly |

## EPIC F — Booking loop & channels · *S3, SB + MB + FE* → 🔵 M2

| Story | Acceptance criteria |
|---|---|
| **F1 Two-phase customer booking (reserve → pay → confirm)** *(SB/MB/FE)* | `409 vehicle_unavailable` on race; idempotent create |
| **F1b Reservation payment-hold + reaper** *(SB)* — ~10-min hold on reserve; a reaper job expires unpaid holds and releases the availability block | an unpaid reservation expires after the hold window and frees the vehicle |
| **F2 Booking-confirm saga (payment + Tajeer + outbox)** *(SB)* — persisted in a **`booking_saga_state`** table | happy + payment-ok/Tajeer-fail (auto-refund) branches pass e2e; saga state is durable/restartable |
| **F3 Moyasar payment integration** *(SB + FE)* — hosted fields, **deposit charged up front (no auth-hold)**, **webhook signature verification** | card data never hits our servers; webhook signature-verified + processed once |
| **F4 External/walk-in/dealer-direct booking entry** *(MB/FE)* | external_aggregator blocks availability, runs compliance, **no commission** |
| **F5 Booking management (view/cancel/extend)** *(MB/FE)* | extend re-checks availability + recomputes price |
| **F6 KYC / driver-license capture** *(MB/FE)* — capture screen + endpoint + fields (license no., expiry, ID, images) on the customer profile/booking | a booking requires captured + valid KYC; expired license blocks confirm |
| **F7 Cancellation / refund / no-show policy** *(MB/FE)* — policy model + endpoints + UI; policy-driven refund amounts; `no_show` transition | cancel within/after the free window refunds the policy amount; `no_show` settles per policy |

## EPIC G — Handover/return, ZATCA, deposits · *S4, SB + MB + FE*

| Story | Acceptance criteria |
|---|---|
| **G1 Inspections + geotagged/timestamped/hashed photos + damage** *(MB/FE)* | booking can't go `active` without min handover photos |
| **G2 ZATCA invoicing (official SDK) + clearance** *(SB)* | signed XML + QR; `pending_clearance` retry doesn't strand the car |
| **G3 Return-settle saga + deposit refund + ledger** *(SB)* — deposit was **charged up front**, so settle = refund of the charged deposit | clean return refunds promptly; damaged → keep evidenced-damage amount, refund the rest; ledger append-only |
| **G3b Ledger append-only DB trigger** *(SB)* — Flyway trigger blocks `UPDATE`/`DELETE` on ledger rows | an attempted ledger row update/delete is rejected at the DB level |
| **G4 Customer booking detail (photos/contract/invoice/refund)** *(FE)* | customer sees compliant invoice + refund status |

## EPIC H — Maintenance & Wasl telemetry reporting · *S5, MB + SB + FE*

> **Live operator tracking (real-time map, telemetry-driven positions, geofencing) is deferred to V1.5.** V1 ingests telemetry **for Wasl compliance reporting only**.

| Story | Acceptance criteria |
|---|---|
| **H1 Maintenance schedules + work orders + nightly recompute** *(MB)* | opening a work order blocks the vehicle; completion recomputes next-due + releases block |
| **H2 Telemetry ingest + partitions (pg_partman via JobRunr)** *(SB/MB)* — stored for Wasl reporting *(live map/geofencing → V1.5)* | ingest dedups out-of-order pings; partition maintenance job runs |
| **H3 Wasl compliance-reporting adapter** *(SB)* | telemetry feeds Wasl in the provider-agnostic shape (Togglz-gated). *Flag: pending confirmation all dealers use Wasl.* |

## EPIC I — Finance, settlement, delivery, one-way · *S5, SB + MB + FE*

| Story | Acceptance criteria |
|---|---|
| **I1 Commission + payout/settlement + daily reconciliation** *(SB)* | payout = captured marketplace − commission; reconciliation alerts on mismatch |
| **I2 Delivery & collection scheduling** *(MB/FE)* | schedule with fee + status flow |
| **I3 One-way rentals** *(MB/FE)* | pickup ≠ return branch supported end-to-end |
| **I4 Finance/settlement UI + plan/billing screens** *(FE)* | — |

## EPIC J — Breadth: notifications, reviews, reports, Saher, wizard · *S6, all*

| Story | Acceptance criteria |
|---|---|
| **J1 Notifications (FCM/SMS/email) + comms log** *(SB/MB/FE)* — incl. **push-notification client + device-token registration endpoint** | lifecycle events notify; every send persisted; a device registers a push token and receives a push |
| **J2 Ratings & reviews — endpoint + screen** *(MB/FE)* | post-completion review persists via the reviews endpoint + renders on the listing; affects ranking |
| **J3 Dealer reports + PDF/Excel (read replica)** *(MB/FE)* | reports query the replica, never primary |
| **J4 Saher traffic-violation passthrough** *(MB/FE)* | fine in rental window attributed to renter on record |
| **J5 Branch permissions + consolidated reporting** *(MB/FE)*; **J6 Self-service connection wizard** *(SB/FE)* — credentials to vault, Togglz-gate; **J7 Admin KPIs** *(MB/FE)* | — |
| **J8 Admin dealership suspend/reject + `admin_action` audit trail** *(MB/FE)* — platform admin can suspend/reject a dealership; every admin action is appended to a platform audit log | suspended dealer is blocked server-side; each admin action writes an immutable `admin_action` row |
| **J9 Dealer Today / Action-Inbox** *(MB/FE)* — home surface of due handovers/returns, expiries, pending items | the dealer's Today view lists actionable items for the day |
| **J10 `/meta` version gate** *(SB/FE)* — min-supported-client endpoint + forced-update handling | an out-of-date client is told to update via `/meta` |

## EPIC K — Hardening & launch · *S6, SB + MBA* → 🔵 M4

| Story | Acceptance criteria |
|---|---|
| **K1 PDPL data-subject workflow** *(SB/MBA)* | — |
| **K2 Security pass** — **app-layer rate limiting (Bucket4j)** on all endpoints, webhook sigs, vault, PII | rate-limited endpoints return `429` past the limit; secrets only in the vault |
| **K3 Load test** to the performance envelope | — |
| **K4 DR/restore drill** | — |
| **K5 Pilot go-live** *(MBA)* | a real rental on staging → prod with migrated history, on the chosen package |
| **K6 Launch landing site** *(MBA content + light FE)* — informal **Next.js** marketing site: what Miqwad is, both sides of the platform, the offering; reuses brand tokens *(not the future customer web app — that is V1.5+)* | the landing site is live at launch, RTL/bilingual, links to the app |

---

## Prep (before W1) — Definition-of-Ready

Deep-design docs reviewed; schema migrates on Testcontainers; CI green on the skeleton; entitlement gating proven on a sample module; off-the-shelf component kit + brand tokens wired into the FE shells; **gov sandbox credentials applied** (Moyasar + Tajeer expected by S1–S3); **packaging matrix signed off**; GCP/CNTXT procurement underway (O8). Only then does the W1 clock start.
