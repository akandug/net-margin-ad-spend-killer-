# Decision and Governance Register

**Project:** Net-Margin Ad Spend Killer (working name)
**Document type:** Governance record — founder decisions, conditions and outstanding requirements
**Recorded:** 2026-11-17
**Status of this document:** A record of decisions and requirements. It is **not** evidence that any control has been implemented, tested or verified.

> **Canonical product name (decided by the founder, 2026-11-17):** **Net-Margin Ad Spend Killer**, with the descriptive subtitle **Net-Margin Ad Spend Protection Platform**. The subtitle describes the product's purpose; the canonical name governs code, schema comments, documentation and merchant-facing copy.

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

**No D decision is APPROVED in full.** Four are **APPROVED WITH CONDITIONS**: D-15 (manual billing-evidence route only, §3.4), **D-13** (approved in principle on 2026-11-20 — conditions in §3.6, G2 advisory in §3.6.1), **D-20** (architectural principles only, for schema preparation — conditions in §3.7), and **D-10a** (allocation principle approved 2026-11-20 — conditions in §3.5.6). For all four the conditions are substantive and the professional and provider reviews remain open; **no decision is closed**. Conditional architectural approval is not implementation or production-readiness approval.

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
| D-13 | Payment processing fee allocation | A — pro-rata over line net revenue including shipping revenue, largest remainder; allocated once at charge time | **APPROVED WITH CONDITIONS** — in principle 2026-11-20 (§3.6); professional and provider reviews open | G2 |
| D-14 | Return provision | A — none in MVP (actual returns only); B post-pilot | **DEFERRED** (conditions in §3.1) | — |
| D-15 | Advertising spend classification | A — three tiers: platform-reported (decision figure), billing-confirmed (reconciliation), settled (deferred) | **APPROVED WITH CONDITIONS** — manual billing evidence only (§3.4) | G3 |
| D-16 | Ad allocation and unallocated spend | A — allocate what is attributable (labelled estimated); retain remainder as store-level cost | OUTSTANDING | G3 |
| D-17 | Tax determination | A — merchant configures jurisdictions, registration ids and tax codes; platform records and reconciles, never computes rates or nexus | PROFESSIONAL REVIEW REQUIRED | G3 |
| D-18 | Accounting-period policy | A — dual management_period + accounting_period (Resolution D, D-R1–D-R10) | PROFESSIONAL REVIEW REQUIRED | G2 |
| D-19 | Materiality and second reviewer | A — per-tenant; default materiality = greater of 1% of period net sales and 5,000 minor units | OUTSTANDING | G3 |
| D-20 | Retention and erasure | A — pseudonymise customer identity in the ledger; crypto-shred the PII key; record an erasure event | **APPROVED WITH CONDITIONS** — architectural principles only (§3.7); key management and retention OUTSTANDING; professional review open | G0 schema / G3 |
| D-21 | Estimate labelling and confidence display | A — provenance badge per figure with drill-down; fixed vocabulary; no fabricated confidence percentage | OUTSTANDING | G3 |
| D-22 | Co-Pilot guardrail defaults | A — the seven-guardrail default table (see §3.2) | OUTSTANDING | G4 |
| D-23 | Attribution presentation | A — platform-reported and internal allocation both stored and shown, never blended | OUTSTANDING | G3 |

**Tally:** APPROVED 0 · APPROVED WITH CONDITIONS 4 (D-10a, D-13, D-15, D-20 — all conditional) · DEFERRED 1 · OUTSTANDING 8 · PROFESSIONAL REVIEW REQUIRED 6.

4 + 1 + 8 + 6 = 19, which is the full D-05–D-23 set. **No decision has moved to APPROVED in full, and none can without the outstanding professional reviews.**

D-20's move out of PROFESSIONAL REVIEW REQUIRED reflects approval of its **architectural principles only**. Its retention and key-management policy is both outstanding and subject to professional review, tracked in §3.7. D-13's move out of OUTSTANDING reflects approval **in principle** only; its accounting, tax and provider questions are unchanged.

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

### 3.4 D-15 — conditions attached to the approved manual billing evidence

The founder approved **manual invoice/statement import** as the pilot route to billing-confirmed advertising spend. The approval is conditional on all six rules in §4.4.1, and on these limits:

- **Approved:** the manual import mechanism, and the separation of platform-reported, billing-confirmed and settled classifications.
- **Not approved:** any claim that settled spend can be derived from an invoice; any implementation of a Meta invoice or billing API; any silent adjustment of platform-reported spend to match an invoice.
- **Still outstanding:** the remaining D-15 presentation rules and any automated billing retrieval, which remain unverified because no endpoint was established (§4.4).

Leaving these conditions unenforced would let an unvalidated document masquerade as confirmed billing data, which is the specific failure this approval is written to prevent.

---

### 3.5 F-1 — Two-view management design (founder-approved management direction)

**Founder direction, 2026-11-17.** Recorded as a **founder-approved management design direction — not a final statutory accounting policy.**

| # | Approved element |
|---|---|
| 1 | **Commercial Management View** — recognise **both revenue and COGS on the order date**, using the applicable cost snapshot, explicitly labelled as an order-date management measure. **Founder's preferred primary view.** |
| 2 | **Fulfilment View** — recognise **both revenue and COGS on the shipment date**, using the shipment cost snapshot. Available separately. |
| 3 | **Preserve the original order and shipment events and their timestamps.** Neither view may overwrite or discard the date the other depends on. |
| 4 | **Statutory accounting treatment remains separately identified and subject to professional accounting review.** Neither management view establishes statutory compliance. |

**What this resolves.** Cross-decision finding F-1 (`g0-g2-decision-preparation.md`): D-06 placed revenue on the order date while D-07 placed COGS on the shipment date, so a period could show revenue at full margin with its cost landing in the next period. Pairing revenue with COGS on a single event date, per view, removes that mismatch *within* each view.

**What this does not resolve.**

- **D-06 and D-07 remain unapproved as accounting policies.** This is a management-information direction, not a recognition policy.
- **The difference between the two views must never be presented as final realised contribution profit.** A Commercial Management View figure where fulfilment has not occurred is not realised.
- **Neither view may be labelled "contribution profit" without a view qualifier.**

**Engine and schema consequences (design requirements, not implemented):**

1. A calculation must carry its **view** (`commercial` | `fulfilment`) as part of its identity, alongside period and calculation version. A snapshot from one view must never be silently compared with, aggregated into, or substituted for the other.
2. Both views must be derivable from the **same preserved events**. If a view requires data the other discards, the design is wrong.
3. Order-date COGS uses the **applicable cost snapshot at order date**; shipment-date COGS uses the **shipment-date snapshot**. A COGS revision between those dates legitimately produces different figures in each view — that is expected, not an error, and must be traceable to the two snapshots.
4. View is a **reporting dimension**, not a mutation: neither view writes over the other's results.

#### 3.5.1 Quantity state: three independent dimensions (INV-26, revised)

Both views must survive orders that ship in parts, are partly cancelled, and are partly refunded. The founder **rejected the single five-state field** originally proposed here, correctly: fulfilment, physical return and money movement are three different questions answered by three different event streams, and forcing them into one mutually exclusive value makes legitimate combinations unrepresentable — an item can be shipped, returned, restocked, and its refund still pending, all at once.

State is therefore derived in **three dimensions**, each folded from its own events and never collapsed into one field:

| Dimension | Derived from | Values |
|---|---|---|
| **Fulfilment** | shipment and cancellation events | `unfulfilled` · `partially_fulfilled` · `fulfilled` · `cancelled_before_shipment` |
| **Return and physical disposition** | return, inspection and write-off events | `not_returned` · `return_in_transit` · `returned_restockable` · `returned_damaged` · `written_off` |
| **Payment and refund** | refund, refund-transaction and reversal events | `paid` · `refund_requested` · `refund_pending` · `refunded` · `refund_reversed` |

**Within a dimension**, quantities partition the ordered quantity and no unit holds two values. **Across dimensions** they do not — the same unit legitimately holds one value in each, and those combinations carry distinct financial meaning (a restocked return differs from a written-off one; a requested refund differs from a completed one).

No dimension is stored as a mutable column; all three are derived (INV-19). **No reverse transitions** within a dimension; a correction is a new event. Impossible combinations — refunded but never paid, restocked but never returned, written off while still unfulfilled — are integrity violations that must fail closed rather than be rendered.

**INV-26 revised:** for every line, at every point in time, quantities within each dimension sum exactly to the ordered quantity, and no unit occupies two values within a dimension. *Status: the event-derived principle is **founder-approved**; this revised specification awaits approval.* Full specification, including the eight required scenario walkthroughs (partial shipment, partial cancellation, return, damaged return, refund requested, refund pending, refund completed, refund reversed) and the combination-to-financial-treatment matrix: `docs/governance/inv26-quantity-state-specification.md`.

#### 3.5.2 Partial shipments and multiple shipments

**Fulfilment View** recognises per **shipment**, not per order:

- `revenue_shipment` = Σ over lines in that shipment of (net revenue per unit × shipped qty), where net revenue per unit = line gross per unit − allocated discount per unit.
- `discount per unit` = line allocated discount ÷ line quantity, rounded by **largest remainder on the final unit** so the per-unit amounts sum exactly to the line's allocated discount. No balancing plug.
- `COGS_shipment` = Σ over lines of (unit cost from the **shipment-date** snapshot × shipped qty).
- `shipping revenue` is allocated across **quantities at order acceptance**, not across shipments at shipment time: the original shipping revenue is spread deterministically over eligible quantities in proportion to their discounted line net revenue, with the largest-remainder step applied **once at allocation** across all units. Each quantity's share is recognised in the Fulfilment View **when that quantity ships**; unshipped shares stay pending. Cancellations, shipping discounts and shipping refunds are explicit release or reversal events. **No residual is forced onto the final shipment.** This is **D-10a**, revised per founder direction — see §3.5.6.
- `outbound shipping cost` and `fulfilment/packaging` are **naturally per shipment**, so a split order genuinely carries two costs. That is a real economic fact the platform should surface, not an allocation artefact: splitting shipments reduces contribution profit.

**Multiple shipments against one order** are independent events, each with its own date and cost snapshot. The same product may therefore carry **different unit costs across two shipments** of the same order if the snapshot moved between them. That is legitimate and must be traceable to two distinct snapshots — not averaged, and not forced to agree.

**Unshipped units are not recognised in the Fulfilment View.** Their revenue and COGS remain pending, visible as unfulfilled value, and are never dropped.

**Commercial Management View is unchanged by shipment state**: full order net revenue and full order-date COGS on the order date. A partial shipment does not alter it. This is the defining difference between the views and the reason the difference must be labelled a timing difference.

#### 3.5.3 Partial cancellations and partial refunds

A cancellation or refund is **its own event on its own date**. It never rewrites the original order-date or shipment-date figure (INV-16, INV-19). Its effect is recognised in each view **according to what that view had already recognised**:

| Event | Commercial Management View | Fulfilment View |
|---|---|---|
| Unit refunded/cancelled **before shipment** | Revenue and order-date COGS were recognised at order date → both reversed on the event date | Nothing was recognised → reduces the pending amount; **no revenue and no COGS reversal** |
| Unit refunded **after shipment** (return) | Reversal on the event date | Revenue and COGS were recognised at shipment → both reversed on the event date |

**The same refund therefore has different view-level effects.** That is correct, not an inconsistency, and it is precisely why the views cannot be merged into one measure.

The refund's merchandise component is bounded by INV-02 at line level and reduced by the unit's allocated discount per the basis established by **F-01** — which is **still open**. Until the captured payload settles whether `subtotalSet` is net or gross of discount, the engine runs the labelled interim assumption recorded in §4.9, and D-09 remains unresolved.

**Period assignment (D-18 interaction).** Reversals land in the period of the event, not the period of the order. A cancellation of a prior-period order is an adjustment in the current period. Whether that is the correct *statutory* presentation is a professional accounting question (§6); the management design records both dates and does not restate silently.

#### 3.5.4 Preventing accidental aggregation or comparison across views

Six enforcement rules (design requirements, not implemented):

1. `calculation_view` is part of **calculation identity** alongside period and calculation version.
2. Snapshot keys include the view. **A read without a view fails closed** — there is no silent default.
3. Every aggregation must group by view. Combining rows from two views is an **error**, never a union.
4. A deliberate cross-view comparison is presented only as a **timing difference**, reconciled by the identity in §3.5.5 — never as a blended or averaged figure.
5. No "combined" or "average of both views" figure is ever stored or displayed.
6. Merchant-facing labels carry the view qualifier. "Contribution profit" without a qualifier is **prohibited** in UI, exports and API responses.

#### 3.5.5 Proposed INV-27 — view reconciliation identity

For a period P and scope S, the two views must differ by an amount **fully explainable by named unit sets**:

```
CP_commercial(P) − CP_fulfilment(P)  =  A(P) + B(P) + C(P)

A = for units still `pending` at period end:
      net revenue recognised in commercial only  −  order-date COGS
B = for units shipped in P:
      shipment-date COGS  −  order-date COGS        (the snapshot difference)
C = period-boundary timing on units refunded or cancelled before shipment
      in a period later than their order date
```

If the left side cannot be reconciled to A + B + C from the event history, the calculation is defective and must fail rather than present either view's figure. *Status: **founder-approved architectural principle**, 2026-11-20 — specification and test design outstanding.* Full identity tuple, fail-closed rules and the reproducible reconciliation artifact: `docs/governance/inv27-calculation-identity-specification.md`.

**Dependencies and remaining gates:** D-06 and D-07 still block **G2** (§5); statutory treatment stays in the professional review register (§6); the view dimension must exist in the G2 schema before any merchant-visible profitability (G3).

**Raised by this extension — one new sub-decision for founder approval:**

| ID | Sub-decision | Proposed default | Status |
|---|---|---|---|
| **D-10a** | How shipping revenue charged once is attributed across multiple shipments of one order | Allocate the original shipping revenue over **quantities** at order acceptance, pro-rata by discounted line net revenue, largest remainder resolved once across all units; recognise each quantity's share on shipment; unshipped stays pending; cancellation / shipping discount / shipping refund are explicit release or reversal events; no residual on the final shipment | **APPROVED WITH CONDITIONS (principle, 2026-11-20)** — conditions C-10a.1–C-10a.5 and the professional-review items remain open (§3.5.6). **C-10a.1 audited 2026-11-20: PARTIALLY SATISFIED — OPEN**; D-10a therefore currently blocks G3 (§5) |

Also noted: D-11 (estimated outbound shipping) and D-12 (fulfilment and packaging) must key on **shipment**, not order, for the same reason.

#### 3.5.6 D-10a — revised allocation rule (re-submitted)

The founder **rejected** the residual-on-final-shipment rule. It was wrong for a reason worth recording: the last shipment became a balancing plug, so its recognised revenue depended on the order of fulfilment rather than on its own contents, and any change to shipment sequencing silently moved revenue between shipments.

The revised rule allocates **at order acceptance, down to individual quantities**, so no unit's share depends on how the order is later split:

1. Determine eligible quantities — those not cancelled and not free.
2. Weight each eligible quantity by its **discounted line net revenue** (line gross per unit less allocated discount per unit).
3. Allocate the original shipping revenue by that weight, with the **largest-remainder step applied once, across all units at allocation time** — so the allocated shares sum exactly to the shipping revenue with no plug anywhere.
4. Recognise each quantity's share in the Fulfilment View **when that quantity ships**. Unshipped shares remain pending and visible.
5. Cancellation, shipping discount and shipping refund are **explicit release or reversal events** on their own dates, never edits to the original allocation.

Full worked examples — multiple shipments, partial cancellation, free shipping, shipping discount, multiple shipping charges, incomplete fulfilment and shipping refund — in `docs/governance/d10a-shipping-revenue-allocation-decision-brief.md`.

#### D-10a — founder approval record (2026-11-20)

**Previous status:** OUTSTANDING (re-submitted after the residual-on-final-shipment rule was rejected). **Status now:** **APPROVED WITH CONDITIONS** — the allocation *principle* only.

**What the founder approved:** shipping revenue is allocated **once at order acceptance** across eligible individual quantities by deterministic largest-remainder arithmetic; each quantity retains its original allocation identity; in the Fulfilment View that quantity's allocated shipping revenue is recognised **when that quantity ships**; cancellations, shipping discounts, subsequent shipping-charge changes and shipping refunds are represented by **explicit adjustment, release or reversal events**; the original allocation is never silently rewritten; **no unexplained residual is assigned to the final shipment.**

**Conditions (all testable, none yet tested):**

| ID | Condition |
|---|---|
| C-10a.1 | The complete specification and all worked examples are **internally consistent** — the same inputs cannot yield different allocations across documents |
| C-10a.2 | Allocation is computed **once**, at order acceptance, and never recomputed later from changed inputs |
| C-10a.3 | Each quantity's share is carried as a distinct identity that survives shipment, cancellation, refund and partial fulfilment without reallocation |
| C-10a.4 | Unshipped allocations remain **pending**, never recognised early and never released silently |
| C-10a.5 | Every change after acceptance — cancellation, shipping discount, charge change, shipping refund — produces an explicit event; no in-place edit of a prior allocation |

**C-10a.1 audit outcome (2026-11-20) — PARTIALLY SATISFIED, REMAINS OPEN**

The independent consistency audit (`docs/governance/d10a-c10a1-consistency-audit.md`) recomputed all seven worked examples from the brief's own rules R1–R8 using exact rational arithmetic, independently of the printed figures.

- The seven worked examples are **arithmetically correct** under their stated assumptions.
- The **anti-balancing-plug requirement is satisfied** — no example assigns an unexplained residual to the final shipment.
- **D-01, D-02 and D-03 remain unresolved specification defects:** the association-basis selector is unspecified and absent from the calculation identity (D-01); E2 is valid only as a fork of E1, not a continuation (D-02); R6 contradicts C-10a.5 on post-acceptance shipping discounts (D-03).
- The additional **shipping-refund ceiling (D-04), multi-charge reconciliation (D-05), allocation-base ambiguity (D-06) and missing scenario coverage (D-07)** must be addressed.
- **D-10a remains APPROVED WITH CONDITIONS.** C-10a.2–C-10a.5 are unaffected by the audit and remain open.
- **The G3 proviso is not currently satisfied.** §5 previously recorded that D-10a no longer blocks G3 *provided* C-10a.1–C-10a.5 are enforced and the specification remains internally consistent. Because C-10a.1 is not satisfied, that proviso fails; the G3 position is restated in §5.

This is a **status update, not a reversal of the founder's approval.** No test has been executed. Every statement above concerns specification consistency only.

**Substantive correction briefs prepared, not applied and not approved:** `p1-shipping-association-selector-decision-brief.md`, `p3-post-acceptance-shipping-discounts-decision-brief.md`, `p4-shipping-refund-ceiling-decision-brief.md`, consolidated in `d10a-p1-p3-p4-founder-decision-package.md`. The editorial corrections P-2, P-6, P-8, P-9 and P-10 are proposed as a reviewable diff in `p-editorial-change-set-proposed-diff.md` and are **not applied**; P-5 and P-7 are excluded from that batch (see the package, §5).

**Still open (unchanged by this approval):** whether shipping revenue is a separate performance obligation for statutory revenue recognition (professional accounting review); tax treatment of shipping charged to customers, including whether shipping tax belongs in the allocation base (professional tax review); treatment of shipping revenue on orders where shipping is charged but goods are never shipped; and the FX basis for shipping revenue in a non-reporting currency.

**Not authorised by this approval:** any engine, schema, migration or test implementation.

---

### 3.6 D-13 — founder approval in principle (2026-11-20)

**Previous status:** OUTSTANDING. **Proposed status:** APPROVED WITH CONDITIONS. **Founder direction:** the 63 minor-unit residual remains an explicit order-level cost rather than being artificially allocated to products; the 444 product-attributable variable fee is allocated by the approved deterministic method.

Seven conditions are attached, each a testable engine property: preserve the complete processor fee; preserve the product-attributable component separately; preserve the order-level residual separately and drillably; **no artificial zero-residual reconciliation**; reconcile against actual processor evidence where available; distinguish estimated / provisional / confirmed fees; and support better evidence later **without silently changing historical results**.

**One clarification the conditions force, recorded because the earlier wording invited double-counting:** the 63 is *inside* the 444, not beside it.

```
fee_total 507
  ├─ tax-related    33   order-level, never in the product allocation base
  ├─ fixed platform 30   order-level, never in the product allocation base
  └─ variable      444   the allocation base
       ├─ to products 381
       └─ residual     63   (12.4% of 507; 14.2% of 444)
```

444 + 33 + 30 = 507 ✓. Reporting 444 allocated *plus* 63 residual would double-count 63 — exactly what C-13.3 and C-13.4 exist to prevent.

**Still open (unchanged):** fee recognition timing across periods; fixed-component classification and monthly platform fees; true-up presentation; fee-refund presentation where the fixed component is retained (INV-22); tax treatment of the tax-related fee; **whether the processor exposes an actual per-charge fee** (unverified — no provider documentation checked and no payload recorded); FX basis for a settlement-currency fee. Professional accounting and tax review remain required.

**Location:** `docs/governance/d13-payment-fee-allocation-decision-brief.md` §10.

### 3.6.1 D-13 — G2 advisory (founder direction, 2026-11-20)

D-13 remains **APPROVED WITH CONDITIONS in principle**, but the founder directed that its unresolved fee-recognition questions be carried as an **explicit G2 advisory** rather than dissolved into the approval. G2 cannot be declared passed while the advisory is neither cleared nor formally accepted as a disclosed limitation.

**Advisory content — the questions that stay open:**

1. **Fee recognition timing** — whether processor fees are recognised at order time (estimated), at capture, or at settlement, and how a fee recognised in one period and settled in another is presented without restatement.
2. **Accounting treatment** of the order-level residual: is it a period cost, a cost of the order, or part of a fulfilment-cost pool?
3. **Tax treatment** of the tax-related component (the 33 in the worked example): is a fee charged on tax deductible in the same proportion as the underlying tax?
4. **Provider evidence** — whether the processor exposes an actual per-charge fee at all. **Unverified.** No provider documentation has been checked and no payload recorded.

**Arithmetic that must be preserved exactly, and must never be presented as additive:**

```
444 = 381 product-attributable variable fee + 63 order-level residual
507 = 444 + 33 variable fee on tax + 30 fixed fee
```

The 63 is **inside** the 444. **Never display 444 allocated plus 63 residual as though they were separate additions to the total fee** — that double-counts 63 and would overstate processor cost by 12.4%. This is a presentation rule with a financial consequence, not a formatting preference.

### 3.7 D-20 — founder-approved architectural principles and the open policy

**Previous status:** PROFESSIONAL REVIEW REQUIRED. **Proposed status:** APPROVED WITH CONDITIONS, principles only.

**Approved:** separation of the immutable financial ledger, erasable identity data and audit records; **random** pseudonymous references rather than email hashes; and the requirement that subject-level key destruction or irreversible revocation **must survive restoration of an older PostgreSQL backup**.

**Still open:** the key-management selection, all retention periods, and the professional review. Per founder direction, **no retention period is asserted anywhere** in this register or its appendices; periods are jurisdiction-specific legal, tax and accounting determinations, and data minimisation applies rather than retaining all personal information for the longest financial-record period.

The specification required before implementation is delivered at `docs/governance/d20-key-management-and-erasure-restore-specification.md`: six key-management options compared, the restore threat scenario, eight protection controls, the durable erasure reconciliation mechanism, and a cost assessment by **structure** rather than invented vendor prices. Its recommendation is a hybrid — register-based blocking for immediacy plus external key destruction for finality — with **no paid KMS selected**.

**Two findings in that specification deserve founder attention now:**

1. **Key material inside the database cannot survive a restore** — including in-database secret facilities, because they are inside the same backup scope. Any such design fails the approved principle by construction.
2. **Erasure must also block re-establishment.** A restored backup restores the identity row; the next webhook then writes *new* personal data for an erased subject, and no read-blocking control fires. Erasure needs a tombstone consulted on **writes**, not only reads.

#### D-20 — founder conditions (2026-11-20), binding

The founder approved the **hybrid architectural direction for schema preparation**: external subject-level key destruction or irreversible revocation, an independently protected erasure register/tombstone, and identity indirection between the financial ledger and erasable identity information. Seven conditions attach, each mapped to controls and tests in the new specification:

| Condition | Requirement | Founder letter |
|---|---|---|
| C-20.1 | Key material and the authoritative erasure record must **not depend solely on PostgreSQL or its backup scope** | a |
| C-20.2 | The erasure register must **resist rollback** and must not be resurrected in an earlier state by a database restore | b |
| C-20.3 | **All** ingestion, webhook and identity-creation paths must consult the register **before** writing personal information | c |
| C-20.4 | Missing or unavailable erasure evidence must cause the affected identity write to **fail closed** | d |
| C-20.5 | A database restoration must trigger **erasure reconciliation before** new identity writes are permitted | e |
| C-20.6 | Delayed webhooks, replayed events and integration retries must **not recreate** erased personal information | f |
| C-20.7 | The architecture must explain how erased subjects are **recognised without retaining their erased PII** | g |

**Scope limits of this approval — restated so it cannot drift:** this is approval of an architectural direction and of the requirement to support identity indirection in future schema design. It is **not** approval of a specific KMS vendor, a paid service, final key-management implementation, any retention period, or production deployment. **No retention period is asserted anywhere in this register**; periods are jurisdiction-specific legal, tax and accounting determinations, and data minimisation applies rather than retaining all personal information for the longest financial-record period.

**Required specification, now delivered:** `docs/governance/d20-restore-and-erasure-threat-model-and-test-specification.md` — 14 threats (T-01–T-14), 7 restore scenarios (R-01–R-07), the monotonic-sequence reconciliation rule, the recognition-without-retention design, condition-to-control-to-test traceability for (a)–(g), 22 test cases (TE-01–TE-22), an evidence standard that must be met before any test may be marked passed, and five open questions requiring founder decision.

**Three findings in that specification deserve founder attention now:**

1. **A rolled-back register must block service, not reconcile silently.** If the database claims to have applied erasure 412 while the register's maximum is 407, the *register* has lost state. The correct response is to refuse identity writes **and** merchant-visible reads, not to guess which side is right. A design that only handles "database is behind" misses this case entirely.
2. **Recognition without retention has a bounded weakness.** Provider customer identifiers are frequently small integers, so a keyed hash alone becomes enumerable if the matching key leaks. The specification adds a per-tombstone salt and a deliberately slow KDF, and states the residual risk plainly rather than claiming zero. `K_match` must be treated as a top-tier secret with rotation support.
3. **The likeliest erasure failure is not a restore — it is scope creep.** Honouring an erasure request by deleting financial events would destroy auditability (T-14). Erasure scope must be **identity only, always**, and the ledger must be structurally incapable of mutation (INV-19).

---

## 4. Provider verification — P-01 to P-05

Verification work was **authorised by the founder and executed on 2026-11-17 as documentation research only**. Every finding below is drawn from official provider documentation read during that work. **No API call was made, no credential was used, and no merchant, sandbox or production account was connected.** A documented field is not proof of live behaviour in a merchant's account.

**Evidence vocabulary (founder-specified):** VERIFIED FROM OFFICIAL DOCUMENTATION · PARTIALLY VERIFIED · NOT VERIFIED · REQUIRES LIVE INTEGRATION TEST · REQUIRES FOUNDER DECISION.

### 4.1 Status matrix

| ID | Verification question | Status | Gate | Practical consequence |
|---|---|---|---|---|
| P-01 | Refund line subtotal net/gross of discounts; per-line discount exposure; separable shipping and tax refunds | **PARTIALLY VERIFIED** — structure verified, discount basis unresolved | G2 | GraphQL Admin API is the primary specification; refund maths is not finalised until the F-01 fixture confirms the basis (§4.9) |
| P-02 | Webhook signature, stable delivery id, retries, ordering, duplicate handling | **VERIFIED** — signature and delivery identity from research; retry and ordering behaviour corrected by the founder against official documentation (§4.3) | G1 | Delivery is duplicate-capable and unordered with bounded retries; reconciliation is mandatory |
| P-03 | Meta spend granularity; availability of invoice, settled-charge or credit data | **PARTIALLY VERIFIED** — delivery spend verified; billing/settled data not found | G3 | MVP may only claim platform-reported spend; billing-confirmed arrives through the approved manual import process (§4.4.1) |
| P-04 | Label-level cost, refunds/voids, webhook coverage, auth, rate vs actual | **PARTIALLY VERIFIED** — Shippo and EasyPost verified on the checklist; ShipStation not verified | G3 | Shippo recommended; ShipStation cannot be recommended on current evidence |
| P-05 | Supabase roles, RLS behaviour, pooling, audit logs, backups/PITR | **PARTIALLY VERIFIED** — RLS and pooling topology verified; role names, audit retention, PITR not verified | G1 | RLS design stands; all live checks deferred by the founder until G0 settles (§4.6.1) |

### 4.2 P-01 — Shopify refunds and discounts

**Source:** Shopify Admin API, *Refund* resource — https://shopify.dev/docs/api/admin-rest/latest/resources/refund (version selector shows 2026-10; the same page records that the REST Admin API became legacy on 2024-10-01 and that new public apps must use the GraphQL Admin API).

Verified structure of a refund:

- `refund_line_items[]` — `id`, `line_item_id`, `quantity`, `restock_type` (`no_restock` | `cancel` | `return` | `legacy_restock`), `location_id`, `subtotal`, `total_tax`, `subtotal_set`, `total_tax_set`.
- `refund_shipping_lines[]` — `subtotal_amount_set` and tax set. The page states that as of 2024-10 refunded shipping is reported here rather than in `order_adjustments`.
- `order_adjustments[]` — `kind` of `shipping_refund` or `refund_discrepancy`; using it for shipping refunds is deprecated as of 2024-10.
- `refund_duties` — `FULL` or `PROPORTIONAL`, plus `total_duties_set`.
- `transactions[]` — the refund's payment transactions, which is the only place a processor fee reversal can be evidenced.
- The nested `line_item` carries `discount_allocations[]` (`amount`, `discount_application_index`) and `tax_lines[]`.

#### 4.2.1 GraphQL Admin API is the primary integration specification

The **GraphQL Admin API is the primary specification** for this integration. The REST findings recorded above are retained only as **supporting historical evidence**, because the REST Admin API became legacy on 2024-10-01 and new public apps must use GraphQL.

- Field semantics are to be established against the GraphQL schema — `Order`, `Refund`, `RefundLineItem`, `OrderAdjustment`, `DiscountApplication`, `OrderTransaction` — before any ingestion contract is written.
- The REST field names above are a **cross-check**, not the contract.
- Where the two disagree, GraphQL governs, and the disagreement is itself recorded as an evidence item.

#### 4.2.2 A refund record is not proof of repayment

A Shopify refund object records an **intended or requested** reversal. It must never be treated, on its own, as evidence that cash was returned to the customer. The financial engine must distinguish three things:

1. the **refund record** — line items, quantities, tax, shipping, adjustments, duties;
2. the **refund transaction** — the processor's own record, carrying its state and amount;
3. the **settled repayment** — a successful transaction and, where available, the processor's settlement evidence.

Contribution profit may recognise a refund as **actual** only when a refund transaction evidences it. A refund record with no successful transaction is **provisional** and must be labelled as such; a pending or failed transaction blocks promotion to actual. This is INV-14 (no silent conversion of estimated into actual) applied to refunds, and it is why the refund fixture in §4.9 captures the transaction separately from the refund record.

**Unresolved:** the reference describes `subtotal` only as "the subtotal of the refund line item". It does not state whether that figure is net or gross of the original discount, nor whether the nested `discount_allocations[]` represent the original order allocation or the reversed portion. **This is the single most important open question in P-01** because D-09 and INV-02 depend on it. Status: REQUIRES LIVE INTEGRATION TEST against the F-01 fixture (§4.9).

**Also not verified:** the order-level discount `allocation_method` (`ACROSS` | `EACH` | `ONE`) that governs how an order discount spreads across lines. The GraphQL `DiscountApplication` reference URL returned 404 during this research, so the fact is recorded as NOT VERIFIED rather than assumed. D-08's pro-rata recommendation stands independently, but the mapping from Shopify's own allocation to ours must be confirmed before G2.

### 4.3 P-02 — Shopify webhooks

**Source:** Shopify *Webhooks — HTTPS delivery* — https://shopify.dev/docs/apps/build/webhooks/subscribe/https

- **Signature:** HMAC-SHA256 over the **raw request body**, base64-encoded, in header `X-Shopify-Hmac-Sha256`, keyed with the app's client secret. Verification must use the unmodified body bytes; re-serialising JSON breaks the comparison, and comparison must be constant-time.
- **Delivery identity:** `X-Shopify-Webhook-Id` identifies a delivery; `X-Shopify-Event-Id` identifies the event. The documentation notes deliveries can be duplicated and advises deduplicating on these identifiers — exactly the stable identity INV-12 requires, and superior to any ordinal.
- **Other headers:** `X-Shopify-Topic`, `X-Shopify-Shop-Domain`, `X-Shopify-Triggered-At`, `X-Shopify-API-Version`.
- **Registration:** REST webhook registration is deprecated; subscriptions are created through the GraphQL Admin API, and mandatory compliance topics apply to public apps.

**Delivery characteristics — corrected by the founder against official documentation:**

- Shopify retries failed deliveries **up to eight times over a four-hour period**.
- Shopify **does not guarantee event ordering**.
- **Duplicate deliveries are possible.**
- Shopify **may stop retrying** once the retry limit is reached, so an event can be lost entirely.

**Delivery is therefore described as duplicate-capable, unordered delivery with bounded retries and mandatory reconciliation.** It is **not** to be described as guaranteed at-least-once, because a bounded retry limit permits permanent loss.

**Design consequences:** deduplicate on `X-Shopify-Webhook-Id` and `X-Shopify-Event-Id`; make every handler idempotent and order-independent; never let a webhook close a financial period; and run a **mandatory scheduled reconciliation sweep** per resource (orders, refunds, fulfilments) that re-reads provider state and ingests anything missed. A missed webhook must never mean a missed financial event.

### 4.4 P-03 — Meta advertising spend

**Sources:** *Ads Insights API* — https://developers.facebook.com/docs/marketing-api/insights/ (updated 2026-04-27); *Ad Account* reference — https://developers.facebook.com/docs/marketing-api/reference/ad-account/ (v25.0, updated 2026-08-03).

- **Delivery spend is verified:** the `spend` metric is returned as a string at ad-account, campaign, ad-set and ad level, with `date_start`/`date_stop`, `date_preset`, attribution-window parameters and asynchronous report jobs; the `ads_read` permission is required.
- **Billing data is not verified, and its absence is the finding:** the current Ad Account reference does not expose a `transactions` edge, and the `ad-account/transactions` reference returned 404 in both URL forms attempted. **No invoice, settled-charge, credit or payment-reconciliation endpoint could be established.**

**Consequence for D-15:** the three-tier model stays, but only tier 1 is reachable through the API. Platform-reported spend is the decision figure and must be labelled provisional. Billing-confirmed spend may only be populated from merchant-supplied invoices or statements (manual import), and settled spend remains unavailable. **No field is to be invented because it would be commercially convenient.**

**Second consequence for D-18:** Meta reports spend on the platform's own day boundary, which need not match the merchant's accounting day or the invoice charge date. Platform-reported spend and any future billing figure must therefore be stored as separate period-scoped facts, never reconciled by overwriting one with the other.

#### 4.4.1 Manual billing evidence — approved for the pilot

The founder has approved **manual invoice/statement import** as the route to billing-confirmed advertising spend for the pilot. It is a documented evidence process, not an API integration.

1. Platform-reported, billing-confirmed and settled spend remain **separate classifications** and are never merged into one figure.
2. An imported invoice may support **billing-confirmed** spend only after validation and reconciliation against platform-reported spend.
3. An imported invoice **never** establishes **settled** spend on its own.
4. The original evidence document is preserved with its **source, reporting period, currency, import date, amount and reconciliation status**.
5. Duplicate imports are **prevented** by a stable identity over source document and period.
6. Unmatched differences **remain visible and unresolved**; platform-reported spend is never silently adjusted to match an invoice.

**No Meta invoice API is to be invented or implemented.** The absence of a verified endpoint (§4.4) stands.

**Not verified:** product-level or catalog/DPA spend attribution. It is listed in Meta's documentation as a reporting capability but the exact fields were not established here, so D-16's allocation logic must not assume it exists.

### 4.5 P-04 — Shipping provider evaluation

**Shippo** — https://docs.goshippo.com (refunds, webhook security, Reporting API `invoice.v1`):

- Refunds are requested **one label at a time**, with statuses `QUEUED`, `PENDING`, `SUCCESS`, `ERROR`; labels are refundable within 90 days; processing is documented as roughly 14 business days; the refund covers label cost, insurance and tracking fee and appears as a **negative line item on the next invoice**.
- The Reporting API's `invoice.v1` returns one row per **invoice item**, carrying `invoice_object_id`, `invoice_number`, `invoice_status`, `invoice_ready_for_charge_at`, `invoice_paid_at`, `invoice_total_amount`, `invoice_charged_amount`, plus `invoice_item_type`, `invoice_item_amount`, `invoice_item_charged_amount`, the FX rate used, `transaction_object_id`, `shippo_account_id` and `is_return`. Refund and credit rows carry negative amounts; one run covers up to 92 days; open and write-off invoices are excluded; data is refreshed periodically.
- Webhook authenticity is documented as IP allowlisting, a per-endpoint auth token and an HMAC signature (`Shippo-Auth-Signature`, `t=`/`v1=`), with the HMAC option requiring a documented setup lead time.

**EasyPost** — https://docs.easypost.com/docs/refunds and /docs/webhooks:

- The `Refund` object is **immutable once created**, links to `shipment_id` and `tracking_code`, and carries carrier-reported status `submitted` | `refunded` | `rejected`. USPS labels are refundable within 30 days if unscanned, with at least 15 days' processing and EasyPost fees also refunded; UPS and FedEx within 90 days.
- Webhooks support HMAC validation via `X-Hmac-Signature` using a `webhook_secret`, or basic authentication; HTTPS is required, a 2XX response is required, and endpoints failing repeatedly may be automatically disabled.

**ShipStation** — https://docs.shipstation.com: the documentation site was reachable and describes label generation, rate comparison, tracking webhooks and return labels, but **label-level cost fields, void/refund semantics and any invoice or billing report were not verified.** Status: NOT VERIFIED.

#### 4.5.1 Evidence preservation and refund lifecycle

- Preserve the **original Shippo invoice item and transaction identifiers** verbatim (`invoice_object_id`, `invoice_number`, `transaction_object_id`, `invoice_item_*`), so every derived figure can be traced back to supplier evidence.
- Five states must be distinguished and never collapsed: **label creation** (label purchased), **charge** (amount billed on an invoice item), **refund request** (submitted), **pending refund** (`QUEUED`/`PENDING`), and **completed refund** (`SUCCESS`).
- A **pending refund is not a completed supplier credit.** It is carried as an expected credit, never netted against cost as though received; only a completed refund, evidenced in supplier billing, reduces cost.
- **Supplier invoice items must not be assumed allocatable to individual orders.** `invoice.v1` links a row to a transaction, but invoice-item-to-order allocation is usable only where a separately evidenced allocation method exists. Until then, store-level reconciliation is the honest ceiling and per-order allocation of invoice items stays unverified.

**Recommendation: Shippo — provisional.** It remains the MVP recommendation because `invoice.v1` is the only verified mechanism that returns the **actual charged amount per transaction**, including negative refund and credit rows and an `is_return` flag. That is what worked example F needs to supersede an estimated outbound shipping cost with an actual one without mutating the estimate (INV-14), and what lets store-period shipping cost reconcile to a supplier invoice (INV-21). The recommendation is **provisional and subject to final commercial and integration validation**, including Shippo account requirements, pricing, webhook HMAC enablement lead time and the invoice-to-order allocation question above. EasyPost is the fallback, with the cleanest refund semantics but no verified equivalent billing report. ShipStation cannot be recommended on the evidence gathered.

### 4.6 P-05 — Supabase and PostgreSQL security

**Sources:** *Row Level Security* — https://supabase.com/docs/guides/database/postgres/row-level-security (updated 2026-11-03); *Connect to your database* — https://supabase.com/docs/guides/database/connecting-to-postgres (updated 2026-11-06); *Supavisor* — https://supabase.com/docs/guides/database/supavisor (updated 2026-07-29).

- **Grants and policies are independent checks.** Adding a policy does not revoke a grant, and enabling RLS does not by itself restrict anything if privileges remain. New tables in the `public` schema may receive broad grants to `anon`, `authenticated` and `service_role`, so the design must revoke first and grant narrowly — this is the documented basis for the least-privilege roles in §4.4 of the architecture review.
- **RLS must be enabled per table**, and `FORCE ROW LEVEL SECURITY` is what makes the policy apply to the table owner as well.
- **`service_role` bypasses RLS entirely.** It must never reach a client, and it must not be the role used by ingestion or workers.
- **Views bypass RLS by default** unless created with `security_invoker = true`; **security-definer functions owned by `postgres` also bypass RLS**. Any reporting view over financial data is therefore a tenant-isolation hazard unless it is explicitly built otherwise.
- **Testing:** pgTAP tests live in `supabase/tests/*.sql` and run through `supabase test db`, which is a viable home for CT-01–CT-12.
- **Pooling:** the connection guide distinguishes direct connection, session pooler and transaction pooler (Supavisor); transaction pooling is the documented choice for serverless or short-lived connections, and the Supavisor page ties the maximum client count to the compute add-on.

#### 4.6.1 Live checks deferred by the founder

All live Supabase checks are **deferred**. **No database is to be connected and no credential requested** at this stage. The following remain **outstanding** until G0 decisions are settled and the founder authorises project provisioning and access:

- Actual database roles and grants.
- RLS and `FORCE ROW LEVEL SECURITY` behaviour on the real tables.
- Transaction-local tenant context (`SET LOCAL`) under the selected pooler.
- Cross-tenant isolation tests (CT-01–CT-12) against a live database.
- Audit logging configuration and retention.
- Actual backup and point-in-time recovery availability on the selected plan.

Until then, every statement in this section is **documented behaviour**, not observed behaviour on this project. In particular, whether `SET LOCAL` tenant context survives transaction pooling **on this project's pooler** is unproven: PostgreSQL semantics are transaction-scoped, but pooler behaviour requires a live test.

### 4.7 Decisions and invariants affected by these findings

| Item | Effect of the evidence |
|---|---|
| D-08 Discount allocation | Recommendation unchanged; Shopify's own `allocation_method` mapping remains unverified |
| D-09 Refund basis | **Cannot be closed** until a fixture settles net vs gross in `refund_line_items[].subtotal` |
| D-11 Shipping cost precedence | **Strengthened** — Shippo `invoice.v1` gives a verified actual-charge source, so option A (estimate superseded by actual revision) is now evidenced |
| D-15 Advertising classification | Tier 1 only via API; tiers 2–3 depend on merchant-supplied evidence or future verification |
| D-16 Ad allocation | Product-level Meta spend must stay unverified; unallocated remainder stays visible |
| D-18 Period policy | Reinforced: platform spend date and supplier invoice date differ, so dual periods are necessary rather than cosmetic |
| INV-02 | Refund ceiling formula stands, but its input basis is blocked on P-01 |
| INV-03 | Supported: refund tax and shipping tax are separately exposed |
| INV-12 / INV-13 | Stable identities now evidenced: Shopify delivery/event ids, Shippo `transaction_object_id`, EasyPost refund id |
| INV-14 | Supported end to end by Shippo invoice items carrying the charged amount per transaction |
| INV-21 | Supported for shipping by invoice items; advertising reconciliation limited to tier 1 |
| INV-22 | Fee-reversal attribution can key on the refund's own `transactions[]` |
| Refund record vs repayment | §4.2.2: a refund record alone cannot promote a value to actual |
| Webhook delivery | §4.3: duplicate-capable, unordered, bounded retries — reconciliation is a required component, not an optimisation |

### 4.8 Rules for this work

Official provider documentation and specifications only; recorded example payloads and fixtures; documented fallback behaviour where a provider fact cannot be verified. **No live merchant credentials, no merchant account connections, no provider integrations.** Preparing the F-01 fixture **specification** is authorised (§4.9); capturing a provider evidence fixture requires a later founder authorisation. No invariant may be marked tested on the basis of this section.

### 4.9 F-01 — Shopify refund fixture specification (authorised, not executed)

**Status: PROCEDURE WRITTEN — NOT EXECUTED. F-01, D-09 and the affected part of INV-02 remain OPEN.** No merchant store was connected, no token requested and no live payload captured.

**Route chosen by the founder (2026-11-17): route B** — the founder will run the capture in their own Shopify development store and supply the redacted payload. The founder has confirmed they **do not currently have access to a development store**; no account may be created on their behalf and no production store or credentials may be used.

**The standalone procedure is written:** `docs/governance/f01-shopify-refund-evidence-capture-procedure.md`. It specifies development-store requirements, the discounted test-order scenario, the partial-refund steps, the pinned API version and endpoint, the exact fields to capture, redaction, validation that the fixture is genuine, minimum access scopes, safe storage and the decision table that interprets the result. It is the operative document; the coverage list below remains the requirement it satisfies.

**Two fixture classes that must never be conflated:**

| Class | What it is | What it can prove |
|---|---|---|
| **Synthetic calculation fixture** | Hand-constructed input records with expected outputs, built from the financial model | Proves the engine's arithmetic and invariants against stated inputs |
| **Provider evidence fixture** | A recorded payload captured from a real Shopify test store | Proves what Shopify actually sends, and settles field semantics |

A synthetic fixture **cannot** settle whether `refund_line_items[].subtotal` is net or gross of discount, because that is a provider fact. **D-09 and the affected part of INV-02 stay open until a provider evidence fixture demonstrates the behaviour.**

**Required coverage of the provider evidence fixture:**

1. Original order carrying a **line-level or order-level discount**.
2. **Partial refund of a discounted line** (quantity less than the line quantity).
3. Original **line price and discount allocations**, as sent by the provider.
4. **Refunded quantity and refund subtotal**, exactly as sent.
5. **Tax treatment** — tax on the refunded line, and whether prices are tax-inclusive.
6. **Refund transaction amount and status**, so the refund record and the repayment can be compared.
7. **Shipping refund**, where applicable.
8. **Expected financial-engine treatment**, stated **before** capture, so the comparison is a test rather than a rationalisation.

**Capture requirements:** the raw request body and all headers for the webhook, plus the GraphQL response used to build the fixture; API version and capture date; the payload stored unmodified as evidence, with any redaction recorded rather than silently applied; and the shop's currency, tax-inclusive setting and discount type.

**Prohibited:** assuming the net/gross basis in code, tests or documentation before the fixture demonstrates it; deriving an expectation from an observed number without recording that it was derived after the fact; treating a refund record as evidence of repayment.

---

## 5. Outstanding founder decisions

The following must be decided by the founder before the named gate can open. Each retains its exact options and recommendation from the delivered register.

**Blocking G2 (financial correctness):** D-06, D-07, D-08, D-09, D-18. **D-13 no longer blocks G2 as an unresolved decision**, having been approved in principle with conditions (§3.6) — provided those conditions are enforced **and** its open fee-recognition, accounting, tax and provider questions are cleared or formally accepted as a disclosed limitation. Those questions are carried as an explicit **G2 advisory** (§3.6.1); G2 cannot be declared passed while that advisory is open and unaccepted.

**Blocking G3 (merchant-visible profitability):** D-10, D-11, D-12, D-16, D-17, D-19, D-20, D-21, D-23. D-15 is approved with conditions (§3.4) and no longer blocks, provided those conditions are enforced. D-20's **architectural principles** are approved (§3.7), but its key-management selection and retention periods still block G3. **D-10a currently blocks G3.** §5 previously recorded that D-10a no longer blocked G3, provided C-10a.1–C-10a.5 were enforced in the engine and the specification remained internally consistent. The 2026-11-20 audit found **C-10a.1 PARTIALLY SATISFIED and OPEN** (defects D-01–D-03 unresolved — §3.5.6), so that proviso fails. D-10a blocks G3 until C-10a.1 is satisfied by an applied and re-audited correction set **and** C-10a.2–C-10a.5 are enforced in the engine.

**Blocking G4 (Co-Pilot):** D-22.

**Already deferred:** D-14, on the three conditions in §3.1.

**Resolved since this register was created:** the canonical product name is fixed (see the header note). P-01–P-05 documentation research is complete per §4, manual Meta billing evidence is approved with conditions (§3.4), and the F-1 two-view management design is approved as a management direction (§3.5). **Still required:** the F-01 provider evidence fixture (§4.9) — the capture procedure is written but no payload exists — and the deferred live Supabase checks (§4.6.1) once G0 settles.

**Decision preparation:** eight G0/G2 decisions (D-05, D-06, D-07, D-08, D-09, D-13, D-18, D-20) now have a preparation report with worked examples, consequences, dependencies and review requirements — `docs/governance/g0-g2-decision-preparation.md`.

**Individual decision briefs prepared for founder review:**

- `docs/governance/d13-payment-fee-allocation-decision-brief.md` — D-13. Variable and fixed fee components, why the tax-related and fixed components are not product-attributable, the 63-of-507 (12.4%) example reproduced and reconciled, the effect of spreading the residual (order and store profit unchanged; product-level profit corrupted), and open accounting/tax/provider questions. **D-13: APPROVED IN PRINCIPLE 2026-11-20 (§3.6)**; §10 records the seven conditions, the corrected reconciliation identity and the evidence grades.
- `docs/governance/d20-data-retention-erasure-decision-brief.md` — D-20. Three-store separation of PII from the ledger, random tenant-scoped references, per-subject envelope encryption and crypto-shredding, the **backup-restore failure mode**, retention as jurisdiction-keyed configuration with **no invented periods**. **D-20: architectural principles approved 2026-11-20 (§3.7)**; retention and key management remain open.
- `docs/governance/d20-key-management-and-erasure-restore-specification.md` — D-20 implementation prerequisites: six key-management options compared on restore survival, immediacy, cost shape and strength if bypassed; the restore threat scenario; eight protection controls; the erasure reconciliation mechanism; cost assessment by structure. **Recommends the hybrid option; no KMS selected.**
- `docs/governance/d10a-shipping-revenue-allocation-decision-brief.md` — **D-10a, revised and re-submitted.** Allocation at order acceptance down to individual quantities, largest remainder resolved once across all units, recognition per quantity on shipment, explicit release/reversal events, no residual on the final shipment. Seven worked examples. **D-10a is APPROVED WITH CONDITIONS (§3.5.6)** — principle approved, conditions C-10a.1–C-10a.5 open, professional-review items open.
- `docs/governance/inv26-quantity-state-specification.md` — **INV-26, revised.** Three independent dimensions (fulfilment, return/disposition, payment/refund), impossible-combination integrity controls, and the eight scenario walkthroughs. **Event-derived principle approved; this specification awaits approval.**
- `docs/governance/inv27-calculation-identity-specification.md` — **INV-27.** The eight-element calculation identity, fail-closed rules, and the reproducible cross-view reconciliation artifact with its four-bucket decomposition. **Principle approved; test design outstanding.**

That report **does not change any status in this register**; it separates product-design conventions from matters requiring professional review, and it records one cross-decision inconsistency (D-06 revenue on order date versus D-07 COGS on shipment date) — **since resolved by founder direction as a two-view management design (§3.5)**, with statutory treatment still subject to professional review.

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
| D-20 (key management) | Whether access revocation alone satisfies erasure; acceptable erasure latency; retention periods by jurisdiction | Legal / privacy |
| D-13 | Fee recognition timing; fixed vs period cost; true-up presentation; fee-refund presentation | Accounting |
| D-13 (tax) | Treatment of the tax-related fee component | Tax |

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
| 2026-11-17 | Canonical product name recorded as **Net-Margin Ad Spend Killer** (subtitle: Net-Margin Ad Spend Protection Platform). P-01–P-05 documentation research executed and recorded in §4: all five PARTIALLY VERIFIED, no item fully verified, no live account or database used. D-09 remains blocked on a refund fixture; Shippo recommended for P-04. |
| 2026-11-17 | **F-1 resolved by founder direction — two-view management design recorded (§3.5).** Commercial Management View (revenue and COGS both on order date, primary) and Fulfilment View (both on shipment date) approved as the **proposed management design, not a statutory accounting policy**. Order and shipment events and timestamps preserved; view becomes a required calculation dimension. D-06/D-07 stay unapproved as accounting policies and still block G2. **F-01:** route B chosen (founder captures in their own development store); founder currently has no development store, so no capture occurred. **Standalone capture procedure written** (`f01-shopify-refund-evidence-capture-procedure.md`) with a pre-stated numeric discriminator, redaction and validation requirements, and minimum read-only scopes. **F-01, D-09 and the affected part of INV-02 remain OPEN.** No code, schema, migration, test, credential, provider connection or deployment. |
| 2026-11-17 | **G0/G2 decision preparation report added** (`g0-g2-decision-preparation.md`): D-05, D-06, D-07, D-08, D-09, D-13, D-18, D-20 each with recommendation, worked example, consequences, dependencies and review requirement. Eight product-design conventions separated from seven matters requiring accounting/tax/legal review; D-09 confirmed unresolvable without F-01. Five cross-decision findings recorded, including the D-06/D-07 period mismatch (F-1) and the requirement that PII stay out of the ledger for D-20 to coexist with INV-19 (F-4). **No status in this register changed.** No code, schema, migration, test, credential or connection created. |
| 2026-11-17 | **F-1 two-view specification extended (§3.5.1–§3.5.5)** per founder direction: quantity state partition for partial shipments, multiple shipments per order, partial cancellations and partial refunds; per-shipment revenue/COGS attribution with largest-remainder discount allocation; the differing view-level effect of a refund before versus after shipment; and six rules preventing cross-view aggregation or comparison, including fail-closed reads and the prohibition on unqualified "contribution profit". **Two invariants proposed, not approved: INV-26 (quantity state partition) and INV-27 (view reconciliation identity).** **One new sub-decision raised: D-10a** (shipping revenue attribution across multiple shipments) — OUTSTANDING, not in the matrix. D-11 and D-12 noted as shipment-keyed. F-01, D-09 and the affected part of INV-02 remain OPEN. No status change, no code, no schema, no migration, no test, no deployment. |
| 2026-11-17 | **D-13 and D-20 founder decision briefs written** (`d13-payment-fee-allocation-decision-brief.md`, `d20-data-retention-erasure-decision-brief.md`) and indexed in §5. D-13 brief reproduces the 63-of-507 example, shows that spreading the residual leaves order and store profit unchanged while corrupting product-level profit and the fee reconciliation, and lists seven open accounting/tax/provider questions. D-20 brief specifies the three-store model, random (not hashed) pseudonymous references, per-subject envelope encryption, and the **backup-restore failure mode that requires key material outside the database backup scope** — flagged as an unverified dependency that may need an external KMS. **No retention period is asserted anywhere**; all periods are marked as jurisdiction-specific legal/tax determinations. **No status changed: D-13 OUTSTANDING, D-20 PROFESSIONAL REVIEW REQUIRED.** No code, schema, migration, test, credential, Supabase connection or deployment. |
| 2026-11-20 | **Founder review of D-13, D-20, D-10a, INV-26 and INV-27. Four status changes.** **D-13:** OUTSTANDING → **APPROVED WITH CONDITIONS (in principle)** — residual stays order-level, seven conditions recorded (§3.6), including **no artificial zero-residual reconciliation** and estimated/provisional/confirmed grading; the double-counting trap (63 is *inside* the 444) recorded explicitly; accounting, tax and provider reviews remain open. **D-20:** PROFESSIONAL REVIEW REQUIRED → **APPROVED WITH CONDITIONS (architectural principles only)** (§3.7) — ledger/identity/audit separation, random pseudonymous references, and erasure surviving backup restore; key management and **all** retention periods remain open, with no period invented. **D-10a:** remains OUTSTANDING, **revised and re-submitted** — the residual-on-final-shipment rule was rejected and replaced by allocation at order acceptance down to individual quantities with the largest-remainder step resolved once, per-quantity recognition on shipment, and explicit release/reversal events. **INV-26:** event-derived principle approved, the five-state partition **rejected** and replaced by three independent dimensions. **INV-27:** approved as an architectural principle subject to specification and test design. Four new documents written; the register's rejected INV-26 model and D-10a rule were **corrected in place**, not merely superseded. F-01, D-09 and the affected part of INV-02 remain OPEN. **No code, schema, migration, test, Supabase provisioning, provider connection, credential or deployment. No control described here is implemented or verified.** |
| 2026-11-20 (second review) | **Founder review of D-10a, D-20 and the D-13 G2 advisory. One status change, one advisory created, one required specification delivered.** **D-10a:** OUTSTANDING → **APPROVED WITH CONDITIONS** (§3.5.6) — allocation once at order acceptance across individual quantities, largest-remainder resolved once, per-quantity recognition on shipment, explicit adjustment/release/reversal events, no residual on the final shipment. Five conditions C-10a.1–C-10a.5 recorded; professional accounting and tax reviews remain open. **D-13:** remains AWC in principle, and its unresolved fee-recognition, accounting, tax and provider questions are now carried as an **explicit G2 advisory (§3.6.1)** that must be cleared or formally accepted before G2 passes; the 444 = 381 + 63 and 507 = 444 + 33 + 30 arithmetic is restated with the rule that 444 allocated plus 63 residual must **never** be displayed as additive. **D-20:** the architectural direction is approved **for schema preparation only**, with seven binding conditions (a)–(g) recorded in §3.7 — key material and the erasure register outside the database backup scope, rollback-resistant register, all identity-write paths consulting the register, fail-closed on missing erasure evidence, reconciliation before identity writes after a restore, no resurrection by replay, and recognition without retained PII. **The required restore-and-erasure threat model and test specification is delivered** (`d20-restore-and-erasure-threat-model-and-test-specification.md`): T-01–T-14, R-01–R-07, TE-01–TE-22 with an evidence standard, and five open questions. **INV-26** and **INV-27** now carry their full test registers (TC-26-01–TC-26-16 with fail-closed rules FC-26-1–FC-26-7; TC-27-01–TC-27-15 with reconciliation rules RR-27-1–RR-27-8). **No KMS vendor, paid service, retention period or implementation is approved. No test in any specification has been executed.** Register tally: APPROVED 0 / AWC 4 / DEFERRED 1 / OUTSTANDING 8 / PRR 6. F-01, D-09 and the affected part of INV-02 remain OPEN. |
| 2026-11-20 (third review) | **D-10a audit outcome recorded and three substantive correction briefs prepared. No status change to D-10a, no decision closed, no test executed.** **C-10a.1:** PARTIALLY SATISFIED — **OPEN** (§3.5.6). The seven worked examples are arithmetically correct under their stated assumptions and the anti-balancing-plug requirement is satisfied, but D-01, D-02 and D-03 remain unresolved specification defects and the shipping-refund ceiling, multi-charge reconciliation, allocation-base ambiguity and missing scenario coverage must be addressed. **G3 proviso restated:** §5 now records that D-10a **blocks G3** because the "specification remains internally consistent" proviso fails; this is a status correction, not a reversal of the founder's approval. **Three substantive decision briefs prepared for founder decision, not applied:** P-1 shipping association selector (basis ladder B-1/B-2/B-3, basis fixed at the allocation instant and immutable, late evidence annotates without reallocating, basis added to the INV-27 identity); P-3 post-acceptance shipping discounts (R6 split into R6.1 order-time and R6.2–R6.5 post-acceptance, resolving the C-10a.5 contradiction); P-4 shipping-refund ceiling (combined order-level ceiling on refunds plus discounts, hard fail at order scope, quarantine at unit scope, with proof that a recognised-revenue-only ceiling wrongly blocks refunds for goods that never shipped). **Four new invariants proposed for founder numbering approval:** INV-28 allocation-basis identity, INV-29 adjustment-event separation, INV-30 combined reduction ceiling, INV-31 shipping bucket partition conservation. **Editorial corrections P-2, P-6, P-8, P-9 and P-10 proposed as a diff and not applied**; P-5 is reclassified as substantive (it changes R8's scope) and P-7 is content-gated on P-1 and P-4 approval, so neither enters the editorial batch. Register tally unchanged: APPROVED 0 / AWC 4 / DEFERRED 1 / OUTSTANDING 8 / PRR 6. F-01, D-09 and the affected part of INV-02 remain OPEN. |
