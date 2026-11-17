# G0 and G2 Founder Decision Preparation Report

**Product:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Scope:** D-05, D-06, D-07, D-08, D-09, D-13, D-18, D-20
**Status:** Preparation only. **No decision status in `decision-register.md` is changed by this report.**
**Date:** 2026-11-17

---

## 0. What this report is, and what it is not

This report prepares eight founder decisions for review. For each decision it provides the recommendation already recorded in the register, a classification of what kind of decision it actually is, a worked example with independently checked arithmetic, the consequences of adopting it, its dependencies, and whether qualified professional review is required.

**This report is not an approval.** A recommendation appearing here does not make a policy approved, and no recommendation is written into the register as decided. The register remains at: **APPROVED 0 · APPROVED WITH CONDITIONS 1 (D-15) · DEFERRED 1 · OUTSTANDING 10 · PROFESSIONAL REVIEW REQUIRED 7.**

**Not performed:** no application code, schema, migration, test, integration, credential, environment variable, Supabase connection or deployment. No provider API was called. No invariant is marked tested.

### 0.1 A limitation you should know before reading

The full A/B/C option sets for these eight decisions were delivered in the earlier review text, but the register stores only each decision's **recommended option** (column "Recommended option"). This report therefore quotes that column **verbatim** and, where it examines an alternative, labels the alternative explicitly as **"alternative examined here — not from the delivered option set"**. Nothing in this report claims to reproduce an option letter that is not in the register. If you want the full option sets restated for the record, that is a separate documentation task.

---

## 1. Classification vocabulary

The founder asked for a clear separation between decisions that are product-design conventions and decisions that need professional advice. Every decision below is classified on three axes, because several of them are not one kind of decision:

| Code | Meaning | Who may decide | Consequence of deciding wrongly |
|---|---|---|---|
| **P — Product-design convention** | A technical or presentational convention with no statutory meaning. Any defensible choice is acceptable provided it is deterministic, stored, versioned and disclosed. | Founder / engineering | Recalculation and re-labelling; no restatement risk |
| **R — Professional review required** | Touches revenue recognition, inventory valuation, tax or legal obligation. The choice has statutory or external-reporting consequences. | Qualified accountant / tax adviser / lawyer, with founder sign-off | Wrong statutory accounts, misstated tax position, regulatory exposure |
| **E — Provider evidence required** | Depends on a fact only the provider can establish (field semantics). Cannot be settled by reasoning, preference or convention. | Nobody can decide it yet; requires a recorded payload | Silent financial error, double-counting, broken invariants |

A single decision can carry more than one classification. That is the normal case here, and conflating them is exactly how a technical convention gets mistaken for an accounting policy.

---

## 2. Five cross-decision findings

These are inconsistencies or risks that only become visible when the eight decisions are examined together. They are surfaced before the individual decisions because two of them change what you are being asked to approve.

### F-1 — D-06 and D-07 place revenue and cost in different periods

D-06 recommends **management revenue on the order date**. D-07 recommends **COGS on the shipment date**. If both are adopted as written, a management period can recognise revenue for an order whose cost lands in the following period. For an order placed on the last day of a month and shipped two days later, the period shows revenue at 100% margin and the next period shows a cost with no revenue.

The magnitude is not academic: for a merchant shipping 1–3 days after order, a period boundary captures roughly one day's orders with no matching cost — on 30-day months and 100 orders/day, that is ~3% of monthly orders mismatched, and the distortion lands entirely in the two periods either side of the boundary.

**RESOLVED — founder direction, 2026-11-17.** The founder approved the two-view structure as the **proposed management design** (not a statutory accounting policy):

1. **Commercial Management View** — recognise **both revenue and COGS on the order date**, using the applicable cost snapshot, and label it explicitly as an order-date management measure. This is the founder's preferred **primary** view.
2. **Fulfilment View** — recognise **both revenue and COGS on the shipment date**, using the shipment cost snapshot, available separately.
3. The original order and shipment events and their timestamps are **preserved**; neither view may destroy the evidence the other needs.
4. **Statutory accounting treatment stays separately identified and subject to professional accounting review.** Neither management view establishes statutory compliance.

**What this changes:** the period mismatch no longer exists *within* either view, because each pairs revenue with COGS on a single event date. **What it does not change:** D-06 and D-07 remain unapproved as accounting policies, and the difference between the two views must never be presented as final realised contribution profit. Neither view may be labelled "contribution profit" without a view qualifier.

**Engine consequence:** a calculation must carry its view (commercial / fulfilment) as part of its identity, so a snapshot from one view can never be silently compared with, aggregated into, or substituted for the other. This is a calculation-versioning requirement, not a display preference.

### F-2 — D-09 cannot be closed by preference, only by evidence

The register recommends option A (net basis). Whether Shopify's refund payload is net or gross of discount is a **provider fact** that the F-01 fixture has not yet established. Adopting option A now would encode an assumption as a policy. **D-09 stays open until F-01 is captured**, and the engine must run under an explicitly labelled, versioned assumption until then.

**2026-11-17 update:** the standalone capture procedure is now written — `docs/governance/f01-shopify-refund-evidence-capture-procedure.md`. It contains the pre-stated numeric discriminator (net basis 54.00 subtotal / 4.32 tax versus gross basis 60.00 / 4.80), so the capture is a test rather than a rationalisation. The founder will run it in their own development store. **No payload has been captured; F-01 and D-09 remain open.**

### F-3 — D-13's allocation basis leaves a real, non-attributable residual

The register basis is "pro-rata over line net revenue including shipping revenue". Payment processing is charged on the **full charged amount, which includes tax**, and it includes a **fixed per-transaction component**. Neither is attributable to a product. Worked example §4.6 shows the residual is 63 of a 507 fee — 12.4%. That residual must be carried as an explicit order-level cost, not spread across products, or product margins are systematically wrong.

### F-4 — D-20 conflicts with append-only unless PII is kept out of the ledger

Erasure requires deletion; INV-19 requires that ledger rows are never mutated or deleted. Both can hold only if **personally identifying data never enters the ledger**. The register recommendation (pseudonymise + crypto-shred) is correct and should be understood as an architectural constraint on the schema, not a later feature: the PII must live in a separate, erasable table from the first migration. If PII is written into order or ledger rows, D-20 becomes unsatisfiable without violating INV-19.

### F-5 — D-05 "conversion at ingestion" must store the original, not the converted value

The register recommendation says rate and rate date are stored on the event, which is right. The refinement to record explicitly: the **original currency amount is the authoritative value**, the converted amount is **derived** and re-derivable. If only the converted figure is stored, a rate correction or a change of reporting currency becomes an unreconstructable rewrite of history.

---

## 3. Canonical worked dataset

All examples below use this dataset so that figures reconcile across sections. Amounts in minor units (cents). Order **O-1001**, store "Aurora Home", store timezone `America/Los_Angeles`, currency USD.

| Element | Value |
|---|---|
| Placed | 2026-03-31 18:00 PDT = 2026-04-01T01:00Z |
| Shipped | 2026-04-02 |
| L1: Linen Throw × 2 @ 6,000 | gross 12,000 · unit COGS 2,100 |
| L2: Ceramic Vase × 1 @ 4,000 | gross 4,000 · unit COGS 1,500 |
| Merchandise gross | 16,000 |
| Order-level discount SPRING10 10% | 1,600 |
| Merchandise net | 14,400 |
| Shipping charged to customer | 900 |
| Tax 8% on merchandise (shipping non-taxable — stated assumption) | 1,152 |
| Total charged | 16,452 |
| Payment fee 2.9% + 30 | 477 + 30 = **507** |
| COGS | 5,700 |
| Actual outbound shipping (Shippo label) | 780 |
| Fulfilment and packaging | 250 |

**Contribution profit before advertising (order level):**

```
  merchandise net revenue        14,400
+ shipping revenue                  900
- COGS                            5,700
- outbound shipping cost            780
- fulfilment and packaging          250
- payment fee                       507
= CP before advertising           8,063
```

Tax of 1,152 is excluded throughout: it is a liability, not revenue. Checked: 14,400 + 900 = 15,300; 15,300 − 5,700 = 9,600; 9,600 − 780 = 8,820; 8,820 − 250 = 8,570; 8,570 − 507 = **8,063**.

---

## 4. The eight decisions

---

### 4.1 D-05 — Reporting currency and FX policy

**Register recommendation (verbatim):** "A — single tenant reporting currency, conversion at ingestion, rate and rate date stored on the event"
**Register status:** PROFESSIONAL REVIEW REQUIRED · **Gates:** G0 schema / G3

**Classification:** P for the storage and derivation model · R for the rate convention and FX presentation · E not required.

**What is actually being decided.** Two separate things are bundled here. (i) A data-model convention: which currency is authoritative, where the rate lives, and how converted values are derived — purely technical, decidable now. (ii) An accounting policy: which rate (transaction-date spot, monthly average, period closing), and how exchange differences are presented — statutory in nature, not decidable by us.

**Recommended split, for your approval as two separate items:**

- **D-05-P (convention, can be decided now):** one reporting currency per tenant; original-currency amount is authoritative; the rate, rate date and rate source are stored immutably on each event; converted amounts are derived and recomputed, never overwritten; every converted figure can be reproduced from (original amount, rate, rate version).
- **D-05-R (policy, professional review):** which rate convention applies to management reporting and to statutory reporting, and whether exchange differences are a separate line or folded into contribution profit.

**Worked example.** EUR store, USD reporting currency.

| Event | Date | EUR | Rate | USD |
|---|---|---|---|---|
| Order O-2001 merchandise net | 2026-03-10 | 1,000 | 1.08 | 1,080 |
| Partial refund | 2026-04-20 | 200 | 1.12 | 224 |

Under transaction-date rates: revenue recognised 1,080; refund reduces revenue by 224. Against the original rate the refund "should" have been 216, so 8 USD is an **exchange difference**, not a margin error. Two treatments:

- *Fold into margin:* refund reduces revenue by 224 → contribution profit falls by 224 (plus COGS reversal at historical rate).
- *Separate FX line:* refund reduces revenue by 216 and an 8 USD exchange loss is reported separately.

Both are defensible; they produce different product-level contribution profit for the same economic event. If the difference is left unlabelled, an 8 USD gap appears in the merchant's numbers with no explanation — precisely the "unexplained balancing adjustment" this project forbids.

**Consequences of adopting the split recommendation.** Reporting currency is fixed per tenant, so a merchant who changes currency later requires a re-derivation pass, not a data rewrite — cheap, because rates and originals are preserved. Adopting transaction-date rates makes period revenue sensitive to daily rate movement; a monthly-average convention smooths it but makes individual order figures non-reproducible from a single stored rate unless the average is itself stored as a rate version.

**Dependencies.** Requires the rate-source decision before the G0 schema is frozen (rate table with source, date, version). Interacts with D-18 (which period a rate belongs to) and with INV-21 aggregation (aggregation must never sum across currencies without conversion at a stored rate).

**Professional review required:** Accounting — yes (rate convention; FX presentation). Tax — yes, if the merchant's tax filings are in a currency other than the store currency. Legal — no. Provider evidence — no.

**Question for the adviser:** *For management reporting of an e-commerce store whose store currency differs from the reporting currency, should revenue be translated at transaction-date spot or a period average, and should exchange differences be shown separately from contribution profit?*

**What would change this recommendation:** a merchant segment that transacts in more than one currency within one store, which would force per-currency reporting rather than a single tenant reporting currency.

---

### 4.2 D-06 — Revenue recognition timing

**Register recommendation (verbatim):** "A — management = order date; statutory = shipment / transfer-of-control date"
**Register status:** PROFESSIONAL REVIEW REQUIRED · **Gate:** G2
**Founder prohibition on record:** option C (statutory timing for management reporting) must not be selected without explicit approval.

**Classification:** R (recognition timing) with P conventions for what must be stored and labelled.

**Worked example — three answers for one order.** Order O-1001 is placed 2026-03-31 18:00 in `America/Los_Angeles`, which is 2026-04-01T01:00Z, and ships 2026-04-02.

| Basis | Period | Why |
|---|---|---|
| Management, store timezone, order date | **March 2026** | Local calendar month of the order |
| Any UTC calendar-month view | **April 2026** | UTC date is 1 April |
| Statutory, shipment date | **April 2026** | Transfer of control proxy |

The lesson is not which answer is right — it is that **a period cannot be derived from a date string**. If `period = substring(order_date, 1, 7)`, this order silently lands in April for every merchant in a negative-offset timezone, and month-end totals disagree with the merchant's own store report. The period must be a stored, explicit `[start, end)` interval with a named timezone, assigned at ingestion and recorded on the event.

**Consequences of adopting the register recommendation.** Management reporting at order date gives the merchant a figure that matches their Shopify dashboard on the day they look at it, which is commercially valuable and reduces "your numbers disagree with Shopify" support load. The cost is that order-date revenue includes orders that are later cancelled or fully refunded, so period revenue must be expected to be revised downward by the reversal, booked in the period the reversal occurs (append-only, never by editing the earlier period). Statutory reporting at shipment date is the conventional transfer-of-control proxy and is what an accountant will expect.

**Consequence to weigh carefully — finding F-1.** As recorded, D-06 (revenue at order date) and D-07 (COGS at shipment date) mismatch revenue and cost in the management view. You are being asked to approve a pairing that splits them. The recommendation here is to keep both views but make each internally consistent.

**Dependencies.** D-18 (period definition), D-07 (cost timing), D-09 (refund timing), INV-16 (period assignment), F-01 (whether a refund record can be dated at all before its transaction settles).

**Professional review required:** Accounting — yes (transfer of control; whether shipment or delivery is the appropriate proxy for the merchant's terms). Tax — yes, where revenue timing affects a tax filing period. Legal — no. Provider evidence — no.

**Question for the adviser:** *For a merchant selling goods shipped by third-party carriers, is shipment or delivery the more appropriate transfer-of-control point, and may management reporting use order date while statutory reporting uses shipment date, with a documented reconciliation between them?*

**What would change this recommendation:** merchants selling digital goods or pre-orders, where order date and transfer of control diverge by weeks — the two views would then need materially different presentation, not just different labels.

---

### 4.3 D-07 — COGS timing and inventory valuation

**Register recommendation (verbatim):** "A — moving weighted average, cost snapshot on each sale event, COGS at shipment date, no retroactive restatement"
**Register status:** PROFESSIONAL REVIEW REQUIRED · **Gate:** G2

**Classification:** R (valuation method and cost timing) with P conventions for the data model and snapshot immutability.

**Worked example — valuation method changes the product decision.**

| Receipt | Date | Qty | Unit cost | Total |
|---|---|---|---|---|
| R1 | 2026-02-01 | 10 | 2,000 | 20,000 |
| R2 | 2026-03-01 | 10 | 2,400 | 24,000 |

Sale of 4 units on 2026-03-10.

*Moving weighted average:* (20,000 + 24,000) ÷ 20 = **2,200**/unit → COGS 8,800; remaining inventory 16 × 2,200 = 35,200. Check: 44,000 − 8,800 = 35,200 ✓.
*FIFO:* COGS 4 × 2,000 = **8,000**; remaining 6 × 2,000 + 10 × 2,400 = 36,000. Check: 44,000 − 8,000 = 36,000 ✓.

Difference: **800 of COGS, i.e. 200 per unit — 10% of unit cost.** For a product whose purpose is to tell a merchant whether a product is profitable, a 10% swing in unit cost is not a rounding detail: it can move a product across the pause threshold. The method is therefore not a bookkeeping preference in this product — it is a margin-determining choice.

**Return and write-off under the snapshot rule.** One unit returned and restocked: COGS reversed at the **stored cost snapshot 2,200**, inventory returns to 17 × 2,200 = 37,400. Under FIFO the reversal would conventionally be 2,400 (most recent layer) or 2,000 (matching the original issue) — the ambiguity must be resolved explicitly, which is why the register's snapshot rule is the right call: the reversal uses the cost actually recognised, so cost is recognised **exactly once**. One unit returned damaged and written off: **no COGS reversal**. The 2,200 remains recognised as COGS and the write-off is a **classification and disclosure** of that same 2,200, not a second expense. Adding an inventory write-off cost on top of retained COGS would double-count the loss.

**Consequences of adopting.** Moving weighted average is simple, deterministic, needs no layer bookkeeping, and is stable under partial returns. It is also less accurate for merchants with volatile input costs, and it is **not** the method a merchant's accountant may have used, so platform figures will differ from statutory figures — which must be disclosed rather than hidden. "No retroactive restatement" keeps history stable and reproducible but means a corrected receipt does not fix past margins; the correction applies prospectively, and the earlier period is restated by an explicit adjustment event.

**Dependencies.** Requires lot/receipt-level data from day one even if WAV is the only method, so the method remains switchable and historical recalculation is possible. Depends on D-06 for the matching period, D-11 for shipping cost precedence, and the inventory-lifecycle invariant INV-20.

**Professional review required:** Accounting — yes (method must be acceptable and consistently applied; WAV vs FIFO vs specific identification). Tax — yes, where inventory valuation affects taxable income. Legal — no. Provider evidence — yes for whether Shopify exposes per-receipt cost and whether inventory item costs carry effective dates (unverified).

**Question for the adviser:** *May the platform present product-level contribution profit using moving weighted average where the merchant's statutory accounts use a different inventory method, provided the difference is disclosed and reconcilable?*

**What would change this recommendation:** merchants with regulated inventory (food, pharma) needing specific identification or expiry-based costing, which WAV cannot express.

---

### 4.4 D-08 — Discount allocation

**Register recommendation (verbatim):** "A — pro-rata by line gross, largest remainder"
**Register status:** OUTSTANDING · **Gate:** G2

**Classification:** P for the arithmetic (fully decidable now) · R for presentation only (whether the discount reduces revenue or is shown as an expense line) · E for one provider question (whether Shopify's own `discount_allocations` already match this basis — if they do, our allocation should follow theirs to avoid two different answers).

**Worked example 1 — residual handling.** Order O-1002, three lines, gross 3,333 / 3,333 / 3,334 (total 10,000), order-level discount 1,000.

Exact shares: 333.3 / 333.3 / 333.4. Floor: 333 / 333 / 333 = 999 — **one cent short**. Largest remainder assigns the remaining cent to the line with the largest fractional part (C, 0.4): **333 / 333 / 334 = 1,000** ✓. Tie-break must be deterministic: largest remainder, then lowest line index.

Without largest remainder, the allocated total does not equal the discount, and the invariant "allocated + unallocated = total" fails by a cent on many orders — which in aggregate becomes a visible reconciliation gap and destroys trust in the totals.

**Worked example 2 — why the basis matters commercially.** Two lines, 1,000 gross each: X with COGS 100 (90% margin), Y with COGS 800 (20% margin). Order discount 200.

| Basis | Discount X | Discount Y | Line margin X | Line margin Y |
|---|---|---|---|---|
| Pro-rata by gross (recommended) | 100 | 100 | 800 | 100 |
| Pro-rata by COGS | 22 | 178 | 878 | 22 |

Both allocations are "reasonable"; they tell opposite stories about which product to keep advertising. Pro-rata by gross is recommended because it matches how the merchant's own storefront allocates the discount and how Shopify reports `discount_allocations`, so platform-reported and internally allocated figures agree — a merchant seeing two different product margins for the same order loses confidence immediately.

**Consequences of adopting.** Deterministic, auditable, matches provider allocation, and needs no per-merchant configuration. The trade-off: pro-rata by gross gives the same percentage discount to a high-margin and a low-margin product, which slightly understates the margin damage to the cheap-to-make product — acceptable for a decision tool, and disclosed.

**Dependencies.** INV-01 (no duplicate discount deduction), INV-02 (refund ceilings at line level), D-09 (refund basis depends on this allocation), D-13 (fee allocation base is net revenue, which depends on discount allocation). F-01 should record the provider's allocation for the same order so we can prove we match.

**Professional review required:** Accounting — only for presentation (contra-revenue vs expense); the allocation arithmetic itself is not a statutory matter. Tax — no, unless discounts affect taxable value in a jurisdiction where they do (merchant-configured under D-17). Legal — no. Provider evidence — yes (alignment with provider allocation).

**Question for the adviser:** *Should order-level discounts be presented as a reduction of revenue, and is that presentation consistent with how the merchant's jurisdictional tax treatment handles discounts?*

---

### 4.5 D-09 — Refund basis (net vs gross + reversal)

**Register recommendation (verbatim):** "A — net (Resolution C)"
**Register status:** OUTSTANDING · **Gate:** G2
**Blocking evidence:** F-01 provider evidence fixture — **not captured**.

**Classification:** **E first** (the provider fact is unknown) · R for presentation · P for the "never infer" rule.

**This decision cannot be responsibly closed today.** Option A may well be correct, but nothing in the official reference states whether `refund_line_items[].subtotal` is net or gross of the original discount. Adopting A now would convert an unverified assumption into a financial policy — the exact failure mode the governance rules exist to prevent.

**Worked example — the same refund, two bases, a 1,200 difference.**

Full refund of L1 (2 × Linen Throw) after the 10% order discount. Gross 12,000; allocated discount 1,200; net 10,800; tax 8% × 10,800 = 864; refund cash = 11,664 (shipping not refunded).

*If the provider sends 10,800 (net):* book merchandise refund 10,800. No discount reversal is needed — the discount was already netted. Revenue reduction **10,800**.

*If the provider sends 12,000 (gross):* the engine must reverse the 1,200 discount as well: 12,000 − 1,200 = 10,800 net. Revenue reduction **10,800**.

*The failure:* a gross figure handled as if it were net, or a net figure plus an additional 1,200 discount reversal, reduces revenue by **12,000 instead of 10,800** — contribution profit understated by 1,200 on a single line, and the merchant is told a healthy product is unprofitable. This is precisely the INV-02 double-count.

**Reconciliation check of the full refund (independent arithmetic):**

```
  remaining L2 net revenue                 3,600
+ shipping revenue retained                  900
- remaining COGS (1,500)                  -1,500
- outbound shipping cost                    -780
- fulfilment and packaging                  -250
- fee still charged                          -169
= CP after full refund of L1               1,801
```
Fee check: total fee 507; refunded variable fee on 11,664 = 338 (2.9% × 11,664 = 338.3 → 338); retained 507 − 338 = **169**, composed of fixed 30 + shipping-related variable 20 + L2 variable 119 = 169 ✓. From the original CP of 8,063: −10,800 + 4,200 + 338 = −6,262 → 8,063 − 6,262 = **1,801** ✓. Both routes agree.

Whether the processor actually returns the variable component on a refund is **provider evidence, not assumed here** — it is flagged unverified and is why the fee refund appears as a separate, evidenced event rather than a rule.

**Recommended interim position (a convention, not a policy):** ingest the refund as received, store it unmodified with the raw payload, and compute under an explicitly named assumption (`refund_basis = 'provider_reported_unconfirmed'`) that is displayed to the merchant and recorded in `calc_version`. The moment F-01 confirms the basis, the assumption is superseded by a versioned recalculation — no history is rewritten. **D-09 stays open until then.**

**Consequences.** Closing D-09 on an assumption risks a 10% error on every refunded discounted line, in the direction that destroys merchant trust fastest (telling them a good product is bad). Leaving it open costs an assumption label in the UI and one recalculation pass after F-01. The second cost is far smaller.

**Dependencies.** F-01 fixture; D-08 (the allocation being reversed); D-13 (fee reversal attribution, INV-22); INV-02 (refund ceiling); §4.2.2 of the register (a refund record is not proof of repayment).

**Professional review required:** Accounting — yes, for presentation (contra-revenue vs expense). Tax — yes, for refunded tax handling. Legal — no. Provider evidence — **yes, mandatory before closure.**

---

### 4.6 D-13 — Payment processing fee allocation

**Register recommendation (verbatim):** "A — pro-rata over line net revenue including shipping revenue, largest remainder; allocated once at charge time"
**Register status:** OUTSTANDING · **Gate:** G2

**Classification:** P for the allocation method · R for recognition timing (charge date vs payout/settlement date) · E for whether the processor's actual per-order fee is retrievable.

**Worked example — allocation with a visible residual (finding F-3).**

Total fee 507 on 16,452 charged. Decompose by what the processor actually charged on:

| Component | Base | Fee |
|---|---|---|
| Variable fee on merchandise + shipping | 15,300 | 2.9% = 443.7 → 444 |
| Variable fee on **tax** | 1,152 | 2.9% = 33.4 → 33 |
| Fixed per-transaction | — | 30 |
| **Total** | | **507** ✓ |

Allocate only the merchandise-and-shipping component, pro-rata line net revenue including shipping revenue:

| Line | Net revenue | Shipping share | Allocation base | Fee |
|---|---|---|---|---|
| L1 | 10,800 | 675 | 11,475 (75%) | 332.775 → **333** |
| L2 | 3,600 | 225 | 3,825 (25%) | 110.925 → **111** |
| Lines total | | | | **444** ✓ |

Largest remainder: floors 332 + 110 = 442, remainder 2, fractional parts .925 and .775 → L2 then L1 → 333 / 111 = 444 ✓.

**Residual carried at order level:** tax-related fee 33 + fixed component 30 = **63**, i.e. 12.4% of the fee. It is not attributable to a product, because tax is not revenue and the fixed component is per charge. It must appear as an explicit order-level cost line. Spreading it across products would silently inflate every product's cost by an amount that has nothing to do with the product — and the merchant could never audit it.

**Per-line contribution (before shipping and fulfilment allocation, which belong to D-11/D-12):** L1 = 10,800 − 4,200 − 333 = **6,267**; L2 = 3,600 − 1,500 − 111 = **1,989**. Order check: 6,267 + 1,989 + 900 − 780 − 250 − 63 = **8,063** ✓ — matches the order-level CP exactly, which is the reconciliation that proves no fee was lost or double-counted.

**Consequences.** "Allocated once at charge time" means a later fee correction is an adjustment event, not a reallocation — stable, reproducible, and consistent with the append-only rule. The trade-off: if the processor's actual fee differs from the configured rate card, product margins carry an estimate until the payout record arrives, so the fee must be labelled **estimated** until then (INV-14). If the actual per-order fee is not retrievable from the provider, every fee is estimated and must say so — that is a material disclosure for a margin product.

**Dependencies.** D-08 (net revenue base), D-09/INV-22 (fee refunds attributed to the refunded lines), D-18 (which period the charge belongs to), and provider evidence on per-order fee availability.

**Professional review required:** Accounting — yes for recognition timing (charge vs settlement). Tax — no. Legal — no. Provider evidence — yes (per-order fee from payout/balance records).

**Question for the adviser:** *Should payment processing fees be recognised when the charge is created or when the payout settles, where the two fall in different reporting periods?*

---

### 4.7 D-18 — Management and accounting periods

**Register recommendation (verbatim):** "A — dual management_period + accounting_period (Resolution D, D-R1–D-R10)"
**Register status:** PROFESSIONAL REVIEW REQUIRED · **Gate:** G2

**Classification:** R for statutory period alignment and restatement policy · P for the period representation, timezone and locking mechanism.

**Worked example — boundary, refund and late invoice in one period.**

March 2026 management period, store timezone `America/Los_Angeles`, containing only O-1001 (placed 2026-03-31 18:00 PDT, i.e. 2026-04-01T01:00Z, shipped 2026-04-02).

| Step | Event | Period affected | CP after advertising |
|---|---|---|---|
| 1 | Order O-1001 recognised | March (store TZ, order date) | 8,063 − 2,000 = **6,063** |
| 2 | Refund of L1 on 2026-04-20 | **April** — March is not reopened | March unchanged; April −6,262 |
| 3 | Meta March invoice imported 2026-05-12, billing-confirmed 2,150 vs platform-reported 2,000 | March, as a **subsequent adjustment** | March restated to **5,913** |

Step 3 is the important one: the March period is not edited. A restatement event is appended, the period is marked `restated` with the reason and the evidence reference, and both versions remain reproducible under their own `calc_version`. The merchant sees the latest figure with a "restated on 2026-05-12" badge and can drill into both.

Had the period been derived from the UTC date string, the order would have fallen in April, the March figure would have been empty, and the merchant's reconciliation against Shopify would have failed at every month end.

**Consequences.** Dual periods satisfy management reporting and statutory reporting without forcing one to distort the other, at the cost of two figures that legitimately differ — which must be labelled, never silently reconciled. Period locking is required: once a period is closed for statutory purposes, changes to it must be adjustments, and the lock itself must be an auditable event. Deferring D-18 is not safe: period assignment is written into the schema and every aggregate depends on it, so a late change is a recalculation of all history.

**Dependencies.** D-06 and D-07 (which event dates define a period), D-05 (rate by period), D-15 (which period an imported invoice belongs to), INV-16, INV-21.

**Professional review required:** Accounting — yes (statutory period definition, restatement policy). Tax — yes (filing periods). Legal — no. Provider evidence — no.

**Question for the adviser:** *May the platform maintain a management period on order date alongside a statutory period on shipment date, and what disclosure is required where the two materially differ?*

---

### 4.8 D-20 — Data retention and erasure

**Register recommendation (verbatim):** "A — pseudonymise customer identity in the ledger; crypto-shred the PII key; record an erasure event"
**Register status:** PROFESSIONAL REVIEW REQUIRED · **Gates:** G0 schema / G3

**Classification:** R (legal obligation) · P (the architectural constraint that makes erasure possible without breaking append-only).

**Worked example — erasure of customer C-77.**

| Data | Treatment | Why |
|---|---|---|
| Name, email, phone, shipping address | **Deleted** (PII key crypto-shredded) | Not required for financial reporting |
| Payment instrument references | **Deleted** | Held by processor, not needed here |
| Order id, dates, amounts, product ids, tax, fee | **Retained** | Financial event evidence; statutory retention |
| Pseudonymous customer token | **Retained, now unlinkable** | Preserves order integrity and aggregation |
| Erasure event (id, timestamp, scope, legal basis) | **Retained** | Proof the request was honoured |

After erasure, contribution profit recomputes to the **same 8,063** — the arithmetic never touched an identifier. The ledger hash chain is intact because no row was mutated or removed; only the separate identity table changed, and that change is itself an event.

**The architectural constraint (finding F-4).** This only works if PII is **never written into order, line or ledger rows** from the first migration. If a customer email or address sits in an order row, erasure requires mutating an immutable row, and D-20 becomes unsatisfiable without violating INV-19. This is a G0 schema requirement, not a later feature.

**Consequences.** Retaining amounts while deleting identity satisfies both the statutory retention duty and the erasure right, and it keeps every aggregate stable — no historical totals move when a customer is erased. The trade-off: a merchant's own support workflow may expect to find the customer on the order; the product must show "customer identity erased on request" rather than pretend the order had no customer. Retention windows must be configurable per jurisdiction, and the default must be the **longest** applicable statutory period, not a marketing-friendly short one.

**Dependencies.** G0 schema (PII separation), INV-19 (append-only), D-17 (jurisdictions, which determine retention periods), the audit log.

**Professional review required:** Accounting — no. Tax — yes (retention periods for tax records by jurisdiction). **Legal — yes, mandatory** (erasure right vs legal-obligation retention, and whether erasure may be deferred). Provider evidence — no.

**Question for the adviser:** *For how long must order-level financial records be retained in our target jurisdictions, and may customer identity be erased on request while the financial amounts are retained, or must erasure be deferred until the retention period expires?*

---

## 5. Classification matrix

| Decision | Product-design convention (decidable now) | Professional review | Provider evidence | Register status (unchanged) | Gate |
|---|---|---|---|---|---|
| D-05 | Storage model: original amount authoritative, rate + date + source stored, conversions derived | Rate convention; FX presentation | — | PROFESSIONAL REVIEW REQUIRED | G0 / G3 |
| D-06 | Store all four dates; label the basis; explicit period interval | Transfer-of-control point; dual-view reconciliation | — | PROFESSIONAL REVIEW REQUIRED | G2 |
| D-07 | Lot-level receipts; immutable cost snapshot; switchable method | Valuation method; cost timing; disclosure of difference | Per-receipt cost availability | PROFESSIONAL REVIEW REQUIRED | G2 |
| D-08 | Pro-rata gross + largest remainder; deterministic tie-break | Presentation (contra-revenue) | Alignment with provider allocation | OUTSTANDING | G2 |
| D-09 | Never infer; label assumption; versioned recalculation | Refund presentation; tax on refunds | **F-01 — mandatory** | OUTSTANDING | G2 |
| D-13 | Allocation basis; explicit residual; allocate once | Recognition timing | Per-order fee retrievability | OUTSTANDING | G2 |
| D-18 | Explicit `[start,end)` interval + timezone; period locking | Statutory period; restatement policy | — | PROFESSIONAL REVIEW REQUIRED | G2 |
| D-20 | PII out of the ledger; crypto-shred; erasure event | **Legal — mandatory**; retention periods | — | PROFESSIONAL REVIEW REQUIRED | G0 / G3 |

### 5.1 What can be settled as product conventions without professional advice

1. **Original-currency amount is authoritative**; converted values are derived and recomputed (D-05).
2. **Every event stores all relevant dates** — order, shipment, refund, charge, settlement — with the reporting basis named explicitly (D-06).
3. **Cost snapshots are immutable** and reversal uses the recognised cost, so cost is recognised exactly once (D-07).
4. **Discount allocation is pro-rata by line gross with largest remainder** and a deterministic tie-break (D-08).
5. **No refund basis is ever inferred**; an unconfirmed basis is a named, labelled assumption with a version (D-09).
6. **Fee allocation excludes tax and the fixed component**, which are carried as an explicit order-level residual (D-13).
7. **Periods are explicit timezone-aware intervals**, assigned at ingestion, closed by an auditable lock (D-18).
8. **PII never enters the ledger**; erasure is a pseudonymisation event, not a mutation (D-20).

These eight are engineering decisions with no statutory meaning. They can be adopted now, provided they are recorded as conventions and not described as accounting policies.

### 5.2 What must not be presented as policy until professionally reviewed

1. Which FX rate convention applies, and where exchange differences are shown (D-05).
2. Whether shipment or delivery is the transfer-of-control point (D-06).
3. Which inventory valuation method is acceptable and how a difference from statutory accounts is disclosed (D-07).
4. Whether discounts and refunds are contra-revenue or expense (D-08, D-09).
5. When payment fees are recognised (D-13).
6. The statutory period definition and restatement policy (D-18).
7. Retention periods and the interaction of erasure with legal-obligation retention (D-20).

### 5.3 What cannot be decided at all yet

**D-09's net-versus-gross basis.** No convention, preference or approval can settle it. It requires the F-01 provider evidence fixture.

---

## 6. Consolidated professional-review brief

If you take one page to an accountant, take this.

| # | Question | Decision | Consequence of no answer |
|---|---|---|---|
| 1 | Transaction-date spot or period-average FX for management reporting; are exchange differences a separate line? | D-05 | Product margins absorb unexplained FX noise |
| 2 | Shipment or delivery as transfer of control; may management use order date with a documented reconciliation? | D-06 | Two revenue figures with no reconciliation the merchant can follow |
| 3 | Is moving weighted average acceptable for product-level margin when statutory accounts use another method, with disclosure? | D-07 | Product margins cannot be published as "profitability" |
| 4 | Contra-revenue or expense presentation for discounts and refunds? | D-08, D-09 | Figures shown to merchants may not reconcile to their accounts |
| 5 | Fees recognised at charge or at settlement? | D-13 | Cross-period fee misstatement |
| 6 | May a management period on order date coexist with a statutory period on shipment date, and what disclosure is required? | D-18 | Statutory reporting cannot be produced from the platform |
| 7 | Statutory retention periods, and may identity be erased while amounts are retained? | D-20 | Erasure requests cannot be honoured lawfully |

**Tax-specific:** refunded tax handling (D-09), discount effect on taxable value (D-08), FX for tax filings (D-05), retention of tax records (D-20).
**Legal-specific:** erasure vs legal obligation (D-20) — mandatory before any real merchant data is stored.

---

## 7. G0 and G2 readiness

**G0 (foundation) is blocked on four items**, all now prepared for decision:

| Item | State |
|---|---|
| D-05 storage model | Convention ready to adopt; rate policy awaiting review |
| D-20 PII separation | Architectural constraint identified; legal position awaiting review |
| Period representation (D-18) | Convention ready to adopt |
| Reporting-currency model | Ready, subject to D-05 |

**G2 (financial correctness) is blocked on five items:**

| Item | State |
|---|---|
| D-06 / D-07 period consistency (F-1) | **Needs your decision** — the register pairing mismatches revenue and cost |
| D-09 refund basis | **Blocked on F-01 provider evidence** |
| D-08 allocation | Ready to adopt as a convention |
| D-13 fee allocation | Ready to adopt, with the residual rule in F-3 |
| D-18 dual periods | Convention ready; statutory alignment awaiting review |

**What this report changes:** nothing in the register. It converts eight undifferentiated "outstanding" items into a decision-ready set, and it identifies that three of them are not single decisions at all but a convention plus a policy that were being asked for as one.

**What it does not do:** it does not close D-09, does not capture F-01, does not change any status, and does not authorise implementation.

---

## 8. Evidence and repository status

- **Verified in this report:** all arithmetic in §3, §4.1–§4.8, reconciled both by direct computation and by delta from the order-level total (§4.5, §4.6). The reconciliation checks are stated so you can reproduce them.
- **Not verified:** every provider fact — refund basis, per-order fee retrievability, per-receipt cost availability, alignment of our discount allocation with the provider's. None was tested against a live payload.
- **Not performed:** no code, schema, migration, test, integration, credential, environment variable, Supabase connection or deployment.
- **Register status:** unchanged by this report — APPROVED 0 · APPROVED WITH CONDITIONS 1 (D-15) · DEFERRED 1 · OUTSTANDING 10 · PROFESSIONAL REVIEW REQUIRED 7. F-1 is a management-design direction, not a D-decision, so it does not alter these counts.
- **Files:** this document added; `decision-register.md` gains a cross-reference only.

---

## 9. Decisions this report asks of you

1. **Adopt the eight product-design conventions in §5.1 as conventions** (not policies).
2. ~~**Resolve F-1:** should the management view match revenue and COGS on the same event, and if so which one?~~ **ANSWERED 2026-11-17** — two-view design approved as the proposed management design, order-date Commercial Management View as the primary measure, fulfilment view separately, statutory treatment still subject to professional review. See F-1.
3. **Confirm D-09 stays open** until F-01 is captured, with the assumption-labelling interim position.
4. **Confirm the D-13 residual rule** (tax-related fee and fixed component stay at order level).
5. **Confirm D-20's PII-separation constraint** as a G0 schema requirement.
6. **Authorise the professional-review brief in §6** to go to an accountant and a lawyer.
7. **Note that no recommendation here is approved**, and that D-06 option C and D-16 option B remain prohibited without explicit approval.
