# Miqwad — Team Roles (V1)

The team is **3 engineers (2 backend + 1 frontend) + the MBA/business owner**; **no dedicated UI/UX designer, QA, or DevOps** — the frontend engineer owns the UI (off-the-shelf component kit), the Senior BE owns CI/CD + infra, everyone runs TDD, and the MBA runs UAT.

| Role | Focus | Week-by-week plan |
|---|---|---|
| **Senior Backend** | Infra + CI/CD + GCP + tenancy + money/ledger + sagas + perf/security; reviewer | [team/senior-backend.md](team/senior-backend.md) |
| **Mid Backend** | Deep domain owner (design author): crown-jewels + breadth + export pipeline | [team/mid-backend.md](team/mid-backend.md) |
| **Frontend** | All three surfaces on an off-the-shelf component kit + minimal brand tokens; owns the UI (no separate designer); bespoke design system deferred | [team/frontend.md](team/frontend.md) |
| **MBA / business** | Legal + government-credentials critical path + packaging + pilot onboarding + PM/UAT | [team/mba.md](team/mba.md) |

**Backend split** (so the two BE engineers rarely block each other): Senior BE = the cross-cutting / infra / money / saga **spine**; Mid BE = the domain **depth + breadth**. They pair only on the sagas and the money/billing ledger.

**Frontend reality** (the plan's #1 risk): one engineer for three surfaces with no designer — protected by the off-the-shelf kit, a thin admin, dealer-OS surfaces first, ruthless reuse, and the generated OpenAPI client. Full V1 scope + the 3-month target are kept; design-system polish is the deferred sacrifice.

> Companion docs: [project-plan.md](project-plan.md) (sprints, milestones, risks), [backlog.md](backlog.md) (epics → stories). Per-person **week-by-week** detail lives in [team/](team/). Weeks map to sprints: W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (M2) · W7–8 = S4 · W9–10 = S5 (M3) · W11–12 = S6 (M4).
