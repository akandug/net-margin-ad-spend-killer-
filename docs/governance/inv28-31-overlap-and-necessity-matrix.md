# INV-28 – INV-31: Overlap and Necessity Matrix

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Status:** PROPOSED FOR FOUNDER REVIEW — **no invariant numbers allocated**
**Founder instruction honoured:** *"Do not create new invariant numbers until the mapping is reviewed and approved."* The four proposals are therefore labelled **PROP-A … PROP-D**. No ID in the INV-28…INV-31 range is claimed.

---

## 1. Purpose and method

Four new invariants were proposed during the P-1/P-3/P-4 work. Before asking you to approve four additions to a register of 27, each was tested against the question: **does an existing invariant already prevent the same failure?**

Method per proposal:
1. Identify the exact failure mode it prevents.
2. Search the approved register for an invariant covering that failure.
3. Determine whether the coverage is genuine or merely adjacent.
4. Decide: fold into an existing invariant, express as a test under one, or keep as new.

## 2. Scope limitation — read this first

This matrix is built from the invariants whose content I **actually inspected** in this review:

| Verified as covering the cited concern | INV-01 (no duplicate discount deduction) · INV-02 (refund ceilings, line level) · INV-03 (tax revenue purity and liability roll-forward) · INV-12 (idempotency identity) · INV-13 (atomic component ingestion) · INV-14 (no silent estimated→actual conversion) · INV-16 (period assignment) · INV-19 (append-only protection) · INV-20 (inventory cost lifecycle) · INV-21 (aggregation; no cross-currency summing) · INV-22 (fee-reversal attribution) · INV-24 (tax balances) · INV-26 (quantity state) · INV-27 (view reconciliation identity) |
|---|---|
| **Not inspected — not cited, not relied upon** | INV-04 – INV-11, INV-15, INV-17, INV-18, INV-23, INV-25 |

**Consequence:** this is a complete mapping against the invariants I examined, and an **incomplete** mapping against the full 27. A definitive overlap determination requires reading the other eleven. I am not asserting that PROP-A…D are the only overlaps with the register — only that they do not overlap with the fourteen I checked, beyond what is stated below.

---

## 3. The matrix

### PROP-A — Shipping association basis is recorded, frozen and part of calculation identity

| Question | Finding |
|---|---|
| Failure mode prevented | Two calculations with identical INV-27 identity but **different per-charge association methods** are aggregated or compared as if equivalent, silently mixing incomparable product-level figures |
| Existing coverage | **INV-27** (calculation identity and view reconciliation) covers *identity completeness*; **INV-21** covers *aggregation integrity* but only currency, period, tenant and calculation version; **INV-19** covers *immutability* of recorded events |
| Is coverage genuine? | **Partly.** INV-19 genuinely prevents mutation, so the *frozen* half is already covered. INV-27 genuinely prevents identity collision — but only for the elements it already lists, and the method vector is not among them. The gap is one missing identity element, not a missing principle |
| Independent failure mode? | **No** — it is a missing element of INV-27, not a new principle |
| Genuinely new requirement | The specific rule that the **vector, not the summary label**, is the identity element, and that a missing vector fails closed |
| Expressible as a test under an existing invariant? | **Yes** — a TC under INV-27: two calculations differing only in method vector must produce different identities |
| MVP or pilot? | **MVP** |
| **Recommendation** | **Fold into INV-27** as a new identity element plus one test. **No new invariant number** |

### PROP-B — Post-acceptance changes are separate events, never in-place edits

| Question | Finding |
|---|---|
| Failure mode prevented | A post-acceptance discount or refund is written over the original allocation, destroying the evidence the original allocation relied on |
| Existing coverage | **INV-19** (append-only protection) directly and completely |
| Is coverage genuine? | **Yes.** This is precisely INV-19's subject |
| Independent failure mode? | **No** |
| Expressible as a test under an existing invariant? | **Yes** — a TC under INV-19 asserting no UPDATE path exists for an allocation, and that a post-acceptance change produces an additional record |
| MVP or pilot? | **MVP** |
| **Recommendation** | **No new invariant.** One TC under INV-19. **Nothing added to the register** |

### PROP-C — Cumulative shipping reductions may not exceed the collectible balance

| Question | Finding |
|---|---|
| Failure mode prevented | Shipping reductions (discounts, refunds, cancellations) cumulatively exceed what the merchant ever collected, and the excess is either absorbed as negative shipping revenue or silently dropped |
| Existing coverage | **INV-02** covers *refund ceilings*, but explicitly at **merchandise line level**, with shipping and tax refunds as **separate components**. **INV-01** covers duplicate deductions. **INV-03/INV-24** cover tax |
| Is coverage genuine? | **No.** INV-02's scope is merchandise refunds at line level and is in any case **blocked on P-01** (the Shopify refund basis is unresolved). Shipping reductions have no ceiling in the current register. INV-01 prevents the *same* deduction twice but not *different* reductions exceeding a total |
| Independent failure mode? | **Yes** — genuinely uncovered |
| Expressible as a test under an existing invariant? | **No.** INV-02 would have to be widened from merchandise to shipping, which changes its scope and its pending P-01 dependency |
| MVP or pilot? | **MVP** |
| **Recommendation** | **Keep as the one new invariant.** Includes: monotone `CB`, three scopes (order/charge/unit), reality-over-ceiling disposition, deferral on missing charge evidence |

### PROP-D — Allocations and reductions reconcile to the collectible balance

| Question | Finding |
|---|---|
| Failure mode prevented | Unit allocations plus reductions fail to reconcile to `CB`; a residual is absorbed somewhere, or a rounding difference accumulates |
| Existing coverage | **INV-01** (no duplicate deduction) covers duplication; **INV-21** covers aggregation paths |
| Is coverage genuine? | **Partly.** Neither is a conservation statement. But conservation is the *test form* of PROP-C, not an independent rule — a ceiling that is never breached and a total that reconciles are two views of the same control |
| Independent failure mode? | **No** — it is PROP-C's conservation test |
| Expressible as a test under PROP-C? | **Yes** |
| MVP or pilot? | **MVP** |
| **Recommendation** | **Fold into PROP-C as its conservation test.** No separate invariant |

---

## 4. Summary recommendation

| Proposal | Disposition | New invariant number? |
|---|---|---|
| PROP-A | Fold into INV-27 as identity element + test | **No** |
| PROP-B | Covered by INV-19; add one test | **No** |
| PROP-C | **Genuinely new** | **Yes — one** |
| PROP-D | Fold into PROP-C as conservation test | **No** |

**Minimum necessary addition: ONE new invariant**, not four.

This is a reduction of 75% from what the working drafts proposed. Two of the four were duplicates of principles the register already holds, and one was a test rather than a rule. Adding them as numbered invariants would have inflated the register without adding protection — and a register whose entries are not independently load-bearing is harder to enforce and easier to quietly ignore.

## 5. What the one new invariant would say (proposed text, not adopted)

> **Cumulative shipping reductions may not exceed the collectible balance.**
> `CB = S_net + Chg_add`, non-decreasing. Every reduction carries a stable identity and is counted once. Reductions are checked at order scope always, and additionally at charge or unit scope where the reduction carries a valid reference. A reduction that exceeds a scope's remaining balance is **recorded in full as a fact** and its excess classified as `shipping_refund_excess`, never rejected, never absorbed. Where the balance is unknown because a charge has not yet been ingested, evaluation is deferred rather than concluded. Conservation: unit allocations plus all reductions equal `CB`. Reduction amounts use largest remainder over the target total, never independent per-unit rounding.
>
> **Fails closed:** a reduction whose scope cannot be determined, or an order whose `CB` cannot be computed, produces no product-level figure for that order.
>
> **Tests:** T1–T11 of the combined P-4/P-5 brief, plus CX-2, CX-5, CX-6 and CX-7.

## 6. Unresolved

| ID | Question | Needs |
|---|---|---|
| OQ-INV-1 | Approve one new invariant (PROP-C), or require a different packaging? | **Founder decision** |
| OQ-INV-2 | Should INV-02 be widened to shipping, or kept merchandise-only with shipping handled by the new invariant? Recommended: keep separate — INV-02 is blocked on P-01 and widening it would spread that blockage | Founder decision |
| OQ-INV-3 | Are INV-04–11, INV-15, INV-17, INV-18, INV-23 and INV-25 free of overlap with PROP-C? | Requires those eleven to be read — not done in this review |
| OQ-INV-4 | If a number is allocated, should it be INV-28 (the next free) or reserved until P-01 closes? | Founder decision |

## 7. Verification status

The T1–T11 arithmetic and the CX-6 rounding failure were recomputed numerically; the CX-6 defect (334 vs 333) was confirmed. **No invariant test has been executed.** No invariant is implemented, and this document does not claim any is.
