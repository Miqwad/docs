# Miqwad — DevOps & Environments

How we build, test, ship, run, and observe Miqwad on GCP (me-central2): environments, CI/CD, runtime services, secrets, observability, and DR. There is no dedicated DevOps role — the **senior backend owns CI/CD + infra**, everyone runs TDD. Cloud topology lives in [../architecture/cloud.md](../architecture/cloud.md); pinned versions in [./stack.md](./stack.md).

> **Status:** ✅ **Decided** for the pipeline and operational discipline. 🟣 me-central2 per-service availability + CNTXT procurement must be confirmed before staging stands up ([../STATUS.md](../STATUS.md) F-1).

---

## 1. Environments

Each environment is its own VPC, Cloud SQL, Memorystore, GCS buckets, Secret Manager, and Keycloak realm. **Credentials never cross environments.**

| Env | Where | Purpose |
|---|---|---|
| **local** | dev machines | Testcontainers (real Postgres/Redis), WireMock for gov sandboxes, Spring run locally |
| **staging** | GCP `miqwad-staging` (me-central2) | mirrors prod; holds gov **sandbox** credentials; mandatory — the four integrations cannot be tested in prod |
| **prod** | GCP `miqwad-prod` (me-central2) | production credentials, isolated |
| **shared** | GCP `miqwad-shared` | Artifact Registry, CI service accounts |

---

## 2. CI pipeline (merge gate)

Runs on every PR (GitHub Actions or Cloud Build). **No merge to main without green CI.**

| Step | What runs |
|---|---|
| 1. Compile + lint | Gradle Kotlin DSL build · Detekt + ktlint · ArchUnit/Konsist module rules |
| 2. Unit | JUnit 5 + MockK — money / availability / state-guard logic |
| 3. Integration | Testcontainers (real Postgres: exclusion constraint + RLS + `@TenantId` + pg_partman/PostGIS) + WireMock (gov sandboxes) |
| 4. Contract + invariants | Contract tests vs the OpenAPI spec · idempotency replay · the two-saga e2e + imported-doc guard + entitlement gating + cross-tenant-leak tests |
| 5. Security | OWASP dependency-check · secret scanning · SBOM. Frontend: lint, type-check, component + Playwright visual tests |
| 6. Image | Build container (Jib or multi-stage Dockerfile) → push to Artifact Registry |

The non-negotiable tests (cross-tenant leak, no-double-book, saga branches, idempotency, imported-doc guard, entitlement gating) are CI gates — see [./testing.md](./testing.md).

## 3. CD pipeline

- **Staging:** auto-deploy on merge to main (new Cloud Run revision).
- **Prod:** **manual approval** gate promotes the **same image** with env-specific config; staging soak first.
- **DB migrations:** Flyway runs on deploy, **forward-only**; documented rollback plan per release; migrations reviewed like code.
- **Mobile:** EAS build/submit pipeline; **OTA** updates for JS-layer fixes; staged store rollout.
- **Feature flags (Togglz):** gate risky / credential-gated features — turn Tajeer/ZATCA/Wasl on **per dealer** as provisioning completes, decoupled from deploys.

---

## 4. Runtime services (Cloud Run)

One image per service, promoted across envs; config via env vars + Secret Manager refs.

| Service | Min instances | Responsibility |
|---|---|---|
| `miqwad-api` | ≥ 2 (multi-zone, autoscale on concurrency) | REST + Elide + webhooks |
| `miqwad-jobs` | ≥ 1 | JobRunr workers + dashboard: outbox dispatch, reconciliation, telemetry, notifications, settlement, dunning, imports, pg_partman maintenance |
| `keycloak` | — (Cloud Run/GKE) | identity |

### Connection pooling — the autoscaling × pool-cap rule (ADR-002)

Each instance runs **HikariCP**, sized deliberately (with coroutines/virtual threads the **pool, not the threads, is the DB-concurrency limiter**). The real risk is Cloud Run autoscaling × per-instance pool exceeding Cloud SQL's connection cap, so:

> Hold `(max-instances of miqwad-api + max-instances of miqwad-jobs) × pool-size` **below** the Cloud SQL `max_connections` (leaving headroom for admin/replica/migration connections).

When headroom gets tight, put **PgBouncer (transaction-pooling mode, ≥ 1.24)** on GCE/GKE in front of Cloud SQL so many app connections multiplex onto few DB connections — **never statement pooling** (RLS uses `SET LOCAL` inside the transaction). Validated at the load-test step: drive the booking path at 2× envelope and confirm no connection-pool exhaustion.

---

## 5. Secrets & config

- **Secret Manager + Cloud KMS (CMEK):** the **dealer gov-credential vault** (Tajeer/Naql + ZATCA CSIDs), gateway keys, DB creds. Fetched at call time, **never logged**, never in Postgres plaintext; access audited.
- **Workload Identity** for Cloud Run → Google APIs; **Workload Identity Federation** for CI → GCP (no long-lived key files).
- Required secrets are validated at startup (fail fast if missing).

## 6. Observability

| Pillar | How |
|---|---|
| **Logging** | Structured JSON (Logback) with a correlation/trace id threaded through every request and external call → Cloud Logging |
| **Metrics** | Micrometer → Prometheus/Cloud Monitoring: request rates, latencies (search p95, booking p95), error rates, queue/outbox depth, integration success/failure, JobRunr job stats |
| **Tracing** 🟡 | Micrometer Tracing/OpenTelemetry across saga steps (booking→payment→Tajeer) — V1.5 |
| **Error tracking** | Sentry on API + both apps |
| **Uptime/synthetics** | On the booking path + each external integration; alerting on SLOs (availability 99.5% yr-1) |
| **Dashboards** | The metric set from the engineering spec; a JobRunr dashboard for jobs |

---

## 7. Resilience & DR

- Cloud SQL **regional HA** + read replica; **PITR** + daily backups (**RPO ≤ 15 min, RTO ≤ 4 h**); a **tested restore runbook** — an untested backup isn't a backup (V1.5).
- GCS **versioning** + lifecycle for media/docs; cross-zone durability.
- **Graceful degradation:** notifications/analytics down ⇒ booking/handover/return still work.
- ≥ 2 API instances, multi-AZ within me-central2; no single point of failure in the app tier.

## 8. Security ops (cloud)

Cloud Armor WAF + rate limits on auth/search/booking · TLS 1.2+/HSTS at the LB · private east-west · VPC-SC perimeter around data services · Cloud Audit Logs on admin + vault · Security Command Center posture · Dependabot + SAST in CI · a focused third-party pen-test of auth/payment/vault before scaling payment volume (V1.5).

---

## 9. Branch & release discipline

Trunk-based with short-lived PR branches · conventional commits · green CI required · staging soak before prod promotion · release notes per deploy. **Incident response & on-call** (severity levels, on-call rota, comms/status page, blameless postmortems) and the runbooks live with the reliability spec.

## 10. Cost / FinOps

Investor-funded, so runway awareness matters.

- **Budgets + alerts:** GCP budget alerts per project (staging/prod); biggest line items watched are Cloud SQL (HA + replica), Cloud Run, Memorystore, egress, GCS.
- **Unit economics:** track **cost-per-booking** and **cost-per-dealer** so infra cost is understood against revenue (commission + subscription).
- **Levers:** Cloud Run min-instances tuned per env; read replica sized for reporting load; GCS lifecycle (photos 24mo, telemetry pruned); non-prod scaled down off-hours.
- **Runway:** monthly cloud spend reviewed against runway; budget alerts are the guardrail against surprise bills.

## 11. Mobile release & forced-update

- **EAS** build/submit for the RN customer app; **OTA** for JS-layer fixes; staged store rollout.
- **Min-version gating:** the app checks its version against the versioned `/v1` API on launch; below the minimum-supported version it forces an update. The contract evolves additively; breaking changes are gated behind a min-version, never pushed onto old apps.
- **OTA policy:** JS-only fixes via OTA; native changes go through the stores; OTA never ships a change that needs a newer native binary.
- **Store & certs:** store-review lead time budgeted into releases; push-notification certs/keys (FCM/APNs) managed as secrets with rotation tracked.

## 12. Prep

- 🟣 Verify me-central2 service availability + complete **CNTXT** procurement before staging stands up ([../STATUS.md](../STATUS.md) F-1).
- **IaC:** Terraform the GCP projects/VPC/Cloud SQL/Cloud Run/Secret Manager so envs are reproducible.
