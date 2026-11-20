# D-10a Editorial Change Set — Proposed Diff

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Decision package:** D-10a substantive correction review, Part D
**Status:** **PROPOSED — NOT APPLIED.** No file listed here has been modified by this change set.
**Authority:** founder instruction 2026-11-20 (third review), Part D

---

## 1. Boundary this change set must respect

The founder's instruction:

> P-2 and P-6 through P-10 may be prepared as a separate editorial-only change set. **Do not make any change that alters financial meaning, allocation behaviour, event timing, calculation identity or approval scope under the label of editorial correction.** Show the proposed diff for review before applying it.

Applying that test to the audit's own list produces a different set from the audit's: **three of the eight items the audit called editorial are not editorial.** They are excluded here and routed to founder approval. That is the honest outcome of the test, and it is recorded first so the reader can see what was removed and why.

---

## 2. Scope triage

| Item | Audit label | Audit description | Classification here | Routed to |
|---|---|---|---|---|
| **P-2** | Editorial | Relabel E2 as an alternative branch from E1 at acceptance | **Editorial — safe** | This change set |
| **P-5** | Editorial | "Restate R8 per charge where R5 applies, with the order-level identity as the aggregate check" | **SUBSTANTIVE** | P-4 §12, R10.3 |
| **P-6** | Editorial | Qualify the E1 base table's "gross" per unit | **Editorial — safe** | This change set |
| **P-7** | Editorial | Add two examples: `Σbase = 0` order; decisive tie-break | **SUBSTANTIVE** | §5 below |
| **P-8** | Editorial | Reword E6 to "the 287 **shipping-revenue component**" | **Editorial — safe** | This change set |
| **P-9** | Editorial | Align INV-26 S2 and D-10a E2 to the same cancelled unit, **or** state explicitly they are separate scenarios | **Editorial — safe** (second option only) | This change set |
| **P-10** | Editorial | "per D-10a E1 mechanics" → "using the D-10a largest-remainder method" | **Editorial — safe** | This change set |

**Result: five items safe to apply as editorial (P-2, P-6, P-8, P-9, P-10); three require founder approval (P-5, P-7, and P-4 which was already substantive).**

---

## 3. Why P-5 is not editorial

Restating R8 "per charge where R5 applies" changes **which refund events are accepted**:

- Under an order-only cap: a 700 refund against charge S1 (600) passes, because 700 ≤ 1,000.
- Under a per-charge cap: the same refund is quarantined, because 700 > 600.

The event's financial treatment changes. That is "allocation behaviour" and "financial meaning" in the founder's own words. It cannot be applied under an editorial label, and it is therefore specified as R10.3 in the P-4 brief.

## 4. Why P-7 is not editorial

The audit proposes two new examples:

- A `Σbase = 0` order (three units, `S_net = 1,000` → 334/333/333).
- A case where the tie-break is decisive (two units with equal fractional parts and a remainder of 1).

The first does not merely illustrate existing behaviour — it **defines behaviour that currently has no rule**. When every unit's net revenue is zero, pro-rata weighting by net revenue is `0/0`, which is undefined. The expected output 334/333/333 implies an **equal-weight fallback**, and that fallback is a new allocation rule. Adding it as an "example" would introduce a rule through the back door.

Independently reconstructed, to show the rule the audit's numbers imply:

```
Σbase = 0, three units, S_net = 1,000, equal weights
  1,000 × 1/3 = 333.3333 → floor 333, each
  floors = 999, remainder 1
  fractions all equal (.3333) → tie across all three
  tie-break by stable unit identity → A takes the unit
  → 334 / 333 / 333, Σ 1,000 ✓
```

```
Decisive tie-break, weights 1/1/2, Σbase 4, S_net 1,002
  A: 1,002 × 1/4 = 250.5  → floor 250
  B: 1,002 × 1/4 = 250.5  → floor 250
  C: 1,002 × 2/4 = 501.0  → floor 501
  floors = 1,001, remainder 1
  fractions: A .5, B .5, C 0 → tie between A and B for one slot
  tie-break by stable unit identity → A
  → 251 / 250 / 501, Σ 1,002 ✓
```

Both are arithmetically sound, but each requires an explicit founder-approved rule:

- **P-7a** — the `Σbase = 0` fallback (equal weighting), and whether it is legitimate to charge shipping on an order whose net merchandise revenue is zero.
- **P-7b** — the tie-break key (stable unit identity, not insertion order, not line order, not quantity).

**P-7 is therefore withheld from this change set** pending those two decisions. It should be specified alongside P-1, because the tie-break key is a reproducibility property of the same family as the association basis.

---

## 5. Proposed diff — P-2 (D-02)

**File:** `docs/governance/d10a-shipping-revenue-allocation-decision-brief.md` · §3, E2 heading and opening

**Defect.** E2 is presented as continuing E1, but E1 ships unit B on 2026-02-18 while E2 cancels B on 2026-02-10. Both cannot be true of one order. The defect is a contradiction between two examples, and the audit offers two repairs: relabel E2 as an alternative branch, or move the cancellation date earlier. The relabel is chosen because it preserves E1's two-shipment arithmetic, which E6 and E7 both depend on.

```diff
-### E2 — Partial cancellation (continues E1)
+### E2 — Partial cancellation (alternative branch from E1 at acceptance)
+
+> **Branch note.** This example branches from E1 **at the point of order acceptance**, not after
+> it. E1's two shipments (2026-02-03 and 2026-02-18) do **not** occur in this branch. The
+> allocation computed at acceptance is identical to E1's — 287 / 287 / 426 — because it is fixed
+> before either branch diverges. Only the subsequent event history differs.
```

**Financial meaning:** unchanged. The allocation 287/287/426 and the total 1,000 are untouched; only the relationship between two examples is clarified.

**Verification after applying:** E6 and E7 still reference E1's shipment 1 (2026-02-03, units A and C, 713) and remain correct. E2's own arithmetic is unchanged.

---

## 6. Proposed diff — P-6 (D-06)

**File:** `docs/governance/d10a-shipping-revenue-allocation-decision-brief.md` · §3, E1 base table

**Defect.** The column headed "Gross" is **per unit**, but reads as a line total. The A row shows `600 (2 units)` in the discount column while the B row shows `—`, which makes the line-level discount appear to belong to A alone. A reader reconstructing the order can reach 12,100 or 13,000 instead of the correct 9,400.

Verified reconstruction of the current table's intent:

| Unit | Line | Gross per unit | Qty | Line gross | Line discount | Net per unit | Line net |
|---|---|---|---|---|---|---|---|
| A | L1 | 3,000 | 1 of 2 | 6,000 (line) | 600 (line total, 300/unit) | 2,700 | 5,400 (line) |
| B | L1 | 3,000 | 1 of 2 | ↑ same line | ↑ same 600 | 2,700 | ↑ same line |
| C | L2 | 4,000 | 1 | 4,000 | 0 | 4,000 | 4,000 |
| | | | | | | **Σbase** | **9,400** |

Check: line L1 net = 6,000 − 600 = 5,400 = 2,700 × 2 ✓. Σbase = 5,400 + 4,000 = **9,400** ✓. The existing `Σbase = 9,400` is correct; only the presentation is ambiguous.

```diff
-| Unit | Line | Gross | Line discount | Discount per unit | Net per unit |
+| Unit | Line | Gross per unit | Line discount (line total) | Discount per unit | Net per unit |
```

```diff
+| |
+| **Note.** "Gross per unit" is the unit's own gross price. A and B are the two units of line L1,
+| so L1's line gross is 6,000 and its line discount is 600 — shown once, on the A row, not twice.
+| Σbase is the sum of net revenue across the three units: 2,700 + 2,700 + 4,000 = 9,400.
```

**Financial meaning:** unchanged. No amount, allocation or rule is altered; the units and scope of two column headings become explicit.

---

## 7. Proposed diff — P-8 (D-09)

**File:** `docs/governance/d10a-shipping-revenue-allocation-decision-brief.md` · §3, E6 closing paragraph

**Defect.** E6 says "The 287 difference between the views is exactly the pending allocation". The 287 is the *shipping* component of the difference, not the whole difference — COGS timing differences also contribute to the total view difference. As written it invites a reader to conclude the views differ only by shipping.

```diff
-The 287 difference between the views is exactly the pending allocation, and it is the
-quantity-and-timing term that INV-27's reconciliation must explain.
+The 287 **shipping-revenue component** of the difference between the views is exactly the
+pending allocation. It is one quantity-and-timing term that INV-27's reconciliation must
+explain; COGS timing differences are a separate term and are reconciled separately.
```

**Financial meaning:** unchanged for shipping. It narrows a claim so it no longer asserts something false about COGS.

---

## 8. Proposed diff — P-9 (D-10)

**File:** `docs/governance/inv26-quantity-state-specification.md` · §S2 opening

**Defect.** D-10a E2 cancels unit **B**; INV-26 S2 cancels unit **C**. Read as a pair, the two documents describe the same scenario with different units, which is the kind of silent divergence that makes cross-document audit unreliable.

The audit offers two repairs: align to the same unit, or state that they are separate scenarios. **Only the second is editorial** — changing which unit is cancelled would change S2's arithmetic and its relation to INV-26 S1. This change set therefore applies the second option and records the first as a founder choice.

```diff
-### S2 — Partial cancellation
-
-Continuing S1, unit C is cancelled and refunded on 2026-02-10 before shipping.
+### S2 — Partial cancellation
+
+> **Scenario note.** S2 continues **INV-26 S1** and is independent of the D-10a worked examples.
+> D-10a E2 cancels unit B of its own order; S2 cancels unit C of S1's order. The different unit is
+> not a contradiction between the documents — they are separate scenarios — and neither example is
+> derived from the other. Where a figure is quoted across both, it is quoted from this
+> specification, not inferred from D-10a.
+
+Continuing S1, unit C is cancelled and refunded on 2026-02-10 before shipping.
```

**Financial meaning:** unchanged. No amount, state or transition is altered.

**Open founder choice.** If the intent was for S2 to *mirror* D-10a E2, aligning the units is a substantive change requiring approval — it would alter S2's worked arithmetic and the INV-26 state table row for the cancelled unit. This change set does not assume that intent.

---

## 9. Proposed diff — P-10 (D-11)

**File:** `docs/governance/inv27-calculation-identity-specification.md` · §4

**Defect.** INV-27 §4 says shipping is allocated "per D-10a E1 mechanics". E1 is one specific order; citing it as the *method* makes the specification depend on an example that may later be relabelled (P-2) or corrected. If E1's numbers ever change, the reference silently changes meaning.

```diff
-Shipping revenue 600 (300 allocated per unit, per D-10a E1 mechanics)
+Shipping revenue 600 (300 allocated per unit, using the D-10a largest-remainder method)
```

**Financial meaning:** unchanged. 300 per unit stands; only the *justification* for it changes from "because E1 says so" to "because the named method produces it".

**Dependency.** After P-1 is decided, this reference should name the **basis** as well as the method — e.g. "using the D-10a largest-remainder method, basis B-3" — because INV-27's identity tuple will then carry `shipping_allocation_basis`. That addition is substantive and is not included here.

---

## 10. Consolidated diff summary

| File | Item | Lines touched | Alters an amount? | Alters a rule? | Alters identity? |
|---|---|---|---|---|---|
| `d10a-...-decision-brief.md` | P-2 | E2 heading + 5 inserted lines | No | No | No |
| `d10a-...-decision-brief.md` | P-6 | 1 header row + 3 inserted lines | No | No | No |
| `d10a-...-decision-brief.md` | P-8 | 3 lines | No | No | No |
| `inv26-...-specification.md` | P-9 | 5 inserted lines | No | No | No |
| `inv27-...-specification.md` | P-10 | 1 line | No | No | No |

**No file in this list has been modified.** The diffs are shown for review; none is applied.

---

## 11. What applying this change set would and would not achieve

**Would achieve:** removal of two cross-example and cross-document ambiguities (P-2, P-9), removal of one reconstruction hazard (P-6), removal of one false claim about COGS (P-8), and removal of one brittle example reference (P-10). All five make the specification *easier to audit correctly*.

**Would not achieve:** C-10a.1 would remain **PARTIALLY SATISFIED — OPEN**. Of the three defects the audit recorded:

| Defect | Fixed by this change set? | Fixed by |
|---|---|---|
| D-01 — association basis selector | **No** | P-1 (awaiting founder decision) |
| D-02 — E2/E1 contradiction | **Yes** | P-2 in this set |
| D-03 — R6 contradicts C-10a.5 | **No** | P-3 (awaiting founder decision) |

So applying the editorial set alone fixes one of three defects. **D-10a continues to block G3 regardless**, because D-01 and D-03 are substantive and neither can be closed editorially. This is the most important sentence in this document: a clean editorial pass is not progress toward G3 on its own.

---

## 12. Recommended application order

1. Founder decides **P-1**, **P-3**, **P-4** (the substantive set).
2. Founder decides **P-5** (per-charge cap) and **P-7** (Σbase = 0 fallback and tie-break key).
3. Apply the substantive decisions and the editorial set **in one pass**, so the specification is never in a state where examples reference rules that have not yet changed.
4. Re-audit C-10a.1 against the applied specification.
5. Only then may C-10a.1 be considered satisfiable.

Applying the editorial set first would leave E6 (reworded by P-8) describing an allocation method (P-10) that P-1 may replace. One pass avoids that.

---

**Prepared:** 2026-11-20 · **Branch:** appjet · **Applied:** nothing · **Implementation authorisation:** not granted · **Tests executed:** none
