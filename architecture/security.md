# Security Architecture

**System-wide security design: how Miqwad authenticates, authorizes, isolates tenants, guards the government-credential vault (the crown jewel), minimizes PCI scope, and governs data under PDPL.**

Pairs with [reliability-dr.md](reliability-dr.md) (observability/DR) and [fraud-trust-safety.md](fraud-trust-safety.md) (economic/behavioural abuse). Cross-links [ADR-003](../decisions/adr-log.md#adr-003--multi-tenancy-shared-schema--dealership_id--tenantid--rls) (tenancy), [ADR-013](../decisions/adr-log.md#adr-013--feature-flags--entitlements-togglz--an-entitlementsubscription-model) (entitlements), [ADR-021](../decisions/adr-log.md#adr-021--auth-keycloak) (auth). Open/provisional items live in [STATUS.md](../STATUS.md).

> **Stack context.** Kotlin / JDK 25 / Spring Boot 4 (Spring Security 7), coroutines-first; multi-tenancy via Hibernate `@TenantId` + Postgres RLS; GCP me-central2 (Dammam) for PDPL residency.

---

## 1. Security principles (non-negotiable)

1. **Least privilege everywhere** — every token, service account, DB role, and bucket grant is scoped to exactly what it needs.
2. **Defence in depth** — no single control is trusted alone. Tenancy is enforced in the ORM *and* the database; payments are tokenized *and* idempotent *and* signature-verified.
3. **Secure by default** — buckets private, DB public-IP off, deny-by-default authorization, TLS-only, secrets never in code or logs.
4. **The credential vault is the crown jewel** — a dealer's Tajeer/Naql credentials and ZATCA CSIDs are the highest-risk asset; their compromise is worse than a data leak, so they get the strongest controls in the system.
5. **Money is never held against a rental that doesn't legally exist** — security and the saga model both serve this invariant.

---

## 2. Authentication ✅

Phone-OTP for customers; RBAC + MFA for staff/platform. The **identity provider is 🔵 Open — Keycloak vs GCIP** (the requirements below hold either way; [STATUS O-2](../STATUS.md), [ADR-021](../decisions/adr-log.md#adr-021--auth)).

| Principal | Mechanism |
|---|---|
| **Customers** | Phone + OTP — rate-limited and attempt-capped; short-lived access token + rotating refresh token; refresh-token reuse detection revokes the whole token family. |
| **Dealer staff & platform users** | The IdP (Keycloak realms or GCIP — O-2) per environment — RBAC, password policy, lockout, and **mandatory MFA for `owner` / `manager` / platform `admin`** roles. |
| **Service-to-service** | Workload Identity (Cloud Run → Google APIs) and Workload Identity Federation (CI → GCP) — **no long-lived key files anywhere**. |

**Tokens.** The JWT carries `sub`, `role`, and `dealership_id`. The API is an OAuth2 resource server; it validates signature, issuer, audience, and expiry on every request. **`dealership_id` is authoritative from the token, never from the request body.**

---

## 3. Authorization — three stacked gates ✅

Every dealer-facing request passes these in order, before any domain logic:

| # | Gate | Enforcement | Denial |
|---|---|---|---|
| 1 | **Tenant scope** | `@TenantId` + RLS bind the request to its `dealership_id` (§4) | RLS rejects rows |
| 2 | **RBAC** | Spring Security authorities mapped from JWT claims; branch-scoping via method security; **deny-by-default** — an unmapped route is forbidden, not open | `403` |
| 3 | **Entitlement** | `EntitlementGuard` resolves the dealership's package/tier → modules/limits ([ADR-013](../decisions/adr-log.md#adr-013--feature-flags--entitlements-togglz--an-entitlementsubscription-model)). A *commercial* gate enforced server-side as a *security* control, so a client can't call a module it hasn't bought | `403 entitlement_package_excluded` / `402 entitlement_limit_reached` |

CRUD endpoints enforce the same via DB-pushed `FilterExpressionCheck`s — never in-memory checks. **Admin impersonation / "view as dealer"** (V1.5) is audited on every use.

---

## 4. Multi-tenant isolation — the worst-case bug ✅

**Threat:** one dealer reads or writes another dealer's data. Isolation is enforced **twice** ([ADR-003](../decisions/adr-log.md#adr-003--multi-tenancy-shared-schema--dealership_id--tenantid--rls)).

- **Layer 1 — Hibernate `@TenantId`:** auto-filters every managed query and auto-populates `dealership_id` on insert; the developer never writes the filter, so they can't forget it.
- **Layer 2 — PostgreSQL RLS (hard backstop):** a policy keyed on `current_setting('app.current_dealership')`, set per transaction via `SET LOCAL`. This catches what the ORM filter misses — **native queries, some fetch paths, and any CRUD misconfiguration**. RLS is non-negotiable and independent of the application code path.
- **Coroutine-safe context:** the tenant id is carried across suspension/thread hops as a coroutine context element (`threadLocal.asContextElement`), with `ScopedValue` for blocking paths. **Background jobs pass `dealership_id` explicitly and never inherit it.** A propagation miss fails *closed* because RLS still rejects the rows.
- **Mandatory test:** the cross-tenant-leak test — set tenant A's session var, query tenant B via repository *and* raw native SQL → **zero rows**. It must pass before any feature work.

**Platform-scoped tables** (`customer`, `platform_user`, `dealership`, `commission_config`) carry no `dealership_id` and sit outside tenant filtering.

---

## 5. The government-credential vault — highest-risk asset ✅

Dealer Tajeer/Naql credentials and ZATCA CSIDs are the crown jewel.

| Aspect | Control |
|---|---|
| **Storage** | Cloud Secret Manager + Cloud KMS (CMEK envelope encryption). Credentials **never** live in the application database, **never** in logs, **never** in code or config files. |
| **Access** | Fetched at call time by the integration adapters, scoped per dealer; every access audited in Cloud Audit Logs. Adapters redact sensitive fields in request/response logs. |
| **Rotation** | Rotatable without redeploy; a suspected exposure triggers the response protocol — **rotate first, investigate second** (§12). |
| **Blast radius** | VPC Service Controls perimeter around Secret Manager + the data services, so even a compromised service account can't exfiltrate secrets off the perimeter. |

---

## 6. Payments & PCI scope minimization ✅

Payments run through **Moyasar** ([ADR-015](../decisions/adr-log.md#adr-015--payments-moyasar)).

- Card data uses the gateway's **hosted fields/SDK** — **card PAN never touches Miqwad servers**, keeping PCI scope to **SAQ-A**.
- Miqwad stores **only gateway references/tokens**. Deposit lifecycle = authorize → capture/refund, all idempotent (**idempotency key on every money call**).
- **Webhooks are signature-verified, replay-protected, and processed once** (idempotent by event id). An unverifiable webhook is rejected and alerted.

> 🟡 Webhook / refund / void / deposit-hold contract detail is provisional pending Moyasar API docs ([STATUS P-5](../STATUS.md)).

---

## 7. Transport, edge & network security ✅

- **TLS 1.2+ / HSTS** terminated at the Global HTTPS Load Balancer; **Cloud Armor** WAF + IP/rate rules in front of Cloud Run (complements app-layer rate limits).
- **Security headers** on web surfaces: `Content-Security-Policy` (nonce-based, no `unsafe-inline` scripts), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` locking camera/mic/geo by default.
- **Private east-west:** Cloud SQL / Memorystore reachable only over private IP via the Serverless VPC connector — **no public DB IP**. Egress to government/payment systems via Cloud NAT with a stable, allow-listed egress IP.
- **VPC Service Controls** perimeter around Cloud SQL, GCS, and Secret Manager to prevent data exfiltration.

---

## 8. Application security — OWASP Top 10 (2021) mapping ✅

| OWASP | Miqwad control |
|---|---|
| **A01** Broken Access Control | Three stacked gates (§3) + RLS (§4); deny-by-default; idempotency keys stop replay |
| **A02** Cryptographic Failures | TLS in transit; CMEK at rest; KYC ID numbers column-encrypted (envelope, KMS); no homemade crypto |
| **A03** Injection | Parameterized JPA/JDBC; Bean Validation at the edge; no string-built SQL; CRUD filters DB-pushed |
| **A04** Insecure Design | Threat-modelled invariants (tenancy, money, sagas); security reviews on auth/payment/vault |
| **A05** Security Misconfiguration | IaC (Terraform) baselines; private-by-default; Security Command Center posture |
| **A06** Vulnerable Components | SBOM + Dependabot + pinned deps + OWASP dependency-check in CI (§11) |
| **A07** Auth Failures | Keycloak (lockout, MFA, password policy); OTP rate/attempt caps; refresh-token reuse detection |
| **A08** Integrity Failures | Signed webhooks; SBOM + pinned deps; forward-only reviewed migrations; outbox integrity |
| **A09** Logging & Monitoring Failures | Structured JSON logs + correlation/trace IDs; audit logs; SLO alerts; Sentry |
| **A10** SSRF | Egress allow-list (only gov/payment/FCM/SMS hosts); no user-supplied URLs fetched server-side |

**Input validation is mandatory at every system boundary** — Bean Validation on DTOs; never trust API responses, webhooks, file contents, or import rows.

---

## 9. Data governance, retention & data-subject rights (PDPL) ✅

**Data classification** (controls scale with class):

| Class | Examples |
|---|---|
| **Restricted** | KYC IDs, government credentials, payment refs |
| **Confidential** | Contracts, invoices, customer PII |
| **Internal** | Operational / business data |
| **Public** | Marketplace listings |

**Retention schedule (authoritative):**

| Data | Retention | Basis |
|---|---|---|
| Handover/return inspection photos | **24 months** (GCS lifecycle) | dispute/evidence window |
| Rental contracts & ZATCA invoices | **≥ 6 years** | ZATCA/tax law |
| Telemetry pings | **90 days hot → 12 months** then prune (pg_partman) | operational + dispute |
| KYC identity data | retained on lawful basis; deleted/anonymized when basis ends | PDPL minimization |
| Audit/ledger events | life of platform (append-only) | financial/audit integrity |

**Data-subject rights (PDPL):** a defined access / export / correction / deletion workflow (partly manual in V1 is acceptable) — export a customer's data on request; delete/anonymize where no lawful basis or regulatory retention requires keeping it. Contracts/invoices are exempt from deletion while the 6-year duty runs. Consent and lawful-basis records are kept.

**Residency:** all personal data stays in-Kingdom (me-central2); backups in-Kingdom. Encryption at rest everywhere; CMEK on the docs/KYC bucket and the credential secrets.

---

## 10. Regulatory obligations beyond the four integrations 🟡

| Obligation | Control |
|---|---|
| **SAMA** (payments oversight) | Payments run through the SAMA-licensed gateway (Moyasar); Miqwad does not hold funds in a way that constitutes regulated money transmission without the right licensing — settlement/escrow design reviewed with legal. |
| **AML / sanctions screening** | Dealer onboarding (CR/TGA/VAT verification) and, where applicable, customer screening; suspicious-activity escalation path. |
| **Consumer protection** | Transparent itemized pricing (rental + deposit + VAT shown separately — a brand promise); clear cancellation/refund terms. |
| **CST — SMS sender-ID & consent** | Registered sender ID, consent capture for marketing messages, opt-out honored; transactional vs marketing separation. |
| **E-invoicing phase obligations** | ZATCA Phase 2 clearance/reporting obligations tracked as they evolve. |

---

## 11. Software supply chain / SBOM ✅

- **SBOM** generated per build (CycloneDX via the Gradle plugin) and stored with the release artifact.
- **Dependabot** (or Renovate) for dependency updates; **pinned dependency versions** (Gradle version catalog + lockfile) so builds are reproducible and a transitive bump can't surprise prod.
- **SAST + secret scanning** in CI; **OWASP dependency-check** for known-vulnerable libraries; container image scanning in Artifact Registry.
- Build provenance via Workload Identity Federation (no static CI keys); images are **promoted, never rebuilt**, across environments.

---

## 12. Security operations & response ✅

- **Monitoring:** Cloud Audit Logs on admin + vault access; Security Command Center posture; anomaly alerts ([fraud-trust-safety.md](fraud-trust-safety.md)) feed the on-call rota.
- **Pen-testing:** a focused third-party pen-test of auth/payment/vault before scaling payment volume (V1.5).
- **Response protocol:** on a suspected secret exposure — **stop, rotate the secret, then investigate**; review the codebase for similar exposure; record a blameless postmortem.

---

## Request flow — the three gates

```
Client + JWT
  → HTTPS LB + Cloud Armor WAF
  → Spring Security resource server (validate sig/iss/aud/exp)
  → Gate 1: Tenant scope (@TenantId + RLS)   — cross-tenant → deny (RLS rejects rows)
  → Gate 2: RBAC (role + branch)             — forbidden    → 403
  → Gate 3: Entitlement (package/tier)       — excluded     → 403 entitlement_package_excluded
                                              — over limit   → 402 entitlement_limit_reached
  → Domain logic
```

---

*See also: [STATUS.md](../STATUS.md) · [decisions/adr-log.md](../decisions/adr-log.md). Tenancy detail in [data-model.md](data-model.md); cloud controls in [cloud.md](cloud.md); fraud/abuse controls in [fraud-trust-safety.md](fraud-trust-safety.md).*
