# Decision Follow-ups — external verifications

These are **not open decisions** — the decisions are made (see [adr-log.md](adr-log.md)). They are external facts to confirm at scaffold time, each with a pre-agreed fallback so none blocks the decision. Mirrored in the project [STATUS register](../STATUS.md).

| # | Item | Decision it backs | Verify | Fallback if it fails |
|---|---|---|---|---|
| F-1 | **GCP me-central2 service availability** | ADR-010, ADR-021 | Confirm per-service availability (Cloud SQL, Memorystore, Cloud Run, GCS, KMS) in Dammam via the KSA reseller **CNTXT**. **Also carries the final me-central2 residency confirmation behind ADR-021** — CNTXT messaged 2026-07-01; **self-hosted Keycloak is the working auth default regardless of the reply** (GCIP was ruled out as not fully in-Kingdom). | Re-evaluate region/service mix while keeping in-Kingdom residency (PDPL) non-negotiable |
| F-2 | **Elide — Spring Boot 4 compatibility** | ADR-016 | 🟡 **Open (2026-07-01)** — latest Elide **7.1.17** still targets **Spring Boot 3.5** (no Boot-4 release; vendor in progress). Re-check before adopting. | CRUD surface uses the interim **Spring Data REST / plain Spring MVC** behind the same `/v1` base — the core domain never depends on Elide |
| F-3 | **Togglz — Spring Boot 4 compatibility** | ADR-013 | ✅ **Resolved 2026-07-01** — **Togglz 4.6.2** targets Spring Framework 7.0.2 (Boot 4), min Java 17. Pin `togglz-spring-boot-starter:4.6.2` (+ console starter). | _(own flag-table fallback no longer needed — adopt Togglz)_ |
| F-4 | **JobRunr — Boot 4 starter** | ADR-008 | ✅ **Resolved 2026-06-30** — `jobrunr-spring-boot-4-starter` 8.7.0 exists (Boot 4 + Jackson 3 supported since 8.3.0) | _(fallback no longer needed: JobRunr core + manual config)_ |
| F-5 | **Dependency BOM re-verification** | [engineering/stack.md](../engineering/stack.md) | The version matrix is a **2026-06-30 snapshot** — re-run latest-version + CVE checks at scaffold time | Bump to the patched/compatible versions current at scaffold time |

> When an item is confirmed, note the verified version/region here, flip any related 🟣 badge in [STATUS.md](../STATUS.md), and (if it changes a decision) update the ADR.
