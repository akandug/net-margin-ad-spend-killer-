# PROP-C — Exact Wording and Invariant Relationship

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision round 2026-11-20 §9
**Status:** PROPOSED FOR FOUNDER DECISION — **no invariant number allocated, not approved, not implemented, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (OPEN)

---

## 1. Exact proposed wording

> **PROP-C — Shipping reduction conservation.**
>
> For every order and every financial calculation identity, the following three clauses must hold.
>
> **(a) Completeness.** Every shipping reduction evidenced by provider records for that order is accounted for in exactly one of three places: applied to a unit allocation, applied at order scope, or classified `confirmed_excess` with its cause recorded. No evidenced reduction may be absent from all three.
>
> **(b) No duplication.** No single evidenced reduction contributes to more than one of those three places, and no stable provider source identity contributes more than once, regardless of how many times the provider delivers it.
>
> **(c) No over-application.** Cumulative reductions applied to any unit never exceed that unit's allocation, and cumulative reductions applied at order scope never exceed the order's collectible balance `CB`. Any excess is classified under (a), never discarded and never clamped to fit.

---

## 2. What each clause prevents

| Clause | Failure it prevents | Concretely |
|---|---|---|
| (a) Completeness | A reduction vanishing between the provider record and the calculation | A refund observed, ingested, and then dropped because it matched no scope and no rule claimed it |
| (b) No duplication | The same money counted twice | A duplicate webhook making 400 of refunds look like 800, producing a false `confirmed_excess` |
| (c) No over-application | A unit or order showing more refunded than was ever collected | A unit with a 426 allocation showing 500 of reductions and a negative shipping revenue |

**Clause (a) is the genuinely new one.** Clauses (b) and (c) restate requirements that P-4/P-5 already specify behaviourally; they are included so that the whole conservation law is testable in one place rather than scattered across a control specification.

---

## 3. Relationship to INV-27

INV-27 and PROP-C are **orthogonal**, and the distinction matters:

| | INV-27 | PROP-C |
|---|---|---|
| **Question answered** | *May these two figures be compared?* | *Does this figure account for every penny?* |
| **Governs** | calculation identity: tenant, store, currency, period, `calculation_view`, `calculation_version`, source-data boundary | conservation of reductions within one identity |
| **Failure mode** | two incomparable figures presented as comparable | a complete-looking figure that has silently lost or duplicated money |
| **Direction** | across identities | within an identity |

A figure can satisfy INV-27 perfectly — correct identity, correctly labelled, never compared across views — and still violate PROP-C, because it lost a refund. Conversely, a figure can satisfy PROP-C and still be unusable for comparison because its identity differs from the figure beside it.

**They do not imply each other, and neither subsumes the other.** That is the test of genuine independence.

One point of contact: PROP-C is scoped "for every calculation identity", so it inherits INV-27's identity boundary rather than defining its own. This is deliberate — it prevents PROP-C from being satisfiable by summing across identities, which would be a way to appear conserved while comparing incomparable things.

---

## 4. Relationship to the existing invariant register

| Existing invariant / rule | Scope | Why PROP-C is not covered |
|---|---|---|
| **INV-02** | Refund ceilings — **merchandise only**, at line level, discount-aware | Silent on shipping. Nothing in INV-02 constrains a shipping reduction. This is the primary independence argument |
| **INV-19** | Append-only protection of records | Protects the *records*; says nothing about whether the *arithmetic* accounts for them. A perfectly append-only ledger can still lose a refund in aggregation |
| **INV-21** | Valid aggregation paths and their integrity | Governs how figures roll up. PROP-C governs conservation within a single node before aggregation |
| **INV-26** | Quantity state partition | About *quantities and their dimensions*, not money |
| **INV-27** | Calculation identity | See §3 |
| **D-10a allocation identity** (`Σ A(u) = S_net`) | The **original** allocation at acceptance | Concerned with `A(u)`, not with post-acceptance reductions. A reduction is not an allocation |
| **P-4/P-5 control** | The **mechanism**: event records, derived status, `CB` completeness, ceiling rule | Specifies behaviour. PROP-C is the testable property that behaviour must satisfy. Clauses (b) and (c) are the invariant form of that control; clause (a) is not present in it |

### 4.1 The honest assessment

The founder asked whether PROP-C is "genuinely independent and does not duplicate an existing conservation rule". The accurate answer:

- **Clause (a) is genuinely new.** No existing invariant requires that every evidenced reduction be accounted for somewhere. The P-4/P-5 control can, in principle, satisfy its own transitions while a reduction classified `unmatched_pending_charge` sits indefinitely and never reaches a financial total. Clause (a) forbids that gap.
- **Clauses (b) and (c) are invariant restatements** of requirements P-4/P-5 already specifies. They add testability, not new semantics.

**Recommendation: retain all three clauses, but record that (b) and (c) are restatements.** Bundling a genuinely new requirement with two restatements is acceptable when the bundle is a single coherent conservation law — and it is, because (a) without (c) would permit "accounted for" to mean "applied beyond what was collected". But the founder should know which clause carries the new obligation, so the decision is not taken on an inflated impression of novelty.

---

## 5. Testability

| Clause | Test shape | Example assertion |
|---|---|---|
| (a) | Inject a reduction, run the calculation, assert it appears in exactly one of the three places | A 200 refund with no observed charge appears as `unmatched_pending_charge`, not nowhere |
| (b) | Deliver the same source identity twice, assert single contribution | `R-77` delivered twice: total 400, not 800 (CX-C) |
| (c) | Apply reductions exceeding a unit allocation, assert spill and no negative balance | 400 + 100 against a 426 allocation → unit 426, order 74, no negative (CX-G) |
| (a)∧(c) | Excess case, assert it is classified and visible | CB final, matched 600 > CB 500 → `confirmed_excess` 100 surfaced (CX-B) |

These are **test designs, not executed tests.** No calculation engine exists.

---

## 6. What PROP-C does not cover

| Out of scope | Where it belongs |
|---|---|
| Merchandise refund ceilings | INV-02 — **open pending F-01 evidence** |
| Whether goodwill is revenue-reducing or an expense | Professional accounting review (OQ-P4-5) |
| Tax treatment of refunded shipping | Professional accounting review |
| Whether the reduction belongs to the same period as the charge | INV-16 period policy |
| Allocation of the original charge | D-10a |
| Advertising spend conservation | Separate unallocated-spend requirement |

---

## 7. Recommendation

Adopt PROP-C as a single new invariant with the three clauses in §1, scoped to a single calculation identity, and with the provenance of each clause recorded per §4.1.

**Do not allocate an invariant number yet** — per your instruction, numbering awaits the mapping review.

**This is a recommendation, not an approved invariant.** Nothing here has been implemented or tested.
