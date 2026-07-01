# Team Plan — Senior Backend

**Role.** Depth partner + infrastructure owner + mentor/reviewer. Owns the scaffold, CI/CD, GCP environments, and the hardest infra (tenancy, pooling, security pass, perf gates). Reviews the Mid BE's crown-jewel work and pairs on the sagas.

**Weeks map to sprints:** W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (**M2**) · W7–8 = S4 · W9–10 = S5 (**M3**) · W11–12 = S6 (**M4**).

## Prep
- Scaffold: Spring Modulith on **Kotlin/Boot 4** (Gradle Kotlin DSL, `kotlin-spring`/`kotlin-jpa`/KSP), Flyway baseline + extensions, **CI/CD + GCP staging**, JobRunr/outbox wiring, Detekt/ktlint + ArchUnit.
- **Distributed tracing from day one** (OpenTelemetry via Micrometer Tracing) wired into the scaffold + CI, so traces exist from the first slice — not retrofitted.
- **Keycloak** (self-hosted, me-central2) provisioned for staff/platform auth; customer phone-OTP path (*O-2 resolved, ADR-021*).
- **OpenAPI "contract covers every IA route" CI gate** stub — fails the build when a documented IA route has no OpenAPI operation.
- Co-author docs **13 / 16 / 17 / 19 / 21 / 25 / 27 / 28**.
- Co-design tenancy + entitlement with the Mid BE; **Boot-4 dependency check (O9)**; **CNTXT/GCP procurement (O8)** with MBA.

## Weeks
- **W1:** **tenancy** — `@TenantId` + RLS + coroutine-safe tenant context (ADR-003) + Keycloak auth + JWT claims (co-own) + **RLS auto-enforcement interceptor + ArchUnit rule** (forbids repos/queries that bypass the tenant filter). The **cross-tenant-leak test must pass before feature work**.
- **W2:** **entitlement enforcement engine** (module/limit/package gates) + branch/staff authz + **admin plan / entitlement / subscription endpoints**; Moyasar adapter start. → **M1**.
- **W3:** **payments (Moyasar) + deposit (charged up front) + append-only ledger** with the **ledger append-only DB trigger** (no UPDATE/DELETE on ledger rows) + **hot-path indexes** (`branch.city`, `rate_plan(vehicle_id)`, `vehicle(category)`, `booking(customer_id)`).
- **W4:** **Tajeer adapter** + Spring-7 core resilience + **outbox dispatcher (JobRunr)** + **`booking_saga_state` table** (persisted saga state machine).
- **W5:** **return-settle saga** + compensation (pairs with the Mid BE's booking-confirm saga) + **~10-min reservation payment-hold + reaper job** (expires unpaid holds, releases the availability block).
- **W6:** reconciliation jobs + the two-saga e2e harness. → **M2**.
- **W7:** ZATCA clearance hardening (support Mid BE) + deposit settlement + signed-URL/photo security.
- **W8:** imported-doc guard (co) + idempotency hardening + **Moyasar webhook signature verification** (verified + processed once).
- **W9:** **connection pooling / PgBouncer** (ADR-002) + **telemetry ingest storage for Wasl reporting** (native partitions/pg_partman job) *(live operator map / geofencing → V1.5)* + read-replica wiring + index/perf tuning.
- **W10:** settlement/billing ledger side + platform ZATCA invoice (co) + **Wasl compliance-reporting adapter** support (provider-agnostic shape). → **M3**.
- **W11:** **security pass** — **app-layer rate limiting (Bucket4j)** on all endpoints, vault, PII, PDPL workflow (doc 27) + free-tier limit enforcement + **`/meta` version-gate endpoint** (min-supported client / forced-update).
- **W12:** **load test + perf-SLO CI gates (doc 28)** + **OpenAPI contract-coverage CI gate enforced** + DR drill (doc 30) + release hardening. → **M4**.

*Pairs with: Mid BE (sagas, money, billing), Frontend (auth/JWT contract, `/meta` gate), MBA (credentials, CNTXT). Detail in docs 16/17/19/25/27/28/30.*
