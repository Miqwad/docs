# Miqwad — System & UML Diagrams

The editable Mermaid source for every system diagram — the source of truth, regenerated into the rendered views. Renders on GitHub, VS Code, and most viewers.

> Cross-refs: ER diagram → [../architecture/data-model.md](../architecture/data-model.md); cloud topology → [../architecture/cloud.md](../architecture/cloud.md); sagas → [../architecture/sagas-outbox-jobs.md](../architecture/sagas-outbox-jobs.md); modules → [../architecture/README.md](../architecture/README.md). Stack: Kotlin · JDK 25 · Spring Boot 4 modular monolith · PostgreSQL + Redis · JobRunr · Moyasar · GCP me-central2.

---

## 1. Component / Architecture

Modules (packaging/entitlements, channels, delivery, reviews, Saher, import), the Elide-for-CRUD vs hand-built domain split, JobRunr/outbox, and GCP data services.

```mermaid
flowchart TB
  subgraph Clients
    RN[Customer app — RN/Expo]
    DW[Dealer portal — React]
    AW[Admin portal — React]
    DEV[GPS/OBD devices]
  end
  subgraph API["miqwad-api (Kotlin · Spring Boot 4 modular monolith)"]
    direction TB
    subgraph HandBuilt["Hand-built domain (Spring MVC)"]
      SEARCH[Search/Availability]
      BOOK[Booking + channels]
      SAGA[Sagas + Outbox]
      PAY[Payments/Deposits/Ledger]
      ZAT[Invoicing/ZATCA]
      SETTLE[Settlement/Commission]
      ENT[Entitlements/Packaging]
      IMP[Import/Migration]
      INTEG[Gov adapters: Tajeer/ZATCA/Wasl/Absher]
    end
    subgraph Elide["Elide JSON:API (fast CRUD)"]
      FLEET[Fleet/Branches/Staff]
      RATE[Rate plans/Categories]
      MAINT[Maintenance/Work orders]
      REV[Reviews/Notifications read]
    end
  end
  JOBS[miqwad-jobs — JobRunr workers]
  subgraph Data["GCP data (private, me-central2)"]
    SQL[(Cloud SQL Postgres<br/>PostGIS/pg_partman)]
    REDIS[(Memorystore Redis)]
    GCS[(Cloud Storage)]
    VAULT[Secret Manager/KMS]
  end
  KC[Keycloak]
  EXT[External: Tajeer/ZATCA/Wasl/Absher/Moyasar/FCM/SMS]

  RN & DW & AW --> API
  DEV --> BOOK
  API --> SQL & REDIS
  API -->|signed URLs| GCS
  API --> VAULT
  API <--> KC
  SAGA --> JOBS
  JOBS -->|outbox dispatch| EXT
  INTEG --> EXT
  ENT -. gates .-> HandBuilt
  ENT -. gates .-> Elide
```

## 2. Booking state machine

Includes `confirmation_failed` (payment ok / Tajeer fail) and channel notes. Guards are hard preconditions.

```mermaid
stateDiagram-v2
  [*] --> pending: reserve (any channel)
  pending --> confirmed: payment captured/authorized + Tajeer registered
  pending --> confirmation_failed: payment ok but Tajeer failed
  confirmation_failed --> pending: auto-refund/void + retry
  pending --> cancelled: customer/dealer cancels
  confirmed --> active: handover inspection (min photos)
  confirmed --> no_show: grace elapsed → fee
  confirmed --> cancelled: within policy
  active --> completed: return inspection + ZATCA invoice cleared/reported
  completed --> [*]
  cancelled --> [*]
  no_show --> [*]
  note right of confirmed
    Commission accrues only if channel = marketplace.
    All channels write an availability_block.
  end note
```

## 3. Sequence — Book & Confirm (saga)

The transactional outbox + JobRunr dispatch + compensation.

```mermaid
sequenceDiagram
  actor C as Customer
  participant APP as App
  participant API as miqwad-api
  participant PAY as Moyasar
  participant DB as Postgres
  participant JR as JobRunr
  participant TAJ as Tajeer
  C->>APP: choose car + dates
  APP->>API: POST /customer/bookings (Idempotency-Key)
  API->>DB: booking=pending + availability_block + outbox(register_contract)
  API-->>APP: payment_intent (rental+deposit hold)
  APP->>PAY: pay (hosted fields)
  PAY-->>APP: payment_ref
  APP->>API: confirm-payment(payment_ref)
  API->>DB: payment=authorized
  JR->>DB: read outbox(pending)
  JR->>TAJ: register contract
  alt Tajeer OK
    TAJ-->>JR: tajeer_ref
    JR->>DB: contract=registered, booking=confirmed, outbox=dispatched
    API-->>C: confirmed + pickup details
  else Tajeer fails
    JR->>PAY: void/refund authorization
    JR->>DB: booking=confirmation_failed, release block, log reason
    API-->>C: payment reversed, please retry
  end
```

## 4. Sequence — Return & Settle (saga)

```mermaid
sequenceDiagram
  actor A as Branch agent
  participant API as miqwad-api
  participant DB as Postgres
  participant JR as JobRunr
  participant ZAT as ZATCA
  participant PAY as Moyasar
  participant TAJ as Tajeer
  A->>API: start return (odometer, fuel, photos, damages)
  API->>DB: inspection=return + damage records, vehicle=available
  API->>DB: outbox(clear_invoice), outbox(settle_deposit), outbox(close_contract)
  JR->>ZAT: clear/report invoice
  alt cleared
    ZAT-->>JR: zatca_uuid + QR
    JR->>DB: invoice=cleared
  else timeout
    JR->>DB: invoice=pending_clearance (retry, car already released)
  end
  JR->>PAY: refund or partial-capture deposit (per damage)
  JR->>DB: ledger entries (deposit, vat, commission)
  JR->>TAJ: close contract
  JR->>DB: booking=completed
```

## 5. Entitlement gating

Every dealer request passes RBAC + tenant scope + entitlement.

```mermaid
flowchart TD
  REQ[Dealer request] --> AUTH{Valid JWT?}
  AUTH -- no --> E401[401 unauthorized]
  AUTH -- yes --> TEN[Set tenant GUC from dealership_id]
  TEN --> RBAC{Role allows action?}
  RBAC -- no --> E403[403 forbidden]
  RBAC -- yes --> ENT[Resolve entitlement (cache)]
  ENT --> MOD{Module enabled?}
  MOD -- no --> EM[403 entitlement_module_disabled]
  MOD -- yes --> PKG{Package allows surface?}
  PKG -- no --> EP[403 entitlement_package_excluded]
  PKG -- yes --> LIM{Create within tier limit?}
  LIM -- no --> EL[402 entitlement_limit_reached + upgrade_to]
  LIM -- yes --> OK[Handle request]
  OK --> RLS[(Postgres RLS backstop)]
```

## 6. External-channel booking (aggregator)

```mermaid
flowchart LR
  T[Booking taken on aggregator] --> AG[Agent opens Miqwad]
  AG --> NB[POST /dealer/bookings channel=external_aggregator, channel_source=aggregator]
  NB --> BLK{Availability free?}
  BLK -- no --> X[409 vehicle_unavailable]
  BLK -- yes --> WB[Write availability_block]
  WB --> COMP[Run Tajeer + ZATCA + handover]
  COMP --> NOC[No Miqwad commission]
  NOC --> DONE[Booking active in Miqwad ops]
```

## 7. Import / migration with gov-doc guard

```mermaid
flowchart TD
  UP[Upload CSV/Excel per entity] --> VAL[Validate rows]
  VAL --> Q{Row valid?}
  Q -- no --> QU[Quarantine row + error]
  Q -- yes --> STG[Stage import_row]
  STG --> CMT[Commit batch idempotently]
  CMT --> TYPE{Entity type?}
  TYPE -- vehicle/customer/booking/maintenance --> LIVE[Create live records]
  TYPE -- contract/invoice --> HIST[source=imported]
  HIST --> GUARD[Excluded from registration/clearance sagas + reconciliation]
  LIVE --> DONE2[Dealer live]
  GUARD --> DONE2
```

## 8. Observability — request → trace → alert

The path from an inbound request through structured logs, metrics, and traces to alerting on the SLOs.

```mermaid
flowchart LR
  REQ[Inbound request<br/>request_id] --> APP[miqwad-api<br/>structured logs + spans]
  JOBS[JobRunr workers] --> APP
  APP -->|logs| LOG[Cloud Logging]
  APP -->|metrics| MON[Cloud Monitoring]
  APP -->|traces| TR[Cloud Trace]
  MON --> SLO{SLO breach?<br/>p95 latency · error rate}
  SLO -- yes --> ALERT[Alert on-call]
  SLO -- no --> OK[Dashboards]
  LOG --> DASH[Dashboards]
  TR --> DASH
```

## 9. Resilience — external-outage isolation

Each government/payment adapter sits behind a Resilience4j circuit breaker so one outage can't cascade; work parks in the outbox and retries.

```mermaid
flowchart TD
  SAGA[Saga step] --> CB{Circuit breaker<br/>per adapter}
  CB -- closed --> CALL[Call Tajeer/ZATCA/Wasl/Moyasar]
  CALL -- ok --> DONE[Mark outbox dispatched]
  CALL -- error --> RETRY[Outbox retry + backoff]
  CB -- open --> PARK[Park in outbox<br/>fail fast, no cascade]
  PARK --> JR[JobRunr re-attempts on schedule]
  RETRY --> JR
  JR --> CB
  CB -- half-open --> PROBE[Probe one call]
  PROBE -- ok --> CALL
  PROBE -- error --> PARK
```

## 10. DR topology — me-central2 (Dammam)

V1 disaster recovery: single-region me-central2 multi-zone HA + point-in-time recovery + tested restore + in-Kingdom backup. A second in-KSA region for cross-region DR is an open item (O10).

```mermaid
flowchart TB
  subgraph MEC2["GCP me-central2 (Dammam)"]
    direction TB
    subgraph ZA["Zone A"]
      RUNA[Cloud Run]
      SQLA[(Cloud SQL primary)]
      REDA[(Memorystore primary)]
    end
    subgraph ZB["Zone B"]
      RUNB[Cloud Run]
      SQLB[(Cloud SQL standby — HA failover)]
      REDB[(Memorystore replica)]
    end
    GCS[(Cloud Storage<br/>in-Kingdom)]
    BAK[(PITR + backups<br/>in-Kingdom)]
  end
  SQLA -. sync replication .-> SQLB
  REDA -. replication .-> REDB
  SQLA --> BAK
  GCS --> BAK
  BAK -.->|tested restore drill| SQLA
  O10[O10 — second in-KSA region<br/>for cross-region DR: open]:::open
  classDef open stroke-dasharray: 5 5
```

## Diagrams carried forward unchanged

The Use-Case map, Class/Domain model (covered by the ER diagram + the low-level design), the Vehicle-Handover activity, and the External-Outage Resilience sequence remain valid from the original set and are regenerated into this file's Mermaid form when edited.
