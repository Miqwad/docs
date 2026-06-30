# Performance & Scalability

> Hard, measured, **CI-gated** performance targets — not aspirations — plus the capacity envelope, caching strategy, connection-pooling rule, DB performance rules, and the load/stress/soak regime that proves them.

**Stack context:** Kotlin / JDK 25 / Spring Boot 4, coroutines-first; PostgreSQL (Cloud SQL) + Redis (Memorystore); search served from the Redis availability cache, never a live cross-join.

Related: [reliability-dr.md](reliability-dr.md) (the same metrics drive both) · [cloud.md](cloud.md) (capacity) · [../decisions/adr-log.md](../decisions/adr-log.md) (**ADR-002** pooling, ADR-011 partitioning) · [../STATUS.md](../STATUS.md).

---

## 1. Latency SLOs ✅ (p95 / p99, server-side)

Measured at the service, excluding third-party gateway/government round-trips (those have their own integration SLOs in §5). Asserted by k6 thresholds in CI (§7).

| Path | p95 | p99 |
|---|---|---|
| Marketplace search | ≤ **800 ms** | ≤ 1.5 s |
| Listing detail + quote | ≤ **600 ms** | ≤ 1.2 s |
| Booking reserve (excl. gateway) | ≤ **1.5 s** | ≤ 2.5 s |
| Booking confirm (excl. external Tajeer/payment) | ≤ **1 s** | — |
| Dealer dashboard | ≤ **1.2 s** | — |
| Fleet / list endpoints | ≤ **800 ms** | — |
| Telemetry ingest (accept) | ≤ **100 ms** | — |
| Heavy reports | **async**, off the read replica (never on the live path) | |

## 2. Throughput & capacity envelope ✅

- **V1 business envelope:** ~**150 dealers · 8,000 vehicles · 1,500 bookings/day · 50,000 customers**.
- **Design headroom: 2×** — the system must meet SLOs at twice the expected load before the first large dealer is onboarded.
- **Telemetry:** ~**270 pings/s** steady (8k vehicles), with **2× burst** headroom; ingest is the highest-volume write path and is isolated (separate LB path + tighter rate limit + native-partitioned table).
- **Autoscaling:** Cloud Run scales `miqwad-api` on concurrency (min-instances ≥ 2, multi-zone). Scaling is **bounded by the connection-pool cap** (§4) — instances can't outrun the database.

## 3. Caching strategy ✅

- **Availability cache (Redis):** per-vehicle, next 90 days, rebuilt from the `availability_block` table; this is what makes marketplace search hit the SLO. **Target hit-ratio ≥ ~95%.** The DB is the source of truth — the cache is always rebuildable.
- **`vehicle_live` cache:** denormalized current position for the live fleet map (fed by telemetry), so the map never queries the partitioned history table.
- **Declarative caching (`@Cacheable`)** over Redis for hot reference reads.
- **Invalidation:** writes that change availability emit events that invalidate/refresh the affected cache keys; **correctness beats staleness on the booking path**.

## 4. Connection pooling & DB concurrency ✅ (ADR-002)

The single most important scalability constraint. On a coroutines-first runtime with virtual threads, the **pool — not the thread count — is the DB-concurrency limiter**.

- **HikariCP per instance**, sized deliberately (never defaults).
- **Cap rule:** `(max-instances of miqwad-api + miqwad-jobs) × pool-size < Cloud SQL max_connections` (minus headroom for migrations, the read replica, and admin). **Cloud Run autoscaling must never be able to exhaust Cloud SQL connections.**
- **PgBouncer (transaction pooling, ≥ 1.24)** on GCE/GKE multiplexes app connections onto few server connections when headroom tightens — **never statement pooling** (RLS uses `SET LOCAL` within the transaction). Validated at the load-test step.

## 5. Availability, durability & async SLOs ✅

- **Availability:** **99.5%** in year 1; a **booking-path success-rate SLO** with a monthly **error budget**; per-integration availability SLOs; graceful degradation keeps the core booking/handover/return working when non-critical deps are down ([reliability-dr.md](reliability-dr.md)).
- **Durability:** **RPO ≤ 15 min, RTO ≤ 4 h** with a *tested* restore ([reliability-dr.md](reliability-dr.md)).
- **Async:** transactional-outbox dispatch lag **p95 ≤ seconds**; reconciliation jobs catch anything that slips.
- **Per-integration SLOs:** synthetic uptime checks on each of Tajeer / ZATCA / Wasl / Moyasar plus the booking path; breaker trips and fallbacks are exercised.

## 6. Database performance rules ✅

- **Per-query budgets** with index coverage on the hot paths (availability lookup, booking by dealer/date, search filters, telemetry by vehicle/time).
- **No N+1 queries** — asserted in tests (fetch plans reviewed; batch / `@EntityGraph` / explicit joins where needed).
- The `tstzrange` + GiST exclusion constraint and partition pruning on `telemetry_ping` keep the two highest-pressure paths fast.
- Heavy / reporting queries run on the **read replica**, never the primary.

## 7. Enforcement — CI perf-gates + monitoring ✅

- **k6 / Gatling** load + **stress** (find the breaking point) + **soak** (sustained, find leaks) against the 2× envelope.
- **CI perf-regression gate:** k6 thresholds (the §1 latency SLOs + error-rate ceilings) **fail the build** if a change regresses performance — **performance is a merge gate, not a post-hoc check**.
- **Continuous monitoring:** Micrometer → Prometheus / Cloud Monitoring dashboards with an **alert per SLO**; synthetic uptime checks on the booking path and each integration. The same metric set feeds the reliability dashboards.

## 8. Core Web Vitals & mobile (frontend) ✅

- **Web (dealer/admin):** LCP < 2.5 s, INP < 200 ms, CLS < 0.1; JS budget per page type; images sized + lazy below the fold.
- **Mobile (customer RN app):** a **cold-start budget**; offline-resilient handover capture; OTA-deliverable JS fixes.
