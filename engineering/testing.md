# Miqwad — Test Strategy & Coverage

What we test, how, and the specific high-value tests that protect the money / compliance / tenancy guarantees. There is no QA role — **TDD + this strategy + pilot UAT** are the safety net.

> **Status:** ✅ **Decided.** Tooling on the Kotlin path: **JUnit 5 + MockK + SpringMockK + Testcontainers + WireMock**. Pinned versions in [./stack.md](./stack.md); CI gates in [./devops.md](./devops.md).

---

## 1. The test pyramid & tools

| Layer | Tools | What it covers |
|---|---|---|
| **Unit** (most — aim highest here) | JUnit 5 + MockK + SpringMockK + AssertJ/Strikt; `kotlinx-coroutines-test` for suspend logic | Domain logic: pricing, availability windows, deposit/VAT/commission math, state-machine guards, entitlement resolution. **This is where money bugs hide.** |
| **Integration** (some) | Spring Boot Test + Testcontainers (real Postgres/Redis) + WireMock (gov/payment) | Repositories, adapters, and the DB-level guarantees: exclusion constraint, RLS, `@TenantId`, pg_partman, PostGIS |
| **Contract** | REST Assured + the OpenAPI spec as source of truth | API conformance |
| **End-to-end** (few) | staging vs sandboxes | The two sagas (book→confirm, return→settle) incl. failure branches |
| **Architecture** (cross-cutting) | ArchUnit / Konsist | Module-boundary rules |
| **Frontend** (parallel track) | Storybook · Playwright visual · e2e happy paths · a11y | Components, visual regression, critical flows |
| **Load** 🟡 | k6 / Gatling | Search + telemetry against the envelope (V1.5, before the first big dealer) |

**MockK on the Kotlin path:** MockK handles Kotlin's `final`-by-default classes, `object`s, and `suspend` functions cleanly (`mockk`/`every`/`coEvery`); **SpringMockK** integrates it with Spring slices (`@MockkBean`). **Mockito is not used.**

---

## 2. The non-negotiable tests (must exist and stay green)

These protect the invariants. They are CI gates.

| # | Test | Asserts |
|---|---|---|
| 1 | **Cross-tenant leak** — *the single most important test* | Set tenant A's GUC, query tenant B's rows via the repository **and** a raw native query → **zero rows** (proves `@TenantId` + RLS) |
| 2 | **No double-booking race** | Two concurrent confirms for overlapping windows on the last car → **exactly one succeeds**, the other gets `vehicle_unavailable` (proves the `tstzrange` + GiST exclusion constraint, not just app logic) |
| 3 | **Booking-confirm saga — happy + failure branches** | pay→confirm→Tajeer-OK; and **payment-OK / Tajeer-fail → auto-refund**, block released, `confirmation_failed` (money never kept); Tajeer-down → charge + queue → window expiry → refund |
| 4 | **Return-settle saga** | return→ZATCA-clear→deposit settle→Tajeer-close; ZATCA-timeout → `pending_clearance` but **car released**; refund-fail → `settlement_pending` + escalation |
| 5 | **Idempotency replay** | Replay every money/external POST (same key) and every webhook (same event id) → **single effect** (no double-charge / register / refund) |
| 6 | **Imported-document guard** | Import a historical contract + invoice (`source=imported`), run all sagas + reconciliation → **zero** Tajeer/ZATCA external calls for them |
| 7 | **Entitlement gating** | Management-only dealer hits a marketplace endpoint → `403 entitlement_package_excluded`; Marketplace-only hits full-ops → `403`; create over tier limit → `402 entitlement_limit_reached` |
| 8 | **Channel commission** | `marketplace` booking accrues commission; `dealer_direct`/`walk_in`/`external_aggregator` accrue **zero**; refund reverses commission proportionally |
| 9 | **Money correctness** | Deposit charge→partial-refund (keep evidenced damage)→full refund; VAT 15%; commission; all integer halalas; ledger is **append-only** (no UPDATE/DELETE path) |
| 10 | **Import dry-run** | Load a real dealer's export into staging → bad rows quarantined (batch not failed) → committed idempotently (re-commit doesn't duplicate) |

---

## 3. Coverage targets

| Area | Target | Why |
|---|---|---|
| Money/ledger, pricing, deposit/VAT/commission | **≥ 95% (highest)** | financial correctness |
| Availability + state-machine guards | highest | the core invariants |
| Saga orchestration + compensation | highest | partial-failure is the real risk |
| Entitlement + tenancy | highest | safety / commercial correctness |
| Integration adapters | high | sandbox + WireMock both branches |
| CRUD (Elide-served) | moderate | framework-generated; test config/permissions, not Elide itself |
| **Overall** | **≥ 80%** | the standing bar |

---

## 4. Frontend testing

- **Storybook** component tests for the shared library (states: hover/focus/active, loading/empty/error, RTL).
- **Playwright** visual regression at 320 / 375 / 768 / 1024 / 1440, **both AR (RTL) and EN**.
- **e2e:** customer search→book→pay→confirmation; dealer confirm + handover capture.
- **a11y:** automated checks (WCAG 2.1 AA on the customer app); reduced-motion.

## 5. Test data & environments

- **Testcontainers** spins real Postgres (with our extensions) + Redis per run — **never an in-memory fake** (the exclusion constraint / RLS only behave on real Postgres).
- **WireMock** fixtures model each gov/payment system's success **and** failure responses.
- **Staging** holds gov **sandbox** credentials for the e2e saga runs; a seeded pilot-dealer dataset for UAT.

## 6. CI gates

No merge without: green unit + integration + contract, the §2 non-negotiable tests, ArchUnit, dependency/secret scan + SBOM, and frontend lint/type/visual. The **two-saga e2e + import dry-run run on staging post-merge**. See [./devops.md](./devops.md) §2.

## 7. UAT (pilot dealers)

- Scripted dealer UAT for onboarding (package selection), import dry-run, handover/return + invoicing, settlement view.
- A pilot dealer completes a **real rental** on staging→prod with migrated history before launch sign-off.

> Business-flow detail that UAT must confirm or correct is tracked as 🟡 Provisional in [../STATUS.md](../STATUS.md) (P-7…P-12).
