# Miqwad — Pricing, Packaging & Entitlements

What a dealer buys, what they get, and how the system enforces it. This doc **owns** the packaging decision: three packages, tiers, commission, billing, and the server-side entitlement contract.

> 🟡 **Provisional numbers** — limits, prices, and the commission rate need **WTP validation / dealer discovery** sign-off (**P-10** in [../STATUS.md](../STATUS.md)). The **structure** (packages, tiers, modules, enforcement) is **decided** ✅; the figures are starting points.

---

## The three packages ✅

| Package | Listed to customers? | Runs dealer's own ops? | Billing |
|---|---|---|---|
| **Marketplace** | Yes | Only the minimum to fulfill a Miqwad booking | **Commission** on marketplace bookings; no/low subscription |
| **Management** | No | Yes — the full operating system | **SaaS subscription** (tiered); **no commission** |
| **Both** | Yes | Yes | **Discounted subscription + commission** on marketplace bookings |

**Why three:** dealers differ. A Telgani-style lead-only dealer wants just bookings; an operationally-mature dealer wants the OS without giving up margin to a marketplace; many want both. A single bundled product would exclude the first two.

**Marketplace-only is not "no system":** the rental is still legally bound by Tajeer/ZATCA/handover, and availability must stay accurate. A Marketplace-only dealer gets the **minimum operating slice** to fulfill a Miqwad booking — vehicle listing, availability, receive/confirm, Tajeer/ZATCA/handover for those bookings, payouts view — but **not** the full ops suite (maintenance, telemetry, multi-branch admin, full analytics, external-channel management).

---

## Tiers — free / standard / pro ✅ (numbers 🟡)

Tiers apply to **Management** and **Both** (Marketplace-only is commission-priced with a minimal fixed feature set). Each tier = **numeric limits** + **enabled modules**.

| Capability | Free (entry) | Standard | Pro |
|---|---|---|---|
| Vehicles | ≤ 5 | ≤ 50 | unlimited\* |
| Branches | 1 | ≤ 3 | unlimited\* |
| Staff users | ≤ 2 | ≤ 10 | unlimited\* |
| Fleet, availability, bookings (all channels) | ✓ | ✓ | ✓ |
| Tajeer / ZATCA / handover / deposits | ✓ | ✓ | ✓ |
| Walk-in + external-aggregator entry | ✓ | ✓ | ✓ |
| Basic reports (utilization, revenue) | ✓ | ✓ | ✓ |
| Maintenance & maintenance tracking | — | ✓ | ✓ |
| Fleet telemetry / live map | — | ✓ | ✓ |
| Delivery & collection scheduling | — | ✓ | ✓ |
| Advanced analytics + PDF/Excel export | — | ✓ | ✓ |
| Multi-branch consolidated reporting | — | — | ✓ |
| Notifications (push/SMS/email) | basic | ✓ | ✓ |
| Priority support / SLA | — | — | ✓ |
| Price (SAR/month) 🟡 | 0 | ~399 | ~999 |

\* "unlimited" = a high soft cap (e.g. 1,000) with a fair-use review trigger, not literally unbounded.

The **free entry tier exists to acquire dealers** (the Shopify-style "solve the operator's problem first" wedge) — enough to run a tiny dealer, with limits that create a natural upgrade path.

---

## Commission ✅ (rate 🟡)

- Charged **only** on `marketplace`-channel bookings. Proposed rate **~10%** (configurable per dealer via `commission_config.base_rate_bps`).
- `dealer_direct`, `walk_in`, and `external_aggregator` accrue **zero** commission — the **fairness rule** (the dealer already pays the external aggregator's cut; Miqwad does not double-dip).
- On refund/cancellation, commission is **reversed proportionally**; the ledger records the reversal.

---

## Billing & settlement ✅

- **Subscription** (Management/Both) is invoiced monthly by Miqwad to the dealer. **This invoice is itself a ZATCA standard (B2B) e-invoice** — signed, cleared/reported, QR — from the same ZATCA module that issues dealer→customer invoices; Miqwad is the supplier.
- **Commission** (Marketplace/Both) is settled **net against payouts**: `dealer payout = captured marketplace revenue − commission` over the period, shown transparently (gross/commission/net/period). The commission line is also reflected on a Miqwad→dealer ZATCA invoice.
- **Dunning:** failed subscription payment → grace window → **feature degradation to the free-tier limit set** (not data deletion) → eventual suspension (scheduled job).
- **Proration & tier changes:** upgrades take effect immediately (prorated); downgrades next period; downgrading **below current usage** (e.g. 60 vehicles → Standard's 50) is **blocked** until the dealer is within the new limit.

---

## The entitlement enforcement contract ✅

**Model** (resolved at request time, cached in Redis/Caffeine):

| Concept | Meaning |
|---|---|
| `plan` | Catalog entry: package + tier → `{ enabled_modules[], limits{} }` |
| `subscription` | A dealer's active plan + status + period + any per-dealer overrides |
| `entitlement` | The **effective** resolved `{modules, limits}` for a dealer, recomputed on subscription change |

### Two enforcement points (both mandatory)

**1. Server-side (authoritative).** Every request resolves the caller's entitlement from `dealership_id`:

| Gate | Triggered when | Response |
|---|---|---|
| **Module gate** | Calling an endpoint whose module isn't enabled | **403 `entitlement_module_disabled`** |
| **Limit gate** | A create that would exceed a numeric limit (e.g. 6th vehicle on Free) | **402 `entitlement_limit_reached`** + `details: {limit, current, upgrade_to}` |
| **Package gate** | Management-only dealer hitting marketplace-listing endpoints, or Marketplace-only hitting full-ops endpoints | **403 `entitlement_package_excluded`** |

**2. Frontend (UX).** Navigation and actions are package/tier-aware: disabled/hidden modules don't appear; limit-bound actions show the upgrade prompt before the call. The FE **never enforces** — the server is authoritative; the FE only avoids dead-ends.

> **Operational flags (Togglz) are separate from entitlements.** Togglz gates *roll-out* of credential-gated integrations (turn Tajeer/ZATCA/Wasl on per dealer as credentials provision) and risky features; entitlements gate *what the dealer paid for*. A feature can be entitlement-on but flag-off (paid for, not yet provisioned). See [../decisions/adr-log.md](../decisions/adr-log.md) (ADR-013).

### Module catalog (the gateable units)

`fleet`, `availability`, `booking_core`, `booking_external_channel`, `tajeer`, `zatca`, `handover`, `deposits`, `marketplace_listing`, `maintenance`, `telemetry`, `delivery`, `reports_basic`, `reports_advanced`, `consolidated_reporting`, `notifications`, `reviews`, `saher`, `wasl`, `connection_wizard`, `import`, `admin`.

The authoritative package/tier → module mapping lives in the `plan` catalog seeded by Flyway.

---

## Worked examples

| Dealer | Package / tier | Outcome |
|---|---|---|
| Small lead-only dealer also on Telgani | **Marketplace** | Listed on Miqwad; pays ~10% on Miqwad bookings; logs Telgani bookings (external channel, no commission) so availability stays accurate and the rental runs through Tajeer/ZATCA. No subscription. |
| Established 40-car agency ditching spreadsheets, not paying commission | **Management / Standard** | ~SAR 399/mo; full ops, telemetry, maintenance, delivery; not listed; zero commission. Imports full history at onboarding. |
| Growth dealer wanting both | **Both / Pro** | Discounted subscription + ~10% on marketplace bookings; unlimited fleet, consolidated multi-branch reporting, priority support. |

See [requirements.md](requirements.md) for the requirement set and [use-cases.md](use-cases.md) for the booking channels and flows.
