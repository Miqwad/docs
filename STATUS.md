# STATUS — What Is Not Yet Final

The single register of everything in these docs that is **not settled**. The backend architecture and the core data/booking/money model are **Decided**; the items below are the deliberate exceptions. Nothing here blocks laying the stable foundation — each is isolated behind an interface or a clearly-marked section.

**Legend:** 🔵 **Open** = a decision not yet made · 🟡 **Provisional** = approach set, detail pending an external input · 🟣 **Follow-up** = decided, but needs an external verification before scaffolding.

_Last updated: 2026-07-01._

> **2026-06-30 milestone — the backend walking skeleton is built and pushed** ([github.com/Miqwad/backend](https://github.com/Miqwad/backend)): a Kotlin 2.3 / Spring Boot 4.1 / JDK 25 Spring Modulith monolith with the cross-cutting kernel + 24 bounded-context modules + a `fleet` vertical slice. Flyway **V1–V6** apply, `/actuator/health` is green, the **non-negotiable cross-tenant RLS isolation test passes** (repo + native SQL → zero rows), the ZATCA SDK signs on JDK 25, and **CI is green**. The whole build runs on JDK 25. The physical DB schema is therefore done; the prioritized build backlog remains the open pre-build deliverable.

> **Team capacity (current): 3 engineers (2 backend + 1 frontend) + a business/MBA owner; no dedicated UI/UX designer.** V1 keeps **full scope on the 3-month target** (accepted as the top delivery risk — see [delivery/project-plan.md](delivery/project-plan.md)). The backend is split: **Senior BE** owns infra/CI-CD, tenancy, money, sagas, perf/security; **Mid BE** owns the deep domain (crown jewels) + breadth. With **one frontend engineer and no designer**, all surfaces (customer app + dealer/admin web) are built **functionally on an off-the-shelf RTL component kit themed with brand tokens** (from the brand doc), **not** a bespoke design system — the **Storybook / Style-Dictionary design system is the explicit sacrifice (deferred post-V1), not the app or its features**. Dealer-OS surfaces lead the build order; the **OpenAPI contract** ([api/openapi.yaml](api/openapi.yaml)) is the frontend's stable seam. **O-1 is now resolved** (2026-07-01): **React (Vite) SPA for the authenticated dealer/admin portals, Next.js confined to the thin public/SEO surface, Expo for the customer app** — see the audit dispositions below and **ADR-022**.

---

## ✅ Formerly-open decisions — now resolved (2026-07-01)

Both remaining 🔵 Open decisions are settled. No open decisions remain.

| # | Decision | Resolution | Appears in |
|---|---|---|---|
| O-1 ✅ | **Web-portal framework: React (SPA) vs Next.js** | **Resolved 2026-07-01 → both, split by surface:** **React (Vite) SPA** for the authenticated dealer & admin portals (behind login, no SEO need); **Next.js only for the thin public/SEO surface** (an informal landing site now, a customer web app later); **React Native/Expo** for the customer app. | [design/frontend-design-system.md](design/frontend-design-system.md), [decisions/adr-log.md](decisions/adr-log.md) (ADR-022) |
| O-2 ✅ | **Auth identity provider: Keycloak vs GCIP** (no hybrid) | **Resolved 2026-07-01 → self-hosted Keycloak in me-central2.** GCIP is **not fully in-Kingdom**, which fails PDPL identity-PII residency; Keycloak self-hosted keeps it in-Kingdom and ships turnkey RBAC. Status: **Accepted — pending final CNTXT residency confirmation** (CNTXT messaged 2026-07-01; Keycloak is the working default regardless of the reply). | [decisions/adr-log.md](decisions/adr-log.md) (ADR-021), [architecture/security.md](architecture/security.md) |

---

## Audit dispositions (2026-07-01)

Decisions closed out in the 2026-07-01 audit pass, kept tight (each already reflected in its owning doc / ADR):

- **Customer acquisition (web):** an **informal landing site now**; a full customer *web* app comes later (built on Next.js when warranted) — not V1 scope.
- **KYC:** customers **enter their ID / driving-licence fields** at booking now; **Absher verification is wired when its API docs arrive** (P-4) — no hard identity gate blocks V1.
- **Dealer onboarding:** a **self-service onboarding wizard**, with a **support-assisted fallback** for dealers who need it.
- **Live fleet tracking:** the real-time fleet map / live-position UI moves to **V1.5** (Wasl telemetry ingest + PostGIS position are still built in V1; the live consumer surface is deferred).
- **Reservation payment-hold:** a **short (~10 min) hold** on a reservation before payment, swept by a **reaper** job (JobRunr) that releases the availability block on expiry.
- **Historical import:** **stays in V1** but **sequenced last** — active/live data first, historical backfill after the live paths are proven.
- **Distributed tracing:** **stays in V1** (not deferred).
- **Wasl usage:** the Wasl integration is **pending confirmation that all dealers actually use it** before it becomes a hard dependency.
- **Future-proof hooks (build now):** land the low-cost seams during V1 — `vehicle.promotion_rank` (for later marketplace ranking/promotion) and the **hot-path indexes** — so later features don't require a migration on hot tables.
- **Tenant propagation:** carried via an **enforced coroutine `ThreadContextElement`** (not a raw `ThreadLocal`, and not `ScopedValue` alone) so `dealership_id` survives suspension/thread hops to both Hibernate's `@TenantId` resolver and the per-transaction RLS `SET LOCAL`.

---

## 🟡 Provisional — pending **vendor API documentation**

**ZATCA (P-2) and Moyasar (P-5) vendor docs were received 2026-06-30 and are now ✅ resolved** (rows below); **Tajeer (P-1), Wasl (P-3), Absher (P-4), and notifications (P-6) remain pending.** The **adapter approach is decided** (provider-agnostic ports behind circuit breakers); refine the still-pending DTOs when their vendor docs + credential onboarding land. Government-credential applications are on the real-world critical path — start them now.

| # | Integration | What's provisional | Unblocked by |
|---|---|---|---|
| P-1 | **Tajeer** (e-contract) | Request/response fields, contract lifecycle states, error codes | Tajeer onboarding + API docs |
| **P-2 ✅** | **ZATCA** Phase 2 (e-invoice) | ✅ **Resolved 2026-06-30** — SDK R4.0.0 + standards received; decided to **embed R4.0.0 in-process on JDK 25**; clearance (B2B std) / reporting (B2C simplified) via Fatoora v2; per-EGS CSID + a Miqwad platform EGS ([integrations.md](architecture/integrations.md), ADR-014) | Done — onboard **GA R4.x** + JDK-25 sign smoke test (**F-6**) |
| P-3 | **Wasl** (fleet tracking) | Telemetry ingest contract, registration payloads | Wasl onboarding + API docs |
| P-4 | **Absher** (identity / e-sign) | Identity verification flow + callbacks | Absher/Tajeer onboarding |
| **P-5 ✅** | **Moyasar** (payments) | ✅ **Resolved 2026-06-30** — API ref + Postman received; halalas 1:1, Mada via `creditcard`, hosted SDK (PCI-safe), webhooks via `secret_token` + re-fetch (idempotent per event id). **Deposit = charge up front, refund on return** (auth-holds too short); commission reversal is our ledger's job ([integrations.md](architecture/integrations.md), ADR-015) | Done — pull Moyasar Mada/decline test cards |
| P-6 | **Email / SMS / push** | Provider selection (e.g. Unifonic for SMS in KSA) + send/delivery contracts | Provider selection |

The still-pending ones (P-1/P-3/P-4/P-6) carry a 🟡 banner in [architecture/integrations.md](architecture/integrations.md); **P-2 and P-5 are now ✅ in hand** there with the real contracts.

---

## 🟡 Provisional — pending **dealer discovery interviews**

Business-flow detail that the current specs *assume* and that two dealer interviews (a big-dealer GM + an operational/branch manager) must confirm or correct. The discovery instrument is `6 - Editable Sources/32_Validation_Kit/09_Business_Flow_Discovery_AR_EN.md` (outside `docs/`).

| # | Area | What needs validation | Appears in |
|---|---|---|---|
| P-7 | **Booking → handover flow** | The real step order, who does what, required fields at each step, exceptions | [product/use-cases.md](product/use-cases.md), [product/requirements.md](product/requirements.md) |
| P-8 | **Handover / return inspection** | Photo/checklist practice, damage-dispute handling, deposit logic | [product/use-cases.md](product/use-cases.md) |
| P-9 | **Maintenance workflow** | How service is scheduled/recorded, what blocks availability | [product/requirements.md](product/requirements.md) |
| P-10 | **Pricing / rate plans** | Real-world pricing rules (seasonal, length-of-rental, deposits) | [product/pricing-packaging.md](product/pricing-packaging.md) |
| P-11 | **Channels** (walk-in / aggregator) | How non-marketplace bookings are taken and reconciled in practice | [product/use-cases.md](product/use-cases.md) |
| P-12 | **Data & documents recorded** | The exact data/documents a dealer keeps per rental (drives schema detail + migration) | [architecture/data-model.md](architecture/data-model.md), [engineering/data-migration.md](engineering/data-migration.md) |

---

## 🟣 Follow-ups — decided, but verify before scaffolding

| # | Item | Verification needed | Tracked in |
|---|---|---|---|
| F-1 | **GCP me-central2 (Dammam)** service availability | Confirm per-service availability via CNTXT (the KSA reseller) for residency | [decisions/follow-ups.md](decisions/follow-ups.md) |
| F-2 | **Boot-4 readiness: Elide** | Confirm a Spring-Boot-4-compatible Elide release at scaffold time; else fall back to Spring Data REST / plain MVC | [decisions/follow-ups.md](decisions/follow-ups.md) |
| F-3 | **Boot-4 readiness: Togglz** | Confirm Boot-4-compatible release; else fall back to an own flag table behind the same interface | [decisions/follow-ups.md](decisions/follow-ups.md) |
| F-4 | **Boot-4 readiness: JobRunr starter** | ✅ Resolved 2026-06-30 — `jobrunr-spring-boot-4-starter` 8.7.0 ships (Boot 4 + Jackson 3 since 8.3.0) | [decisions/follow-ups.md](decisions/follow-ups.md) |
| F-5 | **Dependency BOM re-verification** | ✅ **Resolved 2026-06-30 at scaffold** — full BOM verified building the skeleton (and CI-confirmed). Corrections vs the snapshot: **detekt is `dev.detekt` 2.0** (the `io.gitlab.arturbosch.detekt` id only ships 1.x; ktlint rules via `detekt-rules-ktlint-wrapper`), **Testcontainers 2.0** (`testcontainers-postgresql` / `-junit-jupiter`), MockMvc needs the new `spring-boot-starter-webmvc-test`. The whole build runs on **JDK 25**. | [engineering/stack.md](engineering/stack.md) |
| F-6 | **ZATCA SDK on JDK 25** | ✅ **Resolved 2026-06-30** — the in-process **sign/hash smoke test passes on JDK 25**; no Java-21-toolchain fallback needed. | [architecture/integrations.md](architecture/integrations.md), [decisions/adr-log.md](decisions/adr-log.md) (ADR-014) |
| F-7 | **ZATCA SDK packaging** (new, from scaffold) | The official R4.0.0 fat jar **bundles an entire Spring Framework** (~3,000 classes) + Jackson/Saxon/HttpClient, so it cannot sit on the app classpath (it shadows Spring 7). Before the adapter is built: **repackage/relocate it, or load it in an isolated child-first classloader** behind the `ZatcaSigner` port. In the skeleton it is compile-only + run by an isolated smoke test. | [architecture/integrations.md](architecture/integrations.md), [decisions/adr-log.md](decisions/adr-log.md) (ADR-014) |

---

_When an item is resolved, update its row here, flip the badge in the referencing doc, and (for decisions) record it as an Accepted ADR._
