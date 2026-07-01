# Miqwad — Team Roles (V1)

The team is **3 engineers (2 backend + 1 frontend) + the MBA/business owner**; **no dedicated UI/UX designer, QA, or DevOps** — the frontend engineer owns the UI (off-the-shelf component kit), the Senior BE owns CI/CD + infra, everyone runs TDD, and the MBA runs UAT. **Time is not the constraint** (AI-assisted team): full V1 scope stays and is **sequenced across the plan**, not trimmed to headcount.

| Role | Focus | Week-by-week plan |
|---|---|---|
| **Senior Backend** | Infra + CI/CD + GCP + tenancy + money/ledger + sagas + perf/security + distributed tracing; reviewer | [team/senior-backend.md](team/senior-backend.md) |
| **Mid Backend** | Deep domain owner (design author): crown-jewels + breadth + export pipeline | [team/mid-backend.md](team/mid-backend.md) |
| **Frontend** | All three app surfaces (customer RN incl. discovery feed + KYC, dealer, admin) on an off-the-shelf component kit + minimal brand tokens; owns the UI (no separate designer); bespoke design system deferred; light FE support on the landing site | [team/frontend.md](team/frontend.md) |
| **MBA / business** | Legal + government-credentials critical path + packaging + pilot onboarding + PM/UAT + **owns the launch landing-site content/marketing** | [team/mba.md](team/mba.md) |

**Backend split** (so the two BE engineers rarely block each other): Senior BE = the cross-cutting / infra / money / saga **spine**; Mid BE = the domain **depth + breadth**. They pair only on the sagas and the money/billing ledger.

**Frontend reality** (sequencing, not a capacity ceiling): one engineer for three app surfaces + feed/KYC screens with no designer — the FE stream is **sequenced from W1 across the whole plan** (not piled into the final weeks), on the off-the-shelf kit, a thin admin, dealer-OS surfaces first, ruthless reuse, and the generated OpenAPI client. **Full V1 scope is kept**; only bespoke design-system polish is deferred to post-V1.

**Landing site (launch deliverable).** A small informal marketing site (what Miqwad is, both sides of the platform, the offering) ships for launch as a simple static/Next.js site — **MBA-owned content/marketing** with **light FE support** for build/deploy. A fuller customer web app comes later (V1.5+).

**Resolved framework/auth decisions:** **O-1** — dealer/admin portals on React (Vite) SPA + public/SEO landing site on **Next.js** (ADR-022); **O-2** — self-hosted **Keycloak** in me-central2 (ADR-021).

> Companion docs: [project-plan.md](project-plan.md) (sprints, milestones, risks), [backlog.md](backlog.md) (epics → stories). Per-person **week-by-week** detail lives in [team/](team/). Weeks map to sprints: W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (M2) · W7–8 = S4 · W9–10 = S5 (M3) · W11–12 = S6 (M4).
