# INV-26 — Quantity State: Revised Multi-Dimensional Specification

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Invariant:** INV-26 — quantity state is derived from events, never stored as an asserted field
**Previous proposal:** a single mutually exclusive five-state field per unit
**Founder direction, 2026-11-20:** the **event-derived principle is approved**; the **five-state partition is not approved**. Fulfilment, return/physical disposition, and payment/refund must be **separate dimensions**, each derived from its own events, and must **not** be collapsed into one mutually exclusive state field.
**Status after this document:** **principle approved; specification revised — detailed test design still outstanding**
**Gate:** G2
**Prepared:** 2026-11-20

---

## 1. Why the single-state field had to go

The rejected partition forced one value to describe a unit that is simultaneously in several independent conditions. A unit can genuinely be *shipped*, *in transit back to the merchant*, and *refund still in flight* at the same moment. A single field must either pick one of those (losing the others) or invent a compound value for every combination, which grows as the product of the dimensions and cannot be kept consistent.

The financial consequences are not cosmetic. Revenue reversal depends on **payment/refund** progress; COGS restoration depends on **physical disposition**; revenue and COGS recognition dates depend on **fulfilment**. Collapsing them means a refund that has merely been *requested* can be read as money repaid, or a unit in transit can be read as restocked.

**Three independent dimensions, each with its own event source and its own derivation rule.**

---

## 2. The three dimensions

### Dimension F — Fulfilment

Derived from: shipment and fulfilment events. Evaluated at **unit** level, because a unit is atomically either shipped or not.

| State | Meaning |
|---|---|
| `unfulfilled` | No shipment event covers this unit |
| `fulfilled` | A shipment event covers this unit; carries `shipment_id` and `shipped_at` |

Line- and order-level completeness is **derived, not stored**:

| Derived value | Rule |
|---|---|
| `unfulfilled` | No unit shipped |
| `partially_fulfilled` | At least one, not all, units shipped |
| `fully_fulfilled` | All non-cancelled units shipped |

Derived values must never be written into the unit's dimension field. Storing both a unit state and an order state invites them to disagree.

### Dimension D — Return and physical disposition

Derived from: return authorisation, carrier return events, goods-receipt, inspection and write-off events.

| State | Meaning |
|---|---|
| `not_applicable` | Unit was never fulfilled, so no physical disposition can exist |
| `not_returned` | Fulfilled, no return initiated |
| `return_requested` | Return authorised (RMA) but the customer has not yet dispatched |
| `return_in_transit` | Customer has dispatched; not yet received by the merchant |
| `return_received` | Merchant has received the goods; not yet inspected |
| `restocked` | Inspected and returned to sellable inventory |
| `written_off` | Inspected and not sellable (damaged, lost, destroyed) |
| `disposed_other` | Inspected and disposed of by another route (donated, refurbished, returned to supplier) |

`written_off` and `disposed_other` are deliberately distinct: a write-off is an inventory-value loss, while disposal by another route may carry its own accounting treatment. Merging them would pre-empt a decision that has not been made.

### Dimension P — Payment and refund

Derived from: payment capture events, **refund records**, **refund transactions**, and refund transaction status events.

| State | Meaning | Proves money repaid? |
|---|---|---|
| `unpaid` | No captured payment | — |
| `paid` | Payment captured; no refund activity | — |
| `refund_requested` | A **refund record** exists | **No** |
| `refund_pending` | A **refund transaction** exists, not completed or settled | **No** |
| `refunded` | Refund transaction **completed/settled** for the full refundable amount | **Yes** |
| `partially_refunded` | A completed refund covering less than the full refundable amount | **Yes, in part** |
| `refund_failed` | Refund transaction failed or was voided | **No** |
| `refund_reversed` | A completed refund was itself reversed | **No — reversed** |

**This table is the enforcement point for the founder's earlier correction:** a refund record is not proof of repayment, and `refund_requested` must never be read as `refunded`. Only `refunded` and `partially_refunded` permit revenue reversal recognition.

---

## 3. Derivation rules

**D-26.1 — Derived, never asserted.** Each dimension is computed by folding that dimension's own events in event-time order. No process may write a dimension value directly. A stored dimension value that disagrees with its event fold is a **defect**, detected by recomputation, not corrected by overwriting the field.

**D-26.2 — Independent folds.** A dimension folds only its own event types. A refund event must not move dimension D. A goods-receipt event must not move dimension P.

**D-26.3 — Corrections are events.** A mis-keyed shipment is corrected by a correction event; the dimension re-derives. Nothing is edited in place (INV-19).

**D-26.4 — Unknown is a distinct outcome.** If a dimension's events are absent or contradictory, the result is `indeterminate` — not a default. `indeterminate` blocks merchant-visible profitability for the affected unit and blocks any automated action (per the Co-Pilot guardrails).

**D-26.5 — Determinism.** Folding the same events in event-time order always yields the same three values. Ties in event time are broken by ingestion sequence, which is stable and recorded.

---

## 4. Impossible combinations — integrity controls

The three dimensions are orthogonal, but not every combination is reachable. Rejecting the impossible ones is a genuine control, not pedantry.

| Combination | Verdict | Reason |
|---|---|---|
| `unfulfilled` + `not_returned` / `return_*` / `restocked` / `written_off` / `disposed_other` | **Invalid** | Goods that never shipped cannot be returned or disposed of |
| `unfulfilled` + `not_applicable` | **Valid** | The only valid D for an unfulfilled unit |
| `restocked` / `written_off` / `disposed_other` without a prior `return_received` | **Invalid** | Disposition requires physical receipt |
| `unfulfilled` + `refunded` | **Valid — and important** | A unit may be refunded before it ever ships (cancellation) |
| `fulfilled` + `not_returned` + `refunded` | **Valid** | Refund granted without goods returning (goodwill, or not required back) |
| `fulfilled` + `return_in_transit` + `refund_requested` | **Valid but flagged** | Refund promised before goods received — a real risk position, not a data error |
| `fulfilled` + `return_in_transit` + `refunded` | **Valid but flagged** | Money repaid before goods received |
| `unpaid` + any refund state other than `unpaid` | **Invalid** | Cannot refund an uncaptured payment |
| `refund_failed` followed by no further refund event | **Valid** | The refund record remains visible as failed; no reversal recognised |

**Rule:** an invalid combination is raised as a **data-integrity exception** against the unit, surfaced in the data-quality report, and blocks merchant-visible profitability and automated action for that unit. It is never silently coerced into a "nearest valid" state.

---

## 5. Permitted transitions

Each transition names the event that causes it. A transition not listed is a defect.

**Fulfilment**

| From | To | Cause |
|---|---|---|
| `unfulfilled` | `fulfilled` | Shipment event |
| `fulfilled` | `unfulfilled` | Correction event only (a mis-recorded shipment). Not a business transition. |

**Disposition**

| From | To | Cause |
|---|---|---|
| `not_applicable` | `not_returned` | Fulfilment (D re-derives when F becomes `fulfilled`) |
| `not_returned` | `return_requested` | Return authorisation event |
| `return_requested` | `return_in_transit` | Carrier return dispatch event |
| `return_in_transit` | `return_received` | Goods-receipt event |
| `return_received` | `restocked` / `written_off` / `disposed_other` | Inspection and disposition event |
| `restocked` / `written_off` / `disposed_other` | any | Correction event only |

Terminal dispositions are terminal for business purposes. A reclassification after disposition is a correction and must be auditable.

**Payment / refund**

| From | To | Cause |
|---|---|---|
| `unpaid` | `paid` | Payment capture event |
| `paid` | `refund_requested` | Refund record created |
| `refund_requested` | `refund_pending` | Refund transaction created |
| `refund_pending` | `refunded` / `partially_refunded` | Refund transaction completed/settled |
| `refund_pending` | `refund_failed` | Refund transaction failed or voided |
| `refunded` / `partially_refunded` | `refund_reversed` | Refund reversal event |
| `refund_failed` | `refund_requested` | A further refund record is created |

**Note the deliberate absence** of any transition from `refund_requested` or `refund_pending` to `refunded` that is not driven by a settled transaction. There is no path to `refunded` on a refund *record* alone.

---

## 6. Combination → financial treatment

The treatment of a unit is a function of all three dimensions. This is the table the engine reads.

| F | D | P | Revenue | COGS | Other |
|---|---|---|---|---|---|
| `unfulfilled` | `not_applicable` | `paid` | Commercial only | Commercial only | Shipping allocation **pending** |
| `unfulfilled` | `not_applicable` | `refunded` | Commercial recognises then reverses on refund date | Same | Shipping allocation **released**, never recognised |
| `fulfilled` | `not_returned` | `paid` | Both views | Both views, different snapshots | — |
| `fulfilled` | `not_returned` | `refund_requested` | **No reversal yet** | Unchanged | Flag: refund record, no repayment |
| `fulfilled` | `return_in_transit` | `refund_requested` | **No reversal yet** | Unchanged | Flag: refund promised before receipt; reverse-logistics **estimated** |
| `fulfilled` | `return_in_transit` | `refunded` | Reversed on completion date | Unchanged until disposition known | Flag: repaid before receipt |
| `fulfilled` | `return_received` | `refunded` | Reversed | **Provisional** — disposition unknown | Must be resolved before period close |
| `fulfilled` | `restocked` | `refunded` | Reversed | **Restored** to inventory | Reverse logistics recognised |
| `fulfilled` | `written_off` | `refunded` | Reversed | **Retained — not restored** | **No separate write-off cost added** |
| `fulfilled` | `disposed_other` | `refunded` | Reversed | Per disposal route — policy open | — |
| `fulfilled` | `not_returned` | `refund_reversed` | Re-recognised by compensating event | Unchanged | Original events untouched |
| any | any | `refund_failed` | No reversal | Unchanged | Failed refund remains visible |
| any | `indeterminate` | any | **Blocked** | **Blocked** | Data-quality block |

The `written_off` row encodes the founder-approved correction to the earlier Example E: **the inventory cost is recognised exactly once, as retained COGS. It is never recognised again as an additional write-off cost.** A disposal fee, if one genuinely exists, is a separate and legitimate cost.

---

## 7. Scenario walkthroughs

Eight scenarios, each showing the three dimensions over time and the resulting treatment.

### S1 — Partial shipment

Order of three units: A, B (line L1), C (line L2). A and B ship 2026-02-03; C remains.

| Unit | F | D | P |
|---|---|---|---|
| A | `fulfilled` | `not_returned` | `paid` |
| B | `fulfilled` | `not_returned` | `paid` |
| C | `unfulfilled` | `not_applicable` | `paid` |

Line completeness: `partially_fulfilled` (derived).
Treatment: A and B recognised in both views. C recognised in the Commercial View only, with its shipping allocation (426 per D-10a E1) **pending**. The 426 is the quantity term in INV-27's reconciliation.

### S2 — Partial cancellation

Continuing S1, unit C is cancelled and refunded on 2026-02-10 before shipping.

| Unit | F | D | P |
|---|---|---|---|
| C | `unfulfilled` | `not_applicable` | `refunded` |

Treatment: Commercial View reverses C's revenue and order-date COGS on 2026-02-10. Fulfilment View never recognised C. Shipping allocation **released**, not recognised, not redistributed. C's contribution to the cumulative view difference is permanently zero — if it is not, the reconciliation has a defect.

### S3 — Return, restocked

Unit A returned, received, inspected and restocked; refund completed.

| Unit | F | D | P |
|---|---|---|---|
| A | `fulfilled` | `restocked` | `refunded` |

Treatment: revenue reversed on the refund completion date. **COGS restored** to inventory, because the goods are sellable again. Reverse-logistics cost recognised as an actual or estimated return cost (D-12). The unit's cost has been recognised **once**, across sale and restocking — never twice, never zero.

### S4 — Return, damaged and written off

Unit A returned, received, inspected and found damaged.

| Unit | F | D | P |
|---|---|---|---|
| A | `fulfilled` | `written_off` | `refunded` |

Treatment: revenue reversed. **COGS retained** — the inventory value is gone, so the cost stays in cost of sales and is **not** restored. Critically, **no additional inventory write-off cost is recorded**; adding one would recognise the same inventory value twice. This is the defect corrected in the earlier Example E, now encoded structurally: the difference between `restocked` and `written_off` *is* the COGS restoration decision, so it cannot be double-counted by construction.

### S5 — Refund requested, goods in transit

Customer dispatched the return; merchant has created a refund record but no refund transaction.

| Unit | F | D | P |
|---|---|---|---|
| A | `fulfilled` | `return_in_transit` | `refund_requested` |

Treatment: **no revenue reversal.** A refund record is not repayment. Flagged as *refund promised before goods received* — a genuine commercial risk the merchant should see. Reverse-logistics cost estimated if a carrier rate is known.

### S6 — Refund pending

Refund transaction created, not yet completed or settled.

| Unit | F | D | P |
|---|---|---|---|
| A | `fulfilled` | `return_in_transit` | `refund_pending` |

Treatment: still **no final reversal**. Presented as "refund in flight" with its amount, without netting revenue. If the transaction subsequently fails, P becomes `refund_failed` and no reversal is ever recognised.

### S7 — Refund completed, disposition unknown

Refund settled; goods received but not yet inspected.

| Unit | F | D | P |
|---|---|---|---|
| A | `fulfilled` | `return_received` | `refunded` |

Treatment: revenue reversed on the completion date. **COGS treatment is provisional** because restoration depends on a disposition that does not yet exist. This is a legitimate provisional state, must be labelled as such, and **must be resolved before the period is closed** — a closed period containing provisional COGS is a defect.

### S8 — Refund reversed

A completed refund is itself reversed (for example a refund cancellation or a chargeback reversal).

| Unit | F | D | P |
|---|---|---|---|
| A | `fulfilled` | `not_returned` | `refund_reversed` |

Treatment: revenue **re-recognised by a compensating event** on the reversal date. The original refund event and the original revenue event are untouched — the reversal is a new, additive record (INV-19). It must carry a reference to the refund it reverses, so the chain is auditable end to end.

---

## 8. What this specification requires of the tests

Detailed test design remains outstanding, but the required coverage follows directly:

1. Each dimension folds independently — a refund event never changes D; a receipt event never changes P.
2. Every permitted transition, driven by its named event.
3. Every impossible combination is rejected as a data-integrity exception, not coerced.
4. `refund_requested` and `refund_pending` never produce a revenue reversal.
5. `restocked` restores COGS; `written_off` retains it; neither adds a second cost.
6. Derived line/order completeness is consistent with unit states and is never stored on the unit.
7. `indeterminate` blocks merchant-visible profitability and automated action.
8. S1–S8 reproduce the stated treatments exactly.

### Fail-closed conditions (normative)

These are the conditions under which the engine must refuse rather than degrade. They are listed separately from the test cases because a test can pass while a fail-closed rule is missing.

| ID | Condition | Required behaviour |
|---|---|---|
| FC-26-1 | A unit has no derivable value in any dimension | Refuse; do not default to `not_returned` or `unfulfilled` |
| FC-26-2 | An impossible combination is derived (any row of §4) | Data-integrity exception; quarantine the source event; no coercion to a "nearest valid" state |
| FC-26-3 | Any dimension is `indeterminate` | Block merchant-visible profitability and all automated action for the affected line, order and aggregates containing it |
| FC-26-4 | A refund event arrives for a unit whose P dimension is `unpaid` | Integrity violation; block (T-14-adjacent: cannot refund what was never charged) |
| FC-26-5 | A disposition event arrives without a preceding return event | Integrity violation; block |
| FC-26-6 | Fulfilment quantity exceeds ordered quantity, or disposition quantity exceeds returned quantity | Block; never clamp silently |
| FC-26-7 | Events for one unit cannot be totally ordered (same timestamp, conflicting content) | Block; do not resolve by ingestion order |

### Test cases

Each case states the event sequence, the expected derived dimensions and the expected financial consequence. **None has been executed.**

| ID | Scenario | Events | Expected F | Expected D | Expected P | Expected financial consequence |
|---|---|---|---|---|---|---|
| TC-26-01 | Partial shipment (S1) | order(3 units) → ship(A,B) | `partially_fulfilled` | `not_returned` | `paid` | A,B revenue and COGS recognised at shipment; C pending |
| TC-26-02 | Full shipment, second shipment (S1 cont.) | ship(C) | `fulfilled` | `not_returned` | `paid` | C recognised; line complete |
| TC-26-03 | Partial cancellation (S2) | order(3) → cancel(C) | `partially_fulfilled` | `not_returned` | `partially_refunded` | C's revenue never recognised; cancellation release event, not an edit |
| TC-26-04 | Return, restocked (S3) | ship(A) → return(A) → inspect(restock) | `fulfilled` | `returned_restockable` | `partially_refunded` | COGS **restored**; exactly one cost reversal; no second cost |
| TC-26-05 | Return, damaged (S4) | ship(A) → return(A) → inspect(damaged) | `fulfilled` | `returned_damaged` | `partially_refunded` | COGS **retained**; no additional write-off cost line |
| TC-26-06 | Refund requested, in transit (S5) | ship(A) → refund_requested(A) | `fulfilled` | `return_in_transit` | `refund_requested` | **No revenue reversal yet**; shipping cost incurred |
| TC-26-07 | Refund pending (S6) | … → refund_pending(A) | `fulfilled` | `return_in_transit` | `refund_pending` | **Still no reversal**; provisional status only |
| TC-26-08 | Refund completed, disposition unknown (S7) | … → refund_completed(A) | `fulfilled` | `not_returned` or `indeterminate` | `refunded` | Revenue reversed on refund date; **COGS treatment blocked pending disposition** |
| TC-26-09 | Refund reversed (S8) | … → refund_reversed(A) | `fulfilled` | per prior return state | `paid` | Reversal of the reversal; net position restored via new events |
| TC-26-10 | Dimension independence | ship(A) → refund(A) | unchanged by refund | unchanged by refund | `refunded` | Proves P folds from payment events only |
| TC-26-11 | Independence, other direction | return(A) | unchanged by return | changes | unchanged by return | Proves D and P are independent |
| TC-26-12 | FC-26-2 impossible combination | refund_completed without any payment event | — | — | — | Rejected as integrity exception, **not coerced** |
| TC-26-13 | FC-26-2 restock without return | inspect(restock) with no return event | — | — | — | Rejected |
| TC-26-14 | FC-26-6 over-fulfilment | ship 4 of 3 ordered | — | — | — | Rejected; no clamping |
| TC-26-15 | FC-26-3 indeterminate blocks visibility | any dimension indeterminate | — | — | — | Line excluded from merchant-visible profit; aggregates containing it blocked |
| TC-26-16 | Derivation not stored | any of the above | — | — | — | No unit-state column exists; recomputation from events yields identical values |

**None of these tests exists.** No database, schema or code exists. This is a specification.

---

## 9. Status

| Item | Status |
|---|---|
| Event-derived principle | **Approved by founder** |
| Five-state partition | **Rejected by founder — withdrawn** |
| Three-dimension specification | **Delivered here; awaiting founder approval** |
| Transition rules | Proposed |
| Combination → treatment matrix | Proposed |
| Detailed test design | **Delivered here (TC-26-01 – TC-26-16, FC-26-1 – FC-26-7); not executed** |
| Implementation | None |
| Empirical verification | None |

---

## 10. Boundary statement

Documentation only. No application code, schema, migration, test, Supabase provisioning, provider connection, merchant data or deployment. No provider behaviour is asserted as verified anywhere in this document.
