# Architecture Overview & Invariants

How Miqwad is built, and the **non-negotiable invariants** every module must honor. This is the hub; the sibling docs go deep on each area. Read this first.

Miqwad is a **modular monolith** on **Kotlin 2.x (K2) / JDK 25 / Spring Boot 4** (Spring Modulith, coroutines-first), with PostgreSQL as the system of record and Redis as the hot cache/queue. One module per bounded context (`com.miqwad.<module>`); modules talk **only** through a published event or a module's public `api` interface — never by reaching into internals (enforced by ArchUnit + Modulith tests). Microservice extraction is a V2 option, and the boundaries are drawn so extraction would be mechanical. See [ADR-001](../decisions/adr-log.md#adr-001--backend-kotlin-on-jdk-25--spring-boot-4-modular-monolith).

---

## The non-negotiable invariants

These hold across every module and are the reason to read more than one doc. Each is enforced in code **and** at the database level.

| # | Invariant | How it's enforced | Reference |
|---|---|---|---|
| 1 | **Multi-tenancy** — every tenant-owned row carries `dealership_id`, always from the **JWT claim, never request input** | Hibernate `@TenantId` (auto filter + auto insert) **and** Postgres **RLS** as the hard backstop | [ADR-003](../decisions/adr-log.md#adr-003--multi-tenancy-shared-schema--dealership_id--tenantid--rls) · [data-model](data-model.md) · [security](security.md) |
| 2 | **No double-booking** — a vehicle is available for a window iff no `availability_block` overlaps it | Postgres `EXCLUDE USING gist (vehicle_id WITH =, tstzrange(start_at,end_at) WITH &&)` — DB-level, **all channels** | [ADR-004](../decisions/adr-log.md#adr-004--no-double-booking-via-tstzrange--gist-exclusion-constraint) |
| 3 | **Money** — integer **halalas** + explicit currency, never floats; every movement is an **append-only ledger** entry, balances derived | `Money` `@JvmInline value class`; immutable `ledger_entry` | [ADR-005](../decisions/adr-log.md#adr-005--money-as-integer-halalas--append-only-ledger) |
| 4 | **Keys & schema** — UUIDv7 PKs (UUIDv4 only for public URLs); **Flyway owns the schema** | App-generated UUIDv7; Hibernate auto-DDL off outside local dev | [ADR-006](../decisions/adr-log.md#adr-006--schema--key-conventions-uuidv7-keys-flyway-owns-the-schema) |
| 5 | **Sagas & compensation** — booking-confirm and return-settle are persisted state machines + transactional outbox; compensation (void/release/refund) is first-class | `outbox` table is source of truth; JobRunr dispatches | [ADR-008](../decisions/adr-log.md#adr-008--sagas-via-persisted-state-machine--transactional-outbox-dispatched-by-jobrunr) · [sagas-outbox-jobs](sagas-outbox-jobs.md) |
| 6 | **Idempotency** — every money- or external-system endpoint takes an idempotency key; webhooks are signature-verified and processed once | `IdempotencyService` (Redis/Postgres `idempotency_key`) | [api/reference](../api/reference.md) |
| 7 | **Compliance adapters** — Tajeer, ZATCA, Wasl, Absher are credential-gated, provider-agnostic adapters behind circuit breakers; dealer gov-credentials live in an encrypted vault, **never** in the app DB, never logged | Resilience4j breakers; Cloud KMS / Secret Manager vault | [ADR-014](../decisions/adr-log.md#adr-014--zatca-phase-2-wrap-the-official-sdk) · [integrations](integrations.md) · [security](security.md) 🟡 |

> **Cardinal rule:** money is never held against a rental that doesn't legally exist — if Tajeer registration fails after payment, the authorization is auto-voided/refunded.

---

## Non-functional targets (V1 envelope)

| Dimension | Target |
|---|---|
| Marketplace search (hottest path) | p95 ≤ **800 ms**, p99 ≤ **1.5 s** (city + date-window, ≤5,000 vehicles; served from Redis, never a live cross-join) |
| Dealer dashboard / fleet list | p95 ≤ **1.2 s** |
| Booking creation (reserve) | p95 ≤ **1.5 s** (excl. payment-gateway round trip) |
| Scale envelope | 150 dealerships · 8,000 vehicles · 50,000 customers · 1,500 bookings/day — no re-architecture |
| Telemetry ingest | ~**270 pings/s** sustained (8,000 vehicles × 1 ping/30 s), bursting 2× |
| Availability | **99.5%** year one (→ 99.9% V2); ≥2 API instances, multi-AZ in-region |
| Disaster recovery | **RPO ≤ 15 min · RTO ≤ 4 h**; daily backup + PITR |
| Residency (PDPL) | All personal + operational data **in-Kingdom** (GCP me-central2) |

Detail in [performance.md](performance.md) and [reliability-dr.md](reliability-dr.md).

---

## Bounded contexts (modular monolith)

Each module owns its entities and exposes a surface that is either **hand-built REST** (the complex domain) or **Elide JSON:API CRUD** (data-entry surfaces) — see [ADR-016](../decisions/adr-log.md#adr-016--api-surface-hand-written-rest-v1-for-the-domain--elide-jsonapi-for-crud).

| Module | Owns | Surface |
|---|---|---|
| Identity & Tenancy | auth integration, tenant filter | REST |
| Dealership & Onboarding | `dealership` | REST + Elide |
| Packaging, Entitlements & Billing | `plan`, `subscription`, `entitlement`, `commission_config` | REST |
| Branch · Fleet | `branch`, `vehicle`, images/documents, categories, rate plans | mostly Elide + REST (uploads) |
| Availability & Pricing | `availability_block` | REST (internal) |
| Marketplace Search | — (reads) | REST |
| Booking (all channels) | `booking`, `booking_status_event` | REST |
| Contract (Tajeer) | `rental_contract`, `tajeer_link` | REST + adapter |
| Inspection, Handover & Damage | `inspection`, `inspection_photo`, `damage_record` | REST |
| Payments · Invoicing (ZATCA) · Settlement | `payment`, `invoice`, `payout`, `ledger_entry` | REST + adapters |
| Maintenance · Telemetry/Tracking | `work_order`, `telemetry_ping` (partitioned) | Elide + REST ingest |
| Delivery · Reviews · Saher | `delivery`, `review`, `traffic_violation` | REST + Elide |
| Notifications | `notification` (+ comms log) | Elide read + service |
| Import / Migration | `import_batch`, `import_row` | REST |
| Reporting | — (read replica) | REST |
| Admin (platform) | — (thin) | REST + Elide |

Module-by-module detail (services, events, jobs, guards) lives with each area's doc; the booking state machine is in [product/use-cases.md](../product/use-cases.md).

---

## Cross-cutting model (the shared kernel)

- **Concurrency (ADR-001):** coroutines-first for I/O-bound orchestration (saga steps, external/gov calls); blocking JPA runs on `Dispatchers.IO` / virtual threads; I/O controllers are `suspend`. Never block the request thread on an external call without isolating it.
- **Tenant context (ADR-003):** a request filter reads `dealership_id` from the JWT into a `ThreadLocal`, carried across coroutine suspension via `threadLocal.asContextElement(id)` (primary) with `ScopedValue` (GA in JDK 25) on blocking paths; each transaction pushes `SET LOCAL app.current_dealership` (drives `@TenantId` + RLS). **Background jobs pass `dealership_id` explicitly and never inherit it.** RLS is the hard backstop if any path is missed.
- **Money (ADR-005):** a `Money` `@JvmInline value class` (Long halalas + currency); all arithmetic via it; no floats.
- **Idempotency:** `IdempotencyService` wraps money/external POSTs.
- **Entitlement:** `EntitlementGuard` interceptor resolves modules/limits, throws `entitlement_*` (402/403); cached, recomputed on subscription change.
- **Errors:** Spring `ProblemDetail` → the uniform envelope `{error:{code,message,details},request_id}` via a global `@RestControllerAdvice`.
- **Outbox:** `OutboxService.publish(event)` writes to `outbox` in the same transaction as the state change; JobRunr dispatches.
- **Audit:** Hibernate Envers (or `*_status_event` tables) on booking/contract/payment/inspection.

### Module dependency rules (ArchUnit-enforced)
- `web` → `domain` → repositories; controllers never touch repositories directly.
- Cross-module contact only via the target's `api` interface or a published event.
- `Booking` orchestrates `Contract`/`Payments`/`Invoicing` **via the saga/outbox**, not direct calls into their internals.
- `Entitlement` is a cross-cutting guard depended on by all dealer-facing modules; it depends on none of them.
- Integration adapters sit behind interfaces; domain modules depend on the interface, never the wire details.

---

## What this depends on that isn't final

The architecture is settled; two classes of detail are not (see [STATUS.md](../STATUS.md)):
- 🟡 **Integration contracts** (Tajeer/ZATCA/Wasl/Absher/Moyasar) — adapters are decided, wire-shapes pending vendor docs.
- 🟡 **Business-flow detail** (booking/handover/maintenance/pricing) — pending dealer discovery interviews.
- 🔵 **Web-portal framework** (React vs Next.js) — see [design/frontend-design-system.md](../design/frontend-design-system.md).
