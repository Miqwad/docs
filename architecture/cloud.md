# Cloud Architecture (GCP)

> The target deployment on Google Cloud: topology, service map, networking, data, security, environments, and the data-flow paths that matter.

**Region & residency (the governing constraint):** all personal/operational data lives **in-Kingdom** to satisfy PDPL. Primary region **me-central2 (Dammam)** (CST/NCA Class C licensed). KSA access to me-central2 is via **CNTXT** (Google's exclusive reseller).

> 🟣 **F-1 — pending verification.** Per-service availability in me-central2 must be confirmed via CNTXT before scaffolding (Cloud SQL, Memorystore, Cloud Run, Cloud Armor, Serverless VPC connector, Secret Manager). Where a service lags, the documented fallback applies (e.g. Redis on GKE) — but **data must stay in me-central2**; never fall back to an out-of-Kingdom region for any personal data. See [../STATUS.md](../STATUS.md) (F-1) and [../decisions/adr-log.md](../decisions/adr-log.md) (ADR-010).

Related: [../decisions/adr-log.md](../decisions/adr-log.md) (**ADR-010** cloud/region, **ADR-011** Postgres extensions, **ADR-002** pooling) · [performance.md](performance.md) · [reliability-dr.md](reliability-dr.md) · [../STATUS.md](../STATUS.md).

---

## 1. Principles ✅

- **Residency first** — no personal data leaves me-central2; backups and object storage are in-region.
- **Managed over self-managed** — Cloud SQL, Memorystore, Cloud Run, Secret Manager; the small team runs *features*, not infrastructure.
- **Private by default** — app↔data traffic on private IP; buckets private; the only public surface is the load balancer.
- **Least privilege** — per-service IAM service accounts; the dealer gov-credential vault is the highest-trust asset.
- **Stateless app tier** — horizontal scale; all state in Postgres / Redis / GCS.
- **One artifact, many envs** — the same container promoted local → staging → prod.

## 2. Topology

```
clients → Cloud CDN → HTTPS LB → Cloud Armor → Cloud Run (miqwad-api)
            → { Cloud SQL, Memorystore, GCS, Secret Manager, Keycloak }
Cloud Run (miqwad-jobs / JobRunr) → outbox dispatch → external systems
payment webhooks → miqwad-api (signature-verified, idempotent)
GPS/OBD devices → HTTPS LB (rate-limited path) → miqwad-api (telemetry ingest)
```

- **Clients:** Customer app (React Native/Expo), Dealer portal (React/Next.js), Admin portal (React/Next.js), GPS/OBD devices.
- **Edge:** Global External HTTPS Load Balancer → Cloud Armor (WAF + rate limit) → Cloud CDN (web + media).
- **App tier (Cloud Run, private VPC):** `miqwad-api` (Spring Boot modular monolith) + `miqwad-jobs` (JobRunr workers).
- **Data tier:** Cloud SQL PostgreSQL (HA + read replica, PostGIS/pg_partman), Memorystore Redis (HA), Cloud Storage (private buckets), Secret Manager + Cloud KMS (CMEK), Keycloak.
- **External (egress via NAT, allow-listed):** Tajeer, ZATCA Fatoora, Wasl/TGA, Absher, Moyasar, FCM/SMS/Email.

## 3. Service map ✅

| Concern | GCP service | Notes / fallback |
|---|---|---|
| App runtime | **Cloud Run** (`miqwad-api`) | Serverless, autoscaling; simplest for a modular monolith. Fallback: **GKE Autopilot** if StatefulSets/long-lived workers are needed. |
| Background jobs | **Cloud Run** (`miqwad-jobs`, JobRunr) | Separate min-instances service (JobRunr needs always-on workers + dashboard). |
| Database | **Cloud SQL for PostgreSQL** (regional HA) | PostGIS + pg_partman + btree_gist + pgcrypto. **No TimescaleDB** (ADR-011). Read replica for reporting. |
| Cache / queues | **Memorystore for Redis** (HA) | Availability cache, live-vehicle cache, rate-limit buckets. Fallback: Redis on GCE/GKE (F-1). |
| Object storage | **Cloud Storage** (regional, me-central2) | Private buckets; V4 signed URLs; versioning + lifecycle; CMEK. Photos, docs, imports. |
| Secrets / keys | **Secret Manager + Cloud KMS (CMEK)** | The dealer Tajeer/ZATCA-CSID vault; envelope encryption for KYC columns. |
| Identity | **Keycloak** (self-hosted, Cloud Run/GKE) — ✅ decided (O-2) | Customer OTP, staff/platform SSO, MFA. Runs in-region (residency guaranteed). GCIP was ruled out (not fully in-Kingdom); CNTXT confirmation pending. |
| Edge / WAF | **Cloud Load Balancing + Cloud Armor** | TLS, HSTS, WAF rules, IP/rate limiting (complements Bucket4j). |
| CDN | **Cloud CDN** | Static web + cached public media. |
| Email / SMS / push | external (FCM, Unifonic SMS, email API) | Behind the notification abstraction. |
| CI/CD | **Cloud Build + Artifact Registry** (or GitHub Actions + WIF) | Build/scan/push/deploy. |
| Observability | **Cloud Logging / Monitoring / Trace** + Micrometer/Prometheus/Sentry | Structured logs, metrics, traces, uptime checks, alerts. |
| Mobile builds | **EAS** (Expo) | OTA JS updates; store submissions. |

## 4. Networking ✅

- **VPC** in me-central2; Cloud Run reaches Cloud SQL/Memorystore over **private IP** via a **Serverless VPC Access connector** (no public DB IP).
- **Cloud SQL:** private IP only; IAM DB auth where possible; **no public IP**.
- **Egress** to government/payment systems via **Cloud NAT** with a stable egress IP (some gov portals IP-allow-list) and an **egress allow-list** (Tajeer / ZATCA / Wasl / Absher / Moyasar / FCM / SMS only).
- **Ingress:** single Global External HTTPS Load Balancer → Cloud Armor → Cloud Run. Telemetry ingest shares the LB (separate path + tighter rate limits).
- **VPC Service Controls** perimeter around the data services (Cloud SQL, GCS, Secret Manager) to prevent exfiltration; **Private Google Access** for calls to Google APIs.

## 5. Data architecture ✅

- **Cloud SQL Postgres**, regional **HA** (automatic failover, synchronous standby in a second zone). **Read replica** dedicated to reporting/analytics so heavy dealer reports never touch the primary. **PITR** + automated daily backups (RPO ≤ 15 min, RTO ≤ 4 h — see [reliability-dr.md](reliability-dr.md)), backups in-region.
- **Connection pooling (ADR-002):** each Cloud Run instance runs **HikariCP** (sized deliberately — with coroutines/virtual threads the pool, not the thread count, is the DB-concurrency limiter). Because Cloud Run autoscales, `(max-instances across miqwad-api + miqwad-jobs) × Hikari pool-size` is **capped below Cloud SQL `max_connections`** (with headroom for migrations, the read replica, and admin). When that headroom tightens, **PgBouncer in transaction-pooling mode (≥ 1.24, prepared-statement-safe) on GCE/GKE** multiplexes many app connections onto few server connections — **never statement pooling** (RLS relies on `SET LOCAL` within the transaction). Sizing is proven at the load-test step. Cloud SQL is private IP only; the pooler shares the VPC.
- **Telemetry** in native monthly partitions + pg_partman; partition create/prune via a JobRunr job (ADR-011 — Cloud SQL omits pg_partman's background worker).
- **Memorystore Redis** HA for the availability cache (per-vehicle, next 90 days), `vehicle_live` cache, rate-limit counters, and JobRunr/queue support.
- **GCS buckets:** `miqwad-media-{env}` (inspection photos, vehicle images — private, signed URLs, ≥ 24-month retention via lifecycle), `miqwad-docs-{env}` (contracts/invoices/KYC — CMEK, restricted), `miqwad-imports-{env}` (uploads, short TTL). Versioned; cross-zone durable by default.
- **Encryption:** Google-managed at rest everywhere; **CMEK** (Cloud KMS) for the docs/KYC bucket and the gov-credential secrets; KYC ID numbers additionally column-encrypted (envelope, KMS) before storage.

## 6. Security architecture (cloud layer) ✅

- **IAM:** one **service account per service** with least privilege (e.g. `miqwad-api` reads its secrets + read/writes its buckets + connects to SQL; `miqwad-jobs` similar; neither has project-admin).
- **Workload Identity** for Cloud Run → Google APIs (no key files); **Workload Identity Federation** for GitHub Actions → GCP (no long-lived CI keys).
- **Dealer gov-credential vault** in Secret Manager, CMEK-encrypted, access-audited; the app fetches at call time, never logs, never stores plaintext in Postgres.
- **Cloud Armor** WAF + per-IP rate limits on auth/search/booking; **Cloud Audit Logs** on admin + vault access; **Security Command Center** for posture.
- **TLS 1.2+** everywhere; HSTS at the LB; private east-west traffic.

## 7. Environments ✅

Three **isolated GCP projects** (`miqwad-staging`, `miqwad-prod`, plus `miqwad-shared` for Artifact Registry/CI), each its own VPC, Cloud SQL, secrets, buckets. **Local** = developer machines + Testcontainers (real Postgres/Redis) + WireMock for gov sandboxes. Staging is mandatory (the four government integrations can't be tested in prod). Promotion local → staging → prod by manual approval; gov sandbox credentials in staging, production credentials only in prod (per-env isolation).

## 8. Scaling & resilience ✅

- **Cloud Run** autoscaling on concurrency; `miqwad-api` min-instances ≥ 2 (multi-zone, no cold-start on the booking path); `miqwad-jobs` min-instances ≥ 1 (JobRunr workers). Sized to the V1 envelope (150 dealers / 8,000 vehicles / 1,500 bookings/day / ~270 telemetry pings/s).
- **Cloud SQL** HA failover; read replica for reporting; connection pooling per ADR-002 (§5).
- **Graceful degradation:** if notifications/analytics are down, the booking/handover/return path still works ([reliability-dr.md](reliability-dr.md)).
- **DR:** PITR + tested restore runbook; GCS versioning protects media/docs ([reliability-dr.md](reliability-dr.md)).

## 9. Key data-flow paths ✅

1. **Booking-confirm (saga):** app → LB → `miqwad-api` (reserve → Moyasar hosted payment) → DB commit + `outbox` row → `miqwad-jobs`/JobRunr dispatches Tajeer registration → on success `confirmed`; on Tajeer failure, compensation refunds (deposit is charged up front, never held). Payment webhooks hit `miqwad-api` directly (signature-verified, idempotent).
2. **Telemetry ingest:** device → LB (rate-limited path) → `miqwad-api` (validate/dedup) → partitioned `telemetry_ping` + update `vehicle.live_geo` + Redis `vehicle_live`. Live map reads from Redis; history from partitions (PostGIS).
3. **Media upload:** app requests a **V4 signed URL** from `miqwad-api` → uploads **directly to GCS** (the API never in the byte path) → metadata row persisted. Inspection photos carry geo/timestamp/hash.
4. **Reporting:** dealer report → `miqwad-api` queries the **read replica** → renders PDF/Excel; never touches the primary.
