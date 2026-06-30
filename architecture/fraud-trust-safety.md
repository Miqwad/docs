# Fraud, Abuse & Trust-and-Safety

**The adversarial view: who tries to cheat a two-sided rental marketplace + dealer OS, and the controls + monitoring that stop them.**

Covers the *economic and behavioural* abuse that complements the technical attack surface in [security.md](security.md); monitoring/alerts feed the incident path in [reliability-dr.md](reliability-dr.md).

> **Lesson from the research (Telgani competitive intelligence):** incumbents were burned by **cashback-for-reviews** manipulation. Trust is a marketplace asset — designed in from V1.

---

## 1. Threat catalogue & controls

| Threat | What it looks like | Controls (V1 unless noted) |
|---|---|---|
| **Payment fraud** | Stolen-card bookings, chargeback abuse | Gateway-hosted fields (no PAN on our servers), 3-D Secure / Mada rails, idempotent charge, velocity checks per customer/card-token, charge-up-front → refund lifecycle, refund only to original method |
| **Deposit fraud** | Disputing legitimate deposit charges; "no damage" disputes | Geotagged + timestamped + hashed handover/return photos; itemized deposit ledger; damage evidence trail; clear up-front deposit + refund terms |
| **Account takeover (ATO)** | Credential stuffing, OTP interception | OTP rate/attempt caps + lockout; MFA for privileged dealer/admin roles; refresh-token reuse detection (revoke family); login-anomaly alerts (new device/geo); sensitive-action re-auth |
| **Fake / no-show bookings** | Bogus marketplace bookings to block competitors' cars or farm something | No-show policy + penalty; customer reliability signals; payment (rental + deposit) charged before `confirmed`; pattern detection on cancel/no-show rates |
| **Review manipulation** | Cashback-for-reviews, fake/bulk reviews, competitor smears | **Verified-booking-only reviews** (must map to a completed rental); one review per booking; rate-limit + anomaly detection on review velocity; moderation queue; no incentivized reviews by policy |
| **Inventory scraping** | Bots harvesting dealer fleet/pricing | Rate limits + Cloud Armor; bot detection; auth-gated detail where appropriate; anomaly alerts on read-volume spikes |
| **Multi-tenant abuse** | A dealer probing for another dealer's data | `@TenantId` + RLS ([security.md §4](security.md#4-multi-tenant-isolation--the-worst-case-bug)) — a security control that is also an anti-abuse control |
| **Promo/discount abuse** | Self-referral, code sharing | Deferred with promotions to **V2**; designed so per-customer caps/limits are enforceable when built |
| **Telemetry/Wasl spoofing** | Falsified vehicle positions | Server-side validation, plausibility checks on position deltas, source authentication on the ingest path |

---

## 2. Blocklists & reputation ✅

- **Customer blocklist** — customers barred (fraud, serious damage, non-payment) cannot book; checked at booking creation; reasons audited.
- **Vehicle/dealer flags** — vehicles or dealers under investigation can be suspended from the marketplace **without deleting data**.
- **Reliability signals** — no-show rate, cancellation rate, dispute rate inform risk decisions (transparent, not a hidden social score).

---

## 3. Trust by design (marketplace integrity) ✅

- **Verified transactions only** drive reviews and ratings.
- **Transparent itemized pricing** (rental + refundable deposit + VAT) — a brand promise that also removes a class of "hidden fee" disputes.
- **Clear policies** surfaced at booking time: cancellation window, deposit terms, no-show penalty, fuel/mileage rules.
- **Evidence trail** on every high-dispute moment (handover, return, damage, deposit) so trust decisions rest on records, not he-said-she-said.

---

## 4. Monitoring & response ✅

- **Signals → metrics → alerts:** payment failure/chargeback rates, OTP failure spikes, login anomalies, review-velocity anomalies, read-volume (scraping) spikes, no-show/cancel patterns → Micrometer / Cloud Monitoring.
- **Escalation:** fraud/abuse alerts route to the on-call/incident path; serious cases get a blameless postmortem and a control improvement.
- **Manual tooling:** admin views (audited) to investigate a customer/dealer, place blocks, and moderate reviews; full case-management is **V2**.
- **Feedback loop:** every confirmed abuse case is reviewed to tighten a rule or threshold.

---

## Booking fraud gates

```
Booking request
  → Customer blocklist?        — blocked → reject + audit
  → Velocity / risk checks     — risky   → hold for review
  → Charge payment (rental + deposit) up front
  → Allow confirm
       ↑ no-show / dispute signals feed reliability signals → back into risk checks
```

## Review integrity

```
Review submitted
  → Maps to a completed booking?   — no  → reject
  → Already reviewed this booking?  — yes → reject
  → Velocity anomaly?               — spike → moderation queue
                                     — ok    → publish
```

---

*See also: [security.md](security.md) (technical attack surface) · [reliability-dr.md](reliability-dr.md) (monitoring/incident response) · [STATUS.md](../STATUS.md). Promotions/coupon-abuse is deferred with promotions to V2.*
