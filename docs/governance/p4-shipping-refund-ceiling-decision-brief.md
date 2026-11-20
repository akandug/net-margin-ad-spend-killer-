# P-4 — Shipping-Refund Ceiling

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Decision package:** D-10a substantive correction review, Part B
**Defect addressed:** Shipping-refund ceiling absent (audit finding, C-10a.1)
**Status of this brief:** PROPOSED FOR FOUNDER DECISION — **not applied, not approved, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS — remains AWC)
**Affected conditions:** C-10a.1, C-10a.4
**Authority:** founder instruction 2026-11-20 (third review), Part B, P-4

---

## 1. What is being capped, and why it is not obvious

A shipping refund returns to the customer money the customer paid for shipping. The platform must prevent cumulative shipping refunds from exceeding the legitimate refundable amount, because an uncapped refund stream produces negative net shipping revenue, which flows straight into contribution profit and can make a healthy product look unprofitable or vice versa.

The founder's instruction contains a specific warning:

> Do not use recognised shipping revenue as the only refund ceiling without proving that this does not block legitimate refunds for unshipped goods.

This brief takes that warning seriously and shows that **recognised revenue is the wrong ceiling**, by construction rather than by argument. §3 gives the proof. §4 defines the correct basis.

---

## 2. Base order for this brief

**P4-O1** — the P1-O1 order, pooled basis (B-3):

| Unit | Line | Net per unit | Shipping allocation | State |
|---|---|---|---|---|
| A | L1 | 2,700 | **287** | ships 2026-02-03 |
| B | L1 | 2,700 | **287** | pending |
| C | L2 | 4,000 | **426** | pending |
| | | | **Σ 1,000** | |

`S_net = 1,000`. No order-time discount, no additional charges: `Chg_add = 0`.

---

## 3. Why recognised revenue cannot be the ceiling

### 3.1 Proof — cancelled goods

- A ships 02-03 → A `recognised` 287.
- B and C are cancelled 02-10 → B `released` 287, C `released` 426.
- The customer paid 1,000 for shipping. Two of three parcels never shipped. The merchant refunds the shipping attributable to them: **713**.

| Ceiling basis | Value | Result for a 713 refund |
|---|---|---|
| `Σ recognised` | **287** | **713 > 287 → refund blocked** |
| `S_net + Chg_add` | **1,000** | 713 ≤ 1,000 → refund accepted ✓ |

The recognised-only ceiling blocks a refund the customer is plainly entitled to, and which the merchant has an obligation to make. That is not a conservative error; it is a **wrong** error — it would cause the platform to quarantine or reject a legitimate supplier-initiated refund, and in a system with automated action it would misreport the merchant's net shipping revenue as 287 when the true figure is 287 (A kept) with 713 returned.

### 3.2 Proof — wholly unshipped goods

- Order accepted, nothing shipped. Every unit `pending`. `Σ recognised = 0`.
- The customer cancels the entire order and the merchant refunds all 1,000 shipping.
- Recognised-only ceiling = **0** → every refund is blocked, permanently, for any order that has not yet shipped.
- That is not an edge case. It is the **normal** case for cancellation before dispatch, which is a large share of e-commerce cancellations.

### 3.3 Conclusion

Recognised revenue is a **timing-dependent subset** of what the customer paid. A refund ceiling must be based on what the customer **paid and has not yet had returned**, not on what the merchant has **recognised so far**. Any design that caps on recognised revenue fails the two most common cancellation paths.

---

## 4. Definitions

All cumulative, all in minor units, all per order.

| Symbol | Definition | Notes |
|---|---|---|
| `S_gross` | Original shipping charges as billed to the customer, before any discount | Sum of charge records; each carries an immutable provider identity |
| `D_order_time` | Shipping discount included in the order's own terms at acceptance | Netted before allocation; not an event |
| `S_net` | `S_gross − D_order_time` | The allocated source amount |
| `Chg_add` | Additional shipping charged **after** acceptance (re-ship, upgraded service) | Raises the ceiling; allocated to the units it serves |
| **`CB`** | **Collectible basis** = `S_net + Chg_add` | **The ceiling basis** |
| `R` | `Σ_u refunded_u + Rf_order` — cumulative shipping refunds issued | Refunds of shipping to the customer |
| `Disc` | `Σ_u discounted_u + D_post_order` — cumulative post-acceptance discounts and credits | Waivers, not cash returns |
| `alloc_u` | Unit u's immutable allocation share | From the allocation event |
| `alloc_c` | Charge c's amount, for charge-scoped caps | From the charge record |

**`R` and `Disc` are combined, not capped separately.** A refund and a waiver both reduce net shipping revenue, and the failure mode in §7.5 is precisely a refund followed by a waiver of the same amount. Capping them independently would let the merchant return the same money twice, once as cash and once as a credit.

---

## 5. The rule — R10

**R10.1 — Order ceiling (Tier 1, hard).**

```
R + Disc ≤ CB
```

evaluated after every candidate event. A candidate that would breach this **cannot be economically valid** and is quarantined with a hard alert. It is never partially accepted (§5.3).

**R10.2 — Unit ceiling (Tier 2).**

```
refunded_u + discounted_u ≤ alloc_u + Chg_add_u
```

for every unit u. A candidate that breaches this is quarantined for review, because a unit-scope breach can be a mis-scoped goodwill gesture, a duplicate under a different identity, or a provider defect — all plausible, none safe to auto-accept.

**R10.3 — Charge ceiling (Tier 2), where the basis is B-1 or B-2.**

```
refunds attributed to charge c + discounts attributed to charge c ≤ alloc_c
```

Charge-scoped attribution is available only when the association basis is B-1 or B-2 (P-1, R9). Under B-3 the charge ceiling cannot be evaluated and is skipped — which is one concrete reason B-3 is the inferior basis, and must be disclosed.

**R10.4 — Deterministic ordering.** The ceiling is applied to the event set **ordered by (`effective_date`, provider event identity)**, not by arrival order. The accepted set is therefore a function of the event set, not of the ingestion sequence. §7.7 shows why this matters.

**R10.5 — Monotonic.** `R` and `Disc` only increase. A refund reversal (a returned refund, or a refund that failed to settle) is recorded as a **reversal event** that reduces `R` and is itself auditable — it does not delete the original refund record.

**R10.6 — Fail closed on period close.** An order with any event in quarantine cannot be included in a closed reporting period. The period close for that order is blocked, not silently rounded.

**R10.7 — No silent partial acceptance.** If a 1,200 refund exceeds a 1,000 ceiling, the platform does not accept 1,000. Accepting a portion would conceal that the merchant attempted to return more than was collected — which is either a data error, a currency error, or a genuine business event that needs a human.

---

## 6. Event dispositions

Three terminal dispositions plus one non-event:

| Disposition | When | Financial effect | Visibility |
|---|---|---|---|
| **ACCEPTED** | Passes all applicable ceilings | Bucket/line updated | Normal reporting |
| **QUARANTINED** | Breaches a Tier 1 or Tier 2 ceiling; or an unresolved unit-scope ambiguity | **None** — not applied | Blocks period close for the order; appears in the review queue with the full ceiling arithmetic |
| **REJECTED** | Structurally invalid: negative amount, unknown order/charge/unit reference, amount not an integer minor unit, currency mismatch, malformed provider payload | **None** | Hard error to the integration layer; counted as a data-quality defect |
| **NO-OP** | Duplicate identity (INV-12) | **None** | Logged as a duplicate delivery, not a defect |

Quarantine is deliberately **not** rejection. A quarantined event is evidence that something needs a human decision; discarding it would destroy the evidence, and auto-accepting it would corrupt the ledger. It sits in between, visible and blocking.

---

## 7. The eight required tests

Each test states the ceiling arithmetic explicitly. **None of these has been executed** — they are test specifications.

### T1 — One shipping charge, one refund

- `CB = 1,000`. Order-scoped refund 200, effective 03-01.
- `R = 200`, `Disc = 0` → 200 ≤ 1,000 ✓
- **ACCEPTED.** `Rf_order = 200`. Reported net shipping revenue = 1,000 − 200 = **800**.

### T2 — Multiple charges, multiple refunds

- Charges: S1 = 600 (associated to L1), S2 = 400 (associated to L2). Basis B-1. `CB = 1,000`.
- Refund 300 scoped to S1; refund 200 scoped to S2.
- Order ceiling: `R = 500 ≤ 1,000` ✓
- Charge ceilings: 300 ≤ 600 ✓; 200 ≤ 400 ✓
- **Both ACCEPTED.** Reported = 1,000 − 500 = **500**.
- Note: charge ceilings are evaluated **per charge**, so a refund of 500 against S1 alone would breach the S1 cap (500 ≤ 600 is fine, but 700 would not) even though the order ceiling would still pass. This is the additional protection B-1/B-2 buys.

### T3 — Partial shipping refund

- `CB = 1,000`. Refund 250, unit-scoped to C (whose allocation is 426).
- Order: 250 ≤ 1,000 ✓. Unit: 250 ≤ 426 ✓
- **ACCEPTED.** C: `pending 426 → refunded 250 + pending 176`. Σ buckets = 1,000 ✓ (INV-31)

### T4 — Refund for goods that never ship

This is the case that distinguishes the correct ceiling from the plausible-but-wrong one.

- A ships 02-03 (287 recognised). B, C cancelled 02-10 → 287 + 426 = 713 `released`.
- Refund 713 for the two unshipped parcels.
- Order ceiling: `CB = 1,000`; `R = 713 ≤ 1,000` ✓ → **ACCEPTED**
- Recognised-only ceiling would have given `287`, and the refund would have been **wrongly blocked** (§3.1).
- After acceptance: `released` 713 → `refunded` 713. Buckets: A recognised 287, B refunded 287, C refunded 426. **Σ = 1,000** ✓
- Reported net shipping revenue = 287 − 0 = **287**. Correct: the merchant kept shipping revenue for the one parcel that shipped and returned the rest.

### T5 — Repeated refund requests

- Same provider refund identity delivered twice → second is **NO-OP** (INV-12). `R` stays at the first amount.
- Different identities, cumulative 600 then 500:
  - 600: 600 ≤ 1,000 ✓ ACCEPTED
  - 500: 600 + 500 = 1,100 > 1,000 → **QUARANTINED** (Tier 1)
- Reported remains 1,000 − 600 = **400**. The 500 is not partially accepted at 400.

### T6 — Duplicate provider events

- Provider sends refund event id `rf_8821` three times with byte-identical payloads, and once with a re-serialised but semantically identical payload.
- All four resolve to the same idempotency identity → one ACCEPTED, three NO-OP.
- `R` increases **once**. This tests INV-12's identity stability, not the ceiling.

### T7 — Refunds arriving out of order

- Two refunds: `X` = 300 effective 03-01; `Y` = 800 effective 03-05. `CB = 1,000`. Cumulative 1,100 > 1,000.
- **Arrival order Y then X** (a plausible replay sequence): R10.4 orders by (`effective_date`, identity) → X first (300, accepted), then Y (1,100 > 1,000, quarantined). Accepted total **300**.
- **Arrival order X then Y**: same ordering applied → identical outcome, accepted total **300**.
- The accepted set is now a function of the event set, not the arrival sequence. **This is the property R10.4 exists to guarantee.** Without it, the platform would accept 800 in one replay and 300 in another from identical evidence — the same defect class as the pooling/reallocation problems in P-1 and P-3.
- Reversing the effective dates (Y effective 03-01, X effective 03-05) legitimately changes the outcome to accepted 800, quarantined 300 — because a business fact changed, not because a delivery order changed.

### T8 — Refunds that exceed the remaining refundable balance

- `CB = 1,000`. Prior accepted refund 900. New refund 1,200.
- 900 + 1,200 = 2,100 > 1,000 → **QUARANTINED** (Tier 1), hard alert.
- Variant: prior 0, single refund 1,200 → 1,200 > 1,000 → **QUARANTINED**. Not accepted at 1,000 (R10.7).
- Boundary: refund exactly 1,000 with no prior refunds → 1,000 ≤ 1,000 → **ACCEPTED**. The ceiling is inclusive; an exact-collection refund is legitimate.

---

## 8. Interaction with P-3 discounts

The combined cap is the substance of §4's `R + Disc` decision. Concretely:

| Sequence | Arithmetic | Outcome |
|---|---|---|
| Refund 400, then discount 400 | `R + Disc = 800 ≤ 1,000` | Both accepted. Net revenue 200 |
| Refund 600, then discount 600 | `R + Disc = 1,200 > 1,000` | Discount quarantined. Net revenue 400 |
| Discount 426 scoped to C, then refund 426 scoped to C | Order: 852 ≤ 1,000 **passes** | Unit cap catches it: `refunded_C + discounted_C = 852 > alloc_C = 426` → **QUARANTINED** as a double economic event |
| Refund 1,000 order-scoped, then unit discount 100 on C | Order: 1,100 > 1,000 | Unit discount quarantined |

The third row is the reason R10.2 exists. **Order-level arithmetic alone would accept it**, because 852 is comfortably under 1,000 — and the merchant would have returned the same 426 twice, once as cash and once as a credit. A ceiling that only checks the order total is a ceiling with a hole exactly the shape of a unit.

---

## 9. Additional charges (`Chg_add`)

`Chg_add` raises `CB`, so it must be allocated before it can be refunded.

- Re-ship charge 300 raised 03-05 for unit B. `CB = 1,000 + 300 = 1,300`.
- A refund of 1,200 is now within the ceiling (1,200 ≤ 1,300) where it previously was not.
- The 300 is allocated to B under the same basis rules (P-1 R9), and B's unit cap becomes `alloc_B + 300 = 587`.
- Ordering matters: a refund that references a `Chg_add` which has not yet been ingested is quarantined, then **re-evaluated** when the charge arrives. Re-evaluation is a new decision recorded against the quarantined event; it is not an in-place edit of a prior decision.

---

## 10. Cross-document impact

| Document / section | Change required if P-4 is approved | Kind |
|---|---|---|
| D-10a R7 (shipping refunds) | Replace with R10.1–R10.7 | Substantive |
| D-10a R8 (per-charge caps, per the audit's P-5) | **Not editorial** — see §12 | Substantive |
| D-10a E7 (shipping refund) | Add the ceiling arithmetic explicitly | Substantive |
| D-10a C-10a.4 | Becomes satisfiable once R10 is applied and the tests exist | Status |
| INV-26 S2 / S4 | `released` and `refunded` transitions must respect R10 | Substantive |
| INV-27 §3 | No change; ceiling is order-scoped, not period-scoped | None |
| INV-12 | Refund identity must be stable across re-serialisation (T6) | Substantive |
| D-13 | `Rf_order` mirrors the order-level residual pattern; reference only | Reference |
| New invariant | **INV-30** (§11) | New |

---

## 11. New invariant proposed — INV-30

**Rule.** Cumulative post-acceptance shipping reductions, combining refunds and discounts across unit-scoped and order-scoped lines, may never exceed the collectible shipping basis `S_net + Chg_add`; unit-scoped and charge-scoped reductions may never exceed their own allocation; and the ceiling is applied to an event set deterministically ordered by effective date and provider identity so that the accepted set is independent of arrival order.

| Element | Specification |
|---|---|
| **Scope** | Every shipping refund, discount, credit and reversal, at order, unit and charge scope, in every calculation view |
| **Calculation basis** | `CB = S_net + Chg_add`; `R = Σ refunded_u + Rf_order`; `Disc = Σ discounted_u + D_post_order`; unit cap `alloc_u + Chg_add_u`; charge cap `alloc_c` where basis ∈ {B-1, B-2} |
| **Enforcement** | Ceiling evaluation before any bucket transition; `CB` and `alloc_u` immutable; event set ordered by (`effective_date`, provider identity); cumulative totals recomputed from the event set, never read from a mutable running balance |
| **Failure behaviour** | Tier 1 breach → quarantine + hard alert + period close blocked for the order. Tier 2 breach → quarantine + review queue. Structurally invalid → reject at the integration boundary. Duplicate → no-op. **Never partial acceptance, never silent acceptance** |
| **Test cases** | T1–T8 (§7). Specifically T4 proves the basis is correct and T7 proves determinism; T8 boundary proves inclusivity; §8 row 3 proves the unit cap is load-bearing |

INV-30 is a **proposal**; numbering requires founder approval.

---

## 12. Scope warning — the audit's "P-5" is not editorial

The audit listed a correction labelled **P-5** among items it described as editorial. Its exact wording is:

> **P-5** | D-05 | Restate R8 per charge where R5 applies, with the order-level identity as the aggregate check. | Brief §2 R8

This brief records a disagreement:

- Applying the per-charge cap **changes the set of events that are accepted**. Under an order-only cap, a 700 refund against charge S1 (600) is accepted (700 ≤ 1,000); under a per-charge cap it is quarantined.
- A change that alters which financial events are applied is, by the founder's own Part D boundary — "Do not make any change that alters financial meaning, allocation behaviour, event timing, calculation identity or approval scope under the label of editorial correction" — **substantive**.
- It therefore belongs in this brief (R10.3) and requires founder approval, not an editorial change set.

**P-5 has consequently been excluded from the editorial diff** in `d10a-editorial-change-set-proposed-diff.md`. This is a deliberate scope correction, not an omission. It is covered here as R10.3 and requires founder approval.

### 12.1 Two deliberate deviations from the audit's own P-4 sketch

For completeness, this brief does not adopt the audit's P-4 wording as written:

| Audit P-4 wording | This brief | Why |
|---|---|---|
| "cumulative shipping refunds ... may not exceed `S_net`" | Ceiling is `CB = S_net + Chg_add` | `Chg_add` is money the customer paid for shipping after acceptance. Excluding it would make a re-ship charge permanently unrefundable. §9 |
| "the order-level residual by `S_net − Σ attributable refunded`" | Order-scoped lines capped through the **combined** `R + Disc` order ceiling, with order-scoped amounts excluded from the unit partition | A residual-minus-attributable formulation caps refunds only; it leaves a post-fulfilment *discount* of the same amount uncapped, which is the double-return hole in §8 row 3 |

Both deviations are proposed, not applied. The founder may prefer the audit's narrower `S_net` basis; if so, `Chg_add` needs its own separately named ceiling and the double-return case needs a different guard.

---

## 13. Unresolved questions

| ID | Question | Classification |
|---|---|---|
| OQ-P4-1 | Is a shipping refund that exceeds collected shipping ever legitimate — e.g. a goodwill gesture that includes a service component? | Founder policy. This brief proposes: quarantine, never auto-accept; a goodwill payment above collected shipping should be booked as a separate cost line, not negative shipping revenue |
| OQ-P4-2 | Should a Tier 1 breach block the *merchant's whole period close*, or only that order's inclusion? | Founder policy. This brief proposes: only that order is excluded and flagged; the period may close with the order explicitly excluded and disclosed |
| OQ-P4-3 | Can the target provider supply a stable refund identity across re-serialisation (T6)? | **Provider verification — P-01, P-02.** If not, identity must be derived from a digest of normalised financial fields, which is weaker |
| OQ-P4-4 | Does the shipping provider expose refund state as *pending* versus *completed*, and must pending refunds count toward the ceiling? | **Provider verification — P-04.** Founder direction already states a pending refund is not a completed supplier credit; this brief proposes pending refunds **do** count toward the ceiling (conservative), with the distinction preserved in the event record |
| OQ-P4-5 | Is a refund reversal (returned/failed refund) a reduction of `R` or a separate negative refund line? | This brief proposes a reversal event that reduces `R` while preserving the original — equivalent arithmetically, materially different for auditability |
| OQ-P4-6 | Where `Chg_add` is allocated under B-3, does the unit cap use the pooled share or the full charge? | This brief proposes the pooled share, consistent with the basis |

---

## 14. Recommendation, separated from the proposed decision

### 14.1 AppJet's recommendation

Adopt **R10.1–R10.7** with the collectible basis `CB = S_net + Chg_add`, the combined `R + Disc` cap, unit and charge ceilings, deterministic ordering by effective date and provider identity, and the three-way disposition (accepted / quarantined / rejected) plus no-op.

The recommendation rests on four claims, each independently checkable:

1. **`CB` is the only basis that survives the two cancellation proofs.** Recognised revenue fails both (§3.1, §3.2). No other candidate was found that does not reduce to `CB`.
2. **Combining refunds and discounts is necessary.** Capping them separately lets the same 426 be returned twice (§8, row 3) while every individual cap reads as satisfied.
3. **The unit ceiling is load-bearing, not defensive.** Order-level arithmetic accepts the 852-against-426 double return.
4. **Deterministic ordering converts an order-dependent outcome into a reproducible one.** Without R10.4, identical evidence yields different accepted sets depending on webhook arrival — the same class of defect as D-01 and D-03.

The honest weakness: **quarantine is a queue, and a queue needs an operator.** Tier 1 and Tier 2 breaches produce work for a human, and in a lean MVP that human is the merchant. The alternative — auto-rejecting breaches — is cheaper but destroys evidence and would misclassify the legitimate refund in T4 as fraud-adjacent. The founder should choose quarantine knowing it costs operational effort.

### 14.2 What the founder is being asked to decide

- **Approve** R10 as specified (CB basis, combined cap, unit and charge ceilings, deterministic ordering, quarantine-not-reject);
- **Approve with conditions** — most likely on OQ-P4-1 (goodwill above collected shipping) or OQ-P4-2 (period-close blocking scope);
- **Defer**, in which case C-10a.1 and C-10a.4 stay open, D-10a continues to block G3, and the platform has no defined refund ceiling — which is a state in which no shipping refund may safely be ingested.

OQ-P4-3 and OQ-P4-4 remain **provider verification** items. No part of this brief has been applied, and no test has been executed.

---

**Prepared:** 2026-11-20 · **Branch:** appjet · **Implementation authorisation:** not granted · **Supabase:** not connected · **Tests executed:** none
