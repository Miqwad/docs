# Miqwad — API Reference

The human companion to the machine-readable contract: three surfaces, the wire conventions, the full error catalog, and entitlement-gating semantics. The OpenAPI file [`./openapi.yaml`](./openapi.yaml) is the source of truth (importable into Swagger UI / Postman / Stoplight); this page explains it.

> 🟡 **Some integration-touching endpoints are Provisional.** Anything that calls Tajeer, Wasl, Absher, or SMS/email/push is built behind a stable interface, but the wire-level request/response detail will firm up once vendor docs land — see STATUS [P-1, P-3, P-4, P-6](../STATUS.md). **ZATCA and Moyasar contracts are in hand (2026-06-30) — those DTOs are firm.** The endpoint shapes below are stable; the still-pending integration DTOs behind them are not.

---

## Two API styles (ADR-016)

The surface is split deliberately — see [ADR-016](../decisions/adr-log.md) (REST `/v1` for the domain + Elide JSON:API for CRUD). Both styles sit under the same `/v1` base.

| Style | Used for | Response shape |
|---|---|---|
| **Hand-written REST** (Spring MVC, coroutine handlers) | The complex domain: booking-confirm & return-settle sagas, availability/exclusion logic, marketplace search, payments/deposits/ledger, compliance adapters, entitlement enforcement, commission/settlement, import/migration | The conventions on this page (uniform error envelope, cursor pagination, idempotency keys) |
| **Elide JSON:API** | CRUD-heavy entities: branches, staff, vehicle CRUD + documents, categories, rate plans, maintenance types/schedules, work orders, reviews read/moderate, notifications read, reference data | JSON:API spec (`data` / `attributes` / `relationships`) |

Tenancy stays safe across both: Elide runs over the same Hibernate/JPA layer, so `@TenantId` applies and Postgres RLS remains the hard backstop.

---

## Conventions

| Aspect | Rule |
|---|---|
| **Base URL** | `https://api.miqwad.sa/v1` (prod) · `https://staging-api.miqwad.sa/v1` (staging) |
| **Versioning** | Version in the path (`/v1`). Changes are **additive** within a version (mobile apps can't be force-updated); a breaking change ⇒ `/v2`. |
| **Auth** | `Authorization: Bearer <JWT>`. The dealer token carries a `dealership_id` claim — **authoritative for tenant scope, never read from the body**. Customers authenticate by phone OTP; staff/platform via the IdP (Keycloak or GCIP — provider is Open, [STATUS O-2](../STATUS.md)). |
| **Money** | Integer **halalas** (1 SAR = 100 halalas; `15000` = 150.00 SAR) + explicit currency. **Never floats.** |
| **Timestamps** | RFC3339 UTC (`2026-06-24T09:00:00Z`). |
| **Pagination (REST)** | Cursor-based: pass `?cursor=` and `?limit=` (max 100); responses include `next_cursor` and `has_more`. |
| **Idempotency** | Money/external-system POSTs accept an `Idempotency-Key` header; replays return the original result. Same key + different body ⇒ `409 idempotency_conflict`. |
| **Rate limiting** | Auth / search / booking endpoints are rate-limited ⇒ `429 rate_limited` with a `Retry-After` header. |
| **i18n** | Responses are language-neutral (code + data); the apps localize. `Accept-Language: ar` (default) or `en` selects any server-rendered strings (e.g. notification templates). |

### Error envelope

Every REST error uses one uniform envelope. Internally a Spring `ProblemDetail` (RFC 9457) is produced and serialized into this shape by a single global `@RestControllerAdvice`:

```json
{ "error": { "code": "vehicle_unavailable",
             "message": "The vehicle is no longer available for the selected dates.",
             "details": { } },
  "request_id": "req_01J..." }
```

- `code` — the stable machine key; **clients branch on this**.
- `message` — human-readable, localizable.
- `details` — structured context (varies per code; see catalog).
- `request_id` — correlates with logs/traces.

Stack traces and internal messages are never leaked.

### Status-code discipline

- `2xx` only on real success. A saga that defers external work returns `202`/`200` with the entity in its **in-progress state** (e.g. invoice `pending_clearance`) — not a fake success.
- `4xx` = the caller can fix it (validation, entitlement, conflict). `5xx`/`503` = server/dependency — safe to retry.
- Never `200` with an error body.

---

## Error catalog

| `code` | HTTP | Meaning · `details` |
|---|---|---|
| `unauthorized` | 401 | Missing / invalid / expired token |
| `forbidden` | 403 | Authenticated but role not permitted |
| `not_found` | 404 | Resource absent or not in tenant scope |
| `validation_error` | 400 | `details.fields[]` with per-field messages |
| `conflict` | 409 | State conflict (generic) |
| `vehicle_unavailable` | 409 | Window taken (booking race / overlap) |
| `payment_required` | 402 | Payment needed to proceed |
| `payment_failed` | 402 | Gateway declined; `details.gateway_code` |
| `tajeer_failed` 🟡 | 422 | Contract registration rejected; `details.reason` |
| `tajeer_auth_failed` 🟡 | 422 | Dealer Naql credentials invalid / expired |
| `zatca_failed` | 422 | Invoice clearance / report rejected; `details.reason` |
| `absher_failed` 🟡 | 422 | Identity / e-sign failed |
| `identity_unverified` 🟡 | 422 | Renter KYC not verified |
| `wasl_failed` 🟡 | 422 | Wasl reporting rejected (non-blocking to core flow) |
| `entitlement_module_disabled` | 403 | Dealer's package/tier doesn't include this module |
| `entitlement_package_excluded` | 403 | Surface excluded for the package (e.g. marketplace endpoint for a Management-only dealer) |
| `entitlement_limit_reached` | 402 | Over a tier limit; `details:{limit,current,upgrade_to}` |
| `import_validation_failed` | 400 | Batch has quarantined rows; `details.error_rows` |
| `idempotency_conflict` | 409 | Same idempotency key, different request body |
| `rate_limited` | 429 | `Retry-After` header + `details.retry_after_s` |
| `dependency_unavailable` 🟡 | 503 | External system circuit open; safe to retry later |
| `internal_error` | 500 | Unexpected; quote `request_id` to support |

🟡 = surfaced by integration-touching flows whose vendor contract is still Provisional (Tajeer/Wasl/Absher/notifications — STATUS [P-1, P-3, P-4, P-6](../STATUS.md)). ZATCA + Moyasar contracts are in hand (2026-06-30).

---

## Entitlement gating (402 vs 403)

Every dealer endpoint is gated by the dealership's resolved entitlement (package + tier → modules + limits) **in addition to** RBAC and tenant scope. All layers apply independently.

| Response | Meaning | App behavior |
|---|---|---|
| **402 `entitlement_limit_reached`** | "You can buy more" — a numeric cap is hit | Show an upgrade CTA using `details.upgrade_to` |
| **403 `entitlement_module_disabled`** | The module isn't in your package/tier | Hide the surface |
| **403 `entitlement_package_excluded`** | The whole surface is excluded for the package (e.g. a Management-only dealer calling marketplace-listing endpoints) | Hide the surface |

The server is authoritative regardless of what the app shows. These are distinct from `unauthorized`/`forbidden` (auth/RBAC) and from tenant scope (RLS).

---

## Surface 1 — `/customer` (mobile app)

The consumer marketplace: phone-OTP auth, search across all dealerships, book, pay, track the rental.

| Method | Path | Purpose |
|---|---|---|
| POST | `/customer/auth/request-otp` | Send login code to phone |
| POST | `/customer/auth/verify-otp` | Verify code → tokens |
| GET · PATCH | `/customer/me` | Profile |
| GET | `/customer/search` | **Search available cars** for a date window + city/location; filters: category, price, transmission; sort by price/distance/rating |
| GET | `/customer/listings/{id}` | Listing detail with a **full itemized quote** (rental + refundable deposit + VAT + mileage policy) |
| GET · POST | `/customer/bookings` | List my bookings / **create booking** (returns a `payment_intent`) |
| GET | `/customer/bookings/{id}` | Booking detail incl. contract + handover photos |
| POST | `/customer/bookings/{id}/confirm-payment` | Finalize after the gateway returns |
| POST | `/customer/bookings/{id}/cancel` | Cancel |
| POST | `/customer/bookings/{id}/review` | Rate dealer/vehicle after completion |

**Booking creation is two-phase.** `POST /customer/bookings` reserves a *pending* booking and returns a Moyasar `payment_intent` covering rental + deposit, both **charged up front** (the deposit is refunded in full on a clean return — ADR-015); the app completes payment with the gateway SDK, then calls `confirm-payment`. If the window was taken in between, creation returns `409 vehicle_unavailable` — the live-inventory guarantee in action.

---

## Surface 2 — `/dealer` (SaaS portal)

The operating system — everything a dealership runs daily, all tenant-scoped. Role gates write actions: `owner`/`manager` full; `branch_agent` limited to their branch; `accountant` reads finance.

> Every dealer endpoint is entitlement-gated: a module the package/tier doesn't include returns `403 entitlement_module_disabled`; a create over a tier limit returns `402 entitlement_limit_reached`.

**Auth & dashboard**

| Method | Path | Purpose |
|---|---|---|
| POST | `/dealer/auth/login` | Staff login |
| GET | `/dealer/me` | Staff user + dealership context (Tajeer/ZATCA connection flags) |
| GET | `/dealer/dashboard` | Fleet counts, today's bookings, pending confirmations, revenue, overdue maintenance, utilization |

**Branches · Fleet · Availability**

| Method | Path | Purpose |
|---|---|---|
| GET · POST | `/dealer/branches` | List / create branches |
| PATCH | `/dealer/branches/{id}` | Update |
| GET · POST | `/dealer/vehicles` | List fleet (filter by branch/status) / add vehicle |
| GET · PATCH | `/dealer/vehicles/{id}` | Vehicle detail (docs, images, rate plan, maintenance status) / update |
| PUT | `/dealer/vehicles/{id}/rate-plan` | Set pricing (daily/weekly/monthly, deposit, mileage) |
| POST | `/dealer/vehicles/{id}/images` | Get a presigned upload URL for an image |
| GET · POST | `/dealer/vehicles/{id}/availability` | View calendar / add a manual hold or transfer block |

Bookings and maintenance write availability blocks automatically; the availability endpoint is for manual holds. Overlaps are rejected with `409 vehicle_unavailable` (DB-level GiST exclusion constraint).

**Bookings & Tajeer** 🟡

| Method | Path | Purpose |
|---|---|---|
| GET | `/dealer/bookings` | Incoming/active bookings (filter by status/branch) |
| GET | `/dealer/bookings/{id}` | Detail |
| POST | `/dealer/bookings/{id}/confirm` | Confirm → **triggers Tajeer e-contract registration**; `422 tajeer_failed` surfaces a registration problem |
| POST | `/dealer/bookings` | Create a `walk_in` / `dealer_direct` / `external_aggregator` booking on behalf of a customer. Body carries `channel` (+ `channel_source` for external). Writes an availability block; **no commission** for non-marketplace; runs the normal Tajeer/ZATCA/handover path. `409 vehicle_unavailable` if the window is taken. |

**Handover / inspection** (the trust engine)

| Method | Path | Purpose |
|---|---|---|
| POST | `/dealer/bookings/{id}/inspections` | Create a `handover` or `return` inspection (odometer, fuel) |
| POST | `/dealer/inspections/{id}/photos` | Attach a **geotagged, timestamped** photo (presigned upload) |
| POST | `/dealer/inspections/{id}/damages` | Record a damage finding (area, severity, estimate, photo) |

**Maintenance & work orders**

| Method | Path | Purpose |
|---|---|---|
| GET · POST | `/dealer/vehicles/{id}/maintenance-schedules` | Preventive schedules (every N km or N days) |
| GET · POST | `/dealer/work-orders` | List / open a work order (auto-blocks the vehicle for the service window) |
| PATCH | `/dealer/work-orders/{id}` | Progress / complete (records cost, odometer; recomputes next-due, releases block) |

**Fleet tracking** 🟡

| Method | Path | Purpose |
|---|---|---|
| GET | `/dealer/tracking/live` | Live map: every vehicle's position, speed, ignition, status |
| GET | `/dealer/tracking/vehicles/{id}/history` | Telemetry history for trip/route review |
| POST | `/dealer/tracking/ingest` | Device/gateway → platform ping intake (provider-agnostic; a future Wasl feed uses the same shape) |

**Finance** 🟡

| Method | Path | Purpose |
|---|---|---|
| GET | `/dealer/invoices` | ZATCA e-invoices with clearance status + QR (filter `direction`) |
| GET | `/dealer/payouts` | Settlements (gross, commission, net) |
| GET | `/dealer/subscription` | Current package/tier, entitlement summary, billing status |

**Delivery & collection**

| Method | Path | Purpose |
|---|---|---|
| GET · POST | `/dealer/bookings/{id}/deliveries` | Schedule a delivery or collection (address, geo, window, fee) |
| PATCH | `/dealer/deliveries/{id}` | Update status (`scheduled → en_route → completed`) / assign staff |

**Reviews · Violations · Notifications**

| Method | Path | Purpose |
|---|---|---|
| GET | `/dealer/reviews` | Reviews for this dealership's bookings/vehicles |
| GET · POST | `/dealer/violations` | List / record a Saher fine during a rental window |
| POST | `/dealer/violations/{id}/attribute` | Attribute the fine to the renter on record |
| GET | `/dealer/notifications` | Sent-notification log (communications history) |
| GET · PUT | `/dealer/notification-preferences` | Per-dealer channel preferences |

**Connection wizard** 🟡 · **Import / migration**

| Method | Path | Purpose |
|---|---|---|
| GET | `/dealer/connections` | Tajeer / ZATCA (CSID) / Wasl connection status |
| POST | `/dealer/connections/{system}` | Submit/refresh credentials for a government system (stored in the vault, never returned) |
| GET | `/dealer/imports/templates/{entity}` | Download the CSV/Excel template for an entity |
| POST | `/dealer/imports` | Create an import batch (entity + uploaded file via signed URL) → validation |
| GET | `/dealer/imports/{id}` | Batch status + per-row results (ok / quarantined + error) |
| POST | `/dealer/imports/{id}/commit` | Commit validated rows (idempotent). Imported gov docs flagged `imported`, never re-submitted |

---

## Surface 3 — `/admin` (Miqwad platform team)

Role-gated (`super_admin` / `ops` / `support` / `finance`).

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/dealerships` | All dealerships (filter by status) |
| POST | `/admin/dealerships/{id}/approve` | Approve a pending dealership |
| PUT | `/admin/dealerships/{id}/commission` | Set commission rate + SaaS plan + fees |
| GET | `/admin/overview` | Platform KPIs: active dealerships, GMV, commission, top cities |
| GET | `/admin/plans` | The package/tier catalog |
| PUT | `/admin/dealerships/{id}/subscription` | Assign/change a dealership's package + tier (recomputes entitlement) |
| GET | `/admin/dealerships/{id}/entitlement` | The resolved effective modules/limits for a dealership |

---

## Booking state machine

The heart of the system. State: `pending → confirmed → active → completed`, with branches to `cancelled` / `no_show`. Hard guards:

- **Payment** clears before `confirmed`.
- **Tajeer** registration (`registered`) before `active`.
- **Handover inspection** (with mandatory geotagged, timestamped photos) before `active`.
- **Return inspection + cleared ZATCA invoice** before `completed`.

Commission accrues only on `channel = marketplace` bookings, never `dealer_direct` / `walk_in` / `external_aggregator`.

---

## Integration touchpoints 🟡 (server-side, not public endpoints)

These run inside the platform, referenced by the flows above, behind provider-agnostic adapters with circuit breakers. ZATCA and Moyasar wire contracts are **in hand (2026-06-30)**; Tajeer/Wasl/Absher/notifications stay Provisional pending vendor docs (STATUS [P-1, P-3, P-4, P-6](../STATUS.md)).

| Integration | Trigger / role | STATUS |
|---|---|---|
| **Tajeer** | On booking `confirm`, registers the unified e-contract via the dealership's Naql credentials; status tracked on `rental_contract`. | [P-1](../STATUS.md) |
| **ZATCA Phase 2** | On `return` completion, generates the signed e-invoice (XML + TLV QR), clears/reports, stores the result on `invoice`. | [✅ P-2](../STATUS.md) |
| **Wasl** | Fleet-tracking registration + telemetry reporting; non-blocking to the core flow. | [P-3](../STATUS.md) |
| **Absher** | Renter identity verification / e-sign. | [P-4](../STATUS.md) |
| **Moyasar** | Payment intents, charges (rental + deposit up front), refunds; results land on `payment` via webhook. Deposit refunded on clean return (ADR-015). | [✅ P-5](../STATUS.md) |
| **Email / SMS / push** | Notification delivery (provider TBD, e.g. Unifonic for KSA SMS). | [P-6](../STATUS.md) |

### Inbound webhooks

Payment / Tajeer webhooks: **verify signature, dedupe by event id, process idempotently, respond `2xx` fast** (queue the work). Replays must be safe.

---

## Auth, roles & scope summary

| Token | Reaches | Scope |
|---|---|---|
| **Customer** | `/customer/*` only | Its own data only |
| **Staff** | `/dealer/*` | Scoped to `dealership_id` from the JWT; role gates writes |
| **Platform** | `/admin/*` | Role-gated (`super_admin` / `ops` / `support` / `finance`) |

Tenant isolation is enforced server-side via the token claim **and** Postgres row-level security — never via client input.

---

**See also:** [`./openapi.yaml`](./openapi.yaml) (machine-readable contract) · [ADR-016](../decisions/adr-log.md) (API surface split) · [STATUS](../STATUS.md) (Provisional integration contracts).
