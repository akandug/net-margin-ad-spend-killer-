# Decision and Governance Register

**Project:** Net-Margin Ad Spend Killer (working name)
**Document type:** Governance record — founder decisions, conditions and outstanding requirements
**Recorded:** 2026-11-17
**Status of this document:** A record of decisions and requirements. It is **not** evidence that any control has been implemented, tested or verified.

> The project brief used the working name "Net-Margin Ad Spend Protection Platform". The founder's approval used "Net-Margin Ad Spend Killer". The canonical product name is **unresolved** and requires founder confirmation. This document uses the founder's latest wording without deciding the matter.

> This project is independent of VocalQuery BI. No architecture, schema, files, business rules or development history from that project are imported here.

---

## 1. Approved governance principles

These were explicitly approved by the founder. They establish **design and governance direction only**; they do not constitute evidence that any control has been implemented or passed testing.

| ID | Principle | Status |
|---|---|---|
| G-G1 | The G0–G4 release-gate discipline is adopted. | APPROVED |
| G-G2 | G2 must pass before any merchant-visible profitability is released. | APPROVED |
| G-G3 | Autopilot is excluded from the MVP. The initial product uses Co-Pilot, alerts and controlled manual actions. | APPROVED |
| G-G4 | INV-01 through INV-25 are required financial-integrity specifications. | APPROVED |
| G-G5 | CT-01 through CT-12 are required tenant-security verification tests. | APPROVED |
| G-G6 | P-01 through P-05 are mandatory provider-verification work items. | APPROVED |
| G-G7 | Real merchant financial data must not be introduced before appropriate security, backup and recovery readiness. | APPROVED |
| G-G8 | Financial calculations must remain deterministic and evidence-backed. AI may explain verified results but must never invent, alter or independently calculate financial facts. | APPROVED |
| G-G9 | Accounting, tax and legal treatments remain subject to qualified professional review. | APPROVED |
| G-G10 | Proposed controls and specifications must never be described as implemented, tested or verified unless evidence demonstrates that status. | APPROVED |

---

## 2. Release gates

| Gate | Purpose | Entry condition | Exit condition |
|---|---|---|---|
| **G0** | Foundation: schema, roles, RLS scaffolding, tenant onboarding, ledger append-only structure | Supabase connected; D-05/D-18/D-20 schema decisions settled | Schema deployed to a non-production environment; role model created; no real merchant data |
| **G1** | Ingestion: webhook authenticity, idempotency, replay protection | P-02 verified against official documentation | Delivery identity, signature verification and duplicate suppression demonstrated on fixtures |
| **G2** | **Financial correctness:** calculation engine, invariants INV-01–INV-25, worked examples A–H reproduced exactly | D-06, D-07, D-08, D-09, D-13, D-18 decided; professional review obtained where required | Every invariant has an executed automated test; examples reconcile to the stated figures |
| **G3** | **Merchant-visible profitability** (blocked by G2) | G2 passed; D-10, D-11, D-12, D-15, D-16, D-17, D-19, D-20, D-21, D-23 decided; P-01, P-03, P-04 verified | Profitability figures shown with provenance labelling; CT-01–CT-12 pass |
| **G4** | **Co-Pilot recommendations** | G3 passed; D-22 approved | Recommendations reproducible from stored inputs, thresholds and calc_version; hard blocks demonstrated |

**Hard rule:** G3 cannot open before G2 closes. No dashboard or recommendation work may outrun financial correctness.

---

## 3. Founder approval matrix — D-05 to D-23

Status vocabulary: **APPROVED** · **APPROVED WITH CONDITIONS** · **DEFERRED** · **OUTSTANDING** · **PROFESSIONAL REVIEW REQUIRED**

**No D decision is currently APPROVED. None is APPROVED WITH CONDITIONS.** The founder approved governance principles, not individual decision options.

| ID | Decision | Recommended option | Status | Gate |
|---|---|---|---|---|
| D-05 | Reporting currency and FX policy | A — single tenant reporting currency, conversion at ingestion, rate and rate date stored on the event | PROFESSIONAL REVIEW REQUIRED | G0 schema / G3 |
| D-06 | Revenue recognition timing | A — management = order date; statutory = shipment / transfer-of-control date | PROFESSIONAL REVIEW REQUIRED | G2 |
| D-07 | COGS timing and inventory valuation | A — moving weighted average, cost snapshot on each sale event, COGS at shipment date, no retroactive restatement | PROFESSIONAL REVIEW REQUIRED | G2 |
| D-08 | Discount allocation | A — pro-rata by line gross, largest remainder | OUTSTANDING | G2 |
| D-09 | Refund basis (net vs gross + reversal) | A — net (Resolution C) | OUTSTANDING | G2 |
| D-10 | Shipping revenue treatment | A — separate revenue component, never netted against shipping cost | PROFESSIONAL REVIEW REQUIRED | G3 |
| D-11 | Shipping cost precedence | A — estimate at shipment, superseded by actual invoice as a revision | OUTSTANDING | G3 |
| D-12 | Fulfilment and packaging cost source | A — merchant-configured rules; pool allocation only as an explicit, labelled fallback | OUTSTANDING | G3 |
| D-13 | Payment processing fee allocation | A — pro-rata over line net revenue including shipping revenue, largest remainder; allocated once at charge time | OUTSTANDING | G2 |
| D-14 | Return provision | A — none in MVP (actual returns only); B post-pilot | **DEFERRED** (conditions in §3.1) | — |
| D-15 | Advertising spend classification | A — three tiers: platform-reported (decision figure), billing-confirmed (reconciliation), settled (deferred) | OUTSTANDING | G3 |
| D-16 | Ad allocation and unallocated spend | A — allocate what is attributable (labelled estimated); retain remainder as store-level cost | OUTSTANDING | G3 |
| D-17 | Tax determination | A — merchant configures jurisdictions, registration ids and tax codes; platform records and reconciles, never computes rates or nexus | PROFESSIONAL REVIEW REQUIRED | G3 |
| D-18 | Accounting-period policy | A — dual management_period + accounting_period (Resolution D, D-R1–D-R10) | PROFESSIONAL REVIEW REQUIRED | G2 |
| D-19 | Materiality and second reviewer | A — per-tenant; default materiality = greater of 1% of period net sales and 5,000 minor units | OUTSTANDING | G3 |
| D-20 | Retention and erasure | A — pseudonymise customer identity in the ledger; crypto-shred the PII key; record an erasure event | PROFESSIONAL REVIEW REQUIRED | G0 schema / G3 |
| D-21 | Estimate labelling and confidence display | A — provenance badge per figure with drill-down; fixed vocabulary; no fabricated confidence percentage | OUTSTANDING | G3 |
| D-22 | Co-Pilot guardrail defaults | A — the seven-guardrail default table (see §3.2) | OUTSTANDING | G4 |
| D-23 | Attribution presentation | A — platform-reported and internal allocation both stored and shown, never blended | OUTSTANDING | G3 |

**Tally:** APPROVED 0 · APPROVED WITH CONDITIONS 0 · DEFERRED 1 · OUTSTANDING 11 · PROFESSIONAL REVIEW REQUIRED 7.

### 3.1 D-14 — conditions attached to the accepted deferral

The founder accepted deferral **only if all three conditions hold**:

1. Return provisions are **excluded** from MVP profitability calculations.
2. Their omission is **clearly disclosed** to the merchant.
3. The product does **not imply** that estimated profitability includes a return provision.

Implementation consequences if the deferral stands: contribution profit is an **actual-returns** figure, and every merchant-facing surface that shows CP must carry the disclosure from condition 2. Return-lag consideration in the Co-Pilot guardrails (D-22) is the compensating control and must remain enabled. Option B (a dimensionally correct provision, separate from actual return costs) requires accounting review before it is built.

### 3.2 D-22 — recommended guardrail defaults (not approved)

| Guardrail | Proposed default |
|---|---|
| Minimum spend before evaluation | greater of 3× target CPA and 50,000 minor units |
| Minimum sample | 20 orders **and** 14 days |
| Attribution grace period | 7 days after last spend |
| Return-lag consideration | 30 days, or the merchant's stated return window |
| Hysteresis | pause below −10% of spend; resume only above +5% |
| Cooldown after a recommendation | 72 hours |
| Hard blocks (no recommendation at all) | missing COGS, data stale beyond 48h, attribution coverage below threshold, open data-quality flag, insufficient spend or sample |

### 3.3 Founder prohibitions recorded

- **D-06:** option C (statutory timing for management reporting) must not be selected without explicit founder approval.
- **D-16:** option B (forcing all spend onto products) must not be selected without explicit founder approval.
- All option sets and recommendations in the delivered register are preserved verbatim; no option has been silently changed.

---

## 4. Provider verification requirements — P-01 to P-05

Verification work is **approved**. Status below reflects evidence actually obtained. **No evidence has been obtained for any item**, because no verification has been performed as of this record.

| ID | Verification question | Evidence required | Evidence obtained | Status | Gate |
|---|---|---|---|---|---|
| P-01 | Is Shopify `refund_line_items[].subtotal` net or gross of discounts; how are order-level discount allocations exposed per line; are shipping and tax refunds separable | Official Admin API / webhook reference; recorded fixture | None | AUTHORISED — NOT STARTED | G2 |
| P-02 | Shopify webhook topics for orders/refunds/fulfilments; signature scheme; header carrying a stable delivery id; retry and ordering behaviour | Official webhook documentation; recorded delivery with headers | None | AUTHORISED — NOT STARTED | G1 |
| P-03 | Meta spend granularity and breakdowns; whether any billing, invoice, settled-charge or credit/adjustment data is available via API at all | Official Marketing API reference | None | AUTHORISED — NOT STARTED | G3 |
| P-04 | Shippo / EasyPost / ShipStation: label-level cost detail, refunds and voids, webhook coverage, rate accuracy, auth model — then recommend one | Official API documentation compared on a fixed checklist | None | AUTHORISED — NOT STARTED | G3 |
| P-05 | Supabase role names and privileges; transaction vs session pooling; audit-log availability; PITR availability | Official documentation, then read-only catalog queries on the connected project | None | AUTHORISED — NOT STARTED | G1 |

**Rules for this work:** official provider documentation and specifications only; recorded example payloads and fixtures; documented fallback behaviour where a provider fact cannot be verified. **No live merchant credentials, no merchant account connections, no provider integrations.**

---

## 5. Outstanding founder decisions

The following must be decided by the founder before the named gate can open. Each retains its exact options and recommendation from the delivered register.

**Blocking G2 (financial correctness):** D-06, D-07, D-08, D-09, D-13, D-18.

**Blocking G3 (merchant-visible profitability):** D-10, D-11, D-12, D-15, D-16, D-17, D-19, D-20, D-21, D-23.

**Blocking G4 (Co-Pilot):** D-22.

**Already deferred:** D-14, on the three conditions in §3.1.

**Also outstanding:** the canonical product name (§ header).

---

## 6. Professional review register

These matters cannot be finalised until a qualified professional has reviewed them. Proposed defaults must not be converted into approved accounting policies.

| Item | Subject | Reviewer |
|---|---|---|
| D-05 | FX rate source and rate date | Accounting |
| D-06 | Revenue recognition timing; statutory treatment of refunds against the original sale period | Accounting |
| D-07 | Inventory valuation method and COGS timing | Accounting |
| D-10 | Whether shipping revenue is taxable, and in which jurisdiction | Accounting / tax |
| D-14 | Provision methodology — **before** option B is built, not before MVP ships | Accounting |
| D-17 | Tax determination boundary; nexus and rate responsibility | Tax / legal |
| D-18 | Statutory period assignment, reopen policy, restatement treatment | Accounting |
| D-20 | Erasure vs financial-record retention obligations | Legal / privacy |

---

## 7. Implementation boundary — not authorised

Until explicitly authorised by the founder, the following remain prohibited:

1. Database schemas or migrations.
2. Row-Level Security or authentication implementation.
3. Connecting Supabase.
4. Creating or modifying application code.
5. Connecting Shopify, Meta Ads, shipping or other merchant providers.
6. Ingesting real merchant data.
7. Merchant billing.
8. Deployment.
9. Bypassing any release gate.
10. Treating outstanding decisions as approved.

---

## 8. Change log

| Date | Change |
|---|---|
| 2026-11-17 | Register created. Founder governance principles G-G1–G-G10 recorded. D-05–D-23 statuses assigned (0 approved, 1 deferred, 11 outstanding, 7 requiring professional review). P-01–P-05 recorded as authorised but not started, no evidence obtained. |
