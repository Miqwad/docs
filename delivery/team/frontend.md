# Team Plan — Frontend (sole frontend engineer)

**Role.** Customer RN app (incl. discovery feed + KYC/license capture) + dealer/admin web + every flow across all three app surfaces, **and the UI itself — there is no separate designer.** Works from the typed API client generated from the OpenAPI spec. The dealer/admin portals are React (Vite) SPAs; the public/SEO **landing site is Next.js** (built/deployed with light FE support, content owned by the MBA — *O-1 resolved, ADR-022*). To make one builder viable across the surfaces, the app UI is an **off-the-shelf component library** (a ready-made React + React-Native kit) themed with **minimal brand tokens** (Pine/Brass/Sand, IBM Plex Sans Arabic + Sora) — **not** a bespoke Storybook/Style-Dictionary design system, which is deferred to post-V1. Utilitarian-but-usable beats polished-but-late; **full RTL + basic a11y are non-negotiable**; the kit's built-in states cover the rest. **Admin is kept deliberately thin.**

> **Time is not the constraint** (AI-assisted). This is a **sequencing** plan, not a capacity ceiling: the FE stream is spread **from W1 across the whole timeline** — not piled into the final weeks — protected by the kit, the thin admin, **dealer-OS surfaces first**, ruthless reuse, and the generated API client (never blocked on contracts). **Full feature scope is kept** (customer app + dealer + admin + discovery feed + KYC + reviews + push); only bespoke design-system polish is deferred.

**Weeks map to sprints:** W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (**M2 FE**) · W7–8 = S4 · W9–10 = S5 (**M3**) · W11–12 = S6 (**M4**).

## Prep
- **Pick the component kit** and wire **minimal brand tokens** (web + RN); no bespoke design system.
- Boot the three app shells: RN (Expo Router) customer app + dealer (React/Vite SPA) + admin (React/Vite SPA) with **auth (Keycloak / customer OTP) + package-gated navigation + full RTL**.
- Stub the **Next.js landing site** shell (reuses brand tokens; content lands later with the MBA).
- TanStack Query + Zustand; a typed API client generated from the OpenAPI spec (two small adapters: REST + Elide JSON:API).
- Co-author doc **18 (frontend architecture)**.

## Weeks

Sequenced across the whole plan (not back-loaded). Milestone anchors: M1 W2 · M2 W6 · M3 W10 · M4 W12.

- **W1:** dealer auth (Keycloak) + dashboard shell + **dealer Today / Action-Inbox** stub + admin approval/package-assignment + **admin suspend/reject** control.
- **W2:** fleet screens (vehicle CRUD / images / rate plans). → **M1**.
- **W3:** dealer availability calendar + gated nav; **customer app: anonymous/dateless search + discovery feed shell** (organic ranking; promotion-weight hook stubbed) — brought forward so the customer surface streams early.
- **W4:** dealer booking entry (all channels) + quote; **customer: search results / listing detail + reviews display**.
- **W5:** dealer confirm + Tajeer status + payment wiring; **customer: KYC / license-capture screen** (fields + upload) wired to the KYC endpoint.
- **W6:** customer app: book-a-vehicle + Moyasar pay (**deposit charged up front**) + **`/meta` version gate**. → **M2 (FE side)**.
- **W7:** handover/return capture (camera / geotag / checklist) + damage.
- **W8:** customer booking detail (photos / contract / invoice / refund) + **cancellation / no-show / refund UI** (policy-driven).
- **W9:** customer marketplace filters/sort polish + **reviews submit** (post-completion) + **push-notification client + device-token registration**.
- **W10:** maintenance + delivery + one-way + finance/settlement + plan/billing screens *(live operator tracking map → V1.5)*. → **M3**.
- **W11:** import wizard UI (**active data first; historical last**) + notifications center + connection wizard + package/tier UI + admin KPIs.
- **W12:** **landing-site build/deploy (light FE, MBA content)** + cross-surface cleanup, a11y/RTL pass, offline, EAS build/submit + forced-update policy. → **M4**.

*Reuse relentlessly; lean on the kit's components and states. Pairs with: Mid/Senior BE (API contracts, sagas, marketplace, KYC), MBA (landing-site content). Detail in doc 18; release/forced-update policy in docs 18 §7 / 19 §11.*
