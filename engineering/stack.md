# Miqwad — Stack & Dependency Matrix (BOM)

The decided backend stack, plus a versioned, CVE-vetted dependency matrix to scaffold sprint 1 against — a point-in-time snapshot dated **2026-06-30**.

> **Status:** ✅ **Decided** for the backend, data, app, payments, auth, and cloud layers. 🔵 **Open** for the web-portal framework only (React SPA vs Next.js — [../STATUS.md](../STATUS.md) O-1). The version pins in **Part B** are a snapshot; re-verify at scaffold ([../decisions/follow-ups.md](../decisions/follow-ups.md) F-5).
>
> Decisions trace to the [ADR log](../decisions/adr-log.md) (ADR-001/002/003/005/008/010/011/013/014/015/016/021/022). External readiness checks are tracked as follow-ups F-1 through F-5.

---

## Part A — The Decided Stack

### A.1 Layer table (one page)

| Layer | Choice | Status |
|---|---|---|
| **Backend language & runtime** | **Kotlin 2.x (K2) on JDK 25 (LTS)** — null-safety, data classes, `value class`, coroutines, structured concurrency | ✅ ADR-001 |
| **Framework** | **Spring Boot 4 / Spring Framework 7**, **coroutines-first** (virtual threads complement blocking JPA), **modular monolith** via **Spring Modulith** (bounded contexts: fleet, booking, contract, finance, …) | ✅ ADR-001 |
| **Build** | **Gradle (Kotlin DSL)** + Spring Boot Gradle plugin + `kotlin-spring` (all-open) + `kotlin-jpa` (no-arg) + **KSP** | ✅ ADR-001 |
| **System of record** | **PostgreSQL 17** (Cloud SQL `POSTGRES_17`) — `dealership_id` discriminator, `@TenantId` + RLS, `tstzrange` + GiST exclusion constraint, UUIDv7 keys, Flyway-owned schema | ✅ ADR-002/003/004/006 |
| **Cache / live data / queues** | **Redis** (Memorystore) — availability cache, live-vehicle cache, rate-limit counters, durable queues | ✅ ADR-002 |
| **Connection pooling** | **HikariCP** (deliberately sized) behind **PgBouncer ≥ 1.24** in transaction-pooling mode | ✅ ADR-002 |
| **Customer app** | **React Native (Expo)** | ✅ ADR-022 |
| **Web portals (dealer + admin)** | **React (SPA) vs Next.js — not yet chosen** | 🔵 **Open — O-1** |
| **Payments** | **Moyasar** (Mada, cards, Apple Pay, STC Pay) via REST behind a provider-agnostic adapter; card data never touches Miqwad | ✅ ADR-015 |
| **Auth / identity** | Customer phone-OTP; staff/platform RBAC + MFA; JWT carries subject, role, `dealership_id`. **Provider 🔵 Open — Keycloak vs GCIP** (O-2) | 🔵 ADR-021 / O-2 |
| **Cloud** | **GCP — me-central2 (Dammam)** for PDPL residency | ✅ ADR-010 (F-1) |
| **Object storage** | **Google Cloud Storage (GCS)** — private buckets, V4 signed URLs | ✅ ADR-009/010 |
| **Background jobs / outbox dispatch** | **JobRunr** (Postgres-backed, distributed, dashboard) — the single job engine | ✅ ADR-008 |
| **Compliance adapters** | **Tajeer · ZATCA Phase 2 · Wasl · Absher** — provider-agnostic ports, credential-gated, circuit-broken | ✅ (contracts 🟡 P-1…P-4) |

The customer app, design tokens, and API contract are unaffected by the web-portal O-1 decision; it is isolated to the dealer/admin portals and resolved with a short SSR/SEO spike before frontend scaffolding.

### A.2 The dependency set, by layer

The set is the smallest list of boring, production-proven libraries that covers the requirement — every dependency is a liability to patch and debug. Tags carry over from the deep-dive: **[core]** = V1-essential (sprint 1), **[add]** = added when its feature lands. Exact versions and CVE status are in **Part B**.

**Language, build & Kotlin idioms (ADR-001).** [core] Kotlin 2.x (K2) on JDK 25 · [core] Spring Boot 4 / Spring Framework 7 · [core] Gradle Kotlin DSL + Spring Boot Gradle plugin · [core] `kotlin-spring` (all-open) + `kotlin-jpa` (no-arg) compiler plugins · [core] KSP · [core] `jackson-module-kotlin` (Jackson 3) · [core] `kotlinx-coroutines-core` (+ `-test`) · [core] `kotlin-logging` (SLF4J facade) · [core] Money as a `@JvmInline value class` (ADR-005) · [add, optional] Arrow (`Either`/`Raise`) for pure-domain typed errors only · [add, optional] Kotest + Strikt; Konsist.

**Web / API layer.** [core] `spring-boot-starter-web` · [core] `spring-boot-starter-validation` (Jakarta Bean Validation) · [add] `springdoc-openapi-starter-webmvc-ui` (live Swagger UI from code) · [add] **Elide** for the CRUD-heavy surfaces (JSON:API over the same Hibernate layer; `@TenantId` + RLS still apply — ADR-016).

**Persistence / ORM.** [core] `spring-boot-starter-data-jpa` (Spring Data JPA + Hibernate) · [core] `spring-boot-starter-flyway` + `flyway-database-postgresql` (the schema's source of truth — exclusion constraint, RLS policies, enums, partitions live here as SQL) · [core] PostgreSQL JDBC driver · [core] HikariCP (Boot-managed) · [add] `hypersistence-utils` (Postgres JSONB, ranges, arrays) · [core] Postgres extensions: `postgis`, `pg_partman` + native declarative partitioning, `btree_gist`, `pgcrypto`, `pg_trgm` (ADR-011).

**Security / auth.** [core] `spring-boot-starter-security` (filter chain, `@PreAuthorize`, encoding) · [core] `spring-boot-starter-oauth2-resource-server` (validates the IdP's JWTs — Keycloak or GCIP — extracts roles + `dealership_id`) · [add] the **IdP — Keycloak or GCIP** (external service; RBAC, MFA — provider 🔵 Open, O-2).

**Caching, events, async.** [core] `spring-boot-starter-data-redis` · [core] `spring-boot-starter-cache` (`@Cacheable` over Redis) · [core] Spring `@Async` + `ApplicationEventPublisher` (in-process fan-out, no dependency) · [add] **Spring Modulith** (build-time module boundaries + event-publication registry — a lightweight outbox) · [core] **Togglz** + a DB `plan`/`subscription`/`entitlement` model (ADR-013).

**Resilience & external integrations.** [core] Spring Framework 7 `RestClient` (coroutine-friendly) · [core] Spring Framework 7 core resilience (annotation retries/timeouts/concurrency limits) as the default · [add, optional] **Resilience4j** for advanced circuit breakers/bulkheads once a Boot-4 release is confirmed · [add] per-integration adapter modules behind clean interfaces.

**Saga / outbox / state machine.** [core] Spring `@Transactional` · [core] transactional outbox table (source of truth) dispatched by **JobRunr** · [add] Spring Statemachine (only if the booking lifecycle logic grows complex; a hand-written state service is the default — ADR-008).

**Money, time, mapping.** [core] `Money` as `@JvmInline value class` (integer halalas + currency; never floats — ADR-005) · [core] `java.time` (+ optional `kotlinx-datetime`), all UTC · [core] Kotlin extension-function mapping (`toDto()`/`toEntity()`) · [add, optional] **Konvert (KSP)** for compile-time mappers on the broad CRUD surface.

**Documents — ZATCA e-invoice & PDFs.** [core] **Official ZATCA SDK** wrapped behind the adapter (signed UBL 2.1 XML, cryptographic stamp, TLV QR — ADR-014) · [core] **BouncyCastle** (crypto) + **ZXing** (QR image) + a JAXB/UBL stack at the edges · [add] **OpenPDF** (or PDFBox) for contracts/invoices/reports · [add] **openhtmltopdf** (HTML-templated branded PDFs).

**Files / object storage.** [core] **Google Cloud Storage Java client** (me-central2) — V4 signed URLs, private buckets, PDPL residency.

**Search.** [core] PostgreSQL full-text + trigram (`pg_trgm`) for V1 fleet/vehicle search · [core] **PostGIS** (`geography` + GiST) for nearest-branch/radius search, the live fleet map, geofencing (ADR-011/012) · [add, V2] OpenSearch/Elasticsearch only if marketplace search outgrows Postgres.

**Observability.** [core] `spring-boot-starter-actuator` (health/readiness/liveness/metrics) · [core] **Micrometer** + Prometheus registry · [core] structured logging — **Logback** JSON + **Micrometer Tracing** correlation IDs · [add] **Sentry** (error tracking) · [add] OpenTelemetry export (distributed tracing across saga steps, V1.5).

**Scheduling / background jobs.** [core] **JobRunr (OSS)** — the single job engine (ADR-008): outbox dispatch, reconciliation, telemetry processing + pg_partman partition maintenance, notifications, payout/commission aggregation, deposit-hold expiry, dunning, import batches.

**Notifications.** [add] **Firebase Cloud Messaging** (push) + an **SMS/OTP provider** (Unifonic or similar, KSA) + **email** (Spring Mail / an email API) — behind one swappable abstraction (provider selection 🟡 P-6).

**Rate limiting.** [add] **Bucket4j** (Redis/Lettuce backend) — token-bucket limits on auth, search, booking endpoints.

**Testing.** [core] `spring-boot-starter-test` (JUnit 5, AssertJ, MockMvc) · [core] **MockK + SpringMockK** (Kotlin-native mocking; replaces Mockito) · [core] **Testcontainers** (real Postgres + Redis — the exclusion constraint, RLS, `@TenantId` tested against real Postgres) · [core] **WireMock** (stubs Tajeer/ZATCA/Wasl/Moyasar) · [add] **REST Assured** · [add] **ArchUnit** (architecture rules in tests).

**Build & ops.** [core] Gradle Kotlin DSL + Spring Boot Gradle plugin + `kotlin-spring`/`kotlin-jpa`/KSP + multi-stage Dockerfile (or Jib) · [add] **Jib** (Dockerfile-less images) · [core] `.editorconfig` + **Detekt** + **ktlint** (Kotlin static analysis + formatting; replaces Spotless/Checkstyle/SpotBugs).

**External services (not libraries):** Keycloak or GCIP (IdP — O-2) · Redis · PostgreSQL (+ read replica) · GCS · Moyasar · FCM + SMS + email.

---

## Part B — Versioned Dependency Matrix (BOM)

> ⚠️ **Point-in-time snapshot — 2026-06-30.** Versions and advisories change. **Re-verify latest-version + CVE status at scaffold time** ([../decisions/follow-ups.md](../decisions/follow-ups.md) F-5). Where a library is on the Spring Boot 4 dependency BOM, **do not pin it independently** — inherit the BOM version (noted "managed by Boot 4 BOM"); the version shown is the Boot **4.1.0** managed value (the BOM, not the app, owns the upgrade cadence).
>
> **CVE method.** Each "Known-CVE status" was checked on 2026-06-30 against OSV.dev, the GitHub Advisory Database (GHSA), and NVD via vendor advisories. "No known high/critical (OSV, 2026-06-30)" means none was found **at time of writing** — it is not a guarantee. Where a high/critical CVE exists, the **patched** version is pinned and the CVE id cited. Lines marked **verify at scaffold (F-5)** are ones where a precise patched version could not be confirmed live; the version *line* is given, not an invented point release.

### B.1 Foundation — language, framework, build

| Dependency | Proposed version | Boot-4 / JDK-25 compatible? | Known-CVE status | Source (checked 2026-06-30) | Notes |
|---|---|---|---|---|---|
| **Kotlin (K2 compiler)** | **2.3.x** (line) | ✅ Boot 4.1 baseline is Kotlin 2.3; 2.3 supports JDK 25 | No known high/critical (OSV, 2026-06-30) | InfoQ Spring Boot 4.1; kotlinlang releases | Decided floor is "Kotlin 2.x / JDK 25". Boot 4.0 ships Kotlin 2.2; **Boot 4.1 raises it to 2.3** — pin 2.3.x. Newer 2.4.x exists but is ahead of the Boot baseline; align to the BOM. |
| **JDK** | **25 (LTS)** | ✅ Boot 4 has first-class JDK 25 support; Gradle 9.1+ runs on it | n/a (platform; track JDK CPU advisories) | Spring Boot 4.0 release notes | Decided invariant (ADR-001). `ScopedValue` GA in 25 (ADR-003). |
| **Spring Boot** | **4.1.0** | ✅ self | No known high/critical (OSV, 2026-06-30); Boot 4.1 adds SSRF mitigation | spring.io blog; InfoQ (4.1 GA 2026-06-10) | 4.1.0 is the current GA line (4.0.x also supported to 2026-12-31). The whole BOM below derives from this. |
| **Spring Framework** | **7.0.x** (managed by Boot 4 BOM) | ✅ self | No known high/critical (OSV, 2026-06-30) | spring.io | FW 7.0.8 was current at writing; inherit via Boot. |
| **Gradle** | **9.6.x** (9.6.1) | ✅ JDK 25 supported since Gradle 9.1 | n/a (build tool) | docs.gradle.org release notes | Use the Gradle wrapper; 9.6.1 released 2026-06-27. |
| **Spring Boot Gradle plugin** | **4.1.0** (matches Boot) | ✅ self | n/a | docs.spring.io | Version-locked to the Boot version. |
| **`kotlin-spring` (all-open) plugin** | **2.3.x** (matches Kotlin) | ✅ | n/a | kotlinlang | Applied via the Kotlin Gradle plugin; track the Kotlin version. |
| **`kotlin-jpa` (no-arg) plugin** | **2.3.x** (matches Kotlin) | ✅ | n/a | kotlinlang | As above. |
| **KSP (Kotlin Symbol Processing)** | **2.3.x-1.0.x** (line; tracks Kotlin) | ✅ KSP2 tracks the Kotlin version | n/a | github.com/google/ksp | Pin the KSP release built for the exact Kotlin version — **verify at scaffold (F-5)**. |

### B.2 Kotlin ecosystem

| Dependency | Proposed version | Boot-4 / JDK-25 compatible? | Known-CVE status | Source (checked 2026-06-30) | Notes |
|---|---|---|---|---|---|
| **kotlinx-coroutines-core** (+ `-test`) | **1.11.0** | ✅ companion to Kotlin 2.2.20+; runs on JDK 25 | No known high/critical (OSV, 2026-06-30) | github.com/Kotlin/kotlinx.coroutines | Coroutines is on the 1.x line (does **not** match Kotlin's 2.x). Re-check for a 2.3-aligned release at scaffold (F-5). |
| **jackson-module-kotlin** | **3.x** (managed by Boot 4 BOM) | ✅ Boot 4 defaults to **Jackson 3** | No known high/critical (OSV, 2026-06-30) | spring.io "Jackson 3 support"; Boot 4 migration guide | **Coordinate change:** group is `tools.jackson.module` (Jackson 3), not `com.fasterxml.jackson.module`. Auto-registered on the classpath. `jackson-annotations` stays `com.fasterxml.jackson.annotation` 2.x. |
| **kotlin-logging** | **7.0.3** | ✅ SLF4J facade; runtime-agnostic | No known high/critical (OSV, 2026-06-30) | github.com/oshai/kotlin-logging | **Coordinate change:** group is `io.github.oshai` (was `io.github.microutils`), root package `io.github.oshai.kotlinlogging`. Provide SLF4J yourself (Logback comes via Boot). |
| **Konvert (KSP)** *(optional)* | **latest** (line) | ✅ KSP-based, Kotlin-native | No known high/critical (OSV, 2026-06-30) | github.com/mcarleio/konvert | `[add]` only if hand-written `toDto()` mapping gets repetitive. Pin to a release matching the KSP/Kotlin version — **verify at scaffold (F-5)**. |

### B.3 Persistence

| Dependency | Proposed version | Boot-4 / JDK-25 compatible? | Known-CVE status | Source (checked 2026-06-30) | Notes |
|---|---|---|---|---|---|
| **PostgreSQL JDBC driver** (`org.postgresql:postgresql`) | **42.7.11** (managed by Boot 4 BOM) | ✅ | ✅ **Patched.** 42.7.11 fixes **CVE-2026-42198** (SCRAM PBKDF2 iteration DoS); earlier 42.7.x are vulnerable | jdbc.postgresql.org changelog; Boot 4.1 BOM | Boot 4.1 already ships the patched 42.7.11 — do not downgrade. Adds `scramMaxIterations` (default 100k). |
| **spring-boot-starter-flyway** | **4.1.0** (Boot starter) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io; Boot 4 Flyway autoconfig | **Boot 4 change:** bare `flyway-core` is no longer auto-configured — you must use the **starter** + `flyway-database-postgresql`. |
| **flyway-core / flyway-database-postgresql** | **12.4.0** (managed by Boot 4 BOM) | ✅ | No known high/critical (OSV, 2026-06-30) | Boot 4.1 BOM | Inherited via the starter; the Postgres module is required for PG. |
| **HikariCP** (`com.zaxxer:HikariCP`) | **7.0.2** (managed by Boot 4 BOM) | ✅ | No known high/critical (OSV, 2026-06-30) | Boot 4.1 BOM | Size deliberately; pair with PgBouncer ≥ 1.24 (ADR-002). |
| **Hibernate ORM** (`hibernate-core`) | **7.4.1.Final** (managed by Boot 4 BOM) | ✅ | No known high/critical (OSV, 2026-06-30) | Boot 4.1 BOM | Inherited via `data-jpa`. Provides `@TenantId`. |
| **hypersistence-utils** | **3.15.x** (Hibernate-version-matched artifact) | ✅ supports Hibernate 7.4 | No known high/critical (OSV, 2026-06-30) | github.com/vladmihalcea/hypersistence-utils | **Pick the artifact matching the Boot-managed Hibernate minor** — Boot 4.1 ships Hibernate 7.4, so use `hypersistence-utils-hibernate-74` (or `-73`). Latest line is 3.15.3 (2026-06-02) — **verify the exact `-74` artifact at scaffold (F-5)**. |
| **PgBouncer** (external) | **≥ 1.24** | ✅ transaction-pooling mode; server-side prepared statements need ≥ 1.24 | No known high/critical (OSV, 2026-06-30) | ADR-002 | Infrastructure, not a Java dependency. Never statement pooling (RLS uses `SET LOCAL`). |
| **PostGIS** (Postgres extension) | Cloud SQL allow-listed build | ✅ available on GCP Cloud SQL | Track Cloud SQL security bulletins | ADR-011 (F-1) | Managed by Cloud SQL; not a Maven artifact. |
| **pg_partman** (Postgres extension) | Cloud SQL allow-listed build | ✅ available on GCP Cloud SQL (bg worker omitted → JobRunr runs maintenance) | Track Cloud SQL security bulletins | ADR-011 (F-1) | Managed by Cloud SQL; not a Maven artifact. TimescaleDB is **not** on Cloud SQL. |

### B.4 Web / API, security, resilience

| Dependency | Proposed version | Boot-4 / JDK-25 compatible? | Known-CVE status | Source (checked 2026-06-30) | Notes |
|---|---|---|---|---|---|
| **spring-boot-starter-web** | **4.1.0** (Boot) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io | Embedded Tomcat, Jackson 3. |
| **spring-boot-starter-validation** | **4.1.0** (Boot) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io | Hibernate Validator 9.1 via BOM. |
| **spring-boot-starter-security** | **4.1.0** (Boot) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io | Spring Security 7 via BOM. |
| **spring-boot-starter-oauth2-resource-server** | **4.1.0** (Boot) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io | JWT validation; extracts `dealership_id`. |
| **spring-boot-starter-data-redis** | **4.1.0** (Boot) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io | Lettuce client via BOM. |
| **spring-boot-starter-cache** | **4.1.0** (Boot) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io | `@Cacheable` over Redis. |
| **spring-boot-starter-actuator** | **4.1.0** (Boot) | ✅ self | No known high/critical (OSV, 2026-06-30) | docs.spring.io | Health/metrics/probes. |
| **springdoc-openapi-starter-webmvc-ui** | **3.0.x** (3.0.1) | ✅ explicit Boot 4 + OpenAPI 3.1 support (runs on JDK 25) | No known high/critical (OSV, 2026-06-30) | springdoc.org; github.com/springdoc/springdoc-openapi | v3.0.1 bundles swagger-ui 5.31.0; built against Boot 4.0.1. Re-check for the 4.1-aligned patch (F-5). |
| **Spring Modulith** | **2.1.0** | ✅ released alongside Boot 4.1 (2026-06-10) | No known high/critical (OSV, 2026-06-30) | spring.io; InfoQ | 2.1.0 adds `JobRunrEventExternalizer` — pairs directly with the outbox (ADR-008). |
| **Resilience4j** *(optional)* | `resilience4j-spring-boot4` **2.4.x** | 🟡 Boot-4 artifact exists (since 2.4.0) but was **omitted from the BOM** (issue #2427, fix merged, awaiting release) | No known high/critical (OSV, 2026-06-30) | github.com/resilience4j/resilience4j #2427/#2420 | `[add, optional]` — FW 7 core resilience is the default. Adopt once the BOM fix ships; until then pin the `-spring-boot4` artifact explicitly. **Verify at scaffold (F-5).** |
| **Elide** | **7.x** (line) | 🟡 **Boot 4 release not confirmed** at writing (autoconfigure documents Boot 3) | No known high/critical (OSV, 2026-06-30) | github.com/yahoo/elide | **F-2:** if no Boot-4-compatible Elide at scaffold, the CRUD surface falls back to **Spring Data REST / plain Spring MVC** behind the same `/v1` base — the core domain never depends on Elide. **Verify at scaffold (F-2).** |
| **Togglz** | **4.4.0** | 🟡 4.x targets **Spring Boot 3 / Spring 6**; Boot 4 not confirmed | No known high/critical (OSV, 2026-06-30) | togglz.org; github.com/togglz/togglz | **F-3:** if not Boot-4-ready at scaffold, fall back to an own boolean **flag table** behind the same gating interface (flags are not on the critical path). **Verify at scaffold (F-3).** |

### B.5 Jobs, money/docs, storage

| Dependency | Proposed version | Boot-4 / JDK-25 compatible? | Known-CVE status | Source (checked 2026-06-30) | Notes |
|---|---|---|---|---|---|
| **JobRunr** (`jobrunr-spring-boot-4-starter`) | **8.7.0** | ✅ **Boot 4 + Jackson 3 supported since 8.3.0**; `-spring-boot-4-starter` artifact published | No known high/critical (OSV, 2026-06-30) | jobrunr.io; central.sonatype.com | **F-4 resolved positively** — the Boot-4 starter exists. Fallback (JobRunr core + manual config) no longer needed but remains documented. |
| **BouncyCastle** (`bcprov-jdk18on`, `bcpkix-jdk18on`) | **1.84** (or latest ≥ 1.84) | ✅ JDK 18+ build, runs on JDK 25 | 🟡 1.84 **fixes CVE-2026-0636** (LDAP injection, affected 1.74→1.84). Other 2026 CVEs (CVE-2026-5598 FrodoKEM timing; CVE-2026-3505 PGP AEAD; CVE-2026-5588 PKIX composite) are in code paths Miqwad does not use (ZATCA uses PKCS/TLV signing) | NVD; GHSA-c3fc-8qff-9hwx; Snyk | Pin **≥ 1.84**. Re-check for a release that clears the residual FrodoKEM/PGP CVEs at scaffold (F-5); not blocking since those engines are unused. |
| **ZXing** (`com.google.zxing:core`) | **3.4.0** | ✅ Java 8+ | No known high/critical (OSV, 2026-06-30) | github.com/zxing/zxing; mvnrepository | TLV QR image rendering for ZATCA. Last release 2025-11. |
| **Official ZATCA SDK** (vendor) | per ZATCA portal | n/a (vendor JAR, not on Maven Central) | Track ZATCA advisories | zatca.gov.sa Compliance Enablement Toolbox | Credential/onboarding-gated; wrapped behind the adapter (ADR-014). Exact build provisional — [../STATUS.md](../STATUS.md) P-2. **Verify at onboarding.** |
| **OpenPDF** (`com.github.librepdf:openpdf`) | **3.0.3** | ✅ JDK 17+ | No known high/critical in 3.0.3 (OSV, 2026-06-30) — note: it pulls BouncyCastle for signing (keep ≥ 1.84) | github.com/LibrePDF/OpenPDF | `[add]`. LGPL/MPL iText fork. |
| **openhtmltopdf** (`io.github.openhtmltopdf`) | **1.1.31** | ✅ migrated to PDFBox 3.x | No known high/critical (OSV, 2026-06-30) | github.com/danfickle/openhtmltopdf (community fork) | **Use the active fork `io.github.openhtmltopdf`** — the original `com.openhtmltopdf` is abandoned (~2022). Branded HTML→PDF. |
| **Apache POI** (`poi`, `poi-ooxml`) | **5.5.1** | ✅ JDK 17+ | ✅ Past the duplicate-zip-entry advisory (fixed in 5.4.0); no known high/critical in 5.5.1 (OSV, 2026-06-30) | poi.apache.org | `[core]` for Excel/CSV report export. Not Boot-managed — pin independently. |
| **Google Cloud Storage client** (`com.google.cloud:google-cloud-storage`) | **2.69.0** via `libraries-bom` **26.83.0** | ✅ | No known high/critical (OSV, 2026-06-30) | docs.cloud.google.com; github.com/googleapis/java-cloud-bom | Pin via the GCP `libraries-bom` (26.83.0), not the artifact directly. V4 signed URLs. |

### B.6 Observability, rate limiting, notifications

| Dependency | Proposed version | Boot-4 / JDK-25 compatible? | Known-CVE status | Source (checked 2026-06-30) | Notes |
|---|---|---|---|---|---|
| **Micrometer** (`micrometer-core`) | **1.17.0** (managed by Boot 4 BOM) | ✅ | No known high/critical (OSV, 2026-06-30) | Boot 4.1 BOM | Inherited via actuator. |
| **Micrometer Prometheus registry** (`micrometer-registry-prometheus`) | **1.17.0** (managed by Boot 4 BOM) | ✅ | No known high/critical (OSV, 2026-06-30) | Boot 4.1 BOM | Add the registry to the classpath; Boot auto-configures it. |
| **Micrometer Tracing** (`micrometer-tracing`) | managed by Boot 4 BOM | ✅ | No known high/critical (OSV, 2026-06-30) | Boot 4.1 BOM | Correlation/trace IDs (Sleuth successor). OpenTelemetry 1.62 via BOM. |
| **Logback** (`logback-classic`) | **1.5.34** (managed by Boot 4 BOM) | ✅ | ✅ **Patched.** Boot 4.1 ships 1.5.34, past CVE-2025-11226 / CVE-2026-1225 (ACE) / CVE-2026-9828 (deserialization bypass) | logback.qos.ch news; Boot 4.1 BOM | Inherited via `starter-logging`; do not pin an older Logback. JSON encoding for structured logs. |
| **Sentry** (`sentry-spring-boot-starter-jakarta`) | **8.44.1** | 🟡 8.x is current; explicit Boot-4 starter compat not confirmed at writing | No known high/critical (OSV, 2026-06-30) | github.com/getsentry/sentry-java | `[add]`. Confirm the Boot-4 starter variant at scaffold (F-5). |
| **Bucket4j** (`com.bucket4j:bucket4j_jdk17-core` + `_jdk17-lettuce`) | **8.19.0** | ✅ JDK 17 build; Redis via Lettuce module | No known high/critical (OSV, 2026-06-30) | bucket4j.com; github.com/bucket4j/bucket4j | **Coordinate change:** group is `com.bucket4j`, artifacts are `bucket4j_jdk17-*`. Redis support is split per client (use Lettuce to match Spring's). Community Spring Boot starter available. |
| **Firebase Admin SDK** (`com.google.firebase:firebase-admin`) | **9.9.0** | ✅ | No known high/critical (OSV, 2026-06-30) | github.com/firebase/firebase-admin-java | `[add]` — FCM push. Behind the notification abstraction. |
| **SMS provider (e.g. Unifonic)** | per vendor SDK/REST | n/a (vendor) | Track vendor advisories | ADR / STATUS P-6 | Provider not yet selected (🟡 P-6); integrate via REST behind the abstraction. |

### B.7 Testing & build tooling

| Dependency | Proposed version | Boot-4 / JDK-25 compatible? | Known-CVE status | Source (checked 2026-06-30) | Notes |
|---|---|---|---|---|---|
| **spring-boot-starter-test** | **4.1.0** (Boot) | ✅ self | ✅ Bundled AssertJ is **3.27.7** (managed) — patches **CVE-2026-24400** (AssertJ XXE, affected 1.4.0→3.27.6) | Boot 4.1 BOM; NVD CVE-2026-24400 | JUnit 5 + AssertJ + MockMvc. Boot 4.1 already ships the patched AssertJ — do not downgrade. (Bundled Mockito 5.23.0 is unused on the Kotlin path.) |
| **MockK** (`io.mockk:mockk`) | **1.14.x** (line) | ✅ Kotlin-native; runs on JDK 25 | No known high/critical (OSV, 2026-06-30) | mockk.io; github.com/mockk/mockk | Replaces Mockito (handles `final`-by-default, coroutines). Pin the latest 1.14.x at scaffold (F-5). |
| **SpringMockK** (`com.ninja-squad:springmockk`) | **5.0.1** | 🟡 5.x clones Spring's Mock annotations; explicit Boot-4 compat not confirmed at writing | No known high/critical (OSV, 2026-06-30) | github.com/Ninja-Squad/springmockk | `@MockkBean`/`@SpykBean` for Spring tests. **Verify the Boot-4-compatible release at scaffold (F-5).** |
| **Testcontainers** (`org.testcontainers:*`) | **2.0.5** (or Boot-BOM 1.20.x) | ✅ | No known high/critical (OSV, 2026-06-30) | github.com/testcontainers/testcontainers-java; Boot 4.1 BOM | Boot 4.1 BOM references the 1.20.x line; latest standalone is 2.0.5. Use the BOM-managed version for Spring-test integration, or pin 2.x deliberately. **Verify at scaffold (F-5).** |
| **WireMock** (`org.wiremock:wiremock` / `wiremock-jetty12`) | **3.13.2** | ✅ stable on Jetty 12 module | No known high/critical (OSV, 2026-06-30) | wiremock.org; github.com/wiremock/wiremock | Stable 3.13.x (4.0.0-beta exists but is beta — avoid for V1). Stubs the gov/payment sandboxes. |
| **REST Assured** (`io.rest-assured:rest-assured`) | **6.0.0** | ✅ Java 17+, **adds Spring 7 + Jackson 3 support** | No known high/critical (OSV, 2026-06-30) | rest-assured.io; github.com/rest-assured/rest-assured | `[add]`. 6.0.0 is the Boot-4-aligned line (Spring 7 / Jackson 3); 5.5.7 is the backport branch. |
| **ArchUnit** (`com.tngtech.archunit:archunit-junit5`) | **1.4.2** | ✅ supports up to Java 26 | No known high/critical (OSV, 2026-06-30) | archunit.org; github.com/TNG/ArchUnit | `[add]`. Module-boundary rules in tests (or Konsist as the Kotlin-native alternative). |
| **Detekt** | **1.23.x stable** (2.0.0 is alpha) | 🟡 stable 1.23.x trails Kotlin 2.3; **2.0.0-alpha** targets Kotlin 2.4 + JDK 25 | No known high/critical (OSV, 2026-06-30) | detekt.dev; github.com/detekt/detekt | Pin the latest stable 1.23.x that supports the chosen Kotlin 2.3 compiler; adopt 2.0 once it goes stable. **Verify the Kotlin-2.3-compatible build at scaffold (F-5).** |
| **ktlint** | **via Detekt ktlint ruleset** (or **1.x** standalone) | ✅ | No known high/critical (OSV, 2026-06-30) | detekt.dev; github.com/pinterest/ktlint | Run as the Detekt `formatting` ruleset, or standalone 1.x. Pin to a Kotlin-2.3-compatible release (F-5). |
| **Konsist** *(optional)* | **latest** (line) | ✅ Kotlin-native | No known high/critical (OSV, 2026-06-30) | github.com/LemonAppDev/konsist | `[add, optional]` — Kotlin-native architecture tests (complements/replaces ArchUnit). Verify version at scaffold (F-5). |
| **Jib (Gradle plugin)** (`com.google.cloud.tools.jib`) | **3.5.3** | ✅ builds OCI images for JVM apps | No known high/critical (OSV, 2026-06-30) | plugins.gradle.org; github.com/GoogleContainerTools/jib | `[add]` — Dockerfile-less images. Released 2026-02-05. |

---

## Summary

- **Foundation** is locked to **Spring Boot 4.1.0 / Spring Framework 7 / Kotlin 2.3 / JDK 25 / Gradle 9.6.x**. Everything Spring manages (web, validation, security, oauth2-resource-server, data-jpa, data-redis, cache, actuator, **PostgreSQL 42.7.11, HikariCP 7.0.2, Hibernate 7.4.1, Logback 1.5.34, AssertJ 3.27.7, Micrometer 1.17, Flyway 12.4.0, Jackson 3**) inherits the **Boot 4 BOM** — do not pin those independently.
- **Three follow-ups remain open by design:** **Elide** (F-2 — no confirmed Boot-4 release; Spring Data REST / MVC fallback), **Togglz** (F-3 — 4.x is Boot-3-targeted; own flag-table fallback), and **Resilience4j** (Boot-4 artifact exists but BOM-omitted, #2427). **JobRunr F-4 is resolved positively** (`jobrunr-spring-boot-4-starter` 8.7.0 ships).
- **Coordinate changes to honor:** `jackson-module-kotlin` → group `tools.jackson.module` (Jackson 3); `kotlin-logging` → `io.github.oshai`; Bucket4j → `com.bucket4j:bucket4j_jdk17-*`; openhtmltopdf → `io.github.openhtmltopdf`; Flyway now needs the **starter** + `flyway-database-postgresql`, not bare `flyway-core`.
- **This is a 2026-06-30 snapshot.** Re-run latest-version + CVE checks at scaffold ([../decisions/follow-ups.md](../decisions/follow-ups.md) F-5). The 🟡 lines and every "verify at scaffold" note are the explicit re-verification targets.
