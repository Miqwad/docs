# Miqwad — Brand Identity

The visual and verbal system for **Miqwad (مِقْوَد)** — palette, typography, voice, and design tokens. Arabic-first, full RTL, bilingual AR/EN. Everything here is **✅ Decided** and feeds the shared design system in [frontend-design-system.md](frontend-design-system.md).

---

## The name & positioning

**Miqwad (مِقْوَد)** is the Arabic word for *the steering wheel* — the point of control. It does three jobs: it puts the **dealership** back in control of its whole business; it is the **customer's** moment of taking the wheel; and as a **brand** it is short, dignified, unmistakably Arabic, clean in Latin script, and conflict-free in screening.

- **Tagline (AR):** المقود بين يديك — *"The wheel is in your hands."*
- **Tagline (EN):** *Take the wheel.*
- **Wedge:** *Aggregators solved customer acquisition for dealerships. Miqwad solves everything else — and fills the cars too.*

**Brand pillars:** Control (التحكّم) · Trust (الثقة) · Compliance made effortless (الامتثال بلا عناء) · Local & dignified (محلّي وراقٍ).

> The name is conflict-screened, **not** legally cleared. Before public launch, run "Miqwad / مقود" through SAIP trademark search and confirm `.sa` / `.com` domain availability.

---

## Logo

A minimal geometric **steering-wheel glyph** — an outer ring with three spokes meeting a central hub, single confident stroke weight, the lower spoke subtly elongated to suggest forward motion. Reads as a wheel at any size; reduces to a favicon.

- **Lockups:** horizontal (mark + "Miqwad مِقْوَد"), stacked (mark above bilingual wordmark), icon-only (app icon/favicon).
- **Clear space:** minimum padding equal to the diameter of the central hub.
- **Don'ts:** no tire/car silhouette, no tilt, no gradients on the mark, never on busy photography without the sand or ink backplate.

---

## Color palette

Premium and locally resonant — deep emerald (dignity, trust, a nod to the Kingdom without copying the flag), warm brass (the metal of a key and a dashboard), grounded on warm sand paper rather than clinical white. Deliberately avoids the generic "purple-on-white SaaS" look.

| Role | Name | HEX | Use |
|---|---|---|---|
| Primary | **Miqwad Pine** | `#0F3D33` | Primary brand color, headers, key surfaces |
| Primary-dark | **Deep Ink** | `#0B1F1A` | Text on light, dark sections |
| Accent | **Brass** | `#C8A24C` | CTAs, highlights, the "premium" touch |
| Accent-soft | **Sand Gold** | `#E7D6AC` | Soft fills, badges |
| Surface | **Paper** | `#F6F2E9` | App/page background (warm, not white) |
| Surface-2 | **Bone** | `#FBF9F3` | Cards, raised surfaces |
| Signal-go | **Available Green** | `#2E9E6B` | "Available", success, confirmed |
| Signal-stop | **Busy Red** | `#C2543D` | "Rented/Unavailable", errors |
| Signal-warn | **Service Amber** | `#D9913B` | "In maintenance", warnings |
| Neutral | **Slate** | `#5A6B62` | Secondary text, muted UI |
| Hairline | **Mist** | `#D9D2C2` | Borders, dividers |

**Contrast:** Pine on Paper and Brass on Pine both meet **AA** for UI/large text. Body text uses Deep Ink on Paper for **AAA** comfort.

---

## Typography

Bilingual, Arabic-first, distinctive but professional — explicitly **not** Inter/Roboto/system.

| Role | Family | Weights | Use |
|---|---|---|---|
| AR + Latin workhorse | **IBM Plex Sans Arabic** | 300–700 | UI, body, most headings |
| Latin display accent | **Sora** | 500–700 | English headlines, numerics |
| Numerals | **Sora tabular figures** | — | Prices, odometer, dashboard metrics (columns align) |

**Type scale** (mobile / web, base 16px) — size / line-height:

| Token | px |
|---|---|
| Display | 40 / 48 |
| H1 | 30 / 36 |
| H2 | 24 / 30 |
| H3 | 19 / 26 |
| Body | 16 / 26 |
| Small | 14 / 22 |
| Caption | 12 / 18 |

---

## RTL & bilingual rules

- Arabic is the **default direction** (`dir="rtl"`).
- **Mirror** all directional UI: back arrows, progress, drawers.
- Latin terms (Miqwad, ZATCA, model names) sit **inline LTR** within RTL text.

---

## Voice

- **Confident, not loud.** State what we do plainly; never oversell.
- **Respectful and local.** Arabic-first, the register a Saudi business owner expects.
- **Clear over clever.** A dealer reading a screen at 11pm should never be confused.
- **Honest about money.** Prices, fees, and deposits always shown in full, up front.

**Examples:**

| Surface | Copy |
|---|---|
| Empty fleet | "لا توجد مركبات بعد. أضِف أول سيارة لتبدأ الاستقبال." — *"No vehicles yet. Add your first car to start receiving bookings."* |
| Booking confirmed | "تم تأكيد الحجز. المركبة محجوزة لك." — *"Booking confirmed. The vehicle is reserved for you."* |

---

## The money-itemized promise

Pricing is **always shown itemized — rental + deposit (refundable) + VAT — with the all-in total bold.** This is a **brand promise, not a UI choice**, and it is enforced in the `Money`/`Quote` components of the design system (see [frontend-design-system.md](frontend-design-system.md) §i18n).

---

## Design tokens

These brand values are the source for the Style Dictionary token set generated into CSS variables (web) and a JS theme (RN): the **palette** above, the **type scale**, plus spacing, radii, and durations. Token generation and consumption are described in [frontend-design-system.md](frontend-design-system.md).

---

## Brand in one breath

Miqwad is the steering wheel of the Saudi car-rental business: it puts the dealership in control of its whole operation, and puts the customer in the driver's seat — with trust, transparency, and zero paperwork pain, in Arabic, by design.

---

*Related: [frontend-design-system.md](frontend-design-system.md) · [../STATUS.md](../STATUS.md) · [../decisions/adr-log.md](../decisions/adr-log.md)*
