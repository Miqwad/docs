# Team Plan — Frontend (sole frontend engineer)

**Role.** Customer RN app + dealer/admin web + every flow across all three surfaces, **and the UI itself — there is no separate designer; the FE engineer works as a design-engineer.** Works from the typed API client generated from the OpenAPI spec. The leverage that makes one builder viable across three surfaces is the **shared `@miqwad/design-system`** — **Style Dictionary** tokens (web CSS vars + RN JS theme) and a **Storybook** component library — **built once and reused across all three surfaces** (Pine/Brass/Sand, IBM Plex Sans Arabic + Sora). **Full RTL + a11y are non-negotiable** and ship in every component. **Admin is kept deliberately thin**, composing the shared library rather than growing its own UI.

> **This is the plan's #1 capacity risk** (one builder, three surfaces, no separate designer) — mitigated precisely by the **build-once design system** (the multiplier), the thin admin, **dealer-OS surfaces first**, ruthless reuse, and the generated API client (never blocked on contracts). Feature scope and the 3-month target are kept.

**Weeks map to sprints:** W1–2 = S1 (M1) · W3–4 = S2 · W5–6 = S3 (**M2 FE**) · W7–8 = S4 · W9–10 = S5 (**M3**) · W11–12 = S6 (**M4**).

## Prep
- **Stand up `@miqwad/design-system`:** Style Dictionary tokens (web + RN) + the core Storybook components (Button, Input, Select, Table, Card, StatusChip, Money, DateRange, Modal, …), each shipping hover/focus/active + loading/empty/error + RTL/a11y states. Grow the library as surfaces need it.
- Boot the three shells: RN (Expo Router) customer app + dealer (web) + admin (web) with **auth + package-gated navigation + full RTL**.
- TanStack Query + Zustand; a typed API client generated from the OpenAPI spec (two small adapters: REST + Elide JSON:API).
- Maintain the frontend architecture spec: [design/frontend-design-system.md](../../design/frontend-design-system.md).

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

*Reuse relentlessly; every screen composes `@miqwad/design-system` components. Pairs with: Mid/Senior BE (API contracts, sagas, marketplace). Detail in [design/frontend-design-system.md](../../design/frontend-design-system.md); release/forced-update policy in its React Native section and [engineering/devops.md](../../engineering/devops.md).*
