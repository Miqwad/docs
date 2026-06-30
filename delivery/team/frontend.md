# Team Plan — Frontend (sole frontend engineer)

**Role.** Customer RN app + dealer/admin web + every flow across all three surfaces, **and the UI itself — there is no separate designer.** Works from the typed API client generated from the OpenAPI spec. To make one builder viable across three surfaces, the UI is an **off-the-shelf component library** (a ready-made React + React-Native kit) themed with **minimal brand tokens** (Pine/Brass/Sand, IBM Plex Sans Arabic + Sora) — **not** a bespoke Storybook/Style-Dictionary design system, which is deferred to post-V1. Utilitarian-but-usable beats polished-but-late; **full RTL + basic a11y are non-negotiable**; the kit's built-in states cover the rest. **Admin is kept deliberately thin.**

> **This is the plan's #1 capacity risk** (one builder, three surfaces, no designer) — protected by the kit, the thin admin, **dealer-OS surfaces first**, ruthless reuse, and the generated API client (never blocked on contracts). Feature scope is kept; design-system polish is the sacrifice.

**Weeks map to sprints:** W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (**M2 FE**) · W7–8 = S4 · W9–10 = S5 (**M3**) · W11–12 = S6 (**M4**).

## Prep
- **Pick the component kit** and wire **minimal brand tokens** (web + RN); no bespoke design system.
- Boot the three shells: RN (Expo Router) customer app + dealer (React) + admin (React) with **auth + package-gated navigation + full RTL**.
- TanStack Query + Zustand; a typed API client generated from the OpenAPI spec (two small adapters: REST + Elide JSON:API).
- Co-author doc **18 (frontend architecture)**.

## Weeks
- **W1:** dealer auth + dashboard shell + admin approval/package-assignment.
- **W2:** fleet screens (vehicle CRUD / images / rate plans). → **M1**.
- **W3:** dealer availability calendar + gated nav.
- **W4:** dealer booking entry (all channels) + quote.
- **W5:** dealer confirm + Tajeer status + payment wiring.
- **W6:** customer app shell + book-a-vehicle + Moyasar pay. → **M2 (FE side)**.
- **W7:** handover/return capture (camera / geotag / checklist) + damage.
- **W8:** customer booking detail (photos / contract / invoice / refund).
- **W9:** customer marketplace search / listing / filters / map (discovery).
- **W10:** tracking map (MapLibre) + maintenance + delivery + finance/settlement + plan/billing screens. → **M3**.
- **W11:** import wizard UI + notifications + reviews + connection wizard + package/tier UI.
- **W12:** cross-surface cleanup, a11y/RTL pass, offline, EAS build/submit + forced-update policy. → **M4**.

*Reuse relentlessly; lean on the kit's components and states. Pairs with: Mid/Senior BE (API contracts, sagas, marketplace). Detail in doc 18; release/forced-update policy in docs 18 §7 / 19 §11.*
