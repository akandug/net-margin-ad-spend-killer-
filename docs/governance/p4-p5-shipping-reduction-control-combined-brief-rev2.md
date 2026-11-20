# P-4 / P-5 (Combined, Revised) — Shipping Reduction Control Design

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Supersedes:** `p4-shipping-refund-ceiling-decision-brief.md` (rev 1, 2026-11-20) — P-4 and P-5 treated separately there
**Reason for revision:** founder review 2026-11-20 §4 — treat order-level and per-charge limits as one connected control; construct counterexamples
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not applied, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (OPEN)

---

## 1. Why these are one control, not two

Rev 1 presented an order-level ceiling (P-4) and a per-charge ceiling (P-5) as separate rules with separate tests. They are the same control at two scopes, and presenting them separately hid a defect: rev 1's per-charge rule, applied as a blanket cap, would have rejected legitimate refunds that span charges. §4.3 explains why and constrains it.

There is **one** invariant: cumulative reductions may not exceed the collectible balance, checked at every scope for which the reduction carries a valid reference.

---

## 2. The collectible balance

```
CB = S_net + Chg_add
```

where
- `S_net` = Σ accepted original shipping charges − Σ order-time shipping discounts (the same base the allocation uses)
- `Chg_add` = Σ additional shipping charges raised after acceptance

`CB` is the total shipping the merchant was entitled to collect. It is the ceiling for all shipping reductions.

### 2.1 Monotonicity — the property rev 1 was missing

**`CB` must be non-decreasing.**

Rev 1 allowed charges to be corrected downward, which would have reduced `CB` *after* reductions had already been counted against it. That is a genuine hole: `CB` could fall below committed reductions, making "remaining refundable" negative with no classification.

**Fix:** a downward charge correction is **not** a `CB` decrease. It is a reduction event of type `charge_correction`, counted in `R_total` like any other. `CB` only ever grows.

**Counterexample that breaks rev 1:** `S_net = 600`, `Chg_add = 400` (`CB = 1,000`), refund 800 accepted (remaining 200). Charge then corrected down by 300. If `CB` became 700, remaining = −100 — an unexplained negative. With monotonicity: `CB` stays 1,000, `R_total` becomes 1,100, excess = 100, flagged and classified. The arithmetic is explicable and the evidence is preserved.

### 2.2 The three scopes

| Scope | Applies when | Ceiling |
|---|---|---|
| Order | always | `R_total ≤ CB` |
| Charge | reduction carries a valid charge reference | `Σ R_c ≤ CB_c` (that charge's amount, including its additional charges) |
| Unit | reduction carries a valid unit reference | `Σ R_u ≤ A(u)` (the unit's original allocation) |

A reduction with no reference is checked at order scope only. It is never silently assigned to a scope.

---

## 3. Remaining refundable

```
remaining_order  = CB      − R_total
remaining_charge = CB_c    − Σ R_c     (only where a charge reference exists)
remaining_unit   = A(u)    − Σ R_u     (only where a unit reference exists)
```

---

## 4. Why the ceiling is *not* recognised revenue

Rev 1 considered using recognised shipping revenue as the ceiling and correctly rejected it. Restating the proof briefly because it is load-bearing:

`CB = 1,000`, only unit A (share 287) has shipped, goods for B and C never ship. Merchant refunds 287 for B's shipping. Against recognised revenue (287), the refund consumes the ceiling entirely and appears to breach it. Against `CB` (1,000), it is plainly within. **Using recognised revenue as the ceiling would block a legitimate refund for goods that never shipped** — the exact scenario your §4 flagged.

`CB` is a *collectibility* measure, not a *recognition* measure. Reductions reduce what the merchant keeps; recognition decides when it is reported. They must not be the same number.

### 4.1 The blanket per-charge rule is also wrong

Rev 1's P-5 applied the per-charge cap unconditionally. Consider: `c1 = 600` (covers units A and B), `c2 = 400` (covers unit C). All three units are returned and the merchant refunds 1,000 in a **single transaction with no charge reference**. A blanket per-charge rule must either reject the refund or invent a split between `c1` and `c2` — the latter is an unexplained allocation, which C-10a.1 forbids.

**Rule:** the per-charge cap applies **only** when the reduction carries a valid charge reference. Without one, the order-level cap governs. No split is ever invented.

### 4.2 Why the unit cap is still needed

`CB = 1,000` with `A = 287, B = 287, C = 426`. Only A has shipped. A shipping refund of **500** is attributed to unit A. Order-level: 500 ≤ 1,000, passes. But A's entire shipping was 287; a 500 refund "on A" either covers other units or is goodwill. The unit cap (500 > 287) surfaces 213 as excess for review. Without it, the merchant's product-level shipping revenue for A would go negative with no explanation.

### 4.3 The unit cap also catches double-counting

A unit cancelled before shipment gets a `cancellation_release` of 287. If a `shipping_refund` of 287 also arrives for the same unit, the total against `A(u) = 287` is 574 — breach, 287 excess. That is correct: the same 287 has been reduced twice by two different mechanisms. This is the shipping-side analogue of INV-01's no-duplicate-deduction rule and is why the unit cap is not optional.

---

## 5. Duplicate and out-of-order events

### 5.1 Duplicates

Every reduction carries a `stable_identity`. A repeat of the same identity is **not** a new reduction — it is discarded idempotently, and the discarded copy is recorded. No double counting, no reliance on arrival time.

### 5.2 Out-of-order — and a simplification

Rev 1 required a deterministic ordering rule (its test T7) so that "the accepted set" would be reproducible. **That requirement disappears under the revised disposition model.**

Because an over-ceiling reduction is now *classified* rather than *rejected* (§6), the **totals** are order-independent: `R_total` is a sum, and the excess amount is `max(0, R_total − CB)`, which does not depend on which event arrived first. Only the *label* of "which event is the excess" could vary, and since all excess goes to the same order-level `shipping_refund_excess` line, the label is cosmetic.

**Consequence:** no effective-date ordering, no tie-break and no arrival-sequence rule is needed for the ceiling. This removes a whole class of complexity and a class of non-reproducibility from rev 1. Ordering remains necessary for **allocation** (P-7b), not for the ceiling.

I am recording this as an improvement, not a claim that rev 1's T7 was wrong to be tested — it was testing a requirement that the revised model removes.

### 5.3 Cross-entity ordering — a genuine gap (CX-4)

Reductions and charges are different entity types with independent delivery. **A shipping refund can arrive before the charge it refunds.**

If the charge has not been ingested, `S_net = 0`, so `CB = 0`, and a legitimate 400 refund appears to be 400 excess. The formula is not wrong — its inputs are incomplete.

**Required behaviour:** the order is marked `charge_pending` and the evaluation is **deferred**, not concluded. If the charge is ingested within a bounded window, evaluation re-runs and the refund is correctly accepted. If the window expires, the refund is classified as excess *with the reason `charge_evidence_missing`*, which is distinguishable from a genuine over-refund.

This is not in rev 1 and it would have produced spurious excess classifications on real data, where webhook ordering is not guaranteed (P-02).

---

## 6. Reality over ceiling — the disposition rule

**Rev 1 quarantined over-ceiling refunds as if they could be refused. They cannot.** If money left the merchant's account, refusing to record it produces a ledger that disagrees with the bank.

| Step | Rule |
|---|---|
| 1 | Record the **full** amount as an actual refund fact. Always. It happened |
| 2 | Classify up to the remaining `CB` as a **shipping revenue reduction** |
| 3 | Classify the excess as **`shipping_refund_excess`** — an order-level adjustment, not a revenue reduction, carrying the reason code |
| 4 | Flag the order for review; never absorb the excess silently |
| 5 | If later evidence shows an un-ingested charge, reclassify the excess once, via a new calculation version |

Step 3 is an accounting question, not a technical one: whether the excess is a cost of sale or a liability requires professional review (OQ-P4-1). The system's job is to keep it visible and never to hide it in the revenue figure.

---

## 7. Worked tests

Base: `S_net = 1,000`, `Chg_add = 0` ⇒ `CB = 1,000`. Allocations `A = 287, B = 287, C = 426`.

| ID | Scenario | Arithmetic | Disposition |
|---|---|---|---|
| **T1** | One charge, one refund 300 | `R = 300 ≤ 1,000` | Accept; remaining 700 |
| **T2** | `c1 = 600`, `c2 = 400`; refund 400 (ref `c1`), refund 400 (ref `c2`), refund 300 (ref `c1`) | `R = 1,100 > 1,000` ⇒ excess 100; `c1`: 700 > 600 ⇒ excess 100 | Excess 100, both scopes agree |
| **T3** | Partial refund 426 on C | `R = 426 ≤ 1,000`; unit `426 ≤ 426` | Accept; remaining 574 |
| **T4** | Goods for B never ship; refund 287 for B | `R = 287 ≤ 1,000`; unit `287 ≤ 287` | Accept — the case rev 1's recognised-revenue ceiling would have wrongly blocked |
| **T5** | Same refund identity delivered three times | `R` counted once = 300 | Two copies discarded, recorded |
| **T6** | Duplicate provider event, different arrival timestamps | `R` counted once | Idempotent on stable identity |
| **T7** | T2's three refunds in reverse arrival order | `R_total = 1,100`, excess `= 100` | **Identical totals and identical excess** — order-independent |
| **T8** | Refund 1,200 when `CB = 1,000` | Excess `= 200` | 1,000 revenue reduction + 200 `shipping_refund_excess`, flagged |
| **T9** | Unit B cancelled (release 287) **and** shipping refund 287 for B | Unit `R = 574 > 287` ⇒ excess 287 | Double-count caught |
| **T10** | Refund 400 arrives before its charge; charge arrives within window | Deferred, then `CB = 1,000` | Accepted on re-run, no spurious excess |
| **T11** | Refund 400 arrives; charge never arrives; window expires | `CB = 0` | Excess 400 with reason `charge_evidence_missing` |

T2/T7/T8 arithmetic verified (see §9). T9 is the double-count case and is the strongest justification for the unit cap.

---

## 8. Counterexamples constructed to break the formula

You asked me not to approve a formula because it passes the supplied examples. These are attempts to break it.

| ID | Attack | Outcome |
|---|---|---|
| **CX-1** | Free-shipping order (`CB = 0`); merchant refunds 50 "shipping" as goodwill | Formula flags excess 50. **Not a formula defect** — `CB` cannot distinguish a shipping refund from goodwill. Resolution is a classification question: the refund should be recorded as goodwill, not shipping. Needs a policy decision (OQ-P4-2) |
| **CX-2** | Charged 1,000, goods never ship, merchant refunds all 1,000 | Accept, exact boundary `1,000 = 1,000`. Formula holds |
| **CX-3** | Refund against recognised revenue only (287) | Would block legitimate refunds. Formula holds; rev 1's alternative fails |
| **CX-4** | Refund before charge ingested | **Genuine gap in rev 1.** Fixed by deferral (§5.3). Without it, spurious excess on real webhook data |
| **CX-5** | 500 refund attributed to unit A (share 287) | Unit cap surfaces 213 excess. Formula holds; unit cap is necessary |
| **CX-6** | 33.33% discount on 1,000, computed per unit independently | `round(0.3333×287)=96`, `96`, `round(0.3333×426)=142` ⇒ **334 ≠ 333 target**. **Breaks conservation.** Reductions must use largest remainder over the target total, not per-unit rounding. Largest remainder gives 96/95/142 = 333 |
| **CX-7** | Order-level shipping discount 1,500 on `CB = 1,000` | Excess 500. Formula holds arithmetically but raises a policy question: a shipping discount larger than shipping charged means the merchant subsidised delivery. That is a **cost**, not negative revenue. Needs professional review (OQ-P4-3) |

CX-6 is the one that changes the specification: **no reduction may be computed by independent per-unit rounding.** The largest-remainder discipline that D-10a applies to allocation must apply identically to reductions, or the conservation test fails.

CX-4 and CX-6 are defects I found in my own rev 1. I am recording them rather than presenting the formula as having survived.

---

## 9. Verification status

The arithmetic in T1–T11, CX-2, CX-5, CX-6 and CX-7 was recomputed in a throwaway script from the stated inputs. Largest remainder for CX-6 returns 96/95/142, summing to 333; naive per-unit rounding returns 96/96/142, summing to 334. That confirms the CX-6 defect.

This is **arithmetic verification of this document only**. No specification test has been executed, no code exists, and none of these controls is implemented.

---

## 10. Unresolved questions

| ID | Question | Needs |
|---|---|---|
| **OQ-P4-1** | Is `shipping_refund_excess` a cost of sale or a liability? | **Professional accounting review** |
| **OQ-P4-2** | How is a goodwill payment distinguished from a shipping refund when the provider labels it shipping? | Founder policy + provider evidence |
| **OQ-P4-3** | Is a shipping discount exceeding charges a cost (subsidy) or negative revenue? | **Professional accounting review** |
| **OQ-P4-4** | What is the deferral window for `charge_pending` before classifying `charge_evidence_missing`? | Founder decision (operational) |
| **OQ-P4-5** | How is a reversal of a refund (provider reverses a credit) represented? Recommended: a positive `correction` reduction | Founder decision |

---

## 11. Recommendation

Adopt: **one control, three scopes, monotone `CB`, reality-over-ceiling disposition, order-independent totals, deferral on missing charge evidence, largest-remainder for every reduction amount.**

MVP scope: order and unit scopes, monotone `CB`, reality-over-ceiling, deferral, largest-remainder reductions. Charge scope only if a provider actually exposes charge references (OQ-P5-1) — otherwise it is dead code and should be deferred rather than built.

**This is a recommendation, not an approved decision.** D-10a's status is unchanged and C-10a.1 remains open.
