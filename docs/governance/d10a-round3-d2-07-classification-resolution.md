# Round 3 — D2-07 Classification Resolution

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision clarification 2026-11-20 §2
**Status:** RESOLUTION PROPOSED FOR FOUNDER DECISION — **D2-07 status unchanged pending your ruling**
**Baseline unchanged:** D-10a = APPROVED WITH CONDITIONS · C-10a.1 = PARTIALLY SATISFIED — OPEN

---

## 1. The question asked

D2-07 appears in **both** §3.1 ("Ready for approval — no external evidence, no professional review") and §3.3 ("Requires professional accounting review") of the round-2 table. You asked me to determine whether:

- D2-07 contains multiple distinct decisions that should be separated; or
- one of the two classifications is incorrect.

---

## 2. Finding

**D2-07 contains two distinct decisions, and neither classification is a mistake.** Both lists are defensible because the entry bundles a purely structural requirement with a vocabulary-semantics requirement, and those two have entirely different dependency profiles.

This is a defect in **how I wrote the entry**, not in either classification. The round-2 table's own §1 states that "rows are deliberately not bundled: approving one does not approve its neighbour" — and D2-07 breaks that rule by containing two approvals in one row.

---

## 3. The exact current wording

> **D2-07 — P-3 typed reduction events and orthogonal dimensions**
>
> | | |
> |---|---|
> | **Exact rule** | Four reduction types: `cancellation_release`, `shipping_discount`, `shipping_refund`, `correction`. Three dimensions kept orthogonal: fulfilment (3) × disposition (5) × payment (6). Two separate identities: (a) `Σ effective + Σ unit reductions = CB`; (b) order-scoped reductions as separate lines. |
> | **Financial consequence** | E-5: three reductions on one unit all retained with dates and reasons (100 discount, 200 refund, 126 cancellation) totalling 426. A five-bucket partition records one reason and loses two. |
> | **Classification** | **MVP** (dimensions exist; not every disposition value populated) |
> | **Evidence required** | None external. |
> | **Professional review** | Disposition vocabulary (restocked / damaged / written off) interacts with inventory policy — see D-07, professional review open. |
> | **AppJet recommendation** | Adopt. Identity (a) must exclude order-scoped reductions — adding them double-counts, the defect found during rev-2 verification. |
> | **Decision requested** | Approve the four reduction types, the orthogonal dimensions, and the two-identity separation. |

The **Decision requested** row asks for one approval covering all three elements. The **Professional review** row then attaches a dependency to one of those elements only. That is the contradiction.

---

## 4. Proposed split

### D2-07a — Reduction event types, orthogonal dimensions and identity separation

| | |
|---|---|
| **Exact rule** | Four reduction types: `cancellation_release`, `shipping_discount`, `shipping_refund`, `correction`. Fulfilment, disposition and payment are **independent dimensions**, not a single state field. Two separate reconciliation identities: (a) `Σ effective + Σ unit reductions = CB`; (b) order-scoped reductions as separate lines, never added into (a). |
| **What it does not decide** | Which disposition *values* are authoritative, and what financial event (if any) a disposition value generates. |
| **Financial consequence** | E-5: three reductions on one unit all retained with dates and reasons (100 discount, 200 refund, 126 cancellation = 426). A five-bucket partition records one reason and loses two. Identity (a) excluding order-scoped reductions prevents the double-count found in rev-2 verification. |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | **No** |
| **AppJet recommendation** | Adopt. This is the structural requirement, and it holds for *any* disposition vocabulary. |

### D2-07b — Disposition vocabulary semantics and their inventory-cost consequence

| | |
|---|---|
| **Exact rule (proposed)** | Each disposition value is recorded per unit and retained. **Whether a disposition value generates a financial event, and with what COGS or inventory effect, is not decided here.** |
| **Dependency** | D-07 (COGS timing and inventory valuation), which is already flagged for professional review. |
| **Financial consequence** | A `written_off` disposition and a `restocked` disposition must be distinguishable in the record. Whether the write-off produces an *additional* cost event, or merely reclassifies cost already recognised at shipment, changes reported product contribution profit. Under the corrected Example E the inventory loss is recognised **once** (retained COGS, 4,500), not additionally — but that treatment is the founder-approved reading of a worked example, not a reviewed accounting policy. |
| **Classification** | **MVP** for the requirement to record and distinguish; **professional review** for the financial effect |
| **Evidence required** | None external. |
| **Professional review** | **Yes** — inventory valuation and COGS timing (D-07) |
| **AppJet recommendation** | Record and distinguish disposition values now; do not let any merchant-visible figure assert the COGS consequence of a write-off until D-07 is reviewed. |

---

## 5. Why the two are genuinely separable

This is the test of whether a split is real or merely cosmetic:

| Test | Result |
|---|---|
| Can D2-07a be satisfied without deciding D2-07b? | **Yes.** The dimensions are orthogonal and the identities hold for any vocabulary. Nothing in the structure depends on which values are authoritative. |
| Can D2-07b change without touching D2-07a? | **Yes.** The vocabulary can be revised, extended or reduced with no effect on the reduction types, the orthogonality or the identities. |
| Does one imply the other? | **No.** A correct structure can carry an unreviewed vocabulary, and a reviewed vocabulary can be implemented in a collapsed structure that still loses information. |
| Is there a shared artefact? | Only the *count* — "disposition (5)". See §6. |

---

## 6. Honest caveat on the number five

The round-2 wording says "disposition (5)". **That count is provisional.** It reflects the values needed to express E-4 (returned and restocked) and E-5 (three reductions on one unit) without collapsing information; it is not an approved vocabulary. The mechanism does not depend on the count being five, and D2-07a should be read as requiring *an independent disposition dimension*, not *this particular five-value set*.

I am flagging this because a founder approving "orthogonal dimensions" might reasonably assume the value list is settled. It is not.

---

## 7. Consequence for the round-2 summary

If you accept the split, §3.1 and §3.3 of the round-2 table both become correct as written, with a one-line clarification:

- **§3.1 gains D2-07a** — ready now, no dependency.
- **§3.3 retains D2-07b** — professional review required for the financial effect only.

The round-2 table is **not** amended by this document. It records the state at the time it was written, and I am not permitted to change statuses while preparing this package.

---

## 8. Decision requested

| # | Decision | Recommendation |
|---|---|---|
| R3-D1 | Accept that D2-07 contains two decisions rather than one mis-filed decision | Accept |
| R3-D2 | Approve **D2-07a** (reduction types, orthogonal dimensions, identity separation) as MVP with no professional-review dependency | Approve |
| R3-D3 | Approve **D2-07b** as "record and distinguish disposition values" only, with the COGS/inventory effect referred to D-07 professional review | Approve with that limitation |
| R3-D4 | Confirm whether the provisional five-value disposition vocabulary may be used in MVP pending D-07, or whether it must be marked provisional in merchant-facing output | Your call — I recommend marking it provisional |

**No status has been changed by this document.** D2-07 remains exactly as recorded in the round-2 table until you rule on R3-D1 to R3-D4.
