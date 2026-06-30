# Miqwad — Architecture Decision Records

Each record states a decision and the reasoning behind it, so the next engineer inherits the *why*, not just the result. Format: **Status · Date · Context · Decision · Consequences**.

**Statuses:** `Proposed` (recorded, awaiting founder acceptance) · `Accepted` (ratified) · `Open` (a sub-decision not yet made). **All records are `Accepted` (2026-06-30) except [ADR-021](#adr-021--auth) (Auth), which stays `Open` until the identity-provider choice (Keycloak vs GCIP) is made — [STATUS](../STATUS.md) O-2.** See [README.md](README.md) for the index and the consolidation map, and [follow-ups.md](follow-ups.md) for external checks.

---

## ADR-001 — Backend: Kotlin on JDK 25 + Spring Boot 4, modular monolith
**Status:** Accepted · 2026-06-30

**Context.** The team's deepest strength is the JVM/Spring ecosystem. V1 is a focused build with four hard Saudi government integrations on the critical path, and rental is a time-window resource domain, not e-commerce stock. Spring Boot 4 / Spring Framework 7 is the GA baseline for new projects targeting JDK 25 (LTS).

**Decision.** Build the domain in **Kotlin 2.x (K2) on JDK 25 (LTS) with Spring Boot 4 / Spring Framework 7**, as a **modular monolith** (Spring Modulith) with in-process bounded contexts (fleet, booking, contract, finance, …). **Coroutines-first** for I/O-bound orchestration (saga steps, external calls); virtual threads complement the blocking JPA paths. Build with **Gradle Kotlin DSL** (+ `kotlin-spring` all-open, `kotlin-jpa` no-arg, KSP). Microservice extraction is deferred to V2 only if scale demands.

**Consequences.** No framework-learning tax beyond Kotlin itself; native rental modeling; module boundaries enforced at build time; the rental "plumbing" is hand-built (a smaller surface than e-commerce). Alternatives were weighed during the pre-build phase and not chosen; that rationale is retained in the historical design package, not here.

---

## ADR-002 — PostgreSQL system of record + Redis + connection pooling
**Status:** Accepted · 2026-06-30

**Context.** Money and booking integrity require a relational system of record; the marketplace needs a hot availability read path. On a coroutines-first runtime with virtual threads, the **connection pool — not threads — is the real DB-concurrency limiter**, so pooling is an explicit architectural decision.

**Decision.** PostgreSQL (Cloud SQL) is the system of record; Redis (Memorystore) holds the availability cache, live-vehicle cache, rate-limit counters, and durable queues. **Major version: PostgreSQL 17** (decided 2026-06-30) — newest GA major on Cloud SQL; best native-partition/VACUUM/IO behavior for the telemetry/booking/ledger hot paths; supported to Nov 2029. UUIDv7 is app-generated (ADR-006), so native `uuidv7()` (PG 18-only) is not required; PG 18 is not adopted (Cloud SQL availability lag). Cloud SQL me-central2 availability confirmed via CNTXT (F-1). **HikariCP** always, sized deliberately (never defaults). **PgBouncer in transaction-pooling mode** in front of Cloud SQL — safe because RLS uses `SET LOCAL` inside the transaction — and **never statement pooling**; require **PgBouncer ≥ 1.24** so server-side prepared statements work.

**Consequences.** Relational integrity for money; fast marketplace reads from cache, not live cross-joins. Risk owned here: Cloud Run autoscaling × per-instance Hikari pool can exceed Cloud SQL's connection cap → cap `max-instances × pool-size` below the Cloud SQL limit and introduce PgBouncer at the load-test step.

---

## ADR-003 — Multi-tenancy: shared schema + `dealership_id` + `@TenantId` + RLS
**Status:** Accepted · 2026-06-30

**Context.** A long tail of many small dealerships; a cross-tenant leak is the worst-case bug. On a coroutines-first runtime the tenant id must survive **suspension and thread hops** to reach both Hibernate's resolver (for `@TenantId`) and the per-transaction `SET LOCAL` (for RLS).

**Decision.** Shared DB / shared schema; every tenant-owned row carries `dealership_id`; isolation enforced **twice** — Hibernate `@TenantId` (auto-applied filter + auto-populated insert) **and** PostgreSQL **Row-Level Security** keyed on a per-transaction session var. `dealership_id` is **always derived from the JWT, never request input**. Tenant context lives in a `ThreadLocal` carried into coroutines as a context element (`threadLocal.asContextElement`), with **`ScopedValue`** (GA in JDK 25) for blocking / `StructuredTaskScope` paths; **background jobs pass `dealership_id` explicitly and never inherit it**; an auth filter sets it and a `finally` clears it on non-coroutine paths. **RLS is the hard backstop** regardless.

**Consequences.** The developer cannot forget the filter; RLS catches what the ORM filter misses (native queries, an Elide mis-config). Platform-scoped tables (`customer`, `platform_user`, `dealership`, `commission_config`) carry no `dealership_id`.

---

## ADR-004 — No double-booking via `tstzrange` + GiST exclusion constraint
**Status:** Accepted · 2026-06-30

**Context.** A vehicle is a single resource on a calendar, not a stock count — the core divergence from e-commerce.

**Decision.** Availability = the absence of an overlapping `availability_block`. A Postgres `EXCLUDE USING gist (vehicle_id WITH =, tstzrange(start_at,end_at) WITH &&)` prevents overlaps **at the database level for all channels**. Bookings and maintenance work orders both write blocks.

**Consequences.** Concurrency-safe by construction; the loser of a race gets a clean `vehicle_unavailable`, not a corrupt state. The constraint lives in Flyway (ADR-006), not Hibernate.

---

## ADR-005 — Money as integer halalas + append-only ledger
**Status:** Accepted · 2026-06-30

**Decision.** All amounts are integer **halalas** (1 SAR = 100) + explicit currency; never floats. Money is a Kotlin **`@JvmInline value class`** wrapping integer halalas + currency (type-safe, zero-allocation, impossible to confuse with a raw `Long`). Every money movement is an **immutable ledger entry**; balances are derived, never edited.

**Consequences.** Auditability and dispute defence; no floating-point money bugs; the type system prevents amount/currency mix-ups. Money is always shown itemized in product surfaces (rental + refundable deposit + VAT).

---

## ADR-006 — Schema & key conventions: UUIDv7 keys, Flyway owns the schema
**Status:** Accepted · 2026-06-30 · *(consolidates the former UUIDv7 and Flyway records)*

**Decision.** Primary keys are time-ordered **UUIDv7**, application-generated; **UUIDv4** only where an id is exposed in a public customer-facing URL. The **schema is owned by Flyway migrations** as the source of truth — tables, enums, the GiST exclusion constraint, RLS policies, and partitions live there as raw SQL where they exceed JPA mapping; **Hibernate auto-DDL is disabled** outside local dev.

**Consequences.** Sequential-ish inserts (index locality on hot tables); no creation-time leakage on public ids; reviewable, forward-only migrations; DB features beyond the ORM are first-class.

---

## ADR-008 — Sagas via persisted state machine + transactional outbox, dispatched by JobRunr
**Status:** Accepted · 2026-06-30 · *(consolidates the saga/outbox and background-jobs records)*

**Context.** Booking-confirm (pay → confirm → Tajeer register) and return-settle (return inspection → ZATCA invoice → deposit settle → Tajeer close) span external systems Miqwad does not control.

**Decision.** Model each flow as a **persisted state machine** with a **transactional outbox**; compensation actions (void authorization, release block, refund) are first-class. The **outbox table is the source of truth**; **JobRunr** (Postgres-backed, distributed across instances, retries/backoff, dashboard) is the single job engine and outbox dispatcher — also running reconciliation, telemetry processing, partition maintenance, notifications, payout/commission aggregation, deposit-hold expiry, and import batches. A hand-written state service is the default; Spring Statemachine only if the state logic grows complex.

**Consequences.** Durable, inspectable saga state; a DB commit and an external call never diverge. **Rule: money is never held against a rental that doesn't legally exist** — if Tajeer registration fails after payment, auto-void/refund. One job engine replaces the earlier multi-tool scheduling proposal.

---

## ADR-010 — Cloud: GCP, Dammam (me-central2)
**Status:** Accepted · 2026-06-30 · *(availability verification → [follow-ups](follow-ups.md) F-1)*

**Context.** PDPL requires in-Kingdom data residency.

**Decision.** Google Cloud Platform, **me-central2 (Dammam)**. Service map: Cloud SQL for PostgreSQL, Memorystore for Redis, Google Cloud Storage (V4 signed URLs), Secret Manager + Cloud KMS (the government-credential vault), Cloud Run (default) or GKE, Cloud Build + Artifact Registry (or GitHub Actions → GCP), Cloud Logging/Monitoring/Trace.

**Consequences.** Residency satisfied. Per-service availability in me-central2 (reached via the KSA reseller CNTXT) must be confirmed before scaffolding (follow-ups F-1). Drives the telemetry-storage and DR decisions.

---

## ADR-011 — Postgres data-platform extensions: native partitioning + pg_partman, PostGIS
**Status:** Accepted · 2026-06-30 · *(consolidates the telemetry-storage and geo records)*

**Context.** `telemetry_ping` is high-write time-series (~270 pings/s); Cloud SQL's managed extension allow-list **excludes TimescaleDB** but **includes pg_partman and PostGIS**.

**Decision.** Native Postgres **declarative partitioning by month + pg_partman** for telemetry; partition maintenance runs as a **JobRunr job** (Cloud SQL omits pg_partman's background worker). **PostGIS** `geography` columns + GiST spatial indexes for nearest-branch/radius marketplace search, the live fleet map, and geofencing.

**Consequences.** No managed-Postgres lock-out; current vehicle position denormalized onto `vehicle` + Redis live cache for fast reads; correct, fast spatial queries instead of ad-hoc lat/lng math. TimescaleDB reconsidered only if we ever self-manage Postgres (not planned).

---

## ADR-013 — Feature flags + entitlements: Togglz + an entitlement/subscription model
**Status:** Accepted · 2026-06-30 · *(Boot-4 readiness → [follow-ups](follow-ups.md) F-3)*

**Context.** Per-dealer gating of credential-gated integrations, plus the package/tier model.

**Decision.** **Togglz** (type-safe, embedded, Spring-native) for boolean operational flags (gate Tajeer/ZATCA/Wasl per dealer as credentials provision; gate risky features), **and** a DB `plan`/`subscription`/`entitlement` model for package + tier → enabled-modules + numeric limits, enforced server-side (→ 402/403) and reflected in frontend navigation.

**Consequences.** Unleash (a separate service) avoided as overkill for V1; entitlement gating is first-class in V1. If Togglz is not Boot-4-ready at scaffold time, fall back to an own boolean flag table behind the same gating interface (follow-ups F-3).

---

## ADR-014 — ZATCA Phase 2: wrap the official SDK
**Status:** Accepted · 2026-06-30 · *(SDK + standards received 2026-06-30; runtime decided below)*

**Context.** No turnkey JVM library exists (community libs cover Phase-1 QR only); ZATCA ships an official offline signing/validation SDK + CLI.

**Decision.** Wrap the **official ZATCA SDK R4.0.0**, embedded **in-process** in the JDK 25 monolith — R4.0.0 is built for Java 21 on a modern stack (Santuario 4.0.2) and was verified to load and run on Java 21, so it runs on JDK 25 by backward-compatibility (this removes the Java-version wall the older R3.4.8 hit — no sidecar, no reimplementation). The adapter does the **offline half via the SDK** (CSR, XAdES secp256k1/SHA-256 sign, TLV QR, hash, PIH-chain, local XSD + EN16931 + ZATCA-Schematron validation), then calls the **Fatoora API v2** (`accept-version: v2`; HTTP Basic = PCSID:secret) to **Clear** (Standard/B2B, synchronous) or **Report** (Simplified/B2C, async ≤24h). Each dealer EGS has its own CSR→CCSID→PCSID; **Miqwad runs its own platform EGS + CSID** for platform→dealer (commission/SaaS) B2B invoices. Onboard against the **GA R4.x**. The scaffold-time **JDK-25 sign smoke test** (F-6) **passed** (2026-06-30) — the SDK's sign/hash pipeline runs in-process on JDK 25, so no Java-21-toolchain fallback is needed.

**Consequences.** We do not reimplement cryptographic stamping; platform self-billing is ZATCA-compliant from day one. **Packaging caveat (F-7, found at scaffold):** the R4.0.0 fat jar bundles an entire Spring Framework (~3,000 classes) plus Jackson/Saxon/HttpClient, so it **cannot sit on the application classpath** (it shadows Spring 7) — embed it via an isolated child-first classloader (preferred; keeps the vendor crypto jar byte-for-byte) or a relocated/repackaged jar, behind the `ZatcaSigner` port. In the walking skeleton it is compile-only and run by an isolated smoke test. Exact field/profile detail is provisional pending portal onboarding (STATUS P-2).

---

## ADR-015 — Payments: Moyasar
**Status:** Accepted · 2026-06-30 · *(Moyasar API docs received 2026-06-30; deposit model decided)*

**Decision.** Payments via **Moyasar** (Mada — which rides the `creditcard` rail, identified by `source.company`; plus cards, Apple Pay, STC Pay) over REST (`api.moyasar.com/v1`, HTTP Basic with the secret key) behind a provider-agnostic adapter. Card data is collected by Moyasar's **hosted SDK** (publishable key, posts directly to Moyasar) and never touches Miqwad servers. Amounts are **halalas, 1:1** with our money model; `given_id` gives idempotent payment creation; webhooks are verified by the `secret_token` body value + a re-fetch and processed idempotently per event id. **The refundable deposit is charged up front and refunded on clean return** (kept/partially-refunded on damage) — not an authorization hold, because card auth windows are too short for multi-day rentals and Moyasar does not surface issuer auto-void. **Commission reversal on refund is Miqwad's ledger responsibility** (Moyasar has no commission concept).

**Consequences.** No card data touches Miqwad servers; a future second provider is an adapter change. The charge-up-front deposit ties up the customer's funds for the rental but is reliable across long rentals; pull Moyasar's official Mada/decline test cards before the payment test suite.

---

## ADR-016 — API surface: hand-written REST `/v1` for the domain + Elide JSON:API for CRUD
**Status:** Accepted · 2026-06-30 · *(consolidates the Elide-adoption and API-strategy records; Boot-4 readiness → [follow-ups](follow-ups.md) F-2)*

**Context.** Elide auto-generates JSON:API/GraphQL CRUD + a declarative permission model from JPA entities; much of Miqwad is plain data-entry CRUD where Elide ships features for free, while the core domain is not CRUD.

**Decision.** Use **Elide for the CRUD-heavy surfaces** (branches, staff, vehicle CRUD + documents, categories, rate plans, maintenance types/schedules, work orders, reviews read/moderate, notifications read, reference data) to build fast; **hand-build the complex domain** in Spring MVC (coroutine handlers where I/O-bound): the booking-confirm & return-settle sagas, availability/exclusion logic, marketplace search (custom ranking + PostGIS), payments/deposits/ledger, the compliance adapters, entitlement enforcement, commission/settlement, and import/migration. Both surfaces sit under **`/v1`**; the domain uses the uniform error envelope (Spring `ProblemDetail` → `{error:{code,message,details},request_id}`), cursor pagination, and idempotency-keys on money/external endpoints. **Tenancy stays safe:** Elide runs over the same Hibernate/JPA layer, so `@TenantId` applies, and Postgres RLS remains the hard backstop; Elide `FilterExpressionCheck`s (pushed to the DB) handle tenant + role gating.

**Consequences.** Faster delivery on the broad CRUD surface; two API styles coexist (REST + JSON:API), described by the OpenAPI doc for REST and referencing the Elide-served entities. If Elide is not Boot-4-ready at scaffold time, the CRUD surface falls back to Spring Data REST / plain MVC behind the same `/v1` base — a localized swap, since the core domain never depends on Elide (follow-ups F-2).

---

## ADR-021 — Auth
**Status:** 🔵 Open · 2026-06-30 — the auth **requirements** are accepted; the **identity provider (Keycloak vs GCIP, no hybrid) is undecided** — [STATUS](../STATUS.md) O-2

**Context.** We host on GCP me-central2 (ADR-010), which makes **Google Cloud Identity Platform (GCIP)** — managed, with native phone-OTP, SMS MFA, and multi-tenancy — a real alternative to a self-hosted **Keycloak**. The choice turns on PDPL data residency and RBAC depth, neither yet settled.

**Decided (provider-independent).** Customers authenticate via **phone + OTP** (short-lived access + rotating refresh tokens); dealer staff and platform users get **RBAC + password policy + MFA** for owner/manager/admin roles; the JWT carries subject, role, and `dealership_id`; the API is an **OAuth2 resource server** validating the IdP's JWTs. These hold whichever provider wins.

**🔵 Open — the identity provider (Keycloak vs GCIP; no hybrid).** Decision criteria:
- **PDPL residency (decisive):** identity PII must be stored **in-Kingdom**. Keycloak self-hosted in me-central2 guarantees it; **GCIP's me-central2 residency must be verified** (via CNTXT, ties to F-1) before it qualifies.
- **RBAC depth:** Keycloak ships the owner/manager/branch_agent/accountant role+group model and branch-scoping; on GCIP, RBAC is built app-side on custom claims.
- **Managed vs self-hosted:** Keycloak is free but self-run (operate/scale/patch); GCIP is managed (per-MAU + per-tenant billing).
Resolve before auth scaffolding (S1).

**Consequences.** Tenant scoping and RBAC are enforced server-side from token claims; `dealership_id` is the single source for tenancy (ADR-003). The resource-server design is provider-agnostic, so the choice is localized to the IdP + token-issuance config.

---

## ADR-022 — Frontend: React Native (Expo) customer app; web portals; shared design system
**Status:** Accepted · 2026-06-30 — **web-portal framework is 🔵 Open ([STATUS](../STATUS.md) O-1)**

**Decision.** The customer app is **React Native (Expo)** — decided. The dealer and admin **web** portals run on a framework **not yet chosen between React (SPA) and Next.js** (Open — STATUS O-1); the admin portal is kept deliberately thin. A **shared design system** spans all surfaces (Storybook + Style Dictionary tokens from the brand doc), with MapLibre GL maps, react-i18next + full RTL, and TanStack Query + Zustand.

**Consequences.** One component library across surfaces. The React-vs-Next choice is isolated to the web portals — it does not affect the customer app, the design tokens, or the API — and should be resolved with a short spike before frontend scaffolding (STATUS O-1).

**Sequencing (2026-06-30): the current team is 3 engineers with no dedicated UI/UX designer.** The design-led parts of this ADR — the polished customer **marketplace app** and the **shared design system** — are therefore **deferred until design/frontend capacity is added**. Near-term execution is **backend- and dealer-OS-first**; the [OpenAPI contract](../api/openapi.yaml) is the stable seam the frontends build against later, and any dealer/admin surface needed sooner is a functional, unstyled app-shell against that API (see the [STATUS](../STATUS.md) Team-capacity note).
