# Miqwad — Engineering Documentation

**مِقْوَد** — the two-sided Saudi car-rental platform: a multi-tenant SaaS that runs a rental dealership's whole business (fleet, bookings, Tajeer contracts, ZATCA invoicing, Wasl tracking, payments/settlement) **and** a customer marketplace that fills those cars.

This `docs/` tree is the **canonical, build-facing engineering documentation** — the single source of truth for the team. It is condensed from the original design package; decided things are stated plainly, and anything not yet settled is marked **Provisional** or **Open** and tracked in [STATUS.md](STATUS.md).

> Start here: read [00-overview.md](00-overview.md), then [architecture/README.md](architecture/README.md) for the invariants, then your role path below.

---

## Status legend

Every decision and spec carries one of these:

| Badge | Meaning |
|---|---|
| ✅ **Decided** | Settled. Build to it. |
| 🟡 **Provisional** | Approach is set, but detail will be refined once an external input arrives (vendor API docs, or dealer discovery interviews). Build behind the stable interface; expect the detail to firm up. |
| 🔵 **Open** | A genuine decision not yet made (e.g. web-portal framework). Do **not** assume an answer. Tracked in [STATUS.md](STATUS.md). |

The backend stack is **Decided**; the external-integration contracts and several business-flow details are **Provisional**; the web-portal framework is **Open**. See [STATUS.md](STATUS.md) for the full register of what is not yet final.

---

## Map

```
docs/
  00-overview.md              what Miqwad is · the product · status · the invariants
  STATUS.md                   register of every Provisional / Open item

  architecture/               how the system is built
    README.md                 architecture overview + the non-negotiable invariants
    data-model.md             entities · multi-tenancy · money · physical schema
    integrations.md           Tajeer · ZATCA · Wasl · Absher · Moyasar  🟡
    sagas-outbox-jobs.md      booking/return state machines · outbox · jobs
    security.md               authn/authz · tenant isolation · credential vault
    reliability-dr.md         observability · resilience · disaster recovery
    performance.md            SLOs · caching · pooling · autoscaling
    fraud-trust-safety.md     threat catalogue · controls
    cloud.md                  GCP me-central2 topology · service map

  api/
    openapi.yaml              the machine-readable contract  🟡 (integration paths)
    reference.md              human companion + conventions + error catalog

  product/
    requirements.md           business flow · MoSCoW · packaging · channels  🟡
    use-cases.md              the end-to-end flows  🟡
    pricing-packaging.md      packages · tiers · commission · entitlements

  design/
    brand.md                  palette · type · voice · tokens
    frontend-design-system.md FE architecture + shared design system  🔵 (web framework)

  engineering/
    stack.md                  the decided stack + versioned dependency matrix (BOM)
    devops.md                 environments · CI/CD · secrets
    testing.md                test pyramid · non-negotiable tests · coverage
    data-migration.md         historical import · gov-doc guard · cutover
    onboarding.md             learning guide per library

  decisions/
    README.md                 ADR index + status legend
    adr-log.md                the architecture decision records
    follow-ups.md             external-verification items (GCP / Boot-4 readiness)

  delivery/
    project-plan.md           prep → sprints → milestones → risks
    backlog.md                epics → stories → acceptance criteria
    roles.md                  team split (2 BE + 1 FE + MBA) + index
    team/                     per-person week-by-week plans

  diagrams/                   Mermaid sources
```

---

## Reading paths by role

- **New joiner (any role):** [00-overview.md](00-overview.md) → [architecture/README.md](architecture/README.md) → [engineering/onboarding.md](engineering/onboarding.md) → [STATUS.md](STATUS.md).
- **Backend engineer:** [architecture/README.md](architecture/README.md) → [data-model.md](architecture/data-model.md) → [api/reference.md](api/reference.md) → [sagas-outbox-jobs.md](architecture/sagas-outbox-jobs.md) → [integrations.md](architecture/integrations.md) → [engineering/stack.md](engineering/stack.md) → [decisions/adr-log.md](decisions/adr-log.md).
- **Frontend engineer (sole; owns the UI):** [delivery/team/frontend.md](delivery/team/frontend.md) → [api/reference.md](api/reference.md) → [product/use-cases.md](product/use-cases.md) → [design/brand.md](design/brand.md) (tokens). *(No designer: build on an off-the-shelf component kit; bespoke design system deferred. Web-portal framework is low-stakes 🔵 Open — see [STATUS.md](STATUS.md).)*
- **Product / PM:** [product/requirements.md](product/requirements.md) → [product/use-cases.md](product/use-cases.md) → [product/pricing-packaging.md](product/pricing-packaging.md) → [delivery/](delivery/).
- **Compliance / integrations:** [architecture/integrations.md](architecture/integrations.md) → [architecture/security.md](architecture/security.md) → [STATUS.md](STATUS.md) (provisional vendor contracts).

---

## Conventions

- **Money** is integer **halalas** (1 SAR = 100 halalas) + explicit currency — never floats.
- **Timestamps** are RFC3339 UTC.
- **Tenancy:** every tenant-owned row carries `dealership_id`, always from the JWT claim — never request input.
- Documentation is **Arabic-first product, English-first engineering docs**; UI is full RTL.

> The original design package (the numbered `1 - …` … `8 - …` folders and `6 - Editable Sources/`) is retained as a **historical snapshot** and is no longer the source of truth for engineering. Anything intentionally not carried here (the stack bake-off, market research, the validation kit) still lives there.
