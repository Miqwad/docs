# Miqwad — Frontend Architecture & Design System

How the three frontends are built and how they share one component system, so a **single frontend engineer (no separate designer)** can deliver the customer app, dealer portal, and admin portal. Brand tokens come from [brand.md](brand.md).

> **Status:** All three surface stacks are **✅ Decided** (O-1 **resolved 2026-07-01**): **React Native + Expo** for the customer app, **React (Vite) SPA** for the authenticated dealer & admin portals, and **Next.js** for the thin public/SEO surface — see [§web-portal framework](#web-portal-framework--decided) below, [../STATUS.md](../STATUS.md) **O-1**, and [../decisions/adr-log.md](../decisions/adr-log.md) **ADR-022**.

> **⚠️ V1 design-system scope — RESOLVED (deferral).** V1 ships on an **off-the-shelf, RTL-capable component kit** (a ready-made React + React-Native library) themed with **minimal Miqwad brand tokens** — because the team is **one frontend engineer with no dedicated designer** ([../STATUS.md](../STATUS.md) team-capacity note, [../delivery/team/frontend.md](../delivery/team/frontend.md)). The **bespoke Storybook + Style-Dictionary system and the ~16 hand-built components described below are DEFERRED to post-V1** — they are the explicit sacrifice, not the app or its features. Full **RTL + basic a11y are non-negotiable in V1**; the kit's built-in states cover the rest. Read the "Shared design system", "React Native specifics", and testing sections below as the **post-V1 target**, except where they state a V1 non-negotiable (RTL, a11y, the itemized-`Money` promise). (The "Maps" section is scoped separately by **product** release, not by this design-system deferral: the customer `StaticMapThumbnail` is **V1**; the dealer `FleetMap` on MapLibre is **V1.5**.)

---

## Surfaces & stacks

| Surface | Stack | Status | Notes |
|---|---|---|---|
| **Customer app** | **React Native + Expo** (Expo Router) | ✅ Decided | iOS+Android one codebase; camera/GPS for handover; EAS builds + OTA |
| **Dealer portal** | **React (Vite) SPA** (React Router) | ✅ Decided | Responsive web; the operating-system UI; behind login, no SEO need |
| **Admin portal** | **React (Vite) SPA** | ✅ Decided | Deliberately **thin**; reuses the dealer component library |
| **Public / SEO surface** | **Next.js** | ✅ Decided | Informal landing site now; customer web app later — **only** surface that needs SSR/SEO |

All three are **Arabic-first, RTL by default**, bilingual AR/EN.

---

## Web-portal framework — ✅ Decided

**Resolved 2026-07-01 (O-1):** the web surfaces split by rendering need. The **authenticated dealer & admin portals run on a React (Vite) SPA** (React Router); the **public/SEO surface runs on Next.js** — and only that surface.

| Surface | Framework | Why |
|---|---|---|
| **Dealer & admin portals** | **React (Vite) SPA** + React Router | Behind login, no SEO need; simplest model for authenticated app shells, no SSR machinery, fastest to stand up. The admin portal is deliberately thin and reuses the dealer component library. |
| **Public / SEO surface** | **Next.js** | The **only** surface that needs SSR/SEO/server rendering — an informal marketing/landing site now, a customer *web* app later when one is warranted. Next.js is **not** used for the portals. |

**Scope:** the split is isolated to the web surfaces. It does **not** affect the customer app (React Native/Expo), the design tokens, or the API.

> Tracked in [../STATUS.md](../STATUS.md) **O-1** (resolved) and [../decisions/adr-log.md](../decisions/adr-log.md) **ADR-022**.

---

## Shared component system

**V1 (decided):** one **off-the-shelf, RTL-capable component kit** (React + React-Native) themed with **minimal brand tokens** (Pine, Brass, Sand; IBM Plex Sans Arabic + Sora) is what lets **one frontend engineer** cover three apps. The kit's built-in states (hover/focus/active, loading/empty/error, RTL + a11y) carry the design work; there is **no dedicated designer**. The FE selects the kit in prep and wires the tokens below into it.

**Post-V1 (deferred — the multiplier):** replace the kit with a **bespoke component library** owned in-house, where each item is a production component used by every surface. Deferred because it needs design-engineering capacity the V1 team does not have.

- **Tokens (V1: minimal set into the kit; post-V1: full Style Dictionary)** — generated from [brand.md](brand.md) → CSS variables (web) + a JS theme (RN): Pine `#0F3D33`, Deep Ink `#0B1F1A`, Brass `#C8A24C`, Sand Gold, Paper `#F6F2E9`, Bone; status Go/Stop/Warn (subject to the contrast rules in §Performance & accessibility); `--color-on-accent`; type scale; spacing; radii; durations.
- **Fonts** — IBM Plex Sans Arabic (UI/body), Sora (Latin display + tabular numerals for prices/odometer).
- **Component set (post-V1 bespoke build, ~16 components)** — Button, Input/Form fields, Select, Table, Card, StatusChip (available/rented/maintenance), **Money**, DateRange, Modal, Drawer, Tabs, Toast, EmptyState, Skeleton, PhotoCapture, StaticMapThumbnail (customer), FleetMap (dealer, V1.5). Every component ships **hover/focus/active**, **loading/empty/error**, and **RTL + a11y** states. In V1 these are the **kit's equivalents**, not hand-built.

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

- **Customer app:** Onboarding/OTP → Search (city/date + filters) → Listing + itemized quote (location shown as a **static map thumbnail** that deep-links to native maps — no interactive in-app map) → Book + pay → Booking detail (status, handover photos, contract, invoice, refund) → History → Reviews.
- **Dealer portal:** Dashboard (fleet counts, today's bookings, pending confirmations, overdue maintenance, utilization) → Fleet → Availability → Bookings (incl. **external/walk-in entry**) → Handover/Return capture → Maintenance → Tracking (live `FleetMap` — **V1.5**) → Delivery → Finance (invoices/payouts/subscription) → Reports → Reviews → Violations → Notifications → Connections wizard → Import.
- **Admin portal:** Dealership approvals + package assignment → Plans → Entitlement view → KPIs.

---

## React Native specifics (customer app)

- **Expo Router** file-based navigation; **expo-camera** + **expo-location** + **expo-image-manipulator** for inspection photos — geotagged, timestamped, perceptual-hashed at capture (the trust engine).
- **Offline resilience** on handover/return capture: photos queue locally and upload via GCS signed URL when back online.
- **Pickup location = static map thumbnail + native-maps deep link** (no interactive in-app map in V1) — a rendered thumbnail the user taps to open the device's maps app for directions. The customer app ships **no MapLibre / react-native-maps** dependency.
- **EAS** for builds + store submission; **OTA** for JS-layer fixes; staged store releases.
- **Release & forced-update policy:** on launch the app checks its version against the versioned `/v1` API; below the **minimum-supported version** it forces an update. The API evolves additively, but a breaking change is gated by min-version — never pushed onto deployed old apps. OTA ships JS-only fixes; native changes go through the stores. Store-review lead time and **push-notification certs (FCM/APNs)** are managed as part of release ops.

---

## Maps

Two different treatments, split by surface — do not conflate them:

- **Customer app — `StaticMapThumbnail`, no interactive map (V1).** Listing/booking location is a **static map thumbnail** that **deep-links to native maps** for directions. There is **no interactive in-app map** and no MapLibre dependency on the customer side. (Matches [../product/requirements.md](../product/requirements.md) and [../product/use-cases.md](../product/use-cases.md).)
- **Dealer portal — `FleetMap` on MapLibre, deferred to V1.5.** The dealer live-fleet map (positions from Redis `vehicle_live`), telemetry visualization, and geofencing display use **MapLibre GL** (web) with **marker clustering** for large fleets and open tiles (no Google Maps licensing). This is the **operator live-visibility UI, which moves to V1.5** — the Wasl compliance *registration + reporting* obligation stays V1; only this internal map UI is deferred ([../product/requirements.md](../product/requirements.md), [../product/use-cases.md](../product/use-cases.md)).

---

## RTL & i18n

- `dir="rtl"` default; mirror directional UI (back arrows, drawers, progress); Latin terms (Miqwad, ZATCA, model names, plate Latin) inline LTR within Arabic.
- **react-i18next** with AR (default) + EN bundles; localized numerals, dates, currency.
- **Pricing is always itemized — rental + refundable deposit + VAT — a brand promise, enforced in the `Money`/`Quote` components.** (See [brand.md](brand.md).)

---

## Performance & accessibility

- **CWV targets:** LCP < 2.5s, INP < 200ms, CLS < 0.1. Code-split heavy routes; lazy-load maps/charts; images with explicit dimensions; compositor-friendly animation only.
- **WCAG 2.1 AA — the honest promise.** V1 targets AA for **focus visibility, screen-reader labels, keyboard operability, reduced-motion, and text contrast under the color rules below**. AA text contrast is guaranteed only for the token pairings that pass ≥ 4.5:1 (≥ 3:1 for large text ≥ 24px/18.66px-bold and for UI/graphical objects); the raw status and Brass colors do **not** pass as normal text and are constrained accordingly. Full-catalogue AA across every ad-hoc combination is a post-V1 goal alongside the bespoke system.
- **Color contrast rules (tokens) — these correct real AA failures on our warm palette:**
  - **Status colors** (Go `#2E9E6B`, Stop `#C2543D`, Warn `#D9913B`) **fail AA as normal-size text on Paper `#F6F2E9`.** They may be used only as **large-text, icon, or badge/chip FILL** (paired with a dark-enough on-color), **or** via **darker on-surface text variants** that meet **≥ 4.5:1 on Paper/Bone**. Define `--color-go-text` / `--color-stop-text` / `--color-warn-text` (the darkened variants) and reserve the base tokens for fills, dots, and icons — never for body/label text on the surface.
  - **White on Brass `#C8A24C` fails AA.** Add **`--color-on-accent = Deep Ink #0B1F1A`**; **Brass buttons/CTAs use Deep-Ink labels**, never white. (Brass on Pine, and Deep Ink on Paper, already pass — see [brand.md](brand.md).)
  - Every StatusChip and Money/Quote treatment must resolve to one of the passing pairings above.
- **CI contrast check:** an automated contrast lint (e.g. axe/Pa11y or a token-pair assertion) runs the design tokens against these thresholds in CI, so a non-compliant color pairing **fails the build** rather than shipping.
- Charts via **Recharts**, themed to the design tokens — data-viz is part of the design system, not an afterthought.

---

## Testing (frontend)

Visual regression (**Playwright** screenshots) at **320 / 375 / 768 / 1024 / 1440**, both languages/directions; e2e on the booking happy path; automated **a11y** checks (including the CI contrast check above). Storybook-based component tests are part of the **deferred** bespoke system; in V1, coverage rides on the kit plus the e2e/visual/a11y layers.

---

*Related: [brand.md](brand.md) · [../STATUS.md](../STATUS.md) (O-1) · [../decisions/adr-log.md](../decisions/adr-log.md) (ADR-022)*
