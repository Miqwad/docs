# Team Plan — Senior Backend

**Role.** Depth partner + infrastructure owner + mentor/reviewer. Owns the scaffold, CI/CD, GCP environments, and the hardest infra (tenancy, pooling, security pass, perf gates). Reviews the Mid BE's crown-jewel work and pairs on the sagas.

**Weeks map to sprints:** W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (**M2**) · W7–8 = S4 · W9–10 = S5 (**M3**) · W11–12 = S6 (**M4**).

## Prep
- Scaffold: Spring Modulith on **Kotlin/Boot 4** (Gradle Kotlin DSL, `kotlin-spring`/`kotlin-jpa`/KSP), Flyway baseline + extensions, **CI/CD + GCP staging**, JobRunr/outbox wiring, Detekt/ktlint + ArchUnit.
- Co-author the architecture/ops specs: [data-model](../../architecture/data-model.md), [integrations](../../architecture/integrations.md), [api/reference](../../api/reference.md), [devops](../../engineering/devops.md), [adr-log](../../decisions/adr-log.md), [cloud](../../architecture/cloud.md), [security](../../architecture/security.md), [performance](../../architecture/performance.md).
- Co-design tenancy + entitlement with the Mid BE; **Boot-4 dependency check (O9)**; **CNTXT/GCP procurement (O8)** with MBA.

## Weeks
- **W1:** **tenancy** — `@TenantId` + RLS + coroutine-safe tenant context (ADR-003) + config-driven auth (OAuth2 resource server, issuer-driven — provider is **O-2 Open**: Keycloak vs GCIP) + JWT claims (co-own). The **cross-tenant-leak test must pass before feature work**.
- **W2:** **entitlement enforcement engine** (module/limit/package gates) + branch/staff authz; Moyasar adapter start. → **M1**.
- **W3:** **payments (Moyasar) + deposit + append-only ledger** (money core).
- **W4:** **Tajeer adapter** + Spring-7 core resilience + **outbox dispatcher (JobRunr)**.
- **W5:** **return-settle saga** + compensation (pairs with the Mid BE's booking-confirm saga).
- **W6:** reconciliation jobs + the two-saga e2e harness. → **M2**.
- **W7:** ZATCA clearance hardening (support Mid BE) + deposit settlement + signed-URL/photo security.
- **W8:** imported-doc guard (co) + idempotency hardening + webhook signature verification.
- **W9:** **connection pooling / PgBouncer** (ADR-002) + telemetry storage (native partitions/pg_partman job) + read-replica wiring.
- **W10:** settlement/billing ledger side + platform ZATCA invoice (co) + Wasl adapter support. → **M3**.
- **W11:** **security pass** (rate limits, vault, PDPL workflow — [security.md](../../architecture/security.md)) + free-tier limit enforcement.
- **W12:** **load test + perf-SLO CI gates ([performance.md](../../architecture/performance.md))** + DR drill ([reliability-dr.md](../../architecture/reliability-dr.md)) + release hardening. → **M4**.

*Pairs with: Mid BE (sagas, money, billing), Frontend (auth/JWT contract), MBA (credentials, CNTXT). Detail in [integrations](../../architecture/integrations.md) · [api/reference](../../api/reference.md) · [devops](../../engineering/devops.md) · [cloud](../../architecture/cloud.md) · [security](../../architecture/security.md) · [performance](../../architecture/performance.md) · [reliability-dr](../../architecture/reliability-dr.md).*
