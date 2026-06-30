# Miqwad — Team Roles (V1)

The team is **3 engineers (2 backend + 1 frontend) + the MBA/business owner**; **no dedicated UI/UX designer, QA, or DevOps** — the frontend engineer owns the UI and the shared design system, the Senior BE owns CI/CD + infra, everyone runs TDD, and the MBA runs UAT.

| Role | Focus | Week-by-week plan |
|---|---|---|
| **Senior Backend** | Infra + CI/CD + GCP + tenancy + money/ledger + sagas + perf/security; reviewer | [team/senior-backend.md](team/senior-backend.md) |
| **Mid Backend** | Deep domain owner (design author): crown-jewels + breadth + export pipeline | [team/mid-backend.md](team/mid-backend.md) |
| **Frontend** | All three surfaces + the shared **`@miqwad/design-system`** (Style Dictionary tokens + Storybook components); owns the UI as design-engineer (no separate designer) | [team/frontend.md](team/frontend.md) |
| **MBA / business** | Legal + government-credentials critical path + packaging + pilot onboarding + PM/UAT | [team/mba.md](team/mba.md) |

**Backend split** (so the two BE engineers rarely block each other): Senior BE = the cross-cutting / infra / money / saga **spine**; Mid BE = the domain **depth + breadth**. They pair only on the sagas and the money/billing ledger.

**Frontend reality** (the plan's #1 risk): one engineer for three surfaces with no separate designer — mitigated by the **build-once `@miqwad/design-system`** (the multiplier that lets one builder cover three apps), a thin admin, dealer-OS surfaces first, ruthless reuse, and the generated OpenAPI client. Full V1 scope + the 3-month target are kept.

> Companion docs: [project-plan.md](project-plan.md) (sprints, milestones, risks), [backlog.md](backlog.md) (epics → stories). Per-person **week-by-week** detail lives in [team/](team/). Weeks map to sprints: W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (M2) · W7–8 = S4 · W9–10 = S5 (M3) · W11–12 = S6 (M4).
