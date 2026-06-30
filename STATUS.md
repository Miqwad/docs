# STATUS — What Is Not Yet Final

The single register of everything in these docs that is **not settled**. The backend architecture and the core data/booking/money model are **Decided**; the items below are the deliberate exceptions. Nothing here blocks laying the stable foundation — each is isolated behind an interface or a clearly-marked section.

**Legend:** 🔵 **Open** = a decision not yet made · 🟡 **Provisional** = approach set, detail pending an external input · 🟣 **Follow-up** = decided, but needs an external verification before scaffolding.

_Last updated: 2026-06-30._

> **2026-06-30 milestone — the backend walking skeleton is built and pushed** ([github.com/Miqwad/backend](https://github.com/Miqwad/backend)): a Kotlin 2.3 / Spring Boot 4.1 / JDK 25 Spring Modulith monolith with the cross-cutting kernel + 24 bounded-context modules + a `fleet` vertical slice. Flyway **V1–V6** apply, `/actuator/health` is green, the **non-negotiable cross-tenant RLS isolation test passes** (repo + native SQL → zero rows), the ZATCA SDK signs on JDK 25, and **CI is green**. The whole build runs on JDK 25. The physical DB schema is therefore done; the prioritized build backlog remains the open pre-build deliverable.

> **Team capacity (current): 3 engineers, no dedicated UI/UX designer.** The near-term plan is therefore **backend- and dealer-OS-first** — the engineering-heavy compliance moat (fleet, availability, booking, Tajeer/ZATCA/Wasl, ledger/settlement) that the team can build without design input. The **design-led surfaces are deferred until design/frontend capacity is added**: the polished customer **marketplace app** (React Native/Expo) and the **shared design system** (Storybook / Style Dictionary — ADR-022). The **OpenAPI contract** ([api/openapi.yaml](api/openapi.yaml)) is the stable seam those frontends build against later; any dealer/admin surface needed sooner is a **functional, unstyled app-shell** against that API, not a design-led build. This makes **O-1 non-urgent** (decide the web-portal framework when a portal is actually needed) and defers the consumer-facing parts of **ADR-022**.

---

## 🔵 Open decisions (no answer yet — do not assume one)

| # | Decision | Why open | Decision criteria / next step | Appears in |
|---|---|---|---|---|
| O-1 | **Web-portal framework: React (SPA) vs Next.js** | Customer app (React Native/Expo) is decided; the dealer + admin **web** portals are not. SSR/SEO needs for any public dealer pages vs SPA simplicity is unresolved. | **Non-urgent given current capacity** (3 engineers, no UI/UX designer — see the Team-capacity note above). Defer until a web portal is actually built; then the spike is: do any web surfaces need SSR/SEO or server actions? If yes → Next.js; if purely authenticated app-shells → React SPA. | [design/frontend-design-system.md](design/frontend-design-system.md), [decisions/adr-log.md](decisions/adr-log.md) (ADR-022) |
| O-2 | **Auth identity provider: Keycloak vs GCIP** (no hybrid) | We host on GCP me-central2; GCIP (managed — native phone-OTP, SMS MFA, multi-tenancy) is a real alternative to self-hosted Keycloak. The auth *requirements* are decided; the *provider* is not. | Decisive: confirm GCIP stores identity PII **in-Kingdom** (me-central2, via CNTXT) — Keycloak self-hosted guarantees it. Then weigh RBAC depth (Keycloak turnkey vs GCIP app-side) + managed-vs-self-host/cost. Decide before auth scaffolding (S1). | [decisions/adr-log.md](decisions/adr-log.md) (ADR-021), [architecture/security.md](architecture/security.md) |

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
