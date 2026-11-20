# PROP-C — Clause (a) Independence Assessment and Clause Mapping

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision clarification 2026-11-20 §7
**Status:** PROPOSED FOR FOUNDER DECISION — **no invariant number allocated**
**Affected decision:** D2-16 · D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (OPEN)
**Supersedes nothing:** the wording in `prop-c-wording-and-invariant-relationship.md` §1 stands unchanged

---

## 1. Clause (a) — exact wording, quoted

> **(a) Completeness.** Every shipping reduction evidenced by provider records for that order is accounted for in exactly one of three places: applied to a unit allocation, applied at order scope, or classified `confirmed_excess` with its cause recorded. No evidenced reduction may be absent from all three.

This is the only clause carrying a new obligation. Clauses (b) and (c) are reproduced in full in the source document and are not restated here.

---

## 2. The independent failure mode

An invariant earns its place only if it forbids a failure that nothing else forbids. Clause (a)'s failure mode:

> **A reduction that is observed, ingested, and then never reaches any financial total, while every other rule is satisfied.**

The concrete path — this is not hypothetical, it is what the current specification permits:

1. A shipping refund of 200 is observed for order #1042. `reduction_observed` is written (D2-08 satisfied — immutable, both timestamps).
2. No charge is yet observed for that scope. `CB_status = provisional_incomplete`, so the state is `unmatched_pending_charge` (D2-09 satisfied — no false excess).
3. The charge never arrives. Perhaps the provider never emits it; perhaps the webhook was lost beyond the retry envelope; perhaps reconciliation retrieves the order but not the payment object.
4. `reduction_window` expires. Expiry sets `investigation_due` and **nothing else** (D2-13 satisfied — expiry never reclassifies).
5. The refund sits in `unmatched_pending_charge` indefinitely. It is not in any unit allocation, not in any order-scope total, and not `confirmed_excess`. It is in none of the three places.

At step 5, every rule in the specification has been obeyed. The refund has vanished from every reported figure. Store contribution profit is overstated by 200, and no control has failed — because no control required the reduction to *land* anywhere.

**That is the gap clause (a) closes**, and no existing invariant closes it:

| Candidate protector | Why it does not cover the gap |
|---|---|
| **INV-19** (append-only) | Guarantees the record survives. Says nothing about whether the arithmetic accounts for it. A perfectly append-only ledger loses nothing and still omits the refund from every total |
| **INV-02** (refund ceilings) | Merchandise only. Silent on shipping reductions entirely |
| **INV-12** (idempotency) | Prevents the same reduction twice. Does not require it once |
| **INV-21** (aggregation paths) | Governs how figures roll up, not whether a reduction reached a figure at all |
| **D-10a** (`Σ A(u) = S_net`) | Concerns the original allocation at acceptance, before any reduction exists |
| **D2-09** (CB completeness) | Prevents a *false excess*. Does not prevent an indefinite *pending* |
| **D2-13** (window policy) | Explicitly refuses to reclassify on expiry — by design, so it cannot be the closing mechanism |
| **P-4/P-5 states** | The six states describe where a reduction *is*; none requires that a reduction in `unmatched_pending_charge` eventually leaves that state |

The gap is structural, not an oversight in any one rule: the specification is careful to prevent *wrong* classifications, and clause (a) is what prevents *no* classification.

---

## 3. Clause mapping — (b) and (c) to existing refund-control rules

| Clause | Restates | Exact source rule | What the restatement adds |
|---|---|---|---|
| **(b) No duplication** | D2-08 + INV-12 | Immutable event records with stable provider identity; `reduction_observed` carries the provider's stable source identity, and no single identity contributes twice regardless of delivery count | Makes duplicate-safety a *conservation property* rather than a per-provider implementation detail. Testable once, for all providers |
| **(c) No over-application** | D2-10 + D2-11 + D2-09 | Ceiling is `CB`; excess is classified `confirmed_excess`, never clamped; unit-scoped excess spills to order scope and unit balances never go negative | Ties three separate rules into one arithmetic statement: cumulative reductions ≤ unit allocation, and ≤ `CB` at order scope |

Neither clause introduces new semantics. Both convert behaviour already specified in P-4/P-5 into properties that can be asserted in a single test per clause.

---

## 4. Why bundling is still correct

A defensible objection: if two of three clauses are restatements, why not adopt only clause (a)?

Because (a) alone is satisfiable in a degenerate way — "accounted for in exactly one of three places" permits an implementation that puts everything into `confirmed_excess`, accounting for every penny while accusing the merchant of over-refunding every order. Clause (c) is what makes "applied" mean *applied within what was collected*. And (b) is what stops the three places from being filled by the same penny twice.

The bundle is one conservation law: **every evidenced reduction lands in exactly one place, once, and within its legitimate bound.** Removing any clause makes the remaining two satisfiable while money is still misstated.

---

## 5. Recommendation

**Recommend adoption as one new invariant, with clause provenance recorded as above.** My recommendation rests on the §2 gap: a demonstrated failure mode — a permanently pending reduction — that no existing invariant forbids.

I am not claiming all three clauses are novel. Clause (a) is; (b) and (c) are restatements included for testability and for closure of the degenerate cases.

**No invariant number is allocated.** Numbering awaits your approval of the mapping, per your standing instruction.

---

## 6. Decisions requested

| # | Decision | Recommendation |
|---|---|---|
| R3-D1 | Accept the §2 gap as a genuine, unaddressed failure mode | Approve |
| R3-D2 | Approve PROP-C as one new invariant with all three clauses | Approve |
| R3-D3 | Record the provenance: (a) new, (b)/(c) restatements of D2-08/INV-12 and D2-09/D2-10/D2-11 | Approve |
| R3-D4 | Authorise allocation of the next invariant number (currently INV-28 is unallocated) | Your call — numbering is yours |
| R3-D5 | Decide whether a reduction may remain `unmatched_pending_charge` indefinitely, or must eventually be forced to `confirmed_excess` / an explicit unresolved bucket (OQ-PC-1) | Your call — see §7 |

---

## 7. One question clause (a) does not answer

Clause (a) forbids a reduction being absent from **all three** places. It does not force a resolution *within* them, and it does not set a deadline.

So OQ-PC-1 remains open: when reconciliation is exhausted and a charge is never observed, does the reduction stay pending forever (honest, but permanently unresolved), or move to an explicit `unresolved` bucket that is visible in totals?

My view: it must become **visible in a total** eventually, or clause (a) is satisfied only on paper. But which mechanism — a fifth disposition, or an explicit unresolved bucket — is a design decision I am not taking without your direction. It is not a blocker for approving PROP-C, because clause (a) already forbids the silent version of this failure.

**Nothing here has been implemented or tested. No invariant is marked active.**
