# Decision Follow-ups — external verifications

These are **not open decisions** — the decisions are made (see [adr-log.md](adr-log.md)). They are external facts to confirm at scaffold time, each with a pre-agreed fallback so none blocks the decision. Mirrored in the project [STATUS register](../STATUS.md).

| # | Item | Decision it backs | Verify | Fallback if it fails |
|---|---|---|---|---|
| F-1 | **GCP me-central2 service availability** | ADR-010 | Confirm per-service availability (Cloud SQL, Memorystore, Cloud Run, GCS, KMS) in Dammam via the KSA reseller **CNTXT** | Re-evaluate region/service mix while keeping in-Kingdom residency (PDPL) non-negotiable |
| F-2 | **Elide — Spring Boot 4 compatibility** | ADR-016 | Confirm a Boot-4-compatible Elide release exists at scaffold time | CRUD surface falls back to **Spring Data REST / plain Spring MVC** behind the same `/v1` base — the core domain never depends on Elide |
| F-3 | **Togglz — Spring Boot 4 compatibility** | ADR-013 | Confirm a Boot-4-compatible Togglz release | Own boolean **flag table** behind the same gating interface (flags are not on the critical path) |
| F-4 | **JobRunr — Boot 4 starter** | ADR-008 | ✅ **Resolved 2026-06-30** — `jobrunr-spring-boot-4-starter` 8.7.0 exists (Boot 4 + Jackson 3 supported since 8.3.0) | _(fallback no longer needed: JobRunr core + manual config)_ |
| F-5 | **Dependency BOM re-verification** | [engineering/stack.md](../engineering/stack.md) | The version matrix is a **2026-06-30 snapshot** — re-run latest-version + CVE checks at scaffold time | Bump to the patched/compatible versions current at scaffold time |

> When an item is confirmed, note the verified version/region here, flip any related 🟣 badge in [STATUS.md](../STATUS.md), and (if it changes a decision) update the ADR.
