# Miqwad — Frontend Architecture & Design System

How the three frontends are built and how they share one design system, so a senior frontend + a UI/UX design-engineer can deliver the customer app, dealer portal, and admin portal. Brand tokens come from [brand.md](brand.md).

> **Status:** The customer app stack (React Native + Expo) and the shared design system are **✅ Decided**. The **web-portal framework (dealer + admin) is 🔵 Open** — see [§web-portal framework](#web-portal-framework--open) below, [../STATUS.md](../STATUS.md) **O-1**, and [../decisions/adr-log.md](../decisions/adr-log.md) **ADR-022**.

---

## Surfaces & stacks

| Surface | Stack | Status | Notes |
|---|---|---|---|
| **Customer app** | **React Native + Expo** (Expo Router) | ✅ Decided | iOS+Android one codebase; camera/GPS for handover; EAS builds + OTA |
| **Dealer portal** | Web (framework 🔵 Open) | 🔵 Open | Responsive web; the operating-system UI |
| **Admin portal** | Web (framework 🔵 Open) | 🔵 Open | Deliberately **thin**; reuses the dealer component library |

All three are **Arabic-first, RTL by default**, bilingual AR/EN.

---

## Web-portal framework — 🔵 Open

The dealer and admin **web** portals run on a framework **not yet chosen between React (SPA) and Next.js**. This is a genuine open decision — do not assume an answer.

| Option | For | Against |
|---|---|---|
| **React (SPA)** + Vite + React Router | Simplest model for purely authenticated app shells; no SSR machinery; fastest to stand up | No server-side rendering / SEO for any public-facing dealer pages |
| **Next.js** | SSR/SEO for public dealer pages; server actions; file-based routing | More framework surface and build complexity than an app behind a login needs |

**Decision criteria (the spike):** do any web surfaces need SSR/SEO or server actions? If **yes → Next.js**; if **purely authenticated app shells → React SPA**. Resolve with a short spike **before** frontend scaffolding.

**Scope of the choice:** it is isolated to the web portals. It does **not** affect the customer app, the design tokens, or the API. The **admin portal is deliberately thin** and reuses the dealer component library regardless of the outcome.

> Tracked in [../STATUS.md](../STATUS.md) **O-1** and [../decisions/adr-log.md](../decisions/adr-log.md) **ADR-022**.

---

## Shared design system — ✅ Decided (the multiplier)

One component library across all surfaces is what lets ~2 effective builders cover three apps. The UI/UX person works as a **design-engineer**, building these as production components used by every surface.

- **Tokens** — **Style Dictionary**, generated from [brand.md](brand.md) → CSS variables (web) + a JS theme (RN): Pine `#0F3D33`, Deep Ink, Brass `#C8A24C`, Sand Gold, Paper `#F6F2E9`, Bone; status Go/Stop/Warn; type scale; spacing; radii; durations.
- **Fonts** — IBM Plex Sans Arabic (UI/body), Sora (Latin display + tabular numerals for prices/odometer).
- **Component library in Storybook** — Button, Input/Form fields, Select, Table, Card, StatusChip (available/rented/maintenance), **Money**, DateRange, Modal, Drawer, Tabs, Toast, EmptyState, Skeleton, PhotoCapture, Map. Every component ships **hover/focus/active**, **loading/empty/error**, and **RTL + a11y** states.

---

## State management — ✅ Decided (don't mix concerns)

| Concern | Tool |
|---|---|
| Server data | **TanStack Query** — fetch/cache/refetch; optimistic updates for booking actions with rollback |
| Client/UI state | **Zustand** — wizards, filters-in-progress, ephemeral UI |
| URL state | Route + search params — filters, sort, pagination, active tab (shareable) |
| Forms | React Hook Form + schema validation |

**Rule:** never duplicate server state into Zustand — derive instead.

---

## API client

- A **typed client generated from the OpenAPI contract** for the REST domain endpoints, plus a thin **JSON:API adapter** for the Elide CRUD entities. Both attach the bearer token + `Accept-Language`; money is rendered via the **`Money`** component (halalas → SAR, tabular figures).
- Errors handled by `code`: `entitlement_limit_reached` → upgrade CTA; `vehicle_unavailable` → re-search; `dependency_unavailable` → retry affordance.

---

## Package-gated navigation (entitlements in the UI)

The dealer's resolved entitlement (`GET /dealer/subscription`) drives navigation: modules the package/tier doesn't include are **hidden**, and limit-bound actions show an upgrade prompt **before** the call. **The server remains authoritative** — the FE never enforces; it just avoids dead-ends.

- Marketplace-only dealer → lean fulfillment UI.
- Management-only dealer → never sees marketplace screens.
- "Both" → sees everything.

---

## Information architecture (per surface)

- **Customer app:** Onboarding/OTP → Search (city/date + filters + map) → Listing + itemized quote → Book + pay → Booking detail (status, handover photos, contract, invoice, refund) → History → Reviews.
- **Dealer portal:** Dashboard (fleet counts, today's bookings, pending confirmations, overdue maintenance, utilization) → Fleet → Availability → Bookings (incl. **external/walk-in entry**) → Handover/Return capture → Maintenance → Tracking (live map) → Delivery → Finance (invoices/payouts/subscription) → Reports → Reviews → Violations → Notifications → Connections wizard → Import.
- **Admin portal:** Dealership approvals + package assignment → Plans → Entitlement view → KPIs.

---

## React Native specifics (customer app)

- **Expo Router** file-based navigation; **expo-camera** + **expo-location** + **expo-image-manipulator** for inspection photos — geotagged, timestamped, perceptual-hashed at capture (the trust engine).
- **Offline resilience** on handover/return capture: photos queue locally and upload via GCS signed URL when back online.
- **MapLibre / react-native-maps** for the pickup map.
- **EAS** for builds + store submission; **OTA** for JS-layer fixes; staged store releases.
- **Release & forced-update policy:** on launch the app checks its version against the versioned `/v1` API; below the **minimum-supported version** it forces an update. The API evolves additively, but a breaking change is gated by min-version — never pushed onto deployed old apps. OTA ships JS-only fixes; native changes go through the stores. Store-review lead time and **push-notification certs (FCM/APNs)** are managed as part of release ops.

---

## Maps

**MapLibre GL** (web) / map view (RN) for the dealer live fleet map (positions from Redis `vehicle_live`), the customer search map, and geofencing display. **Marker clustering** for large fleets. Open tiles — no Google Maps licensing.

---

## RTL & i18n

- `dir="rtl"` default; mirror directional UI (back arrows, drawers, progress); Latin terms (Miqwad, ZATCA, model names, plate Latin) inline LTR within Arabic.
- **react-i18next** with AR (default) + EN bundles; localized numerals, dates, currency.
- **Pricing is always itemized — rental + refundable deposit + VAT — a brand promise, enforced in the `Money`/`Quote` components.** (See [brand.md](brand.md).)

---

## Performance & accessibility

- **CWV targets:** LCP < 2.5s, INP < 200ms, CLS < 0.1. Code-split heavy routes; lazy-load maps/charts; images with explicit dimensions; compositor-friendly animation only.
- **WCAG 2.1 AA** for the customer app (contrast, focus, screen-reader labels); reduced-motion respected.
- Charts via **Recharts**, themed to the design tokens — data-viz is part of the design system, not an afterthought.

---

## Testing (frontend)

Visual regression (**Playwright** screenshots) at **320 / 375 / 768 / 1024 / 1440**, both languages/directions; component tests in **Storybook**; e2e on the booking happy path; automated **a11y** checks.

---

*Related: [brand.md](brand.md) · [../STATUS.md](../STATUS.md) (O-1) · [../decisions/adr-log.md](../decisions/adr-log.md) (ADR-022)*
