# P-7a — Unallocated Shipping Revenue: Display Specification

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision clarification 2026-11-20 §6
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not implemented**
**Affected decisions:** D2-14 (O-C retained) · D2-01 · R3-A4

---

## 1. Position retained

Order-level treatment (**O-C**) remains the recommended MVP approach: when `Σ weight = 0` for a charge scope, the revenue is held in an explicit `unallocated_shipping_revenue` bucket, visible at order and store level, and excluded from product margins **with disclosure**.

This document adds only the display rules. It does not reopen the O-C choice.

---

## 2. When the bucket arises — and when it must not

The distinction matters because two situations look similar and require opposite treatment:

| Situation | Correct treatment | Why |
|---|---|---|
| Eligible quantities exist but every weight is zero (e.g. all items 100% discounted, so discounted line net revenue is zero) | **Unallocated bucket** | Money was collected; no defensible allocation basis exists |
| **No** eligible quantities — every quantity cancelled or released before acceptance | **Cancellation release**, not unallocated | The shipping charge is being released; parking it as unallocated would hide a refundable amount |
| Association evidence invalid or conflicting | **Pooled**, not unallocated | A basis exists in principle; the evidence for it is unusable |

Conflating these three would produce the worst outcome available: a refundable amount reported as unallocated revenue, which inflates contribution profit on an order where the merchant owes money back.

---

## 3. Display at order level

The bucket is shown **per charge scope**, not merged into a single order figure (OQ-P7a-1, R3-A4).

```
Order #1042 — shipping revenue
  Charge c1  (label, 700)      → allocated: direct to line 1
  Charge c2  (label, 500)      → UNALLOCATED (zero allocation base)
                                   reason: all eligible items have zero net revenue
  Total shipping revenue collected          1,200
  Allocated to items                        700
  Unallocated (not attributable)            500
```

Per-charge display is recommended because orders routinely carry charges with different bases — a direct label plus a pooled repack charge, for example. Merging them at order level would make a partly-attributed order look wholly unattributed.

---

## 4. Display at store level

The bucket appears as a **named line in the contribution build-up**, positioned so the merchant can see it is inside the store total and outside the product margins:

```
Product contribution profit (all items)                3,000
+ Unallocated shipping revenue (not attributable)      1,200
− Advertising expenditure                                500
= Store contribution profit before unallocated ad spend 3,700
```

The disclosure requirement is explicit and must appear wherever the product figure appears:

> **Product contribution margins exclude 1,200 of shipping revenue that could not be attributed to any item.**

Without that sentence the merchant will add product margins, fail to reach 3,700, and conclude the platform has lost money — or worse, trust the product margins as the whole picture.

---

## 5. Prohibited presentations

| Prohibited | Consequence if allowed |
|---|---|
| A per-product row showing `shipping revenue 0` | Implies the item was assessed and scored zero, rather than never attributable |
| Distributing unallocated revenue equally by count (O-B) | Credits a fully discounted item with revenue and makes it appear contribution-positive — 334 of a 1,000 charge across three zero-revenue units |
| Labelling the bucket "pending allocation" | It is not pending. It is final for that calculation identity; no future step allocates it |
| Netting it silently against shipping cost | Hides collected revenue inside a cost line; destroys the gross-vs-net distinction |
| Excluding it from the store total to make margins reconcile | Understates store profit and breaks `Σ direct + Σ pooled + Σ unallocated = S_net` |
| Reclassifying it as direct after late evidence, without a new version | Violates D2-04: late evidence annotates and may propose a correction; it never applies one |

**On "pending":** a correction *may* later establish direct association and move the amount through a new `allocation_version`. That is a versioned correction with a new calculation identity, not a pending step in the current one. The current figure is complete and final for its identity.

---

## 6. The accounting question — explicitly open

Two readings both produce the same store contribution profit and differ in what the line *means*:

| Reading | Treatment | Store total | Difference |
|---|---|---|---|
| **A — Revenue outside product margins** | Recognised as shipping revenue at order level, excluded from product margins | 3,700 | Gross margin includes it; product margins do not |
| **B — Order-level recovery of shipping cost** | Presented as an offset against order-level shipping cost rather than as product revenue | 3,700 | Same bottom line; different gross presentation and different "shipping revenue" figure |

**The bottom line is identical; the reported shipping revenue and gross margin are not.** That is precisely why this is not a presentation preference and cannot be settled by product design.

**OQ-P7a-2 remains open and requires professional accounting review.** I am not recommending A over B. My recommendation is limited to: hold the amount in a named bucket, keep it reconciled, disclose its exclusion from product margins, and let the accounting treatment determine which reading the statutory reports use.

---

## 7. Decisions requested

| # | Decision | Recommendation |
|---|---|---|
| R3-C1 | Confirm O-C order-level treatment as the MVP approach (re-affirming D2-14) | Approve |
| R3-C2 | Confirm per-charge-scope display rather than per-order aggregation (OQ-P7a-1) | Approve |
| R3-C3 | Confirm the mandatory disclosure sentence at §4 | Approve |
| R3-C4 | Confirm the prohibited list at §5, including that the bucket is never described as "pending" | Approve |
| R3-C5 | Confirm OQ-P7a-2 stays open for professional review, with no reading selected now | Approve |

No interface exists, and no figure in this document was produced by a running system.
