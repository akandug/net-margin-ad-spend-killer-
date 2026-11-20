# P-1 (Revised) — Shipping Association Selector

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Supersedes:** `p1-shipping-association-selector-decision-brief.md` (rev 1, 2026-11-20)
**Reason for revision:** founder review 2026-11-20 §2 — reassess the B-1/B-2/B-3 ladder for mutual exclusivity, sufficiency and deterministic selection
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not applied, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (PARTIALLY SATISFIED — OPEN)

---

## 1. What this revision changes, and why

Rev 1 proposed three association "bases" and defended them. Re-examining them against your question, **two of rev 1's central claims do not survive**:

| Rev 1 claim | Revised finding |
|---|---|
| B-1, B-2 and B-3 are three bases | They are **three labels for one mechanism**. There is exactly one decision per charge — direct or pooled. B-2 is not a third algorithm; it is "some charges direct, some pooled" |
| The three are collectively sufficient | **No.** Four association shapes exist that rev 1's ladder cannot express (§3) |
| The summary basis is a sufficient identity element | **No.** Two different per-charge vectors can share the label B-2, so the label cannot separate them in INV-27 (§7) |

Rev 1 also had no correction path at all. It said the basis is immutable, which is correct, but it never said what happens when the original association is *wrong* — the case you asked about in §2. §6 supplies it.

---

## 2. Are the three bases mutually exclusive and collectively sufficient?

### 2.1 Mutually exclusive — yes, but trivially so

If a basis is defined as *the fraction of charges carrying a valid association*, the three are exclusive by construction: 1, between 0 and 1, and 0. Exclusivity is not the problem.

The problem is that this is a partition of **order-level summaries**, not of the **mechanism**. The mechanism has one decision per charge. Defining a "basis" at order level and then describing B-2 as a distinct case invites the reader to think there are three algorithms to build, verify and test. There are not. There is one, applied per charge.

### 2.2 Collectively sufficient — no. Four gaps

| Gap | Shape rev 1 could not express | Consequence if unhandled |
|---|---|---|
| **G1** | A charge associated to a **line holding several units** | Direct-to-line still needs a within-line split. Rev 1 assumed direct = no arithmetic needed. It is needed |
| **G2** | A charge associated to **some units of a line**, not all | Rev 1 had no way to say "direct for units 1–2, pooled for unit 3" |
| **G3** | A charge associated to a **shipment**, not a line | A shipment may span lines. Rev 1's "associated ⇒ direct to a line" is simply false here |
| **G4** | An association record that is **present but invalid** | References a line not in the order, or the charge amount does not match the association. Rev 1's test was "association exists", which accepts garbage |

G4 matters most for financial integrity: a present-but-invalid association would be trusted, producing a confidently wrong direct allocation. Rev 1 would have accepted it.

### 2.3 The revision: one decision per charge, with a stated granularity

Replace the ladder with a **per-charge method**:

| Field | Values |
|---|---|
| `method(c)` | `D` (direct) or `P` (pooled) |
| `granularity(c)` | `G-line` · `G-unit` · `G-shipment` · `none` |
| `evidence_ref(c)` | the payload record relied upon |
| `snapshot_id` | the frozen payload snapshot the decision was made from |

**Selection rule R9′** — at the allocation instant, for each charge `c`:

1. If the frozen snapshot contains an association record that **passes the validity test** (§4), set `method = D` with its granularity.
2. Otherwise set `method = P`, `granularity = none`.
3. A charge may never be split between D and P. If part of a charge is associated and part is not, the charge is `P` and the partial association is recorded as evidence, not applied.

Within a `D` charge, the amount is distributed to the associated units by largest remainder over those units only. `G-unit` is exact; `G-line` and `G-shipment` require the within-scope largest remainder, which is where G1/G3 are handled.

**Order-level summary label** (for reporting and the identity, derived, never used to compute):

| Label | Condition |
|---|---|
| `A1` all-direct | every charge `D` |
| `A2` mixed | at least one `D` and at least one `P` |
| `A3` all-pooled | every charge `P` |

These labels replace B-1/B-2/B-3. They are *summaries*, and §7 explains why the label alone is not an identity.

---

## 3. Is selection deterministic using only information available at the allocation instant?

**Yes, with one condition that must be stated explicitly: the allocation instant must be bound to a frozen payload snapshot.**

R9′ is a pure function of (charge set, snapshot contents). No later event can change it. The condition is that "the allocation instant" cannot mean "whatever the database currently holds" — if the order payload is re-fetched or re-ingested later, the snapshot must be the one referenced at allocation, identified by `snapshot_id`, so that a re-run reproduces the same decision.

| Situation | Deterministic outcome |
|---|---|
| Association present in the acceptance payload | `D`, granularity from the record |
| Association absent at acceptance, arrives later | `P`, frozen. Late record becomes an annotation (§5) |
| Association present but fails validity | `P`, with `evidence_invalid` recorded |
| Association present for only part of a charge | `P` (rule 3), partial association recorded |
| Two conflicting association records in one snapshot | `P` + `association_conflict` flag. Not resolved by preferring one |

The last row is a deliberate choice: when the provider contradicts itself, guessing produces a confidently wrong direct allocation, which is worse than pooling and flagging. It fails toward the *less attributed* state, not the more confident one.

---

## 4. The validity test (new in this revision)

An association record is valid only if all hold:

1. It references a line, unit or shipment that **exists in this order's accepted set**.
2. The referenced scope's unit count is greater than zero.
3. The sum of association amounts for the charge does not exceed the charge amount.
4. It carries a **stable provider identity** — not a positional index into a payload array.
5. It is present in the frozen snapshot used at allocation.

Failure of any test ⇒ `method = P` plus a recorded reason. No partial credit, no inference.

Test 4 is the one that matters for reproducibility: an association identified only as "the second entry in `shipping_lines`" is not evidence, because array order is not contractual.

---

## 5. Late-arriving evidence — annotation, never reallocation

| Property | Rule |
|---|---|
| Effect on allocation | **None.** The per-charge method is frozen |
| Representation | `association_evidence_late` annotation naming the record, its arrival time and the charge |
| Data-quality flag | `A3_contested` if late evidence contradicts a frozen `P` decision |
| Merchant-visible effect | The flag travels with the product-level figures; it does not change them |
| What it may trigger | A **correction request** (§6) — a proposal, not an automatic act |

Rev 1 asserted immutability without saying what the evidence does instead. It annotates and it can propose. It cannot act.

---

## 6. Correcting an incorrect original association (new in this revision)

This is the case rev 1 omitted entirely. The requirement is: investigate and correct **without destroying the original record**.

| Step | What happens |
|---|---|
| 1. Investigation | An authorised user or the founder reviews the evidence. Investigation is read-only and produces no financial change |
| 2. Correction authorisation | A `correction_authorised` event records: actor, timestamp, reason, evidence reference, and the `allocation_version` being superseded. This is a deliberate act, never automatic |
| 3. New allocation version | The engine computes `allocation_version n+1` with a **new INV-27 calculation identity** (new `calculation_version`). It is a new record |
| 4. Coexistence | Version `n` is retained unchanged and remains independently reproducible. Nothing is updated or deleted |
| 5. Reporting | The current version is shown; the superseded version remains retrievable and the change is disclosed |

**The financial consequence of a correction is narrower than it appears, and this should be stated to merchants:** the store-level shipping total is invariant under any reallocation — only the *product-level distribution* moves. A correction cannot change store contribution profit. It changes which product looked healthy. That makes corrections financially low-risk and reputationally high-risk, which is exactly why step 2 is an authorised, audited act rather than a background job.

A correction that changes the *total* would not be an association correction at all — it would be a charge or reduction correction, handled by P-4's control.

---

## 7. Inclusion in the INV-27 calculation identity

Rev 1 said "add `shipping_allocation_basis`". **That is insufficient**, and the reason is worth recording.

Two different orders can both summarise to `A2` while having entirely different method vectors:

| Order | Charge 1 | Charge 2 | Summary label |
|---|---|---|---|
| X | `D` | `P` | `A2` |
| Y | `P` | `D` | `A2` |

Their product-level shipping distributions are not comparable, yet the label is identical. Using the label as an identity element would let the engine aggregate or compare them as if equivalent — precisely the silent-comparison failure INV-27 exists to prevent.

**Revised identity element:** `association_method_vector` — the canonical serialisation of the ordered set `[(charge_identity, method, granularity), …]`, hashed. Two calculations may share an identity only if their vectors are identical.

Rules:
1. A calculation whose vector is absent **fails closed**. No default vector.
2. Aggregation across differing vectors is **prohibited**.
3. A correction produces a new vector, therefore a new identity, therefore an explicitly separate calculation.
4. The summary label may be displayed; it may never substitute for the vector.

---

## 8. Comparison under identical inputs

The same order, the same charges, three methods. Amounts in minor units.

**Order P1-R2** — two units, deliberately unequal so the effect is visible:

| Unit | Line | Merchandise net revenue |
|---|---|---|
| A | L1 | 100 |
| B | L2 | 9,900 |

**Charges:** `c1 = 1,000` associated to L1 · `c2 = 200` associated to L2 · `S_net = 1,200`

### 8.1 Allocation

| Method | A | B | Total | Arithmetic |
|---|---|---|---|---|
| All-direct (`A1`) | **1,000** | **200** | 1,200 | each charge to its line |
| All-pooled (`A3`) | **12** | **1,188** | 1,200 | 1,200 × 100/10,000 = 12; 1,200 × 9,900/10,000 = 1,188 |
| Mixed (`A2`: c1 direct, c2 pooled) | **1,002** | **198** | 1,200 | 1,000 + (200 × 100/10,000 = 2); 0 + 198 |

All three reconcile exactly to 1,200. No rounding was required — the weights divide evenly.

### 8.2 The consequence that matters

| Method | A contribution before COGS and ad spend | B contribution | Store total |
|---|---|---|---|
| All-direct | 100 + 1,000 = **1,100** | 9,900 + 200 = **10,100** | **11,200** |
| All-pooled | 100 + 12 = **112** | 9,900 + 1,188 = **11,088** | **11,200** |
| Mixed | 100 + 1,002 = **1,102** | 9,900 + 198 = **10,098** | **11,200** |

**Two findings:**

1. **The store total is identical in every case.** The association method is not a P&L decision. It cannot change store contribution profit by one unit. It is purely a *product-level attribution* decision.
2. **Product A's reported contribution differs by 988** — from 112 to 1,100, a factor of 9.8 — on a single 1,000 charge.

Rev 1 illustrated this with a ±13/−26 difference on a 1,400 order and thereby **understated the stakes by two orders of magnitude**. A merchant deciding whether to keep advertising product A would reach opposite conclusions under the two methods.

Which number is right? Neither, in the abstract. If the merchant genuinely charged 1,000 shipping on a 100-revenue item (bulky freight, for example), all-direct is the truth. If the 1,000 was an order-level delivery charge unrelated to that item, all-pooled is the truth. **The method encodes a real commercial fact, which is why it must be evidence-derived and frozen rather than chosen for convenience.**

---

## 9. MVP requirement versus pilot workflow

| Behaviour | Classification | Reasoning |
|---|---|---|
| Per-charge method recorded and frozen | **MVP mandatory** | Without it, results are unreproducible |
| Method vector in INV-27 identity | **MVP mandatory** | Without it, incomparable figures share an identity |
| Pooled default when no valid association | **MVP mandatory** | The only safe default |
| Validity test (§4) | **MVP mandatory** | Prevents trusting garbage; cheap to implement |
| Fail closed when vector absent | **MVP mandatory** | No silent default |
| `G-line` granularity | **MVP mandatory** | The common shape |
| Within-scope largest remainder | **MVP mandatory** | G1/G3 are not edge cases |
| Late-evidence annotation + `A3_contested` flag | **Pilot** | Affects presentation, not the calculation |
| Correction / new allocation version (§6) | **Pilot** | Needed only once real merchants dispute a figure |
| `G-unit` / `G-shipment` granularity | **Pilot** | Only if a provider actually exposes them (OQ-P1-1) |
| Merchant-selectable basis | **Later — and only with disclosure** | A merchant choosing the basis can choose the number |
| Automatic reallocation on late evidence | **Not recommended at any stage** | Destroys reproducibility; rev 1's immutability argument stands |

**The load-bearing unknown:** OQ-P1-1/P1-2 — whether any target provider exposes a genuine line-level association. If none does, `A1` and `A2` are unreachable in the MVP and every order is `A3`. The *mechanism* above should still be built, because it is what makes `A3` a recorded decision rather than an accident, but the merchant-facing claim must be honest: product-level shipping attribution is pooled, and the product does not currently offer a better basis.

---

## 10. Unresolved questions

| ID | Question | Needs |
|---|---|---|
| OQ-P1-1 | Does Shopify or any MVP provider expose line-level shipping-charge association? | P-01/P-02 provider documentation |
| OQ-P1-2 | Is that association stable across payload revisions, or positional? | Provider docs + fixture |
| OQ-P1-3 | For `G-shipment`, how are shipments identified before shipping occurs? | Provider docs |
| OQ-P1-4 | Should a correction be merchant-initiated, founder-approved, or both? | Founder decision (governance, not accounting) |
| OQ-P1-5 | Is a charge-level association conflict an error state for the merchant or a silent fallback to pooled? | Founder decision (product behaviour) |

None of these is an accounting question. They are provider and governance questions.

---

## 11. Recommendation

**Adopt R9′ — one method per charge with stated granularity, the validity test, the vector (not the label) in the identity, and correction by new version rather than edit.**

This is smaller than rev 1, not larger: one decision per charge replaces three bases, and the validity test removes a whole class of confidently-wrong allocations that rev 1 would have trusted.

MVP scope: §9's mandatory rows only. The correction workflow is a pilot addition and should not delay G2.

**This is a recommendation, not an approved decision.** D-10a's status is unchanged and C-10a.1 remains open.
