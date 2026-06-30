# Miqwad — Tech Learning Guide

For every framework and library in the stack: a plain-English **what**, **why we chose it for Miqwad**, the **official docs**, and the **key things to know to use it in this system**. The goal is to get any team member productive on a tool from one or two tutorials.

> **How to use:** skim the *what / why / key* for everything; go deep on a library when you own that area.
> **Exact pinned versions live in [./stack.md](./stack.md)** — this guide is conceptual. Decisions trace to [../decisions/adr-log.md](../decisions/adr-log.md).

---

## A. Backend core

### Kotlin (on JDK 25) — the backend language
- **What:** a concise, null-safe JVM language with data classes, `value class`, extension functions, and coroutines. Runs on the same JVM/Spring/Hibernate foundation the team knows, with far less boilerplate than Java.
- **Why:** null-safety, coroutines for clean async, `@JvmInline value class` for type-safe money, no Lombok needed (ADR-001).
- **Docs:** [kotlinlang.org/docs](https://kotlinlang.org/docs/home.html) · [Kotlin for Java devs](https://kotlinlang.org/docs/comparison-to-java.html)
- **Key for Miqwad:** prefer `val` + immutable data classes; `@JvmInline value class Money`; the `kotlin-spring` (all-open) + `kotlin-jpa` (no-arg) compiler plugins make Spring proxies + JPA entities work without hand-written `open`/no-arg constructors.

### Spring Boot 4 / Spring Framework 7 — the framework
- **What:** wires everything (web, DB, security, config) with sensible defaults so you write features, not plumbing. Boot 4 / FW 7 targets JDK 25 and is the new-project baseline.
- **Why:** the team's deepest strength; models the rental domain natively (ADR-001).
- **Docs:** [Spring Boot + Kotlin tutorial](https://spring.io/guides/tutorials/spring-boot-kotlin/) · [docs.spring.io/spring-boot](https://docs.spring.io/spring-boot/index.html)
- **Key for Miqwad:** modular-monolith packaging (one module per bounded context); constructor injection; `application.yml` per env; profiles for local/staging/prod; keep controllers thin (`suspend` handlers where I/O-bound).

### Kotlin coroutines — concurrency the coroutines-first way
- **What:** lightweight suspendable computations for concurrent, non-blocking code in a sequential style (`suspend`, `launch`/`async`, structured concurrency).
- **Why:** the backend is **coroutines-first** for I/O-bound orchestration (saga steps, gov/payment calls, parallel fan-out); virtual threads complement blocking JPA (ADR-001).
- **Docs:** [coroutines overview](https://kotlinlang.org/docs/coroutines-overview.html) · [guide](https://kotlinlang.org/docs/coroutines-guide.html) · [context & dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)
- **Key for Miqwad:** the **tenant id is carried across suspension via `threadLocal.asContextElement`** (ADR-003); blocking JPA goes on `Dispatchers.IO`/virtual threads; test suspend logic with `kotlinx-coroutines-test`; never block the request thread on an external call without isolating it.

### Spring Modulith — keeping the monolith clean
- **What:** enforces module boundaries at build time and provides structured in-process events.
- **Why:** we ship a modular monolith; this keeps modules independent so V2 microservice extraction is mechanical (ADR-001).
- **Docs:** [spring-modulith reference](https://docs.spring.io/spring-modulith/reference/)
- **Key for Miqwad:** one module per context (fleet, booking, finance…); modules talk via events or published APIs, never reach into each other's internals; `ApplicationModuleTest` per module; the event-publication registry is a backup to our JobRunr outbox.

### Spring Data JPA + Hibernate — database access & multi-tenancy
- **What:** maps Kotlin entities to tables and generates queries; Hibernate is the engine underneath.
- **Why:** standard, mature, and its **`@TenantId`** gives automatic per-dealer data isolation — our #1 safety rule (ADR-003).
- **Docs:** [spring-data/jpa](https://docs.spring.io/spring-data/jpa/reference/) · [Hibernate ORM](https://hibernate.org/orm/documentation/)
- **Key for Miqwad:** annotate the tenant column `@TenantId` once — **never hand-write the dealer filter**; the tenant context must be coroutine-safe; **Flyway owns the schema**, set `ddl-auto=none` outside local; map money to a `Money` `@JvmInline value class`, never `double`; the `tstzrange` exclusion constraint and RLS live in SQL, not JPA.

### Yahoo Elide — fast CRUD APIs from your models
- **What:** point Elide at your JPA entities and it auto-creates JSON:API endpoints (list/get/create/update/delete) + a permission system — no controller code for plain data.
- **Why:** much of Miqwad is data-entry CRUD (branches, staff, rate plans, categories, maintenance types…); Elide builds that fast so we spend time on the hard domain (ADR-016).
- **Docs:** [elide.io](https://elide.io/) · [Spring guide](https://elide.io/pages/guide/v7/01-start.html)
- **Key for Miqwad:** use Elide **only** for the CRUD list in ADR-016; the booking sagas, availability, money, gov integrations, entitlements, and import are **hand-built Spring MVC**. Tenancy: Elide runs over the same Hibernate layer, and **Postgres RLS is the hard backstop** underneath it — use Elide `FilterExpressionCheck`s (pushed to the DB) for tenant/role gating, never in-memory checks on big tables.

### JobRunr — background jobs & the saga engine
- **What:** schedule background work in plain Kotlin/JVM (fire-and-forget, scheduled, recurring), stored in Postgres, with automatic retries and a web dashboard.
- **Why:** one tool for outbox dispatch, reconciliation, telemetry, notifications, settlement, partition maintenance, dunning, imports (ADR-008).
- **Docs:** [5-minute intro](https://www.jobrunr.io/en/documentation/5-minute-intro/) · [docs](https://www.jobrunr.io/en/documentation/)
- **Key for Miqwad:** jobs must be **idempotent** (they retry); the `outbox` table is the source of truth, JobRunr is the dispatcher; run workers in the separate `miqwad-jobs` service; use recurring jobs for nightly maintenance + pg_partman partition maintenance; secure the dashboard.

### Resilience (Spring Framework 7 core; Resilience4j optional) — isolating the government integrations
- **What:** wraps calls to external systems with timeouts, retries-with-backoff, and concurrency limits so one slow dependency can't hang the app. **Spring Framework 7 ships core resilience** (annotation-based retry/timeout/concurrency-limit) as the default; **Resilience4j** is optional for advanced circuit breakers.
- **Why:** Tajeer/ZATCA/Wasl/Absher/payment are outside our control; each needs isolation.
- **Docs:** [Spring Framework reference](https://docs.spring.io/spring-framework/reference/) · [Resilience4j](https://resilience4j.readme.io/)
- **Key for Miqwad:** isolate **per external system** (a Tajeer outage must not trip ZATCA); pair with the outbox so a tripped breaker fails into the queued/retry path, not a user error; log every external call (sensitive fields redacted).

### Flyway — database migrations
- **What:** versioned `.sql` files that build/evolve the schema reproducibly across environments.
- **Why:** the schema is the source of truth, not Hibernate auto-DDL (ADR-006).
- **Docs:** [Flyway documentation](https://documentation.red-gate.com/flyway)
- **Key for Miqwad:** **forward-only**; the exclusion constraint, RLS policies, PostGIS/pg_partman setup, and the plan seed all live in migrations; **never edit a released migration — add a new one.**

### Kotlin idioms instead of Lombok/MapStruct — data classes + extension-function mapping
- **What:** **data classes** give getters/equals/copy for free; **extension functions** (`fun Entity.toDto()`) do entity↔DTO mapping explicitly. Optional **Konvert** (KSP) generates mappers if hand-written ones get repetitive.
- **Why:** less magic, full control, no annotation-processor surprises; Lombok/MapStruct are dropped (ADR-001).
- **Docs:** [data classes](https://kotlinlang.org/docs/data-classes.html) · [extension functions](https://kotlinlang.org/docs/extensions.html) · [Konvert](https://github.com/mcarleio/konvert)
- **Key for Miqwad:** **never expose entities directly** — map to DTOs at the API edge with `toDto()`/`toEntity()`; keep DTOs as immutable data classes; reach for Konvert only on the broad CRUD surface if mapping volume justifies it.

### Togglz — feature flags & entitlement gating
- **What:** turn features on/off at runtime, per dealer if needed, with a small admin console.
- **Why:** gate the gov integrations per dealer as credentials provision, and back the package/tier entitlements (ADR-013).
- **Docs:** [togglz.org](https://www.togglz.org/documentation/)
- **Key for Miqwad:** Togglz = boolean **operational** flags; the **`entitlement` table** handles numeric limits/modules — keep them distinct; entitlement checks are authoritative server-side (402/403), Togglz gates rollout.

---

## B. Integrations & money

### ZATCA Phase 2 (official SDK) — e-invoicing
- **What:** Saudi e-invoicing — signed UBL XML + TLV QR, cleared/reported to ZATCA's Fatoora platform.
- **Why:** legally mandatory; the official SDK does the cryptographic stamping (ADR-014).
- **Docs:** [ZATCA developer portal](https://zatca.gov.sa/en/E-Invoicing/SystemsDevelopers/)
- **Key for Miqwad:** wrap the official SDK behind our adapter; the module issues **both** dealer→customer and Miqwad→dealer invoices; **imported historical invoices are never re-cleared**; CSIDs live in the vault.

### Moyasar — payments (Mada, cards, Apple Pay, STC Pay)
- **What:** Saudi payment gateway; we use its hosted fields/SDK so card data never touches our servers.
- **Why:** Mada + KSA rails; clean REST API (ADR-015).
- **Docs:** [docs.moyasar.com](https://docs.moyasar.com/)
- **Key for Miqwad:** behind a provider-agnostic adapter; deposit = charged up front, refunded on return (partial on damage); **idempotency keys** on every money call; webhooks signature-verified + processed once.

### Tajeer / Wasl / Absher — government platforms
- **What:** Tajeer = the unified e-rental contract; Wasl = fleet tracking reporting to TGA; Absher = renter identity/e-sign (in the Tajeer flow).
- **Why:** all mandatory; the compliance moat.
- **Docs:** credential-gated government APIs with sparse public docs — learn from the official onboarding packs and our adapter specs. [TGA](https://tga.gov.sa)
- **Key for Miqwad:** only the lessor operates the Tajeer contract (dealer's own Naql credentials, from the vault); **build against WireMock until sandbox access lands**; each behind a circuit breaker; **imported historical contracts are never re-registered**.

> 🟡 The wire-level contracts for Tajeer/Wasl/Absher are still **Provisional** pending vendor docs — see [../STATUS.md](../STATUS.md) (P-1, P-3, P-4). **ZATCA and Moyasar contracts are in hand (2026-06-30)** — those are firm.

---

## C. Data, cache, search

### PostgreSQL — the system of record
- **What:** our relational database; holds all money, bookings, and the integrity constraints.
- **Why:** relational integrity for money/bookings; the exclusion constraint for no-double-booking is a Postgres strength (ADR-002).
- **Docs:** [postgresql.org/docs](https://www.postgresql.org/docs/)
- **Key for Miqwad:** halalas as `bigint`; `tstzrange` + GiST exclusion constraint; RLS for tenancy; UUIDv7 PKs; on GCP it's **Cloud SQL** (managed extension allow-list — no TimescaleDB).

### PostGIS — geospatial
- **What:** a Postgres extension for maps/locations — nearest-branch, radius search, geofencing.
- **Why:** marketplace search by location + the live fleet map need real spatial queries (ADR-011).
- **Docs:** [postgis.net](https://postgis.net/documentation/)
- **Key for Miqwad:** `geography(Point,4326)` columns + GiST index; `ST_DWithin` for radius, `ST_Distance` for nearest; available on Cloud SQL.

### pg_partman — time-series partitioning (telemetry)
- **What:** automatically splits a huge table into monthly chunks so it stays fast.
- **Why:** `telemetry_ping` is high-write; TimescaleDB isn't available on Cloud SQL (ADR-011).
- **Docs:** [pg_partman](https://github.com/pgpartman/pg_partman)
- **Key for Miqwad:** Cloud SQL omits pg_partman's background worker → run `partman.run_maintenance()` from a **JobRunr** scheduled job; native declarative RANGE partitions on `recorded_at`.

### Redis (Memorystore) — cache & fast reads
- **What:** an in-memory store for caching and counters.
- **Why:** marketplace search must be fast — availability is served from cache, not live joins (ADR-002).
- **Docs:** [redis.io/docs](https://redis.io/docs/latest/)
- **Key for Miqwad:** caches = availability (per-vehicle 90 days), `vehicle_live`, rate-limit buckets; the DB is the source of truth — cache is rebuildable; on GCP it's **Memorystore** (HA).

---

## D. Testing & quality

> Full strategy + the non-negotiable tests in [./testing.md](./testing.md).

### Testcontainers — real-DB integration tests
- **What:** spins up a real Postgres/Redis in Docker during tests, so you test against the real thing.
- **Why:** the exclusion constraint, RLS, and `@TenantId` only behave correctly on real Postgres.
- **Docs:** [java.testcontainers.org](https://java.testcontainers.org/)
- **Key for Miqwad:** every persistence/adapter test uses it; assert tenant isolation (cross-tenant leak test), the no-double-booking race, and the imported-doc guard here.

### WireMock — stub the external systems
- **What:** a fake HTTP server that mimics Tajeer/ZATCA/payment so tests run without real sandboxes.
- **Why:** dev can't be blocked on government sandbox access.
- **Docs:** [wiremock.org/docs](https://wiremock.org/docs/)
- **Key for Miqwad:** model each gov system's success + failure responses; pair with the saga tests (payment-ok/Tajeer-fail branch).

### JUnit 5 + MockK + ArchUnit — testing
- **What:** the test framework, **MockK** (Kotlin-native mocking — handles `final`-by-default classes, `object`s, and `suspend` functions; **SpringMockK** integrates it with Spring tests), and architecture-rule tests. **Mockito is not used on the Kotlin path.**
- **Why:** money/availability/state-guard logic is where bugs hide; ArchUnit/Konsist keeps module boundaries honest.
- **Docs:** [mockk.io](https://mockk.io/) · [SpringMockK](https://github.com/Ninja-Squad/springmockk) · [ArchUnit](https://www.archunit.org/) · [JUnit 5](https://junit.org/junit5/docs/current/user-guide/)
- **Key for Miqwad:** AAA structure; `mockk`/`every`/`coEvery` for suspend functions; `@MockkBean` in Spring slices; ArchUnit/Konsist rules ("no controller touches a repository", "modules don't reach into each other").

---

## E. Frontend

### React — dealer & admin web
- **What:** the library for building web UIs from components.
- **Why:** standard, large talent pool; powers the dealer + admin portals (ADR-022).
- **Docs:** [react.dev/learn](https://react.dev/learn)
- **Key for Miqwad:** package/tier-aware navigation (hide modules the dealer isn't entitled to); shared component library; RTL-first.

> 🔵 The web-portal framework (React SPA vs Next.js) is **Open** — see [../STATUS.md](../STATUS.md) (O-1). The customer app, design tokens, and API are unaffected.

### React Native + Expo — customer app
- **What:** build the iOS+Android app from one React codebase; Expo adds tooling, camera, location, and OTA updates.
- **Why:** one codebase, fast iteration, native camera/GPS for inspection photos (ADR-022).
- **Docs:** [docs.expo.dev](https://docs.expo.dev/) · [reactnative.dev](https://reactnative.dev/docs/getting-started)
- **Key for Miqwad:** Expo Router for navigation; `expo-camera`/`expo-location`/`expo-image-manipulator` for geotagged/timestamped/hashed handover photos; EAS for builds + OTA JS fixes; offline-resilient handover capture.

### TanStack Query — server data in the apps
- **What:** handles fetching, caching, and refreshing API data so you don't hand-roll loading/error/refetch.
- **Why:** the apps are data-heavy; this removes a class of bugs.
- **Docs:** [tanstack.com/query](https://tanstack.com/query/latest)
- **Key for Miqwad:** server state via Query, **don't** duplicate it into Zustand; optimistic updates for booking actions with rollback; one typed API client.

### Zustand — client state
- **What:** a tiny state store for UI/client state (filters, wizards).
- **Why:** simple, no boilerplate; the right tool for the *client* half of state.
- **Docs:** [zustand.docs.pmnd.rs](https://zustand.docs.pmnd.rs/)
- **Key for Miqwad:** client state only (URL state for filters/sort/pagination where shareable); keep server state in TanStack Query.

### MapLibre GL — maps (live fleet, search)
- **What:** an open-source map renderer (no Google Maps licensing).
- **Why:** dealer live map + customer search map without per-load fees (ADR-022).
- **Docs:** [maplibre.org](https://maplibre.org/maplibre-gl-js/docs/) · RN → `react-native-maps`
- **Key for Miqwad:** live positions from Redis; cluster markers for big fleets; a KSA-appropriate tile source.

### react-i18next + RTL — Arabic-first bilingual
- **What:** translation + language switching; RTL = right-to-left layout for Arabic.
- **Why:** Arabic-first is a product requirement.
- **Docs:** [react.i18next.com](https://react.i18next.com/)
- **Key for Miqwad:** default `dir="rtl"`; mirror directional UI; Latin terms (Miqwad, ZATCA, model names) inline LTR; localized numerals/dates/currency.

### Storybook + Style Dictionary — the design system
- **What:** Storybook develops/showcases UI components in isolation; Style Dictionary turns design tokens into code for web + RN.
- **Why:** one shared component library + tokens across all three surfaces (ADR-022).
- **Docs:** [storybook.js.org](https://storybook.js.org/docs) · [styledictionary.com](https://styledictionary.com/)
- **Key for Miqwad:** tokens from the brand doc (Pine/Brass/Sand, IBM Plex Sans Arabic + Sora); every component has hover/focus/active + RTL + a11y states.

---

## F. Platform / DevOps

> Full pipeline + environments in [./devops.md](./devops.md).

### Docker — containers
- **What:** package the app + its dependencies into one runnable image.
- **Why:** identical artifact local→staging→prod.
- **Docs:** [docs.docker.com](https://docs.docker.com/) · JVM images → [Jib](https://github.com/GoogleContainerTools/jib)
- **Key for Miqwad:** multi-stage build or Jib (no Dockerfile); one image, config via env per environment.

### GCP — Cloud Run, Cloud SQL, GCS (our cloud)
- **What:** Cloud Run runs containers serverlessly; Cloud SQL = managed Postgres; GCS = object storage.
- **Why:** managed services + the Dammam region for PDPL residency (ADR-010).
- **Docs:** [Cloud Run](https://cloud.google.com/run/docs) · [Cloud SQL](https://cloud.google.com/sql/docs/postgres) · [GCS](https://cloud.google.com/storage/docs)
- **Key for Miqwad:** me-central2 region (verify per-service availability, F-1); private IP to Cloud SQL via Serverless VPC connector; V4 signed URLs for media; Secret Manager + KMS for the vault; `miqwad-api` + `miqwad-jobs` services.
