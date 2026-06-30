# STATUS — What Is Not Yet Final

The single register of everything in these docs that is **not settled**. The backend architecture and the core data/booking/money model are **Decided**; the items below are the deliberate exceptions. Nothing here blocks laying the stable foundation — each is isolated behind an interface or a clearly-marked section.

**Legend:** 🔵 **Open** = a decision not yet made · 🟡 **Provisional** = approach set, detail pending an external input · 🟣 **Follow-up** = decided, but needs an external verification before scaffolding.

_Last updated: 2026-06-30._

---

## 🔵 Open decisions (no answer yet — do not assume one)

| # | Decision | Why open | Decision criteria / next step | Appears in |
|---|---|---|---|---|
| O-1 | **Web-portal framework: React (SPA) vs Next.js** | Customer app (React Native/Expo) is decided; the dealer + admin **web** portals are not. SSR/SEO needs for any public dealer pages vs SPA simplicity is unresolved. | Short spike: do any web surfaces need SSR/SEO or server actions? If yes → Next.js; if purely authenticated app shells → React SPA. Decide before frontend scaffolding. | [design/frontend-design-system.md](design/frontend-design-system.md), [decisions/adr-log.md](decisions/adr-log.md) (ADR-022) |
| O-2 | **Auth identity provider: Keycloak vs GCIP** (no hybrid) | We host on GCP me-central2; GCIP (managed — native phone-OTP, SMS MFA, multi-tenancy) is a real alternative to self-hosted Keycloak. The auth *requirements* are decided; the *provider* is not. | Decisive: confirm GCIP stores identity PII **in-Kingdom** (me-central2, via CNTXT) — Keycloak self-hosted guarantees it. Then weigh RBAC depth (Keycloak turnkey vs GCIP app-side) + managed-vs-self-host/cost. Decide before auth scaffolding (S1). | [decisions/adr-log.md](decisions/adr-log.md) (ADR-021), [architecture/security.md](architecture/security.md) |

---

## 🟡 Provisional — pending **vendor API documentation**

We do not yet hold the official integration docs/credentials for the external systems. The **adapter approach is decided** (provider-agnostic ports behind circuit breakers); the **wire-level contracts are not**. Build the ports; refine the DTOs when the vendor docs + credential onboarding land. Government-credential applications are on the real-world critical path — start them now.

| # | Integration | What's provisional | Unblocked by |
|---|---|---|---|
| P-1 | **Tajeer** (e-contract) | Request/response fields, contract lifecycle states, error codes | Tajeer onboarding + API docs |
| P-2 | **ZATCA** Phase 2 (e-invoice) | Exact XML/UBL profile fields, clearance/reporting responses, sandbox specifics | Official ZATCA SDK + portal onboarding |
| P-3 | **Wasl** (fleet tracking) | Telemetry ingest contract, registration payloads | Wasl onboarding + API docs |
| P-4 | **Absher** (identity / e-sign) | Identity verification flow + callbacks | Absher/Tajeer onboarding |
| P-5 | **Moyasar** (payments) | Webhook payloads, refund/void/capture response shapes, deposit-hold semantics | Moyasar account + API docs |
| P-6 | **Email / SMS / push** | Provider selection (e.g. Unifonic for SMS in KSA) + send/delivery contracts | Provider selection |

These are flagged with a 🟡 banner in [architecture/integrations.md](architecture/integrations.md) and with `#`/`description` notes in [api/openapi.yaml](api/openapi.yaml).

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
| F-5 | **Dependency BOM re-verification** | The version matrix in [engineering/stack.md](engineering/stack.md) is a 2026-06-30 snapshot — re-run version + CVE checks at scaffold time | [engineering/stack.md](engineering/stack.md) |

---

_When an item is resolved, update its row here, flip the badge in the referencing doc, and (for decisions) record it as an Accepted ADR._
