# Reliability, Observability & Disaster Recovery

> How Miqwad stays observable, contains faults, recovers cleanly in steady state, and survives a catastrophe without losing money, contracts, or compliance data. Steady-state resilience and DR/business-continuity in one place.

**Stack context:** Kotlin / JDK 25 / Spring Boot 4 (Spring Framework 7 core resilience; Resilience4j Boot-4-ready), coroutines-first; JobRunr outbox dispatcher; PostgreSQL (Cloud SQL) + Redis (Memorystore); GCP **me-central2 (Dammam)**, in-Kingdom.

Related: [performance.md](performance.md) (the same metrics drive the SLOs) · [cloud.md](cloud.md) (topology, backups) · [sagas-outbox-jobs.md](sagas-outbox-jobs.md) (the sagas) · [../decisions/adr-log.md](../decisions/adr-log.md) (ADR-008, ADR-010) · [../STATUS.md](../STATUS.md).

---

## 1. Observability ✅

Principle: **every request and every external call is traceable end-to-end**, and the metrics that define the SLOs ([performance.md](performance.md)) are the metrics on the dashboards. **All four pillars — structured JSON logging, correlation/trace IDs, Sentry, and distributed tracing — are in from the start; none is deferred to V1.5.** This is a deliberate build-for-the-long-run decision: an unobservable saga is undebuggable, and retrofitting tracing later is far more expensive than wiring it in on day one.

| Pillar | How |
|---|---|
| **Structured logs** | JSON (Logback) with a **correlation/trace id** threaded through every request, saga step, and external call → Cloud Logging. PII redacted; gov credentials never logged. |
| **Metrics** | Micrometer → Prometheus / Cloud Monitoring: request rates, latencies (search/booking p95/p99), error rates, **plus JobRunr job stats, transactional-outbox lag, and per-integration success/failure rates**. |
| **Distributed tracing** | Micrometer Tracing / OpenTelemetry **across the saga steps** (booking → payment → Tajeer → confirm; return → ZATCA → settle → close) — a stuck saga is a visible span, not a guess. |
| **SLO dashboards + alerts** | One dashboard per SLO with **an alert per SLO**; the booking-path success-rate burns a monthly error budget. |
| **Error tracking** | Sentry on the API and both apps. |
| **Audit logging** | Admin + vault access (Cloud Audit Logs); booking/contract/payment/inspection status events (Hibernate Envers or `*_status_event` tables). |
| **Retention & redaction** | Log retention per the data-retention schedule; PII redaction at the logging layer. |

## 2. Fault tolerance & resilience ✅

Each external dependency (Tajeer / ZATCA / Wasl / Absher / Moyasar / notifications) is isolated so its failure is contained.

- **Per-integration circuit breakers** (Spring FW 7 core resilience; Resilience4j optional) — a Tajeer outage must not trip ZATCA. Each adapter has its own breaker, **timeout**, and **bounded retries with backoff + jitter**.
- **Bulkhead isolation:** concurrency limits per integration so one slow dependency can't exhaust shared threads/pool.
- **Rate limiting + load shedding:** Bucket4j (app) + Cloud Armor (edge) protect auth/search/booking/telemetry; under overload, non-critical work sheds first.
- **Transactional outbox + saga compensation + idempotency:** the booking-confirm and return-settle sagas commit state and queue external calls atomically; compensation (refund / release) is first-class; every money/external call is idempotent. **Money is never held against a rental that doesn't legally exist** — if Tajeer registration fails after payment, auto-refund.
- **Reconciliation jobs:** JobRunr jobs reconcile payment/Tajeer/ZATCA state so anything that slips through a transient failure is detected and repaired.
- **Dead-letter / poison-message handling:** a message that keeps failing is parked in a dead-letter path with an alert, not retried forever.

### Graceful degradation

If notifications / analytics / telemetry or other **non-critical** deps are down, **booking / handover / return still work**; the failed side-effects queue and replay when the dependency recovers.

### Explicit fallbacks

| Dependency down | Fallback |
|---|---|
| **Tajeer** | Charge + queue the registration; the booking waits in a safe state. Auto-refund if registration never completes — money is returned, never kept against a non-existent contract. |
| **Elide** not Boot-4-ready / lagging | Spring Data REST or plain controllers (the core never depends on Elide — see F-2). |
| **Togglz** lagging | Own boolean flag table behind the same interface (F-3). |

## 3. Reliability & availability ✅

- **Redundancy:** ≥ 2 `miqwad-api` instances, **multi-zone within me-central2**; no single point of failure in the app tier.
- **Database:** Cloud SQL **regional HA** (synchronous standby, automatic failover) + a **read replica** for reporting.
- **Health probes:** Spring Boot Actuator **health / readiness / liveness** — readiness gates traffic during startup/migration; liveness restarts a wedged instance.
- **Zero-downtime deploys:** new Cloud Run revision + forward-only, backward-compatible Flyway migrations; traffic shifted after readiness passes.
- **Availability target:** 99.5% year 1; booking-path success-rate SLO with a monthly error budget.
- **Chaos / failure-injection (V1.5):** deliberately kill a sandbox dependency and confirm the breaker trips, the fallback engages, work queues, and no data is lost.

## 4. Incident response & on-call ✅

| Severity | Scope | Response |
|---|---|---|
| **Sev1** | Booking path or payments down / data-integrity risk | Immediate; status page; senior backend escalation |
| **Sev2** | A single integration degraded | Prompt; templated comms |
| **Sev3** | Non-critical | Tracked, next business day |

- **On-call rota:** lightweight (small team) — primary + backup, with the senior backend owning infra escalation; documented runbooks per known failure (Tajeer down, payment-webhook backlog, outbox lag, DB failover).
- **Comms + status:** internal incident channel + a customer/dealer status page for Sev1/Sev2; templated updates.
- **Blameless postmortems:** every Sev1/Sev2 gets a written, blameless postmortem with action items tracked to closure.
- **Escalation to security:** suspected secret exposure follows the credential-exposure protocol (rotate → investigate).

---

# Disaster Recovery & Business Continuity

> The catastrophe plan: zone outage, corrupted database, lost region, or a key person unavailable. Owns the **residency-vs-DR tension**.

**Stack context:** Cloud SQL regional HA + read replica; GCS versioned buckets; PITR; all data in-Kingdom (PDPL). See [../decisions/adr-log.md](../decisions/adr-log.md) (ADR-010, ADR-011).

## 5. Objectives ✅

| Objective | Target |
|---|---|
| **RPO** (max data loss) | **≤ 15 minutes** |
| **RTO** (time to restore) | **≤ 4 hours** |
| **In-Kingdom always** | Recovery must not move personal data out of Saudi Arabia (PDPL) — constrains the DR topology (§8). |
| **Compliance integrity** | A restore must not re-register a contract or re-clear an invoice with the government (the imported/historical-doc guard also protects restores). |

## 6. Backup strategy ✅

- **Cloud SQL:** automated daily backups + **Point-In-Time Recovery (PITR)** via write-ahead logs → meets RPO ≤ 15 min. Backups stored **in-Kingdom**.
- **GCS:** object **versioning** + lifecycle on the media/docs/import buckets; cross-zone durable by default; CMEK on docs/KYC.
- **Secrets/keys:** Secret Manager versions retained; KMS keys are regional and recoverable; the gov-credential vault is part of the continuity plan (its loss blocks compliance operations).
- **Configuration/infra:** environments are **Terraform-defined** (IaC) so a project can be rebuilt reproducibly; no snowflake config.
- **Backup verification:** a backup is not a backup until a restore has succeeded — see §7.

## 7. Failure scenarios & restore drills ✅

| Scenario | Response | Target |
|---|---|---|
| Single **zone** outage | Cloud SQL regional HA fails over to the standby zone; Cloud Run is already multi-zone; monitor only | within RTO, ~zero data loss |
| **App** instance wedged / bad deploy | Liveness restart; roll back to the previous Cloud Run revision (same image, prior tag) | minutes |
| **Database corruption / bad migration** | PITR restore to just before the event; replay outbox/reconciliation; verify ledger integrity | within RTO, RPO ≤ 15 min |
| **Accidental data deletion** | PITR or GCS object-version restore for the affected scope | RPO ≤ 15 min |
| **Region loss** (me-central2) | The hard case — see §8 | per §8 outcome |
| **Vault / key compromise** | Rotate per the credential-exposure protocol; restore from prior secret versions if needed | minutes–hours |

**Drills (tested, not assumed):**

- A **documented failover/restore runbook** with step-by-step recovery for each scenario and explicit ownership (senior backend leads recovery; business lead handles external/dealer comms).
- A **game-day** exercise (V1.5, before scaling) that actually executes a PITR restore into a clean project and measures real RPO/RTO against the targets.
- Each drill asserts: ledger balances reconcile, no gov doc is re-submitted, RLS/tenancy intact, and the booking path works end-to-end post-restore.

## 8. The residency-vs-DR tension 🔵

PDPL requires personal data to stay **in-Kingdom**, but classic DR wants a **second, geographically separate region**. These pull in opposite directions.

- **Verify** whether a **second in-KSA GCP region/zone** is available via CNTXT for cross-region DR (ties to STATUS **F-1**).
- **If a second in-Kingdom region exists:** asynchronous cross-region replica + cross-region backup, both in-Kingdom — full DR.
- **If not (the likely V1 reality):** V1 DR = **me-central2 multi-zone HA + PITR + tested restore + an off-region-but-still-in-Kingdom backup copy**. This is a *conscious* residency-driven limitation, not an oversight, and is revisited when in-Kingdom DR options expand.
- **Never** fail over or back up to a region outside Saudi Arabia for personal data.

## 9. Business continuity (beyond the database) ✅

- **Data export / portability:** dealers can export their data (fleet, bookings, contracts, invoices, customers) — both a PDPL/contractual right and a continuity hedge if Miqwad itself is unavailable; the inverse of the import module.
- **Government-integration continuity:** if a gov platform is down for an extended period, bookings continue in a safe state with queued registration (§2 fallback); dealers are notified; money charged up front is auto-refunded if a contract never registers — never kept against an unregistered contract.
- **People continuity:** no single person is the only one who can recover the system — the runbook, IaC, and credential-recovery procedure are documented and at least two team members can execute them.
- **Vendor continuity:** the provider-agnostic payment adapter and the provider-agnostic integration adapters mean a single vendor outage is degraded service, not an outage.
