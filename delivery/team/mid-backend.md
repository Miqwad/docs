# Team Plan — Mid Backend

**Role.** Deep domain owner — the author of this design, with the deepest system knowledge. Takes the hard tasks (crown jewels) **and** carries breadth, consults the Senior BE on the hardest calls, and is on a growth path. Owns the **export pipeline** (O2) so the rendered docs stop going stale.

**Weeks map to sprints:** W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (**M2**) · W7–8 = S4 · W9–10 = S5 (**M3**) · W11–12 = S6 (**M4**).

## Prep
- Lead-author docs **12 (low-level design) / 13 (physical schema) / 14 (backlog) / 15 (sagas) / 22 (packaging, with MBA) / 23 (migration) / 26 (diagrams)**.
- Repo conventions: module layout, DTO/mapping reference (Kotlin extension functions / Konvert), the **export pipeline (O2)**.
- Co-design tenancy + entitlement with the Senior BE.

## Weeks
- **W1:** dealership onboarding + **entitlement model/data + package assignment** (co-own entitlement with Senior BE) + **admin dealership suspend/reject + `admin_action` platform audit trail**.
- **W2:** vehicle/fleet CRUD + docs/images + rate plans + categories + expiry alerts. → **M1**.
- **W3:** **availability core** + `tstzrange`+GiST exclusion constraint + Redis availability cache + idempotency store + **anonymous / dateless search** query path + **import module foundation** (staging + validation/quarantine, so active data can load early).
- **W4:** **booking core** (state machine, all channels) + minimal listing/quote→book + external/walk-in entry + **KYC / license-capture endpoint + fields**.
- **W5:** **booking-confirm saga** (payment + Tajeer via outbox/JobRunr, deposit charged up front) + channel-aware commission.
- **W6:** **ZATCA adapter** + invoice issue/clearance; booking-loop e2e. → **M2**.
- **W7:** trust core — inspections + geotagged photos + damage; return flow.
- **W8:** imported-doc guard + booking management + **cancellation / refund / no-show policy + endpoints** (policy-driven refund amounts).
- **W9:** **full marketplace search/listing/quote** + **customer discovery feed** with a **promotion-weight ranking hook (organic in V1)** + telemetry **ingest for Wasl reporting** (PostGIS) *(live operator map / geofencing → V1.5)* + `marketplace_compliance` flag.
- **W10:** **Wasl compliance reporting** + maintenance + delivery + one-way + **settlement/commission + subscription billing** (+ platform ZATCA invoice with Senior BE). → **M3**.
- **W11:** **reviews endpoint** + Saher + branch perms + consolidated reporting + admin KPIs + export pipeline; **import/migration: active data first**.
- **W12:** **import/migration full historical (sequenced last)** + reports/exports; coverage to target. → **M4**.

*Pairs with: Senior BE (tenancy, money, return-settle saga, billing ledger, admin/audit), Frontend (booking/confirm/marketplace/feed/KYC APIs), MBA (packaging, migration data, UAT). Detail in docs 12/13/14/15/16/22/23.*
