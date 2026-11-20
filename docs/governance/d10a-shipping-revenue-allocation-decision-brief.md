# D-10a — Shipping Revenue Allocation: Revised Decision Brief

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Decision:** D-10a — how shipping revenue charged to a customer is attributed across the order's quantities and shipments
**Previous proposal:** pro-rata by shipped line net revenue, **residual on the final shipment**
**Founder direction, 2026-11-20:** **the previous wording is not approved.** The residual-on-final-shipment rule is rejected. Refine and return for approval.
**Status after this brief:** **OUTSTANDING — revised proposal awaiting founder approval**
**Gate:** G2
**Prepared:** 2026-11-20

---

## 1. Why the previous proposal was wrong

The rejected rule resolved rounding **at shipment time**: allocate across shipments, and give the last shipment whatever was left over.

Two defects, and the second is fatal:

1. **The final shipment carries a plug.** Its recognised shipping revenue is whatever the arithmetic happened to leave, which is not the same as what its contents were actually charged. The figure is unfalsifiable — there is nothing to check it against.
2. **Shipment order becomes economically meaningful.** Re-ship the same units in a different sequence and the per-shipment figures change, with no change in the underlying order. A financial engine whose output depends on the order in which warehouse events happen is not deterministic in the sense INV-15 requires.

The root cause is that allocation and recognition were conflated. **They must be separated:** allocate once, at order acceptance, down to individual quantities; recognise each quantity's share when that quantity ships. Rounding is then resolved once, across the whole order, where it is checkable — and every shipment recognises exactly the sum of its units' allocations, with no plug anywhere.

---

## 2. Revised rule

**R1 — Capture the shipping components separately.** Gross shipping revenue charged, shipping discount, and net shipping revenue are three distinct values (INV-04). **Allocation operates on net shipping revenue.** The discount is never allocated to products as a separate deduction: it is already reflected in the net, and allocating it again would double-count it.

**R2 — Allocate once, at order acceptance, across eligible quantities.** The allocation base for each unit is its **discounted line net revenue** — line gross less the line's allocated discount, expressed per unit after the line-level largest-remainder discount allocation. Every ordered unit is eligible at allocation time.

**R3 — Largest remainder at allocation time, across all units.**

```
raw_i      = S_net × base_i / Σbase
alloc_i    = floor(raw_i)
remainder  = S_net − Σ alloc_i
```

Distribute the `remainder` minor units one each to the largest fractional parts, ties broken by a stable deterministic key (unit identifier, ascending). Result: **Σ alloc_i = S_net exactly.** No plug is created and none is needed later.

**R4 — Recognition on shipment.** When a unit ships, its allocation is recognised as shipping revenue in the Fulfilment View on that shipment's date. Unshipped units' allocations remain **pending**, reported as a distinct balance, never recognised and never dropped.

**R5 — Direct association overrides pooling.** Where a shipping charge is explicitly associated with a specific shipment or line (for example a separate charge for a second parcel), that charge is allocated **only across the quantities it covers**, using the same base and the same largest-remainder method. Only shipping revenue with no explicit association is pooled across all eligible quantities. Two charges produce two independent allocations. This is more faithful than pooling, because the charge was actually incurred for those goods.

**R6 — Every reduction is an explicit event, never a re-allocation.**

| Reduction | Treatment |
|---|---|
| **Partial cancellation** | A **release** event for the cancelled units' allocations. Released, not recognised, not redistributed. |
| **Shipping discount** | Captured at order time and netted **before** allocation. Not a later event. |
| **Shipping refund** | A **reversal** event on the refund date, attributed to the unit or charge that caused it. |
| **Late adjustment** | A further explicit event. |

A later event **never** re-allocates amounts already recognised. Re-allocating after a cancellation would silently change figures the merchant has already seen, which INV-19 forbids and which would make historical reports unreproducible.

**R7 — Degenerate bases.** If `S_net = 0` (free shipping), all allocations are zero and nothing is recognised — the shipping **cost** is then a real cost with no offsetting revenue, and must be visible as such. If `Σbase = 0` (a fully discounted order) with `S_net > 0`, allocate equally by unit count with largest remainder. Neither case may divide by zero or silently drop revenue.

**R8 — Incomplete fulfilment.** Pending allocations are a first-class reported balance. At any point in time: **recognised + released + refunded + pending = S_net.**

---

## 3. Worked examples

All amounts in minor units. Common order: three units, two on line L1 and one on line L2.

| Unit | Line | Gross | Line discount | Discount per unit | Net per unit |
|---|---|---|---|---|---|
| A | L1 | 3,000 | 600 (2 units) | 300 | **2,700** |
| B | L1 | 3,000 | — | 300 | **2,700** |
| C | L2 | 4,000 | — | — | **4,000** |
| | | | | Σbase | **9,400** |

### E1 — Multiple shipments, pooled allocation

Shipping gross 1,000, discount 0, **S_net = 1,000**.

```
A: 1,000 × 2,700 / 9,400 = 287.234 → 287
B: 1,000 × 2,700 / 9,400 = 287.234 → 287
C: 1,000 × 4,000 / 9,400 = 425.532 → 425
                            floors = 999, remainder 1
fractions: A .2340, B .2340, C .5319 → C takes the extra unit
```

| Unit | Allocation |
|---|---|
| A | 287 |
| B | 287 |
| C | **426** |
| Σ | **1,000** ✓ |

Recognition:

| Event | Date | Units | Shipping revenue recognised |
|---|---|---|---|
| Shipment 1 | 2026-02-03 | A, C | 287 + 426 = **713** |
| Shipment 2 | 2026-02-18 | B | **287** |
| | | Σ | **1,000** ✓ |

Shipment 2 recognises 287 because that is unit B's allocation — **not** because it was last. Reordering the shipments to B first would give 287 then 713, with identical totals. Determinism restored.

### E2 — Partial cancellation (continues E1)

Unit B cancelled 2026-02-10, before shipment.

- **Release event:** 287, allocated to the cancellation date.
- Recognised 713 + released 287 = **1,000** ✓
- A stays 287 and C stays 426. Nothing is redistributed.

**Contrast with the rejected approach.** Re-allocating the 1,000 across A and C alone would give A ≈ 403 and C ≈ 597 — silently rewriting figures already recognised for A and C. That is exactly the mutation this rule exists to prevent.

### E3 — Free shipping

Shipping gross 0 (or gross 1,000 with a 1,000 discount) → **S_net = 0**.

| Unit | Allocation | Recognised |
|---|---|---|
| A, B, C | 0 | 0 |

If the actual outbound cost is 780, contribution profit falls by 780 with **no offsetting revenue**. That is the honest representation of a free-shipping promotion, and it must be visible rather than absorbed. INV-04 still requires the gross charge and the discount to be represented separately even when they net to zero.

### E4 — Shipping discount

Shipping gross 1,200, shipping discount 200 → **S_net = 1,000**.

Allocation is **identical to E1**: 287 / 287 / 426.

The 200 discount is **not** allocated to products. It is already inside the 1,000. Allocating it again would charge the products 200 twice — once through the reduced revenue and once as a cost.

### E5 — Multiple shipping charges, direct association

Same three units, shipped as two parcels with **separate charges**:

- Charge 1: 600, covering parcel 1 = units A and C
- Charge 2: 400, covering parcel 2 = unit B

**Charge 1** — base 2,700 + 4,000 = 6,700:

```
A: 600 × 2,700 / 6,700 = 241.791 → 241
C: 600 × 4,000 / 6,700 = 358.209 → 358
                       floors = 599, remainder 1
fractions: A .7910 > C .2089 → A takes it
```

A = **242**, C = **358**, Σ = 600 ✓

**Charge 2** — base 2,700 (unit B alone) → B = **400** ✓

| Unit | Pooled (E1) | Direct association (E5) | Difference |
|---|---|---|---|
| A | 287 | 242 | −45 |
| B | 287 | 400 | +113 |
| C | 426 | 358 | −68 |
| Σ | 1,000 | 1,000 | 0 ✓ |

**The two methods give different product-level answers, and the difference is material.** Direct association is more faithful because the 400 was incurred for B's parcel specifically. This is the one genuinely open sub-question in D-10a — see §5.

### E6 — Incomplete fulfilment at period end

E1's order, period closed after shipment 1 only.

| Balance | Amount |
|---|---|
| Recognised (A, C) | 713 |
| Pending (B, unshipped) | 287 |
| Released | 0 |
| Refunded | 0 |
| **Σ** | **1,000** ✓ |

The Commercial Management View has already recognised the full 1,000 at the order date. The 287 difference between the views is exactly the pending allocation, and it is the quantity-and-timing term that INV-27's reconciliation must explain. The two specifications close on each other.

### E7 — Shipping refund

After both shipments (1,000 recognised), the customer returns unit C and the merchant refunds the shipping attributable to it.

| Event | Amount |
|---|---|
| Recognised to date | 1,000 |
| Shipping refund — unit C | −426 |
| **Recognised net** | **574** |

The 426 ties exactly to C's original allocation, so the refund is attributable and auditable.

If the merchant instead refunds 500, then 426 reverses the unit's allocation and the remaining **74 is an order-level shipping refund adjustment**, explicitly labelled as such. It is neither a re-allocation nor a silent absorption.

---

## 4. What the rule now guarantees

| Property | How it is enforced |
|---|---|
| Exact reconciliation | Σ allocations = S_net at allocation time; recognised + released + refunded + pending = S_net at all times |
| No plug | Rounding resolved once, at allocation, across all units |
| Shipment-order independence | Recognition is the sum of units' allocations; sequence cannot change any figure |
| No double-counted discount | Allocation base is net; the discount is represented once, separately |
| No silent history change | Reductions are new events; nothing already recognised is recomputed |
| Pending is never lost | Pending is a reported balance, reconciled in the identity |
| Degenerate cases safe | Zero net and zero base both specified, neither divides by zero |

---

## 5. The one question this brief does not settle

**When shipping revenue has no explicit association, should it be pooled (E1) or should an association be inferred?**

| Option | Consequence |
|---|---|
| **A — Pool all unassociated revenue** (recommended default) | Simple, deterministic, no inference. Per-unit figures are an average, which is honest when the charge genuinely covers the whole order. |
| **B — Infer association from shipment contents** | More faithful where charges are really per-parcel, but the inference is unverifiable and changes per-unit figures depending on shipment grouping. |

**Recommendation:** option A as the default, with **R5 direct association applied wherever the provider actually exposes the link**. That is: use real evidence when it exists, never invent the link when it does not. Whether a shipping provider exposes charge-to-shipment association is a **provider verification item — not established here**.

---

## 6. Reconciliation to the shipping cost side

Shipping revenue and shipping cost are separate and stay separate (INV-04). Two consequences this rule makes visible:

1. **Splitting an order increases cost without increasing revenue.** Two parcels carry two actual label costs against one shipping charge. The merchant should see this as the real economic penalty it is.
2. **Shipping cost is naturally per shipment** (D-11 estimated, replaced by actual invoices), so D-11 and D-12 must key on **shipment**, not order. This is unchanged from the previous direction and remains a design requirement.

---

## 7. Status and what is being asked

**D-10a — OUTSTANDING.** This is a revised proposal, not an approved policy.

**Requested:** approval of R1–R8, and a decision on §5 (pooled default with direct association where evidenced).

**Not claimed:** no provider was called. Whether any shipping provider exposes a charge-to-shipment association is **unverified**. No code, schema, migration or test exists.

---

## 8. Boundary statement

Documentation only. No application code, schema, migration, test, Supabase provisioning, provider connection, merchant data or deployment. All arithmetic above is derived from stated inputs and reconciled in-line; it is a specification worked example, not output from an implemented engine.
