# P-3 — Two Recognition States and Typed Reduction Events: Model Validation

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Derived from:** `p3-post-acceptance-shipping-discounts-decision-brief-rev2.md` (rev 2)
**Responds to:** founder decision round 2026-11-20 §4
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not implemented, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (OPEN) · INV-26 · INV-27

---

## 1. The claim to be validated

Rev 2 replaced a five-bucket mutually-exclusive partition with **two recognition states plus a typed reduction event list**. This document tests whether that replacement preserves the three independent facts you named — fulfilment, return and physical disposition, payment and refund — and whether it remains coherent against both management views.

The claim under test is narrow and falsifiable:

> Every fact the five-bucket model recorded is recoverable from the two-state model, and at least one fact the five-bucket model *destroyed* is recoverable from the two-state model.

If the second half fails, the revision is unnecessary complexity and should be rejected.

---

## 2. The model

### 2.1 Recognition state — derived from shipment events only

```
recognised(u) ∈ { pending, recognised }
recognised(u) = true  ⟺  a shipment event covering u has occurred
```

That is the entire state machine. There is no third value, and no non-shipment event can change it. A return does not un-recognise. A refund does not un-recognise. A cancellation of a *different* unit does not affect it.

### 2.2 Reduction events — an append-only typed list

| Type | Cause | Reduces |
|---|---|---|
| `cancellation_release` | unit cancelled before shipment | that unit's allocation |
| `shipping_discount` | post-acceptance shipping discount | as scoped by the event |
| `shipping_refund` | money returned to the customer for shipping | as scoped by the event |
| `correction` | reversal or correction of any of the above | as scoped by the event |

Each event carries: type, amount, scope (unit / charge / order), provider timestamp, ingestion timestamp, stable identity, reason code, and evidence reference.

### 2.3 Derived quantities

```
effective_allocation(u) = A(u) − Σ unit-scoped reductions on u
recognised_revenue(u)   = recognised(u) ? effective_allocation(u) : 0
net_shipping_revenue    = Σ_u recognised_revenue(u) − Σ order-scoped reductions
```

**Two identities, not one** (this is the correction made to rev 2 after numerical verification):

```
(a) Allocation identity:  Σ_u effective_allocation(u) + Σ_u unit-scoped reductions(u) = CB
(b) Order-scoped reductions are separate lines, deducted from net revenue,
    period-assigned once to their effective date
```

Order-scoped reductions never touch a unit's allocation. Adding them to identity (a) double-counts — the defect found and fixed during verification of rev 2.

---

## 3. Proof that the three dimensions remain independent

### 3.1 The information-theoretic argument

A mutually-exclusive partition assigns each unit exactly one label from a set `S`. Independent dimensions assign a tuple from `D₁ × D₂ × D₃`.

The five-bucket model had `|S| = 5`. The two-state model plus orthogonal dimensions yields:

| Dimension | Values | Source |
|---|---|---|
| Fulfilment | `unshipped` · `partially_shipped` · `shipped` | shipment events |
| Return / physical disposition | `none` · `return_requested` · `returned_restocked` · `returned_damaged` · `written_off` | return and disposition events |
| Payment / refund | `unpaid` · `paid` · `refund_requested` · `refund_pending` · `refunded` · `refund_reversed` | payment events |

`|D₁ × D₂ × D₃| = 3 × 5 × 6 = 90` distinguishable combinations, against 5 in the partition.

**This is not merely more states.** The critical difference is the direction of the map. From an event history `H`, the partition produces a single label; the tuple produces three. The tuple map is injective on the facts we need to report, because each coordinate is computed from a disjoint event family. The partition map is not injective: multiple distinct histories collapse to one label, and the collapsed information is unrecoverable.

### 3.2 The decisive case: shipped, then returned

Unit A ships 10 February and is returned and restocked 20 March.

| Model | Fulfilment | Disposition | Payment | Was it shipped? | When? |
|---|---|---|---|---|---|
| Five buckets | *one bucket only* — must choose `returned` | — | — | **Lost** | **Lost** |
| Two-state + dimensions | `shipped` | `returned_restocked` | `refunded` | **Retained** | **10 Feb** |

The five-bucket model forces a choice between "shipped" and "returned". Whichever it picks, the other fact is gone. The two-state model records both because they come from different event families and neither overwrites the other.

**This is the fact the five-bucket model destroyed, and it is not a corner case — it is the normal life of a returned item.** The second half of the claim in §1 therefore holds.

### 3.3 Why `recognised` is derived from shipments and nothing else

`recognised(u)` answers one question: *has this unit's cost and revenue been realised?* Only a shipment event answers it. If a refund could set `recognised = false`, then a unit shipped in February and refunded in March would vanish from February's Fulfilment View — retroactively changing a closed period. That is exactly the silent restatement INV-16 and INV-27 forbid.

The refund is recorded as a reduction with its own date. The February view keeps its revenue; March records the reversal. Neither period is rewritten.

---

## 4. Reconciliation against both management views

The F-1 decision established two views that must not be conflated. The two-state model must serve both without being either.

| | Commercial Management View | Fulfilment View |
|---|---|---|
| **Date basis** | order date | shipment date, per unit |
| **Revenue recognised** | order-accepted merchandise net revenue + shipping revenue at order date | per-unit merchandise net revenue + that unit's `effective_allocation` on its shipment date |
| **COGS basis** | cost snapshot at order date | cost snapshot at shipment date |
| **Quantities included** | all accepted quantities | only `recognised(u) = true` units |
| **Requires per-unit shipment timestamps?** | No | **Yes** |
| **Treatment of unshipped units** | included in the order-date measure | excluded until shipped |
| **Treatment of returns** | separate adjustment event at return date | separate adjustment event; the shipment remains recognised |

### 4.1 The structural point

**The Fulfilment View cannot be computed from a single collapsed state field.** It needs, per unit, *whether* it shipped and *when*. A partition that overwrites `shipped` with `returned` destroys the input the Fulfilment View requires. This is why the two-state model is not merely tidier — it is a precondition for the view the founder approved in F-1.

### 4.2 Non-aggregation

`calculation_view` is an element of the INV-27 identity. A Commercial View figure and a Fulfilment View figure have different identities and **must never be summed, compared or averaged**. Their difference is not "realised profit" — it is a timing difference that requires an explicit reconciliation calculation to explain, never a silent subtraction.

---

## 5. Worked examples

**Order O-1**, accepted 20 January. Units A, B, C. Merchandise net revenue: A 2,700 · B 2,700 · C 4,000 (total 9,400). COGS: A 1,000 · B 1,000 · C 1,500. Shipping revenue `S_net = 1,000`, allocated by largest remainder over the 9,400 base: **A 287 · B 287 · C 426**. `CB = 1,000`.

### E-1 — Partial shipment

A ships 10 February; B and C ship 5 March.

| View | Period | Revenue | Shipping revenue | COGS |
|---|---|---|---|---|
| Commercial | January | 9,400 | 1,000 | 3,500 |
| Fulfilment | February | 2,700 | 287 | 1,000 |
| Fulfilment | March | 6,700 | 713 | 2,500 |

Fulfilment February + March = 9,400 revenue and 1,000 shipping revenue, reconciling to the Commercial January figure — because no reductions have occurred. Units B and C are `pending` throughout January; their exclusion is what makes February's figure 2,700 rather than 9,400.

### E-2 — Cancellation before shipment

C is cancelled 1 February, before shipping. `cancellation_release` 426 scoped to unit C.

| Unit | `recognised` | `effective_allocation` | Contributes to Fulfilment View? |
|---|---|---|---|
| A | pending | 287 | not yet |
| B | pending | 287 | not yet |
| C | pending | 426 − 426 = **0** | never — it never ships |

Identity (a): `(287 + 287 + 0) + 426 = 1,000 = CB` ✓. Net shipping revenue once A and B ship: `287 + 287 = 574`. The 426 is released, not lost.

### E-3 — Refund before shipment

B is refunded 2 February, before shipping. Payment dimension: `refunded`. Fulfilment dimension: `unshipped`. `shipping_refund` 287 scoped to unit B.

| Fact | Retained? |
|---|---|
| B never shipped | Yes — `recognised(B) = false` |
| B was refunded | Yes — payment dimension |
| B's shipping revenue was never recognised | Yes — `recognised_revenue(B) = 0` |
| 287 was returned to the customer | Yes — reduction event with date |

**The five-bucket model would record "refunded" and lose "never shipped".** That distinction matters: a refund before shipment means no COGS was ever incurred for B; a refund after shipment means it was. Conflating them misstates both revenue and cost.

### E-4 — Return after shipment

A ships 10 February, returned and restocked 20 March.

| Date | Event | Effect on Fulfilment View |
|---|---|---|
| 10 Feb | shipment | A recognised: revenue 2,700 + 287; COGS 1,000 |
| 20 Mar | return + restock | `shipping_refund` 287 scoped to A; merchandise refund handled under INV-02; disposition `returned_restocked` |

| View | Period | Shipping revenue | Note |
|---|---|---|---|
| Fulfilment | February | 287 | Unchanged by the March return — the period is closed |
| Fulfilment | March | 0 for A | The 287 reduction is dated March |

`recognised(A)` remains `true` after the return. **This is the case that proves the model**: A is still identifiable as having shipped on 10 February, while also being returned, restocked and refunded.

### E-5 — Multiple reduction events on one unit

Unit C receives three reductions:

| Date | Type | Amount | Scoped to |
|---|---|---|---|
| 1 Feb | `shipping_discount` | 100 | C |
| 1 Mar | `shipping_refund` | 200 | C |
| 5 Mar | `cancellation_release` | 126 | C |

`effective_allocation(C) = 426 − (100 + 200 + 126) = 0`.

| Question | Two-state model | Five-bucket model |
|---|---|---|
| Total reduction? | 426 | 426 |
| Which reasons applied? | **All three, with dates** | **One bucket — two reasons lost** |
| Was C discounted, refunded, or cancelled? | **All three, separately** | Cannot say |
| Identity (a) | `(287 + 287 + 0) + 426 = 1,000` ✓ | Holds, but unexplained |

**No information is lost when several events affect the same quantity** — this is the direct answer to your §4 requirement. The three events are independent records with independent dates; their sum is derived, not stored. A partition must choose one, and the choice is the loss.

### E-6 — Reduction arriving before its unit ships

C's `shipping_discount` of 100 arrives 1 February; C ships 5 March.

At 1 February, `recognised(C) = false`, so `recognised_revenue(C) = 0` regardless of the reduction. At 5 March, C is recognised at `effective_allocation(C) = 426 − 100 = 326`.

The reduction did not need to wait for the shipment, and the shipment did not need to wait for the reduction. **The two dimensions are genuinely orthogonal** — this example is only computable because they are.

---

## 6. Conservation and failure behaviour

| Check | Rule | Failure behaviour |
|---|---|---|
| Allocation identity (a) | `Σ effective + Σ unit reductions = CB` | Calculation fails closed |
| Order-scoped separation (b) | order-scoped reductions never enter (a) | Fails closed |
| Recognition monotonicity | `recognised` never returns to `pending` | Fails closed |
| Reduction amount | must use largest remainder over the target total, never independent per-unit rounding | Fails closed (CX-6: naive rounding gives 334 against a 333 target) |
| Reduction ceiling | per P-4/P-5 control | Excess classified, never discarded |
| Missing scope on a reduction | treated as order-scoped, never invented | Recorded, flagged |

---

## 7. What this model does not solve

| Not solved here | Where it belongs |
|---|---|
| Whether a return provision should exist | D-14 — **deferred, and its omission must be disclosed** |
| Statutory revenue recognition timing | Professional accounting review (F-1 keeps this open) |
| Merchandise refund ceilings | INV-02 — **still open pending F-01 evidence** |
| Whether `effective_allocation` may go negative | Prevented by the unit cap in P-4/P-5, not by this model |
| Whether an order-scoped reduction belongs to product-level margin reporting | OQ-P7a-3 — professional review |

---

## 8. Verdict on the claim

| Claim | Result |
|---|---|
| Every fact the five-bucket model recorded is recoverable | **Yes** — the two states plus typed events are a superset |
| At least one fact it destroyed is recoverable | **Yes** — E-4: a returned unit remains identifiable as shipped, with its date |
| The model serves both management views | **Yes**, and the Fulfilment View is *only* computable with it (§4.1) |
| No information is lost under multiple events | **Yes** — E-5 |
| The model is simpler | **Yes** — one binary state plus an event list, replacing five buckets and their transition rules |

The revision is justified and is not unnecessary complexity.

---

## 9. Recommendation

Adopt the two recognition states plus typed reduction events, with the three dimensions kept orthogonal and `recognised` derived from shipment events alone.

**MVP:** recognition state, typed reductions, identity (a) and (b), recognition monotonicity, largest-remainder reduction amounts.
**Pilot:** the full disposition vocabulary (restocked / damaged / written off) — MVP needs the dimension to exist, not every value populated.

**This is a recommendation, not an approved rule.** D-10a and INV-26 are unchanged; C-10a.1 remains open. No test has been executed and no model has been implemented.
