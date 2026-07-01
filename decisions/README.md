# Decisions (ADRs)

Architecture Decision Records for Miqwad live in [adr-log.md](adr-log.md). External-verification items pulled out of the records are in [follow-ups.md](follow-ups.md).

## Status legend

| Status | Meaning |
|---|---|
| `Proposed` | Recorded and reasoned; **awaiting founder acceptance**. |
| `Accepted` | Ratified — build to it. Carries the acceptance date. |
| `Open` | A genuine sub-decision not yet made (tracked in [STATUS.md](../STATUS.md)). |

## Index

| ADR | Title | Status |
|---|---|---|
| [001](adr-log.md#adr-001--backend-kotlin-on-jdk-25--spring-boot-4-modular-monolith) | Backend: Kotlin/JDK 25/Spring Boot 4, modular monolith | Accepted |
| [002](adr-log.md#adr-002--postgresql-system-of-record--redis--connection-pooling) | PostgreSQL + Redis + connection pooling | Accepted |
| [003](adr-log.md#adr-003--multi-tenancy-shared-schema--dealership_id--tenantid--rls) | Multi-tenancy: shared schema + `@TenantId` + RLS | Accepted |
| [004](adr-log.md#adr-004--no-double-booking-via-tstzrange--gist-exclusion-constraint) | No double-booking (GiST exclusion constraint) | Accepted |
| [005](adr-log.md#adr-005--money-as-integer-halalas--append-only-ledger) | Money: integer halalas + append-only ledger | Accepted |
| [006](adr-log.md#adr-006--schema--key-conventions-uuidv7-keys-flyway-owns-the-schema) | Schema & key conventions (UUIDv7, Flyway) | Accepted |
| [008](adr-log.md#adr-008--sagas-via-persisted-state-machine--transactional-outbox-dispatched-by-jobrunr) | Sagas: state machine + outbox + JobRunr | Accepted |
| [010](adr-log.md#adr-010--cloud-gcp-dammam-me-central2) | Cloud: GCP me-central2 (Dammam) | Accepted |
| [011](adr-log.md#adr-011--postgres-data-platform-extensions-native-partitioning--pg_partman-postgis) | Postgres extensions: partitioning + PostGIS | Accepted |
| [013](adr-log.md#adr-013--feature-flags--entitlements-togglz--an-entitlementsubscription-model) | Feature flags + entitlements (Togglz) | Accepted |
| [014](adr-log.md#adr-014--zatca-phase-2-wrap-the-official-sdk) | ZATCA Phase 2: wrap the official SDK | Accepted |
| [015](adr-log.md#adr-015--payments-moyasar) | Payments: Moyasar | Accepted |
| [016](adr-log.md#adr-016--api-surface-hand-written-rest-v1-for-the-domain--elide-jsonapi-for-crud) | API: REST `/v1` + Elide JSON:API for CRUD | Accepted |
| [021](adr-log.md#adr-021--auth) | Auth: self-hosted Keycloak (me-central2) | Accepted* |
| [022](adr-log.md#adr-022--frontend-react-native-expo-customer-app-web-portals-shared-design-system) | Frontend: Expo app + React (Vite) SPA portals + Next.js public surface | Accepted |

\* ADR-021 is **Accepted — pending final CNTXT residency confirmation** (2026-07-01); Keycloak is the working default regardless of that reply.

## How this set was curated (from the pre-build draft log of 23)

This is a **deliberate curation** of the original 23-record draft log down to the decisions that are genuinely *architecture* decisions. The mapping:

- **Consolidated** — small, tightly-related records merged: UUIDv7 + Flyway → **ADR-006**; telemetry-partitioning + PostGIS → **ADR-011**.
- **Folded in** — the background-jobs (JobRunr) record into **ADR-008**; the API-strategy record into **ADR-016**.
- **Relocated** — decisions that belong in a product/spec doc, not the architecture log:
  - Product packaging (3 packages) → [product/pricing-packaging.md](../product/pricing-packaging.md)
  - Multi-channel bookings → [product/use-cases.md](../product/use-cases.md)
  - Migration / historical import → [engineering/data-migration.md](../engineering/data-migration.md)
  - V1/V2 re-scope → [delivery/backlog.md](../delivery/backlog.md)

Every record was rewritten in present-tense decided voice; the pre-build "we used to think…" pivot-notes and the version-readiness hedges were removed (the latter moved to [follow-ups.md](follow-ups.md)).
