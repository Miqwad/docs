# Miqwad — Data Migration & Import

How a dealer moves from spreadsheets or another system onto Miqwad with **full historical data**, safely and repeatably: the import module, per-entity templates, validation/quarantine, idempotent commit, the government-document guard, ordering, and the cutover runbook.

> **Status:** ✅ **Decided.** This document **owns the migration decision** (below). Schema tables and the gov guard are referenced from [../architecture/data-model.md](../architecture/data-model.md) and [../architecture/sagas-outbox-jobs.md](../architecture/sagas-outbox-jobs.md).

---

## 0. The decision — import full historical data at onboarding

**Decision.** A dealer onboards with their **full historical** dataset — not just live state. The import module ingests vehicles (+docs), customers (+KYC), active bookings, **and** historical contracts, invoices, and maintenance/work-order history, through staging tables with validation/quarantine and idempotent re-runnable commits.

**Why.** Historical records make a dealer's reports and compliance view **complete from day one**; active bookings keep availability accurate immediately. The alternative (live-state-only) would leave reporting and compliance blank at launch and force a parallel-run on the old system.

**Consequence.** Imported government documents (historical Tajeer contracts, ZATCA invoices) describe filings that **already exist** with the government — so they are tagged `source=imported` and **excluded from all registration/clearance sagas and reconciliation** (the gov-doc guard, §4). Two-way sync with a legacy system is **out of scope** — Miqwad becomes the system of record at cutover.

---

## 1. Scope (full historical)

| Imported at onboarding | Notes |
|---|---|
| **vehicles** (+ documents) | istimara / insurance / operating-card with expiries |
| **customers** (+ KYC) | KYC numbers encrypted on import (envelope, Cloud KMS) |
| **active bookings** | open/upcoming → create `live` bookings + availability blocks |
| **historical contracts** | → `rental_contract` with `source=imported` |
| **historical invoices** | → `invoice` with `source=imported` |
| **maintenance / work-order history** | → `source=imported` |

## 2. The import module

- **Flow:** download template → upload file (to GCS via signed URL) → **validate** (creates `import_batch` + `import_row`s) → review results → **commit** (JobRunr job) → batch `completed`. **Re-runnable.**
- **Validation layers:** schema (required columns, types, enums) · referential (branch/category/customer exists or is in the same batch) · business rules (plate uniqueness, date sanity, money as halalas) · dedup keys. **Invalid rows are quarantined** with an error message — **the batch is never failed wholesale**; the dealer fixes those rows and re-uploads.
- **Idempotency:** each row carries a natural **dedup key** (vehicle = `plate_number`; customer = `phone`; booking = external ref + vehicle + dates). **Re-committing a batch upserts rather than duplicates**; `target_id` records what was created.
- **Throughput:** large files processed in chunks by JobRunr; progress on `import_batch.{total,ok,error}_rows`.

---

## 3. Per-entity templates & order

Import in **dependency order** (later entities reference earlier). Each template is a CSV/Excel with documented columns, an example row, and an enum legend, downloadable per entity (`GET /dealer/imports/templates/{entity}`).

1. **branches** (if not already created in onboarding)
2. **vehicle_category** (or use platform reference codes)
3. **vehicles** (+ documents with expiries)
4. **customers** (+ KYC docs — encrypted, KSA-resident)
5. **active bookings** (open/upcoming) → create `live` bookings + availability blocks
6. **historical contracts** → `rental_contract` `source=imported`
7. **historical invoices** → `invoice` `source=imported`
8. **maintenance history / work orders** → `source=imported`

---

## 4. The government-document guard (critical correctness)

> Miqwad must **never** re-register a contract or re-clear an invoice that already exists with the government.

- Historical **contracts and invoices** are records of documents **already filed** with Tajeer/ZATCA before the dealer joined. On import they are written with **`source = imported`** and are **excluded from the registration/clearance sagas and every reconciliation job**.
- **Enforced two ways:** a `source != imported` predicate on **every gov-driving query**, **and** a dedicated test ([./testing.md](./testing.md) §2.6) that imports a historical contract + invoice, runs all sagas/reconcilers, and asserts **zero** external Tajeer/ZATCA calls for them.
- **Active bookings (step 5) are `live`** and **do** run normal compliance going forward — their *future* return clears a real new invoice; their already-issued opening contract, if imported, stays `imported`.

---

## 5. Data quality & mapping

- **Source mapping:** each dealer's export (spreadsheet/other software) is collected in prep and its columns mapped to our template. Common transforms: currency→halalas, date formats→RFC3339, status vocab→our enums, plate/VIN normalization.
- **Quarantine report:** per-batch downloadable list of failed rows + reasons, so a non-technical dealer admin can correct and re-upload.
- **PII:** KYC numbers are encrypted on import (envelope, Cloud KMS); only masked values appear in operational tables.

## 6. Cutover runbook (per dealer)

| # | Step | Detail |
|---|---|---|
| 1 | **Prep** | Dealer exports their data; map to templates; pick a cutover date |
| 2 | **Dry-run on staging** | Import the real files into staging; review the quarantine report; fix mappings |
| 3 | **Freeze** | Dealer stops entering new data in the old system at the cutover moment |
| 4 | **Final import** | vehicles → customers → active bookings → history, in order; verify counts |
| 5 | **Verify** | Spot-check fleet, open bookings (availability blocks present), a few historical invoices (QR renders, **not** re-cleared), reports populate |
| 6 | **Go live** | Dealer operates on Miqwad; old system read-only for reference |
| 7 | **Rollback** | A wrong batch is idempotent — re-import after correction; imported rows are tagged by `import_batch_id` for targeted cleanup |

## 7. Deferred

- Aggregator **API sync** (auto-pulling Telgani bookings) is **V2**; V1 external-aggregator bookings are entered manually.
- **Two-way sync** with a legacy system is out of scope — Miqwad becomes the system of record at cutover.

> 🟡 The exact data/documents a dealer keeps per rental (which drives schema + template detail) is **Provisional** pending dealer discovery — see [../STATUS.md](../STATUS.md) (P-12).
