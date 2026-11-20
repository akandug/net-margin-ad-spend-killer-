# P-3 — Post-Acceptance Shipping Discounts

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Decision package:** D-10a substantive correction review, Part B
**Defect addressed:** D-03 (R6 contradicts C-10a.5)
**Status of this brief:** PROPOSED FOR FOUNDER DECISION — **not applied, not approved, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS — remains AWC)
**Affected conditions:** C-10a.2, C-10a.3, C-10a.5
**Authority:** founder instruction 2026-11-20 (third review), Part B, P-3

---

## 1. The contradiction

D-10a contains two rules that cannot both hold:

- **R6** (as written) states that a shipping discount is applied to the original shipping revenue, which implies the allocated amount changes.
- **C-10a.5** states that every change after acceptance — including a shipping discount — produces an explicit event and that no prior allocation is edited in place.

If R6 is read literally, the discount is netted into S_net and the allocation is recomputed. That directly violates C-10a.2 (allocation computed once, never recomputed from changed inputs), C-10a.3 (each quantity's share survives without reallocation) and C-10a.5 (no in-place edit). If C-10a.5 governs, R6 as written is wrong.

The founder's approval of D-10a explicitly includes both "cancellations, shipping discounts ... are represented by explicit adjustment, release or reversal events" **and** "the original allocation is never silently rewritten". So C-10a.5 expresses the approved intent and R6's wording is the defect. This brief resolves it by splitting R6 into an order-time rule and a post-acceptance rule, and by making the post-acceptance rule operate on a bucket model that cannot reallocate.

**Why this matters beyond wording.** A post-acceptance shipping discount is a reduction in money the merchant will collect. If it is netted into S_net and reallocated, every unit's shipping share changes — including units already shipped, whose revenue is already recognised and may already have been reported to the merchant. That is the same history-rewriting failure the founder rejected in the residual-on-final-shipment rule, arriving through a different door.

---

## 2. The resolution — R6 split into five rules

**R6.1 — Order-time shipping discount.** A shipping discount that is part of the order's own terms at acceptance is netted into S_net **before** allocation. It is not a later event and produces no adjustment record. This is E4's case and is unchanged by this brief.

**R6.2 — Post-acceptance shipping discount, order-scoped.** A discount granted after acceptance that is not attributable to specific units is recorded as an **order-level adjustment line**. It is **not** pushed into any unit's share. The original allocation is untouched.

**R6.3 — Post-acceptance shipping discount, unit-scoped.** A discount that the provider or merchant attributes to specific units is recorded as a **unit-scoped adjustment** that moves the affected amount out of that unit's `pending` or `recognised` bucket into a `discounted` bucket. The original allocation record still shows the original share; the adjustment is a separate, dated record referencing it.

**R6.4 — No pending balance remaining.** If a "shipping discount" arrives when the affected scope has no pending balance, it is **not** a price reduction — the goods have shipped and the money has been collected. It is reclassified as a **post-fulfilment shipping credit** and falls under the R10 ceiling (P-4). It is never applied as negative revenue without that ceiling check.

**R6.5 — Period attribution.** In the management views, an order-scoped post-acceptance discount is deducted in the period of its **effective date**, in full. Statutory treatment is **not** established by this rule and remains subject to professional accounting review (§10).

---

## 3. The bucket model

The allocation is immutable; the *state* of each unit's shipping revenue moves through a partition. Five buckets, mutually exclusive, single membership at any instant:

| Bucket | Meaning | Entered from |
|---|---|---|
| `pending` | Not yet shipped, not cancelled, not reduced | Initial state at allocation |
| `recognised` | Shipped; shipping revenue earned and retained | `pending` on shipment |
| `released` | Cancelled; shipping revenue never recognised | `pending` on cancellation |
| `refunded` | Shipping revenue returned to the customer | `recognised` or `released` on a completed refund |
| `discounted` | Shipping revenue waived by a unit-scoped discount | `pending` or `recognised` on a unit-scoped discount |

**Order-level lines** (not buckets, because they have no unit owner):

| Line | Meaning |
|---|---|
| `D_post_order` | Cumulative order-scoped post-acceptance discounts |
| `Rf_order` | Cumulative order-scoped shipping refunds |

### 3.1 Conservation and reporting

**INV-31 (proposed) — partition conservation:**

```
Σ_u (pending_u + recognised_u + released_u + refunded_u + discounted_u) = S_net + Chg_add
```

for every order, at every instant, in every calculation view. `Chg_add` is additional shipping charged after acceptance (re-ship, upgraded service).

**Reported net shipping revenue:**

```
reported = Σ_u recognised_u − D_post_order − Rf_order
```

Order-scoped lines are deliberately excluded from the unit partition. The precedent is D-13: the 63-unit payment-fee residual was approved as an explicit order-level cost rather than being artificially allocated to products. An order-scoped discount has no unit owner for exactly the same reason, and inventing an owner would be the artificial allocation D-13 rejects.

The consequence is that product-level contribution profit does not absorb an order-scoped discount, while store-level contribution profit does. That is a deliberate, disclosed asymmetry, not an omission.

---

## 4. Effective date versus event date

Both are recorded, on every adjustment:

| Field | Meaning | Used for |
|---|---|---|
| `effective_date` | The date the discount applies commercially | Management-view period attribution (R6.5) |
| `event_date` | The date the merchant granted it, per the source | Audit trail and ordering |
| `ingested_at` | The date the platform learned of it | Late-arrival detection |

R6.5 attributes to `effective_date`. If a discount is backdated across a closed period, the deduction lands in the closed period in the statutory view — which is a restatement — but in the management view it lands in the period of the effective date and the restatement is disclosed. **This distinction is precisely the open question the founder reserved for the accounting adviser (D-18 / F-1)** and this brief does not resolve it.

---

## 5. Treatment by quantity state

Allocation P3-O1 throughout: A 287, B 287, C 426; S_net 1,000; B-3 pooled basis.

| State of the affected scope | Order-scoped discount | Unit-scoped discount |
|---|---|---|
| All `pending` | `D_post_order` increases; buckets unchanged; reported revenue reduced in the effective period | Target unit's `pending` → `discounted`; conservation holds because the `discounted` bucket absorbs it |
| Some `recognised` | `D_post_order` increases; deducted in full in the effective period (front-loaded — §6.3) | Target unit: `pending` first, then `recognised` → `discounted`; a reduction from `recognised` is a revenue reversal in the current period |
| Target `released` (cancelled) | `D_post_order` increases; order ceiling still applies | Allowed: `released` → `discounted`; the unit never contributed revenue, so no revenue reversal arises |
| Target already `refunded` | Ceiling applies (P-4) | **Quarantined** — the same economic amount would be reduced twice (§7.4) |
| Target `discounted` already | Ceiling applies | Further discount allowed up to the unit's remaining share; excess quarantined (P-4, Tier 2) |

---

## 6. Worked examples

### 6.1 P3-1 — Order-scoped discount before any shipment

- Allocation (immutable): A 287, B 287, C 426. Σ 1,000.
- 2026-02-01: order-scoped shipping discount **200**, effective 02-01, nothing shipped.
- Recorded: adjustment event, `scope = order`, `amount = 200`, `type = shipping_discount_post_acceptance`.
- Buckets: unchanged — A 287 pending, B 287 pending, C 426 pending. Σ 1,000 ✓
- `D_post_order` = 200.

| Period | Event | Σ recognised | D_post_order deducted | Reported |
|---|---|---|---|---|
| Feb | A ships 02-05 | 287 | 200 | **87** |
| Mar | B ships 03-02, C ships 03-10 | 287 + 426 = 713 | 0 (already taken) | **713** |
| | | Σ 1,000 | Σ 200 | **800** |

Check: 800 = 1,000 − 200 ✓. The 200 is deducted **once**, at order level, and never from a unit share — no double count.

Note the shape of the answer: February reports 87 of shipping revenue while 287 was recognised. That is the front-loading consequence of R6.5 (§6.3 compares the alternatives).

### 6.2 P3-2 — Unit-scoped discount during partial fulfilment

- A ships 02-05 → A `recognised` 287.
- 2026-02-10: shipping discount **100** attributable to unit C.
- Recorded: adjustment event, `scope = unit C`, `amount = 100`. Original allocation record for C still states 426.
- Bucket movement: C `pending` 426 → `discounted` 100 + `pending` 326.
- Buckets at 02-10: A recognised 287, B pending 287, C pending 326, C discounted 100. **Σ = 1,000** ✓ (INV-31 holds)
- March: B ships 287, C ships 326 → Σ recognised = 287 + 287 + 326 = 900.
- Reported = 900 − 0 − 0 = **900** = 1,000 − 100 ✓

The original 426 is still on the allocation record and still reproducible; the 100 is a dated adjustment referencing it. Nothing was rewritten.

### 6.3 P3-3 — Order-scoped discount after full shipment

- All three units shipped by 03-10: Σ recognised = 1,000.
- 2026-03-20: shipping discount **150** granted, order-scoped.
- No pending balance remains → **R6.4 applies**: reclassified as a post-fulfilment shipping credit, subject to the R10 ceiling.
- Ceiling check (P-4): `Σ_u(refunded + discounted) + Rf_order + D_post_order = 0`; `S_net + Chg_add = 1,000`. Adding 150 → 150 ≤ 1,000 ✓ → **accepted**.
- Recorded as `Rf_order` = 150 (order-scoped credit). Reported March = 1,000 − 150 = **850**.
- Cumulative net shipping revenue = **850** = 1,000 − 150 ✓

**Variant — ceiling exhausted.** If 900 had already been refunded order-scoped, remaining headroom = 1,000 − 900 = 100 < 150. The 150 is **rejected and quarantined** for review (P-4 Tier 1). It is not partially accepted, because silently accepting 100 of a 150 credit would under-refund the customer and hide the anomaly.

### 6.4 P3-4 — Duplicate delivery

The same 150 credit is delivered twice by webhook retry, with the same provider credit identity.

- Idempotency identity (INV-12) matches → the second delivery is a **no-op**.
- `Rf_order` remains 150, not 300. Reported remains 850, not 700.

### 6.5 P3-5 — Discount against an already-refunded unit (double economic event)

- Unit C's shipping 426 is refunded → C `recognised`/`pending` → `refunded` 426.
- A 426 "discount" scoped to unit C then arrives.
- Unit C's remaining un-reduced share = 426 − 426 (refunded) = 0. A further 426 would reduce the same economic amount twice.
- Disposition: **quarantined** as `duplicate_economic_event`. Not accepted, not permanently rejected — a human must determine whether this is a duplicate with a different identity, a mis-scoped goodwill gesture, or a provider defect.

This case is the reason the P-4 ceiling cannot be order-level only: at order level, `refunded 426 + discounted 426 = 852 ≤ 1,000` would **pass**, and the merchant would lose 426 twice.

---

## 7. Alternatives considered

### 7.1 For the discount's effect on unit shares

| Option | Mechanism | Advantages | Disadvantages | Verdict |
|---|---|---|---|---|
| **B-1** | Recompute S_net and reallocate across all units | Each unit's share stays "correct" for the new total | Rewrites recognised figures; violates C-10a.2, C-10a.3, C-10a.5; destroys reproducibility | **Rejected** — this is the literal reading of R6 and it is the defect |
| **B-2** | Keep shares; apply the discount pro rata across units as a derived deduction | Product-level figures absorb the discount proportionally | Requires an allocation of the discount, i.e. a new allocation basis for a quantity with no owner; the pro-rata basis is arbitrary for an order-level concession | Rejected for order-scoped; see §7.2 for the period variant |
| **B-3** | **Keep shares immutable; order-scoped discount as an order-level line; unit-scoped discount as a bucket movement** | Never rewrites history; mirrors the approved D-13 residual precedent; product and store totals both remain correct and explainable | Product-level figures do not absorb order-scoped discounts; requires disclosure of the order-level line | **Recommended** |

### 7.2 For period attribution of an order-scoped discount

| Option | Feb (A ships, 287 recognised) | Mar (B, C ship, 713) | Total | Assessment |
|---|---|---|---|---|
| **C-1 — Deduct in full in the effective period** | 287 − 200 = **87** | **713** | 800 ✓ | Deterministic, no allocation arithmetic, no plug. Front-loads the deduction |
| **C-2 — Pro rata to recognised gross** | 200 × 287/1,000 = 57.4 → **57**; Feb = 230 | 200 − 57 = 143; Mar = 713 − 143 = **570** | 800 ✓ | Smoother; but requires allocating a discount across periods, and the rounding residue needs a home — the same balancing-plug problem the founder rejected for shipments |
| **C-3 — Deduct only when the order is fully fulfilled** | **287** (no deduction) | 713 − 200 = **513** | 800 ✓ | Defers the deduction; makes an open order's shipping revenue overstated until completion |

**C-1 is recommended for the management view.** It needs no allocation step, so it cannot produce a plug; it is trivially reproducible; and it is conservative (it does not overstate revenue in an open period). **C-2 is the treatment most likely to be required in the statutory view**, because a transaction-price reduction is normally allocated to the performance obligations it relates to — which is an accounting question, not a product-design question, and is reserved for professional review.

The choice between C-1 and C-2 **does not affect** order-level totals, unit shares, or lifetime revenue. It affects only the period split and the product-level presentation. That is why it can be deferred without blocking the ledger design, and why it must nonetheless be disclosed rather than left implicit.

---

## 8. Reproducibility across calculation versions

- The adjustment event set is part of the **source-data boundary** in the INV-27 identity. A calculation version bump that changes the boundary produces a different identity, not a different answer to the same question.
- Recomputing any period from the same allocation records plus the same adjustment events must produce byte-identical bucket balances and reported figures. This is testable arithmetic, not an assertion.
- Adjustments are never deleted, only superseded by a further adjustment. A reversal of a discount is itself an adjustment event with its own date.
- A recalculation must **not** consult the current state of an open adjustment; it must replay the event set bounded by the calculation's source-data boundary.

---

## 9. Cross-document impact

| Document / section | Change required if P-3 is approved | Kind |
|---|---|---|
| D-10a R6 | Split into R6.1–R6.5 | Substantive |
| D-10a E4 (shipping discount) | Confirm it is R6.1 (order-time) and add a cross-reference | Editorial |
| D-10a E7 (shipping refund) | Cross-reference R6.4 for post-fulfilment credits | Editorial |
| D-10a C-10a.5 | Restate as satisfied by R6.2/R6.3 rather than contradicted by R6 | Substantive |
| INV-26 S2 | Cancellation → `released` bucket; add `discounted` bucket to the state table | Substantive |
| INV-27 §3 aggregation | Add order-level `D_post_order` / `Rf_order` as legitimate non-unit lines | Substantive |
| INV-27 §4 worked example | Unaffected — no discounts in O1/O2 | None |
| INV-12 idempotency | Add discount/credit identity, distinct from refund identity | Substantive |
| D-13 | Cited as the precedent for order-level lines; no change | Reference |
| D-18 | Period attribution (R6.5 vs C-2) is a D-18 question | Dependency |
| New invariants | INV-29, INV-31 (proposed, §11) | New |

---

## 10. Unresolved questions and professional review

| ID | Question | Classification |
|---|---|---|
| OQ-P3-1 | Should an order-scoped post-acceptance discount be deducted in full in the effective period (C-1) or allocated pro rata to recognised gross (C-2)? | **Professional accounting review** — revenue-recognition timing. Management view may use C-1; statutory view may require C-2 |
| OQ-P3-2 | Is a post-fulfilment shipping credit (R6.4) revenue reduction or a customer refund liability? | **Professional accounting review** |
| OQ-P3-3 | Does a backdated discount require a restatement of a closed statutory period? | **Professional accounting review** — D-18 |
| OQ-P3-4 | Should product-level contribution profit absorb order-scoped discounts in *any* presentation, e.g. a footnote line at product level? | Founder product decision. This brief proposes: no absorption, disclose the order-level line only |
| OQ-P3-5 | Does the target provider distinguish a shipping *discount* from a shipping *refund* in its payload, or must we infer it from timing? | **Provider verification — P-01.** If the provider does not distinguish them, R6.4's reclassification rule needs a payload-based test rather than a semantic one |
| OQ-P3-6 | What is the maximum plausible number of adjustments per order before a review threshold should fire? | Implementation policy; not measurable now |

---

## 11. New invariants proposed

### INV-29 — Adjustment-event separation

**Rule.** No post-acceptance change to shipping revenue may mutate an allocation record. Every change is an adjustment event with its own identity, date and scope; derived balances are computed as the allocation plus its adjustment event set, and must be recomputable from those records alone.

| Element | Specification |
|---|---|
| Scope | All shipping revenue adjustments: discounts, charge changes, releases, refunds, credits |
| Calculation basis | Derived bucket balance = allocation share, plus the ordered set of adjustment events bounded by the calculation's source-data boundary |
| Enforcement | Append-only allocation records (INV-19); no UPDATE path for allocation amounts; adjustment events carry a reference to the allocation they affect; recomputation test compares derived balances against stored balances |
| Failure behaviour | An adjustment that cannot reference a valid allocation is **rejected** at ingestion. A recomputation that does not reproduce the stored balance is a **hard defect** that blocks calculation output |
| Test case | TC-29-01: P3-2 — after the 100 unit-scoped discount, recompute C's buckets from allocation + adjustment → expect pending 326, discounted 100, and the allocation record still reading 426. TC-29-02: attempt an in-place update of C's allocation → expect refusal at the privilege layer. TC-29-03: replay the event set with the discount omitted → expect a detectable divergence |

### INV-31 — Shipping bucket partition conservation

**Rule.** For every order, `Σ_u (pending + recognised + released + refunded + discounted) = S_net + Chg_add` at every instant, with single membership per unit bucket and only the defined transitions permitted.

| Element | Specification |
|---|---|
| Scope | Every order, every calculation view, every instant |
| Calculation basis | Per-unit bucket membership; order-level `D_post_order` and `Rf_order` excluded from the partition by design |
| Enforcement | Bucket transitions restricted to the declared graph (`pending→recognised`, `pending→released`, `pending→discounted`, `recognised→refunded`, `recognised→discounted`, `released→refunded`, `released→discounted`); conservation asserted after every transition; `Chg_add` raises the target |
| Failure behaviour | Conservation violation → transaction rolled back; the adjustment is **not** applied. Illegal transition → rejected |
| Test case | TC-31-01: P3-1 — after the 200 order-scoped discount, Σ buckets must still be 1,000, because the discount is an order-level line. TC-31-02: P3-2 — after the 100 unit-scoped discount, Σ buckets must still be 1,000 with 100 in `discounted`. TC-31-03: attempt `recognised→pending` → expect rejection |

Both are **proposals**. Numbering requires founder approval.

---

## 12. Recommendation, separated from the proposed decision

### 12.1 AppJet's recommendation

Adopt the **R6 split** (R6.1–R6.5), the **bucket model** in §3, **B-3** for the effect on unit shares, and **C-1** for period attribution in the management view. Adopt INV-29 and INV-31.

The reasoning, stated so it can be challenged:

1. **The contradiction cannot be resolved in R6's favour.** Netting a post-acceptance discount into S_net and reallocating rewrites recognised revenue, which the founder explicitly approved against.
2. **The bucket model makes the invariant mechanical.** With five mutually exclusive buckets and a declared transition graph, "did anything get double-counted?" becomes an arithmetic check (INV-31) rather than a review.
3. **Order-scoped discounts belong at order level.** This is not a new principle — it is the D-13 residual decision applied consistently. Allocating an ownerless concession to products would be the artificial allocation D-13 rejects, and it would make product-level figures move for reasons unrelated to the product.
4. **C-1 is chosen for reproducibility, not for elegance.** It requires no allocation of a discount across periods, so it cannot produce a rounding plug — the specific failure mode the founder rejected in the residual-on-final-shipment rule.

The honest weakness of this recommendation: **product-level contribution profit will not absorb order-scoped discounts**, so a product that triggered a goodwill shipping discount will look better than it is at product level while the store total is correct. The mitigation is disclosure of the order-level line, not allocation. The founder may reasonably prefer the pro-rata alternative; if so, that is a different decision with a different set of consequences and should be recorded as such.

### 12.2 What the founder is being asked to decide

- **Approve** the R6 split and the bucket model, with B-3 and C-1;
- **Approve with conditions** — most likely naming C-2 (pro rata) instead of C-1, or requiring a product-level disclosure line;
- **Defer**, in which case D-03 stays open, C-10a.5 remains contradicted by R6, and C-10a.1 cannot be satisfied.

OQ-P3-1, OQ-P3-2 and OQ-P3-3 remain **professional accounting review** items regardless of which option is chosen. OQ-P3-5 remains a **provider verification** item under P-01.

No part of this brief has been applied, and no test has been executed.

---

**Prepared:** 2026-11-20 · **Branch:** appjet · **Implementation authorisation:** not granted · **Supabase:** not connected · **Tests executed:** none
