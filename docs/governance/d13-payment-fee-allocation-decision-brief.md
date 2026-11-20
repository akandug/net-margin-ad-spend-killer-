# D-13 — Payment Processing Fee Allocation: Founder Decision Brief

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Decision:** D-13 — Payment processing fee allocation
**Register recommendation:** A — pro-rata over line net revenue including shipping revenue, largest remainder; allocated once at charge time
**Status:** **OUTSTANDING — unchanged.** This brief prepares the decision. It does not approve it, and it does not change the register.
**Gate:** G2 (financial correctness)
**Prepared:** 2026-11-17

---

## 0. What you are being asked to decide

One question, with a narrow scope:

> **Should the part of a payment fee that cannot be attributed to any product be spread across products, or carried as an explicit order-level cost?**

Everything else in D-13 (the pro-rata base, largest-remainder rounding, allocate-once-at-charge-time) is already the register recommendation and is not in dispute. The disputed part is the **residual**.

The recommendation is to carry it at order level. The argument for that is set out below, including the case against it.

---

## 1. Why a payment fee has both variable and fixed components

A card payment fee is not a percentage of your revenue. It is a sum of charges, each computed on a **different base**:

| Component | Base it is charged on | Varies with |
|---|---|---|
| Variable rate | The **full amount charged to the card**, including tax and shipping | Order value |
| Fixed per-transaction amount | Nothing — it is per charge | **Number of charges**, not value |

Three consequences follow, and they are the whole of D-13:

1. **The variable component is charged on tax.** Tax is collected on behalf of a tax authority and is not merchant revenue (INV-03). A fee computed on tax is therefore a cost with no corresponding revenue. It cannot be attributed to a product, because the base it was computed on is not a product's revenue.
2. **The fixed component is per charge, not per product or per unit of value.** It does not scale with the order, so no product-level allocation base can represent it.
3. **Both are charged on the order as a whole.** The processor does not know which product caused the charge.

The merchant pays one fee; the fee's components have three different bases. Any allocation must decide what to do with the two components that have no product base.

---

## 2. Why part of the fee cannot be reliably attributed to a product

Attribution requires a defensible causal link between the cost and the product. For payment fees:

| Component | Attributable to a product? | Why |
|---|---|---|
| Variable fee on merchandise revenue | **Yes, proportionally** | Charged on product revenue; scales with it; a product-level base is meaningful |
| Variable fee on **shipping revenue** | **Yes, proportionally** | Charged on shipping revenue, which is attributable to lines |
| Variable fee on **tax** | **No** | Tax is not revenue; the base is a liability, not product economics |
| **Fixed per-transaction** | **No** | Incurred once per charge regardless of which products are in it |

So the residual is not an accounting nicety. It is the arithmetic consequence of two components whose bases are structurally not product-level.

**Note what this means for a merchant selling zero-rated goods.** A merchant selling both standard-rated and zero-rated products (food, books, children's clothing in some jurisdictions) pays a tax-related fee only on the rated lines. Spreading that fee across all products by revenue charges the zero-rated products for a tax they never incurred. That is not a rounding difference; it is a wrong cost on a whole product category.

---

## 3. The proposed treatment

**Allocate the merchandise-and-shipping component; carry the remainder as an explicit, named order-level cost line.**

Rules:

1. The allocation base is **line net revenue including shipping revenue** — the same base the fee was actually charged on, minus tax.
2. Rounding is **largest remainder**, so allocated fees sum exactly to the allocatable component. No rounding drift.
3. The residual is **named, not hidden**: a distinct order-level line, drilled down to its two causes (tax-related fee, fixed component).
4. The fee is allocated **once, at charge time**. A later correction is an adjustment event, never a reallocation — consistent with the append-only rule (INV-19) and with reproducible snapshots.
5. The fee is labelled **estimated** until the processor's actual per-charge fee is obtained (INV-14). If it is never obtainable, every fee is estimated and must say so.
6. **Reconciliation check, enforced:** allocated fees + residual = total fee, and Σ line contribution + order-level items = order contribution. If either fails, the calculation is wrong and must fail loudly rather than present a figure.

---

## 4. Worked example — 63 of 507 (12.4%)

This is the example already recorded in `g0-g2-decision-preparation.md` §4.6, reproduced here so the decision can be taken on the numbers.

**Inputs.** Order charged **16,452** minor units. Total fee **507**. Two lines, shipping revenue 900, shipping cost 780, fulfilment 250, COGS L1 4,200 / L2 1,500.

**Step 1 — decompose the fee by the base it was actually charged on.**

| Component | Base | Fee |
|---|---|---|
| Variable 2.9% on merchandise + shipping | 15,300 | 443.7 → **444** |
| Variable 2.9% on **tax** | 1,152 | 33.4 → **33** |
| Fixed per-transaction | — | **30** |
| **Total** | 16,452 ✓ | **507** ✓ |

**Step 2 — allocate only the allocatable component (444).**

| Line | Net revenue | Shipping share | Allocation base | Fee |
|---|---|---|---|---|
| L1 | 10,800 | 675 | 11,475 (75%) | 333 |
| L2 | 3,600 | 225 | 3,825 (25%) | 111 |
| **Allocated** | | | 15,300 ✓ | **444** ✓ |

Largest remainder check: floors 332 + 110 = 442; remainder 2; fractional parts .775 (L1) and .925 (L2) → L2, then L1 → 333 / 111 = 444 ✓.

**Step 3 — the residual.**

```
tax-related fee      33
fixed component      30
                    ---
residual             63     = 12.4% of the 507 fee
```

**Step 4 — contribution, and the reconciliation that proves nothing was lost.**

| Line | Net revenue | COGS | Allocated fee | Line contribution |
|---|---|---|---|---|
| L1 | 10,800 | 4,200 | 333 | **6,267** |
| L2 | 3,600 | 1,500 | 111 | **1,989** |

```
6,267  + 1,989          = 8,256    lines
        + 900  shipping revenue
        - 780  shipping cost
        - 250  fulfilment
        -  63  fee residual
                          --------
                            8,063   order contribution  ✓
```

The 8,063 is reached identically whether the residual is carried at order level or spread across lines — see §5. That is the crucial point, and it is why this decision is about **product-level truth, not about order profit**.

---

## 5. What happens if the residual is allocated to products instead

This is the strongest objection to the recommendation, so it deserves a precise answer.

**The order and store totals do not change.** Spreading 63 pro-rata (largest remainder) gives L1 −47 and L2 −16:

| Treatment | L1 contribution | L2 contribution | Order contribution |
|---|---|---|---|
| Residual at order level (**recommended**) | 6,267 | 1,989 | **8,063** |
| Residual spread to products | 6,220 | 1,973 | **8,063** |

So the merchant's total profit is identical. **The only thing that changes is product-level profit** — which is exactly the number the merchant uses to decide which product to scale and which campaign to pause. The damage is concentrated precisely where this product makes its money.

Three distortions follow, in increasing order of severity:

**(a) Pro-rata spreading hides the fixed component's true incidence.** Spreading by revenue gives every product the same *percentage* burden. But the fixed fee is per charge, so its burden is per *transaction*, which is a much larger share of a cheap product's revenue than of an expensive one. Illustration — one order containing one unit each of a 500 product and a 50,000 product:

| Product | Revenue | Fixed fee 30 spread pro-rata | True fixed incidence (per unit) | Error |
|---|---|---|---|---|
| Cheap | 500 | 0.27 | 15 | **under-costed by 14.73** (2.9% of its revenue) |
| Premium | 50,000 | 29.73 | 15 | over-costed by 14.73 |

The cheap product looks better than it is, systematically, in every catalogue. The merchant then over-scales low-value items and under-scales premium ones — a direct, repeating commercial error, not a rounding artefact.

**(b) It charges products for tax they never incurred.** The 33 tax-related fee belongs to the rated lines only. Spread across the catalogue, zero-rated products absorb a cost caused by a tax they do not attract.

**(c) It destroys auditability.** `507 = 444 + 63` is checkable in one line. Once 63 is spread, the merchant cannot reconcile the processor's statement to the product costs, and — worse — a genuine fee error becomes undetectable, because the reconciliation that would have caught it is forced to zero by construction. You lose the control at the same moment you lose the transparency.

**The honest counter-argument.** A residual carried at order level is a cost the merchant may overlook when judging a product, and at very low order counts it can look like an orphan. The answer is **visibility, not allocation**: name it, show it in store-level contribution profit, and let it be drilled into. That keeps the cost impossible to ignore without corrupting product economics.

---

## 6. How the residual flows into store-level contribution profit

The residual is **already inside order contribution** (8,063 above), so it rolls up automatically. Nothing needs to be added at store level. What must exist is a **named drill-down** so it stays visible rather than dissolving into a total:

```
Store contribution profit
├── Σ order contribution profit
│    └── order 1
│         ├── Σ line contribution            (fees allocated to lines only)
│         └── order-level items
│              ├── shipping revenue
│              ├── shipping cost
│              ├── fulfilment and packaging
│              └── payment fee residual  ← named, drilled into two causes
│                   ├── tax-related fee
│                   └── fixed component
└── − unallocated advertising spend   (D-16 — must never be silently dropped)
```

Two properties this gives you:

1. **At any scope**, `total fee = allocated to lines + order-level residual`. The processor statement reconciles at store level as well as at order level.
2. **Scale makes the residual material, and visible.** At 30 minor units fixed per charge, 10,000 orders carry 300,000 minor units of pure fixed fees. That is not noise; it is a line the merchant should see and can act on (fewer, larger orders). Carried at order level, it is visible and attributable to order count. Spread across products, it disappears into margin and can never be managed.

---

## 7. Remaining questions requiring professional review

These are open and are **not** answered by this brief.

**Accounting:**

1. **Recognition timing.** Should the fee be recognised when the charge is created, or when the payout settles, where the two fall in different reporting periods? This affects D-18 period assignment and is a genuine policy choice, not a technical detail.
2. **Is the fixed component an order cost or a period cost?** Treating it as per-order is defensible when it truly is per charge. But processors also levy **monthly/platform fees**, which are period costs of a different class and must not be pulled into per-order allocation. The boundary between the two needs a policy.
3. **True-up presentation.** When the actual fee differs from the configured rate card, is the correction a **revision** of the original estimate or a **separate adjustment event**? The register's "allocate once" rule implies adjustment; the accounting presentation should be confirmed.
4. **Fee refunds where the fixed component is retained** (INV-22): the reversal must be attributed to the transaction that caused it, and the retained fixed component must not be reversed. Confirmation of the presentation is required.

**Tax:**

5. Whether the tax-related fee is recoverable or deductible, and whether it must be reported separately for tax purposes in the relevant jurisdiction.

**Provider evidence — not verified, and not claimed:**

6. **Does the processor expose an actual per-charge fee?** This brief does not assert that it does. If it does, fees can move from *estimated* to *actual*. If it does not, every fee stays estimated and that is a material disclosure for a margin product. This requires verification against official provider documentation and recorded payloads — the same discipline as P-01–P-05. **Nothing here has been verified against a live provider.**

**Currency (D-05 interaction):**

7. The fee is charged in the settlement currency while the order may be in a presentment currency. Which rate, and which date, applies to the fee conversion — and does the residual convert on the same basis?

---

## 8. Recommendation

**Adopt option A as recorded, with the residual carried as an explicit, named order-level cost.**

Reasoning in one line: the residual is not attributable to a product because its bases are not product revenue, and spreading it does not change order or store profit — it only corrupts the product-level figure that drives scale and pause decisions, while destroying the reconciliation that would catch fee errors.

**Conditions if adopted:** the reconciliation check in §3.6 must be enforced as a hard failure, the residual must be named and drilled down rather than absorbed, and the fee must be labelled estimated until actual per-charge evidence exists.

**Not approved by this brief.** At the time of writing, D-13 was **OUTSTANDING** pending your review. Following founder review on **2026-11-20** the **principle is approved in principle** — see §10, which records the conditions and the reviews that remain open. Items in §7 remain open regardless of your decision.

---

## 9. Boundary statement

Documentation only. No code, schema, migration, test, provider connection, credential, environment variable, Supabase provisioning or deployment. No fee figure here was obtained from a live processor; all arithmetic is reproduced from the recorded worked example in `g0-g2-decision-preparation.md` §4.6 and re-verified by the reconciliation in §4 step 4.

---

## 10. Founder direction, 2026-11-20 — approval in principle

**Status change: OUTSTANDING → APPROVED WITH CONDITIONS (in principle).** The principle is approved; the professional accounting and provider reviews remain **open**.

### 10.1 What was approved

The founder approved the principle that **the 63 minor-unit residual remains an explicit order-level payment-processing cost** rather than being artificially allocated to products, and that the **444 product-attributable variable fee is allocated using the approved deterministic method**.

### 10.2 Conditions attached to the approval

These are mandatory requirements, not preferences. Each is a testable property of the engine.

| # | Condition | Consequence if breached |
|---|---|---|
| **C-13.1** | **Preserve the complete processor fee** as its own recorded amount | The fee can no longer be reconciled against processor evidence |
| **C-13.2** | **Preserve the product-attributable component** separately from the total | Product-level margin becomes unauditable |
| **C-13.3** | **Preserve the order-level residual separately**, named and drillable | The residual becomes invisible and gets absorbed into product figures |
| **C-13.4** | **No artificial zero-residual reconciliation** — the engine must never force allocated + residual to equal the total by fabricating an allocation | The reconciliation control that catches fee errors is destroyed |
| **C-13.5** | **Reconcile against actual processor evidence where available** | Estimated fees are never trued up, and the estimate silently becomes the reported figure |
| **C-13.6** | **Distinguish estimated, provisional and confirmed fees** | A merchant cannot tell an estimate from an actual |
| **C-13.7** | **Support better evidence later without silently changing historical results** | Historical figures mutate, violating INV-19 |

**On C-13.4 specifically:** the founder's phrasing — avoid artificial zero-residual reconciliation — is stronger than the brief's original requirement. The brief asked that allocated + residual + tax-related + fixed equal the total (which they do, exactly: 444 + 63 + 33 + 30 = 570 ≠ 507 — see §10.3). The condition additionally forbids *ever* manufacturing an allocation to make a residual disappear. The residual is a fact to be displayed, not a defect to be smoothed.

### 10.3 Reconciliation identity, restated to match the conditions

The approved structure must satisfy, for every charge:

```
fee_total              = 507   (preserved in full)              C-13.1
fee_product_allocated  = 444   (deterministic, per §3)          C-13.2
fee_order_residual     =  63   (explicit, named)                C-13.3
fee_tax_related        =  33   (not product revenue based)
[`fee_fixed_platform`] =  30   (not product revenue based)
```

with the check:

```
fee_total  =  fee_product_allocated + fee_tax_related + fee_fixed_platform
507        =  444 + 33 + 30  =  507   ✓

fee_order_residual (63) is the portion of the 444 that is not product-attributable,
and is reported as an order-level cost. It is NOT an additional amount.
```

**This is the clarification the conditions demand.** The 63 is *inside* the 444, not beside it. Allocated 444 = product-attributable 381 + order-level residual 63. The 33 and 30 were never in the product allocation base, so they never enter the residual either. Stating it the other way — 444 allocated *plus* 63 residual — would double-count 63, and that is precisely the error C-13.3 and C-13.4 exist to prevent.

Restated fully:

```
fee_total               507
  ├─ fee_tax_related      33   order-level, not product-allocated
  ├─ fee_fixed_platform   30   order-level, not product-allocated
  └─ fee_variable        444   the product-allocation base
       ├─ allocated to products   381
       └─ order-level residual     63   (12.4% of 507, 14.2% of 444)
```

### 10.4 The evidence-grade requirement (C-13.5, C-13.6, C-13.7)

A fee figure carries exactly one grade at any time:

| Grade | Source | May it be reported as the fee? |
|---|---|---|
| `estimated` | Configured rate card applied to the charge | Yes, labelled estimated |
| `provisional` | Processor reports a pending or unsettled amount | Yes, labelled provisional |
| `confirmed` | Settled or charged amount from processor evidence | Yes, labelled confirmed |

Rules:

1. A grade may move **upward** (estimated → provisional → confirmed) only by recording a **new event** that references the evidence. It never mutates the prior event (INV-19).
2. A higher grade does **not** retroactively rewrite a published historical figure. It creates a **new calculation** under a new calculation version; the old calculation remains reproducible.
3. If a confirmed amount contradicts the estimate, the difference is an explicit **adjustment**, never a silent correction.
4. **The founder's forward-looking requirement:** do not assume future processors cannot supply more detailed fee evidence. The design must accommodate a processor that later exposes per-charge or per-line fee detail, without changing historical results silently. This means the allocation method and the evidence grade are separate, versioned concerns — a better-evidenced fee is a new event with a higher grade, not a rewrite of the method.

### 10.5 What remains open

Approval in principle does **not** close these. They are unchanged from §7.

| Open item | Owner | Blocks |
|---|---|---|
| Recognition timing — charge date vs settlement date across periods | Professional accounting review | D-18 interaction, G2 |
| Fixed component: order cost vs period cost; monthly platform fees | Professional accounting review | G2 |
| True-up presentation: revision vs adjustment event | Professional accounting review | G2 |
| Fee refund presentation where the fixed component is retained (INV-22) | Professional accounting review | G2 |
| Tax treatment of the tax-related fee | Tax review | G3 |
| **Does the processor expose an actual per-charge fee?** | Provider verification (official docs + recorded payload) | Fee evidence grade; G2/G3 |
| FX basis for a fee charged in settlement currency | D-05 decision | G0/G3 |

### 10.6 Implementation status

**None.** No allocation method, evidence-grade mechanism, reconciliation check or adjustment path has been implemented or tested. This section records an approved principle and its conditions, not a working control.
