# Miqwad — Team Roles & Plans (V1)

One concise section per role, condensed from each person's week-by-week plan. The team is **3 engineers (2 backend + 1 frontend) + the MBA/business owner**; **no dedicated UI/UX designer, QA, or DevOps** — the frontend engineer owns the UI (off-the-shelf components), the Senior BE owns CI/CD + infra, everyone runs TDD, and the MBA runs UAT.

> Companion docs: [project-plan.md](project-plan.md) (sprints, milestones), [backlog.md](backlog.md) (epics → stories). Weeks map to sprints: W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (M2) · W7–8 = S4 · W9–10 = S5 (M3) · W11–12 = S6 (M4).
>
> **Backend split (so the two BE engineers rarely block each other):** Senior BE owns the cross-cutting/infra/money/saga spine; Mid BE owns the domain depth + breadth. They pair only on the sagas and the money/billing ledger.

---

## Mid Backend — deep domain owner

**Responsibilities.** The author of the design, with the deepest system knowledge. Takes the crown-jewel tasks **and** carries breadth, consulting the Senior BE on the hardest calls. Owns the export pipeline (O2) so the rendered docs stop going stale.

**Prep.** Lead-author the low-level design, physical schema, backlog, sagas, packaging (with MBA), migration, and diagrams docs. Set repo conventions (module layout, Kotlin extension-function / Konvert mapping reference). Build the export pipeline. Co-design tenancy + entitlement with the Senior BE.

**Focus by sprint.**
- **S1** — dealership onboarding + entitlement model/data + package assignment (co-own entitlement); vehicle/fleet CRUD + docs/images + rate plans + categories + expiry alerts. → **M1**
- **S2** — availability core (`tstzrange` + GiST exclusion constraint) + Redis cache + idempotency store; booking core (state machine, all channels) + minimal listing/quote → book + external/walk-in entry.
- **S3** — booking-confirm saga (payment + Tajeer via outbox/JobRunr) + channel-aware commission; ZATCA adapter + invoice issue/clearance; booking-loop e2e. → **M2**
- **S4** — trust core (inspections + geotagged photos + damage) + return flow; imported-doc guard + booking management (cancel/extend/no-show/late).
- **S5** — full marketplace search/listing/quote + telemetry ingest + live-map API (PostGIS) + `marketplace_compliance` flag; Wasl + maintenance + delivery + one-way + settlement/commission + subscription billing (+ platform ZATCA invoice with Senior BE). → **M3**
- **S6** — import/migration (full historical) + export pipeline + reports/exports; reviews + Saher + branch perms + consolidated reporting + admin KPIs; coverage to target. → **M4**

**Pairs with.** Senior BE (tenancy, money, return-settle saga, billing ledger); Frontend (booking/confirm/marketplace APIs); MBA (packaging, migration data, UAT).

---

## Senior Backend — depth partner & infrastructure owner

**Responsibilities.** Owns the scaffold, CI/CD, GCP environments, and the hardest infra (tenancy, pooling, security pass, perf gates). Reviews the Mid BE's crown-jewel work and pairs on the sagas. Acts as mentor/reviewer.

**Prep.** Scaffold Spring Modulith on Kotlin/Boot 4 (Gradle Kotlin DSL, `kotlin-spring`/`kotlin-jpa`/KSP), Flyway baseline + extensions, CI/CD + GCP staging, JobRunr/outbox wiring, Detekt/ktlint + ArchUnit. Co-author the schema, adapters, errors, devops, ADR, cloud, security, and performance docs. Co-design tenancy + entitlement with the Mid BE; run the Boot-4 dependency check (O9); drive CNTXT/GCP procurement (O8) with MBA. *(The walking-skeleton scaffold is done — see [github.com/Miqwad/backend](https://github.com/Miqwad/backend).)*

**Focus by sprint.**
- **S1** — tenancy (`@TenantId` + RLS + coroutine-safe tenant context per ADR-003) + Keycloak auth + JWT claims (co-own); the cross-tenant-leak test must pass **before feature work**. Entitlement enforcement engine (module/limit/package gates) + branch/staff authz; Moyasar adapter start. → **M1**
- **S2** — payments (Moyasar) + deposit + append-only ledger (money core); Tajeer adapter + Spring-7 core resilience + outbox dispatcher (JobRunr).
- **S3** — return-settle saga + compensation (pairs with the Mid BE's booking-confirm saga); reconciliation jobs + the two-saga e2e harness. → **M2**
- **S4** — ZATCA clearance hardening (support Mid BE) + deposit settlement + signed-URL/photo security; imported-doc guard (co) + idempotency hardening + webhook signature verification.
- **S5** — connection pooling / PgBouncer (ADR-002) + telemetry storage (native partitions / pg_partman job) + read-replica wiring; settlement/billing ledger side + platform ZATCA invoice (co) + Wasl adapter support. → **M3**
- **S6** — security pass (rate limits, vault, PDPL workflow) + free-tier limit enforcement; load test + perf-SLO CI gates + DR drill + release hardening. → **M4**

**Pairs with.** Mid BE (sagas, money, billing); Frontend (auth/JWT contract); MBA (credentials, CNTXT).

---

## Frontend — all three surfaces (sole frontend engineer)

**Responsibilities.** Builds the customer RN app + dealer + admin web and every flow across them, **and owns the UI directly — there is no separate designer.** Works from the typed API client generated from the OpenAPI spec. To make one builder viable across three surfaces, the UI is an **off-the-shelf component library** (a ready-made React + React-Native kit) themed with **minimal brand tokens** (Pine/Brass/Sand, IBM Plex Sans Arabic + Sora) — **not** a bespoke Storybook/Style-Dictionary design system, which is deferred to post-V1. Utilitarian-but-usable beats polished-but-late; **full RTL + basic a11y are non-negotiable**, the kit's built-in states cover the rest. **Admin is kept deliberately thin.**

**Prep.** Pick the component kit and wire minimal brand tokens (web + RN). Boot the three shells — RN (Expo Router) customer app + dealer (React) + admin (React) — with auth + package-gated navigation + full RTL. TanStack Query + Zustand; typed API client (REST + Elide JSON:API adapters). Co-author the frontend doc.

**Focus by sprint.**
- **S1** — dealer auth + dashboard shell + admin approval/package-assignment; fleet screens (vehicle CRUD / images / rate plans). → **M1**
- **S2** — dealer availability calendar + gated nav; dealer booking entry (all channels) + quote.
- **S3** — dealer confirm + Tajeer status + payment wiring; customer app shell + book-a-vehicle + Moyasar pay. → **M2 (FE side)**
- **S4** — handover/return capture (camera / geotag / checklist) + damage; customer booking detail (photos / contract / invoice / refund).
- **S5** — customer marketplace search / listing / filters / map; tracking map (MapLibre) + maintenance + delivery + finance/settlement + plan/billing. → **M3**
- **S6** — import wizard UI + notifications + reviews + connection wizard + package/tier UI; cross-surface cleanup, a11y/RTL pass, offline, EAS build/submit + forced-update policy. → **M4**

**The plan's #1 risk.** One builder + no designer across three surfaces — protected by the off-the-shelf kit, the thin admin, the dealer-OS-first order, ruthless reuse, and the generated API client (never blocked on contracts). Feature scope is kept; design-system polish is the sacrifice.

**Pairs with.** Mid/Senior BE (API contracts, sagas, marketplace).

---

## MBA — business, legal & credentials

**Responsibilities.** Owns the legal + government-credential critical path (Path A), the packaging/pricing matrix, pilot-dealer onboarding, and acts as PM + UAT lead. Credential applications have approval queues on the critical path, so this role starts at **week 0**.

**Prep (urgent — week 0).** Initiate Tajeer / ZATCA CSID / Wasl / Moyasar applications (longest lead time — start first). Drive GCP / CNTXT procurement (O8) with the Senior BE. Path A legal: PPA + Founders' Agreement with the equity-conversion option. Author the pricing/packaging matrix + limits + commission (with Mid BE). Recruit 2–3 pilot dealers; collect + map their historical data.

**Focus by sprint.**
- **S1** — credentials → "sandbox granted"; finalize package / per-mode terms (incl. `marketplace_compliance`). → **M1**
- **S2** — production credential pathway; pilot import data sets; PDPL data-subject workflow.
- **S3** — M2 demo with a pilot dealer; commission/SaaS terms; sprint board + risk log (acting PM). → **M2**
- **S4** — dealer UAT scripts; ZATCA/Tajeer verification; real historical-migration dry-run.
- **S5** — settlement/billing terms + reconciliation; Wasl confirmation with the TGA. → **M3**
- **S6** — pilot go-live checklist; support/escalation; legal review of customer + dealer terms; pilot onboarded staging → prod (migrated history, chosen package + mode); launch sign-off; investor demo. → **M4**

**Open items owned here.**

| Item | Status |
|---|---|
| **O5** — tier prices/limits: propose, get investor sign-off | 🔵 Open |
| **O8** — CNTXT/GCP procurement (with Senior BE) | 🔵 Open |
| **O10** — residency-vs-DR: confirm in-KSA DR options (with Senior BE) | 🔵 Open |

**Pairs with.** Every role on requirements/UAT; Mid BE on packaging + migration; Senior BE on credentials + cloud procurement.
