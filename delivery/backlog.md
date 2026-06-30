# Miqwad — Build Backlog (V1)

The prioritized, sprint-mapped backlog that bridges design → code: epics → stories → acceptance criteria, sequenced across the six 2-week sprints (W1–W12).

> Companion docs: [project-plan.md](project-plan.md) (sprints, milestones, risks), [roles.md](roles.md). Owners: **SB** senior backend · **MB** mid backend · **FE** frontend (sole; owns UI on an off-the-shelf kit) · **MBA** business/PM.

---

## V1 vs V2 scope (this doc owns the re-scope decision)

✅ **Decided.** Everything in the epics below is **in V1**. V1 is the complete platform — the rental operating system, the customer marketplace, all four government integrations (Tajeer, ZATCA, Wasl, Absher), packaging/entitlements, multi-channel bookings, and full historical migration. Nothing listed here is cut.

**Deferred to V2** — explicitly out of V1 scope:

| 🔵 Deferred to V2 |
|---|
| Microservice extraction from the modular monolith (only if scale demands) |
| The three additional V2 feature items from the requirements (Part 4) — not represented in any epic below |

Every story traces to a Must-have requirement and to an acceptance test. Nothing in V1 scope is unrepresented; the V2 items are deliberately absent.

## Sprint map

Vertical slices; integrate continuously.

| Sprint | Theme | Milestone |
|---|---|---|
| **S1** | tenancy + entitlements + onboarding + fleet | 🔵 M1 walking skeleton |
| **S2** | availability + search + import | |
| **S3** | booking loop + channels | 🔵 M2 booking e2e |
| **S4** | handover/return + ZATCA + deposits | |
| **S5** | fleet ops + finance + historical migration + billing | 🔵 M3 ops + monetization |
| **S6** | breadth + packaging + hardening | 🔵 M4 full V1 pilot-ready |

> Acceptance criteria are written so a test can assert them. **"Done"** = code + tests + green CI + demoed.

---

## EPIC A — Platform foundation & tenancy · *Prep + S1, SB*

| Story | Acceptance criteria |
|---|---|
| **A1 Scaffold** — modular-monolith skeleton (Spring Modulith), Flyway baseline, CI/CD, GCP staging | schema migrates on Testcontainers; CI green; one module boots |
| **A2 Tenancy** — `@TenantId` + RLS + tenant GUC filter | cross-tenant-leak test passes (repo + native query return zero) |
| **A3 Auth** — Keycloak (staff/platform), customer OTP, JWT → roles + `dealership_id` | role/tenant claims enforced; OTP rate-limited |
| **A4 Outbox + JobRunr** — outbox table + dispatcher + dashboard | an event written in a tx is dispatched once; retry/backoff on failure |

## EPIC B — Packaging, entitlements & billing · *S1 core, S5 billing — SB*

| Story | Acceptance criteria |
|---|---|
| **B1 Plan/subscription/entitlement model** + resolver + cache | changing a subscription recomputes entitlement |
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
| **D3 Marketplace search + listing + quote** *(MB/FE)* — PostGIS radius/nearest, filters, itemized quote | p95 ≤ 800ms on the test fleet; only `marketplace_listing`-entitled dealers appear |
| **D4 Customer app shell + search/listing UI** *(FE)* | RN app boots with RTL + gated nav; search returns live inventory |

## EPIC E — Migration / import · *S2 fleet/customers/bookings, S5 history — MB + FE + MBA*

| Story | Acceptance criteria |
|---|---|
| **E1 Import module + staging + validation/quarantine** *(MB)* | bad rows quarantined, batch not failed; idempotent re-commit |
| **E2 Templates (vehicle/customer/booking) + import wizard UI** *(FE)* | dealer bulk-loads fleet; counts verified |
| **E3 Historical import (contracts/invoices/maintenance)** *(MB, S5)* with **imported-doc guard** | imported gov docs flagged; the imported-doc-guard test passes (zero gov calls) |
| **E4 Cutover dry-run** *(MBA, ~W8)* | a real dealer's data imports on staging cleanly |

## EPIC F — Booking loop & channels · *S3, SB + MB + FE* → 🔵 M2

| Story | Acceptance criteria |
|---|---|
| **F1 Two-phase customer booking (reserve → pay → confirm)** *(SB/MB/FE)* | `409 vehicle_unavailable` on race; idempotent create |
| **F2 Booking-confirm saga (payment + Tajeer + outbox)** *(SB)* | happy + payment-ok/Tajeer-fail (auto-refund) branches pass e2e |
| **F3 Moyasar payment integration** *(SB + FE)* — hosted fields, deposit charge (up front) | card data never hits our servers; webhook idempotent |
| **F4 External/walk-in/dealer-direct booking entry** *(MB/FE)* | external_aggregator blocks availability, runs compliance, **no commission** |
| **F5 Booking management (view/cancel/extend)** *(MB/FE)* | extend re-checks availability + recomputes price |

## EPIC G — Handover/return, ZATCA, deposits · *S4, SB + MB + FE*

| Story | Acceptance criteria |
|---|---|
| **G1 Inspections + geotagged/timestamped/hashed photos + damage** *(MB/FE)* | booking can't go `active` without min handover photos |
| **G2 ZATCA invoicing (official SDK) + clearance** *(SB)* | signed XML + QR; `pending_clearance` retry doesn't strand the car |
| **G3 Return-settle saga + deposit refund + ledger** *(SB)* | clean return refunds promptly; damaged → keep evidenced-damage amount, refund the rest; ledger append-only |
| **G4 Customer booking detail (photos/contract/invoice/refund)** *(FE)* | customer sees compliant invoice + refund status |

## EPIC H — Maintenance & telemetry · *S5, MB + SB + FE*

| Story | Acceptance criteria |
|---|---|
| **H1 Maintenance schedules + work orders + nightly recompute** *(MB)* | opening a work order blocks the vehicle; completion recomputes next-due + releases block |
| **H2 Telemetry ingest + partitions (pg_partman via JobRunr) + live map + history** *(SB/MB/FE)* | live map renders from Redis; dedup of out-of-order pings; partition maintenance job runs |
| **H3 Wasl adapter** *(SB)* | telemetry feeds Wasl in the provider-agnostic shape (Togglz-gated) |

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
| **J1 Notifications (FCM/SMS/email) + comms log** *(SB/MB/FE)* | lifecycle events notify; every send persisted |
| **J2 Ratings & reviews** *(MB/FE)* | post-completion review affects listing ranking |
| **J3 Dealer reports + PDF/Excel (read replica)** *(MB/FE)* | reports query the replica, never primary |
| **J4 Saher traffic-violation passthrough** *(MB/FE)* | fine in rental window attributed to renter on record |
| **J5 Branch permissions + consolidated reporting** *(MB/FE)*; **J6 Self-service connection wizard** *(SB/FE)* — credentials to vault, Togglz-gate; **J7 Admin KPIs** *(MB/FE)* | — |

## EPIC K — Hardening & launch · *S6, SB + MBA* → 🔵 M4

| Story | Acceptance criteria |
|---|---|
| **K1 PDPL data-subject workflow** *(SB/MBA)* | — |
| **K2 Security pass** — rate limits, webhook sigs, vault, PII | — |
| **K3 Load test** to the performance envelope | — |
| **K4 DR/restore drill** | — |
| **K5 Pilot go-live** *(MBA)* | a real rental on staging → prod with migrated history, on the chosen package |

---

## Prep (before W1) — Definition-of-Ready

Deep-design docs reviewed; schema migrates on Testcontainers; CI green on the skeleton; entitlement gating proven on a sample module; off-the-shelf component kit + brand tokens wired into the FE shells; **gov sandbox credentials applied** (Moyasar + Tajeer expected by S1–S3); **packaging matrix signed off**; GCP/CNTXT procurement underway (O8). Only then does the W1 clock start.
