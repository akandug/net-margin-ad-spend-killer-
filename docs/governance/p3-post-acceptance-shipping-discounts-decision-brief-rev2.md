# P-3 (Revised) — Post-Acceptance Shipping Discounts

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Supersedes:** `p3-post-acceptance-shipping-discounts-decision-brief.md` (rev 1, 2026-11-20)
**Reason for revision:** founder review 2026-11-20 §3 — challenge the five-bucket partition for necessity, overlap and redundancy
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not applied, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.5 · R6.1–R6.5

---

## 1. The challenge, and what it found

You asked three questions about the five buckets (`pending`, `recognised`, `released`, `refunded`, `discounted`). The answers:

| Your question | Answer |
|---|---|
| Are all five necessary? | **No.** Three of them are not states |
| Are any overlapping or redundant? | **Yes** — and worse, they are *lossy* |
| Can fewer, clearer event-derived states do the same job? | **Yes** — two states and a typed event list |

Rev 1's partition conflates **two different dimensions** into one mutually exclusive field:

1. **Recognition** — has this unit's allocation been recognised as revenue yet? (driven by shipment)
2. **Reduction** — has something reduced what is owed back to the merchant? (driven by cancellation, discount, refund, correction)

Because the five buckets are mutually exclusive, a unit can occupy only one at a time. That means **a refunded unit loses the fact that it was recognised**. The Fulfilment View needs both facts simultaneously: the merchant must be able to see that a unit shipped (and its revenue was recognised) *and* that a shipping refund subsequently reduced it. Rev 1's model cannot express this. That is a genuine defect, not a presentational one — it would have forced a join back to the raw event stream to answer a question the state model claimed to answer.

Rev 1 also mislabels reasons as states. `released`, `refunded` and `discounted` are three different *reasons a reduction happened*. They differ in meaning and audit trail, not in state.

---

## 2. The simpler model

### 2.1 Dimension 1 — recognition (2 states, from shipment events)

| State | Derivation |
|---|---|
| `pending` | The unit's accepted quantity has not shipped |
| `recognised` | The unit's accepted quantity has shipped |

Monotone: `pending → recognised` on the shipment event. **Never reverses.** A return does not un-ship goods. A refund does not un-ship goods. This is the property that makes the model able to hold both facts at once.

### 2.2 Dimension 2 — reductions (typed event list, additive)

Each reduction event carries: `type`, `amount`, `effective_date`, `stable_identity`, and optionally `scope_ref` (a unit or charge).

| Type | Meaning | Scope |
|---|---|---|
| `cancellation_release` | Goods will never ship; their allocation stops being owed | unit |
| `shipping_discount` | Merchant reduced the shipping charge after acceptance | unit or order |
| `shipping_refund` | Shipping money returned to the customer | unit or order |
| `correction` | Authorised correction of a prior allocation or reduction | unit or order |

Reductions are **additive and order-insensitive in total**. They are never edits to the allocation.

### 2.3 Derived values

```
effective_allocation(u) = A(u) − Σ reductions with scope u
recognised_revenue(u)   = recognised(u) ? effective_allocation(u) : 0
net_shipping_revenue    = Σ_u recognised_revenue(u) − Σ order-scoped reductions
```

### 2.4 Conservation

```
Σ_u effective_allocation(u) + Σ_u reductions(u) + Σ order-scoped reductions = CB
```

This is the conservation test proposed as PROP-D in the invariant matrix.

### 2.5 What this replaces

| Rev 1 bucket | Expressed instead as |
|---|---|
| `pending` | recognition = `pending` |
| `recognised` | recognition = `recognised` |
| `released` | reduction event, type `cancellation_release` |
| `refunded` | reduction event, type `shipping_refund` (unit or order) |
| `discounted` | reduction event, type `shipping_discount` (unit or order) |

**Five mutually exclusive buckets → one boolean and a typed list.** Fewer concepts, no state machine, and no lost information.

---

## 3. Does the simplification preserve behaviour?

Yes where rev 1 was right, and strictly better where rev 1 lost information.

| Case | Rev 1 (5 buckets) | Revised (2 dims) | Same net revenue? |
|---|---|---|---|
| Unit never ships, cancelled | `released`, revenue 0 | `pending` + `cancellation_release` | Yes, 0 |
| Unit ships, nothing else | `recognised`, revenue A | `recognised`, no reductions | Yes, A |
| Unit ships, then refunded | `refunded`, revenue 0 — **recognition fact lost** | `recognised` + `shipping_refund` | Yes, 0 — **recognition fact kept** |
| Unit ships, discount before shipment | `discounted`, revenue A−d | `recognised` + `shipping_discount` | Yes, A−d |

Net revenue is identical in every case. The revised model additionally retains the recognition fact in row 3, which rev 1 discarded.

---

## 4. R6.1–R6.5 under the revised model

| Rule | Status | Revised expression |
|---|---|---|
| **R6.1** Order-time discount netted before allocation, no event | Preserved | Unchanged. It reduces `S_net` before allocation; it is not a reduction event |
| **R6.2** Post-acceptance **unit-scoped** discount → explicit event | Preserved | `shipping_discount`, `scope_ref = unit` |
| **R6.3** Post-acceptance **order-scoped** discount → no unit owner | Preserved | `shipping_discount`, no `scope_ref`, order-level line (D-13 residual precedent) |
| **R6.4** Discount and refund distinguished | Preserved | Distinguished by `type`, never by amount or sign |
| **R6.5** Effective date from the provider event | Preserved, **and partly retired** | Effective date is retained for *period assignment* only. It is **not** needed for ordering the ceiling — see the combined P-4/P-5 brief §5 |

The R6/C-10a.5 contradiction is resolved the same way rev 1 resolved it — original allocation preserved, every post-acceptance change an explicit event — but with the event vocabulary reduced from three buckets to three types on one list.

---

## 5. Worked examples

Base order **E1** (from the D-10a brief): units A, B, C with net revenue 2,700 / 2,700 / 4,000, `S_net = 1,000`. Largest-remainder allocation: **A = 287, B = 287, C = 426** (sum 1,000).

| ID | Events | Recognition | Reductions | Net shipping revenue by period |
|---|---|---|---|---|
| **P3-R1** | Order-scoped discount **200**, effective Feb, before shipment. A ships Feb; B and C ship Mar | A recognised Feb; B, C recognised Mar | order-scoped 200, Feb | Feb = 287 − 200 = **87**; Mar = 713; total **800** |
| **P3-R2** | Unit-scoped discount **100** on C, effective Feb, before C ships. A ships Feb; B, C ship Mar | as above | C 100 | Feb = 287; Mar = 287 + (426−100) = 613; total **900** |
| **P3-R3** | Both: order-scoped 200 (Feb) + unit-scoped 100 on C (Mar) | as above | order 200, C 100 | Feb = 287 − 200 = **87**; Mar = 287 + 326 = 613; total **700** |
| **P3-R4** | C ships Mar, then shipping refund **426** on C | all recognised | C 426 | Mar = 287 + 287 + 0 = 574; total **574** |
| **P3-R5** | C ships Mar, then partial shipping refund **300** on C | all recognised | C 300 | total = 1,000 − 300 = **700** |
| **P3-R6** | B cancelled before shipment | A, C recognised; B `pending` | B `cancellation_release` 287 | Feb = 287; Mar = 426; total **713** |
| **P3-R7** | C ships Mar, refunded 426, then a second refund request of 426 | all recognised | C 426 + duplicate | Second request rejected as duplicate by stable identity; if genuinely a second execution it becomes excess per P-4 §6; total stays **574** |
| **P3-R8** | C shipped, returned and restocked, shipping refunded 426 | all recognised | C 426 | total **574** — identical to R4, because a return does not change the shipping treatment |

Every figure reconciles to `Σ effective + Σ reductions = CB = 1,000`. Verified arithmetically (§7).

**Note on R1/R3:** the order-scoped discount is deducted in its **effective period** (Feb), not when goods ship. That is a period-assignment choice, recorded as OQ-P3-1. It does not affect the total, only the split between Feb and Mar.

---

## 6. Unresolved questions

| ID | Question | Needs |
|---|---|---|
| **OQ-P3-1** | Should an order-scoped post-acceptance shipping discount be period-assigned to its effective date (as here) or spread across shipments? Effective-date assignment is simpler and is what these examples use | **Professional accounting review** — it changes reported period margin |
| **OQ-P3-2** | Is a post-fulfilment shipping credit a **revenue reduction** or a **liability/cost**? These examples treat it as a revenue reduction because it reduces what the merchant keeps | **Professional accounting review** |
| **OQ-P3-3** | Does a discount backdated to a closed period trigger a restatement, or is it recognised in the open period with disclosure? | **Professional accounting review** (interacts with D-18) |
| **OQ-P3-4** | Should `correction` be a distinct reduction type or a new allocation version? Recommended: a distinct type for corrections to a *reduction*, a new allocation version for corrections to the *allocation* (P-1 §6) | Founder decision |

---

## 7. Verification status

Arithmetic for P3-R1 to P3-R8 was recomputed from the stated inputs in a throwaway script and reconciles to 1,000 in every case, with no balancing plug. That is **arithmetic verification of this document**, not execution of any specification test, and it is not evidence that the model is implemented.

The claim that the revised model is behaviour-preserving (§3) is an argument, not a test result.

---

## 8. Recommendation

**Adopt the two-dimension model: recognition (`pending`/`recognised`) from shipment events, plus a typed reduction event list. Retire the five mutually exclusive buckets.**

Rationale: it is smaller (2 + 4 types versus 5 exclusive states), it removes the state machine that C-10a.5 would otherwise require, and it fixes the information loss in rev 1's row 3 — a refunded unit retains its recognition fact, which the Fulfilment View needs.

MVP scope: recognition states, the four reduction types, order-scoped reductions, conservation. The `correction` type is pilot scope.

**This is a recommendation, not an approved decision.** D-10a's status is unchanged and C-10a.1 remains open.
