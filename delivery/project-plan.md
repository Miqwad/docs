# Miqwad — Project Plan (V1)

Ship the full V1 — rental operating system **and** customer marketplace — in **3 months / six 2-week sprints**, preceded by a flexible prep phase that ends at a Definition-of-Ready.

> Companion docs: [backlog.md](backlog.md) (epics → stories), [roles.md](roles.md) (per-role plans), [../decisions/adr-log.md](../decisions/adr-log.md), [../STATUS.md](../STATUS.md).

---

## Goal & stack

V1 is the complete platform: the rental OS, the customer marketplace, all four Saudi government integrations, packaging/entitlements, multi-channel bookings, and full historical migration. Nothing is cut from V1; what is deferred to V2 is already decided — see [backlog.md](backlog.md).

| Layer | Stack |
|---|---|
| Backend | Kotlin 2.x (K2) · JDK 25 · Spring Boot 4 / Spring Framework 7 · coroutines-first · Spring Modulith |
| API | Hand-built domain (Spring MVC) + Elide JSON:API for CRUD, both under `/v1` |
| Data | PostgreSQL (Cloud SQL · PostGIS · pg_partman) + Redis (Memorystore) |
| Jobs | JobRunr (Postgres-backed) + transactional outbox |
| Frontend | React Native (Expo) customer app · React dealer + admin portals |
| Payments | Moyasar |
| Cloud | GCP me-central2 (Dammam) |

## The team

One person per role; no dedicated QA or DevOps. Senior BE owns CI/CD + infra; everyone runs TDD; MBA runs UAT.

| Role | Focus |
|---|---|
| UI/UX (design-engineer) | Design system owner + builds screens |
| Senior Backend | Depth partner + infra/CI-CD + mentor/reviewer |
| Mid Backend | Deep domain owner (design author); crown-jewels + breadth |
| Senior Frontend | Customer RN app + dealer/admin web + complex flows |
| MBA | Legal/credentials critical path + packaging + onboarding + PM/UAT |

Per-person week-by-week plans are condensed in [roles.md](roles.md).

---

## 1. Prep phase → Definition-of-Ready (DoR)

The flexible run-up absorbs all design and credential lead time. Prep is "done" when the DoR is met:

| ✅ DoR item | Owner |
|---|---|
| Deep-design docs reviewed; physical schema **migrates on Testcontainers**; CI green | SB / MB |
| **Tenancy + entitlement gating proven** on a sample module (cross-tenant-leak test passes) | SB / MB |
| Design system covers the core flows | UX |
| **Government sandbox credentials applied for** (Moyasar + Tajeer expected to land in S1–S3) | MBA (from week 0) |
| **GCP / CNTXT procurement underway** (O8); **Boot-4 dependency check done** (O9) | SB + MBA |
| **Packaging matrix signed off** | MBA + MB |

Only when the DoR is met does the W1 clock start.

## 2. Logical build order (core-first)

Build outward from the core in strict dependency order. Sprints map onto these layers.

| Layer | What | Sprint |
|---|---|---|
| **L0** Foundation | Modular-monolith scaffold, Flyway baseline + extensions, CI/CD, GCP staging, JobRunr/outbox | S1 |
| **L1** Tenancy & identity | `@TenantId` + RLS + coroutine-safe tenant context, Keycloak, JWT → roles + `dealership_id` (cross-tenant-leak test passes **before feature work**) | S1 |
| **L2** Entitlements/packaging | plan/subscription/entitlement model + gating engine (cross-cutting, early) | S1 |
| **L3** Tenant org & fleet | dealership → branch → staff → vehicle (+docs/images/categories) → rate plans | S1 |
| **L4** Availability core | `availability_block` + `tstzrange` + GiST exclusion constraint + Redis cache | S2 |
| **L5** Booking core | state machine across all channels + minimal listing/quote → book | S2 |
| **L6** Money core | payment + deposit + append-only ledger | S3 |
| **L7** Compliance core + sagas | Tajeer + ZATCA, booking-confirm + return-settle sagas, outbox/JobRunr, idempotency, compensation | S3 |
| **L8** Trust core | handover/return inspections + geotagged photos + damage | S4 |
| **L9** Marketplace & channels | full search/listing/filters/map, external-aggregator + walk-in, `marketplace_compliance` flag | S5 |
| **L10** Fleet ops & finance | telemetry + Wasl + live map, maintenance, delivery, one-way, settlement/commission + subscription billing + platform ZATCA invoice | S5 |
| **L11** Breadth & onboarding | import/migration (full historical), reports + exports, notifications, reviews, Saher, branch perms + consolidated reporting, admin KPIs, connection wizard | S6 |
| **L12** Hardening & launch | security pass, perf/load/soak to the SLOs, DR drill, pilot | S6 |

## 3. Sprints & milestones

| Sprint | Layers | Milestone |
|---|---|---|
| **S1** | L0–L3 | 🔵 **M1 — Walking skeleton:** scaffold + tenancy + entitlements + fleet, on staging, CI green |
| **S2** | L4–L5 | Availability + booking core across channels |
| **S3** | L6–L7 | 🔵 **M2 — Booking loop e2e:** reserve → pay (Moyasar sandbox) → confirm → Tajeer (sandbox), with payment-ok/Tajeer-fail auto-refund |
| **S4** | L8 | Full lifecycle: handover → return inspections + damage |
| **S5** | L9–L10 | 🔵 **M3 — Ops + monetization:** marketplace + channels live; settlement/commission + subscription billing produce a platform ZATCA invoice |
| **S6** | L11–L12 | 🔵 **M4 — Pilot-ready:** breadth + migration + hardening; a pilot dealer completes a real rental staging → prod on its chosen package + compliance mode |

**Continuous through every sprint:** TDD + Testcontainers + WireMock; the two-saga e2e from S3 on; weekly demo to the MBA + pilot dealer; the priority-order checkpoint that protects the demo.

## 4. Risks & mitigations

| Risk | Mitigation |
|---|---|
| **Full V1 in 3 months** (largest risk) | Flexible prep absorbs design + credentials; libraries delete work (JobRunr, pg_partman, PostGIS, Togglz); **core-first sequencing** (booking loop live by S3/W6); weekly priority-order checkpoint |
| **One FE dev for three surfaces** | UI/UX as design-engineer (~2 effective builders); **thin admin**; shared design system. Accepted |
| **Full historical migration** (compliance-sensitive) | Validate + quarantine; **imported gov docs never re-submitted**; cutover runbook; MBA dry-runs real data by W8 |
| **Gov-credential + GCP/CNTXT lead times** | MBA starts week 0; build vs WireMock; Togglz-gate per dealer |
| **Entitlement gating is cross-cutting** | Built in prep/S1, before breadth |
| **Spring Boot 4 newness / Elide lag** | O9 confirmed core deps; Elide → Spring Data REST fallback; Resilience4j → Spring 7 core; Togglz → own flag table |
| **Performance must never fall behind** | Enforced SLOs with CI perf-gates + continuous monitoring + load/soak before the first big dealer |
| **No QA/DevOps role** | Senior BE owns CI/CD; TDD + Testcontainers + the two-saga/entitlement/channel/import e2e tests; MBA runs UAT |

## 5. Open items

| Item | Status | Owner |
|---|---|---|
| **O5** — tier prices/limits (structure already set in packaging matrix) | 🔵 Open | MBA proposes / investor signs off |
| **O8** — verify GCP me-central2 per-service availability + complete CNTXT procurement | 🔵 Open | MBA + Senior BE (prep) |
| **O10** — residency vs DR: verify a second in-KSA GCP region for cross-region DR; if none, V1 DR = me-central2 multi-zone HA + PITR + tested restore + in-Kingdom backup | 🔵 Open | MBA + Senior BE |

✅ **Resolved/closed:** O1 (scheduling), O2 (export pipeline → Mid BE), O3 (no contract FE), O4 (legal Path A), O6 (`marketplace_compliance`), O7 (Elide adopted), O9 (Boot 4 confirmed).
