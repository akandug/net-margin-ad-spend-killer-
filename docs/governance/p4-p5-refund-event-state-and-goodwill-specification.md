# P-4 / P-5 — Shipping Reduction Control: Event State, Goodwill Classification and Deferral Window

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Derived from:** `p4-p5-shipping-reduction-control-combined-brief-rev2.md` (rev 2)
**Responds to:** founder decision round 2026-11-20 §5, §6, §7
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not implemented, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (OPEN) · INV-26 · INV-27

---

## 1. The problem this control solves

A refund of shipping money is an external fact: the processor moved money. Whether that refund was *legitimate* depends on charges that may not have been observed yet. Any control that decides legitimacy at the moment the refund arrives will misclassify refunds that arrive before their charge — which, given unordered duplicate-capable webhook delivery (P-02), is routine rather than exceptional.

The design therefore separates **what happened** from **what it means**.

---

## 2. Two layers, deliberately not one

### 2.1 Layer 1 — immutable event records

Two event families are recorded as observed, with both timestamps:

| Event | Carries |
|---|---|
| `charge_observed` | charge identity, amount, scope, `provider_ts`, `ingestion_ts`, stable source identity, evidence ref |
| `reduction_observed` | reduction type, amount, scope, `provider_ts`, `ingestion_ts`, stable source identity, evidence ref |

Neither is ever updated. A later reversal is a new `reduction_observed` of type `correction`, not an edit.

### 2.2 Layer 2 — derived reconciliation status

Status is **not stored as mutable state**. It is a pure function of the complete event set:

```
status(reduction) = f( event set, balance state )
```

Because `f` depends only on the *set* of events and not on the order they were processed, status is **order-independent by construction**. This is the answer to your requirement that "all calculations remain order-independent without losing event chronology":

- **Order-independence** comes from `f` being a set function.
- **Chronology** is preserved on the events themselves via `provider_ts`.
- Neither property is obtained at the expense of the other, because they live in different places.

Storing status as a mutable column would destroy this: it would be order-dependent (the last writer wins) and it would be unverifiable (no way to recompute it).

---

## 3. The six states

| State | Layer | Entry condition | Financial effect |
|---|---|---|---|
| `money_moved` | **Event fact** | the refund exists in provider evidence | Always recorded. Never suppressed, never conditional |
| `expected_authorised` | Derived | an authorised refund expectation exists for this scope | None yet — an expectation is not money |
| `matched` | Derived | a reduction is paired with an observed charge covering its scope, and the balance permits it | Counted against CB; legitimate |
| `unmatched_pending_charge` | Derived | the reduction's scope has no observed charge, or CB is `provisional_incomplete` | **Recorded, not counted as excess.** Awaits evidence |
| `confirmed_excess` | Derived | CB is `final` **and** cumulative matched reductions exceed the applicable ceiling | Classified, surfaced, **never silently discarded** |
| `duplicate_suppressed` | Derived | the same stable source identity has already been ingested | Counted once. The duplicate is retained as evidence of the duplicate |

**`money_moved` and the derived states coexist.** A refund can be `money_moved` and `unmatched_pending_charge` simultaneously — the money did move, and we cannot yet say whether it was within the ceiling. Collapsing these into one label is what makes naive implementations treat an early refund as fraud.

### 3.1 The five states in your §5, mapped

| Your label | Where it lives here |
|---|---|
| Actual external money movement | `money_moved` (event fact) |
| Expected or authorised refund | `expected_authorised` (derived, non-financial) |
| Matched refund | `matched` |
| Unmatched refund awaiting charge evidence | `unmatched_pending_charge` |
| Confirmed over-ceiling refund | `confirmed_excess` |
| Duplicate provider event | `duplicate_suppressed` |

---

## 4. Balances

### 4.1 Order-level collectible balance

```
CB(order) = S_net + Σ additional_charges − Σ charge_corrections
```

`S_net` is the shipping revenue allocated at acceptance (D-10a). `additional_charges` are post-acceptance shipping charges. `charge_corrections` reduce the charge.

**`CB` is monotone within a completeness state.** It never grows by refunding and never shrinks by recognising. Reductions are compared *against* it; they do not modify it.

### 4.2 CB completeness — the fix that makes the control correct

```
CB_status ∈ { provisional_incomplete, final }
```

`CB_status = provisional_incomplete` while any charge the order is expected to carry has not been observed, or while ingestion is within its provider retry envelope. `final` once all expected charges are observed and the retry envelope has closed.

**This is the single most important addition to rev 2.** Without it, CB for an order whose charges have not yet arrived is `0`, and every refund against that order is arithmetically an excess. With it, CB is *unknown*, and unknown is not zero.

### 4.3 The ceiling rule

```
excess(r) is confirmed  ⟺  CB_status = final
                        AND  Σ matched reductions (order scope) > CB
                        AND  r's own scope balance is exhausted
```

A reduction may only be classified `confirmed_excess` when **both** conditions hold. Otherwise it is `unmatched_pending_charge`.

### 4.4 Per-charge and per-unit balances

```
refundable(charge c) = c.amount − Σ matched reductions scoped to c
refundable(unit u)   = A(u) − Σ matched reductions scoped to u
```

Unit-scoped refundable may not go negative; a reduction exceeding a unit's allocation spills to the order scope rather than being forced onto the unit. This is the unit cap verified as CX-5 (excess 213 in that example).

---

## 5. Transitions

```
reduction_observed
        │
        ├─ duplicate source identity? ──▶ duplicate_suppressed (retained, not counted)
        │
        ├─ scope charge observed AND CB_status = final
        │        ├─ within balance ──▶ matched
        │        └─ over balance   ──▶ confirmed_excess
        │
        └─ scope charge unobserved OR CB_status = provisional_incomplete
                 └─▶ unmatched_pending_charge
                          │  (later: charge_observed arrives, CB → final)
                          ├─ within balance ──▶ matched
                          └─ over balance   ──▶ confirmed_excess
```

`unmatched_pending_charge` → `confirmed_excess` is **permitted only after evidence arrives**. It is never reached by the passage of time (§8).

---

## 6. Counterexamples that break a naive formula

These are constructed deliberately to attack the design, as you required.

**Setup for all:** `CB = 1,000`, allocation 287 / 287 / 426 to units A / B / C.

### CX-A — Refund arrives before its charge

Charge `c1 = 1,000` not yet observed. A shipping refund of 200 arrives.

| Formula | CB | Verdict | Correct? |
|---|---|---|---|
| Naive (`refund > CB ⟹ excess`) | 0 | `confirmed_excess 200` | **No** — the charge exists; we have not seen it |
| Specified | `provisional_incomplete` | `unmatched_pending_charge` | **Yes** |

When `c1` is observed: CB = 1,000, refund 200 ≤ 1,000 → `matched`. No merchant is ever told they over-refunded because of webhook ordering.

### CX-B — Charge correction reduces CB below refunds already matched

CB = 1,000. Refund 600 matched. A `charge_correction` reduces `c1` to 500, so CB = 500.

Cumulative matched 600 > CB 500 → `confirmed_excess` of **100**, surfaced with the correction as its cause.

**The design must not clamp the 600 down to 500.** Clamping would silently discard 100 of real money movement, which is precisely the double-counting/discarding failure INV-19 and the unallocated-spend rule forbid. The 100 is an evidenced discrepancy requiring investigation, and it stays visible.

### CX-C — Duplicate provider event

The same refund (stable identity `R-77`, amount 400) is delivered twice.

| Handling | Cumulative refunds | Verdict |
|---|---|---|
| No dedupe | 800 | `confirmed_excess 400` — **false** |
| Dedupe on stable source identity | 400 | `matched` — correct |

Both deliveries are retained; the second is `duplicate_suppressed`. The duplicate is evidence *that* the provider duplicated, which is operationally useful and must not be deleted.

### CX-D — Two distinct refunds, identical `provider_ts`, delivered in reverse order

Refunds 300 and 200, both timestamped 12:00:00, identities `R-91` and `R-92`. Total 500 ≤ CB 1,000.

Delivered in either order, the derived status is `matched` for both, because status is a function of the set. Under mutable last-writer state, the 200 processed first might consume a scope balance and the 300 be flagged. **Order-independence is not a convenience here; it is what makes the same evidence produce the same answer twice.**

### CX-E — Legitimate refund exceeding recognised revenue

Units B and C never ship. Only A's 287 is recognised. The customer is refunded **713** for shipping on goods that never shipped.

| Ceiling basis | Ceiling | Refund 713 | Verdict |
|---|---|---|---|
| Recognised revenue | 287 | exceeds | **Wrongly rejected** |
| `CB` | 1,000 | within | `matched` — correct |

**This is the counterexample to your own earlier caution**, and it confirms it: recognised shipping revenue cannot be the ceiling. Money was legitimately collected for shipping that was never provided; refusing to refund it because the revenue was never recognised would block the correct accounting outcome. The ceiling is the **collectible balance**, not the recognised amount.

### CX-F — Additional charge after a full refund

CB = 1,000, full refund 1,000 matched. Then an additional charge of 200 is observed → CB = 1,200.

`refundable = 1,200 − 1,000 = 200`. Not an excess; not a new refund. The balance is simply available again.

### CX-G — Two reductions on the same unit, exceeding that unit's allocation

Unit C allocation 426. `shipping_refund` 400 and `shipping_discount` 100 → 500 > 426.

Unit scope exhausted at 426; the residual 74 spills to the order scope. CB = 1,000, matched order-scope total 500 ≤ 1,000 → both `matched`. **No negative unit balance is created**, and no reduction is lost.

---

## 7. Goodwill and free-shipping classifications (CX-1)

### 7.1 Separate event types

| Type | Nature | Ceiling applies? |
|---|---|---|
| `shipping_refund` | return of shipping money actually collected | **Yes** — CB |
| `delivery_goodwill` | payment to a customer for a poor experience | **No** |
| `fulfilment_compensation` | payment for a fulfilment failure (late, damaged, lost) | **No** |
| `other_service_adjustment` | other customer-service credit | **No** |

### 7.2 The CX-1 failure

Order with free shipping (`CB = 0`). The merchant pays a 300 goodwill credit.

| Treatment | Result |
|---|---|
| Misclassified as `shipping_refund` | 300 > CB 0 → `confirmed_excess 300`. **The merchant is accused of over-refunding money they never collected** |
| Correctly classified as `delivery_goodwill` | Recorded as an operating cost line of 300. Ceiling not applied |

The arithmetic of contribution profit is the same (−300), but the classification is not: one is a refund against collected revenue, the other is a cost. Reporting them identically would overstate refund rates, understate operating costs, and corrupt any refund-rate metric the merchant is shown.

### 7.3 Technical treatment versus accounting classification

| Question | Status |
|---|---|
| Which event type is it? | **Technical, decided here** — by the evidence of what the payment was for |
| Does the ceiling apply? | **Technical** — no, for non-refund types |
| Is goodwill revenue-reducing or an expense? | **Professional accounting review** — not decided here |
| Is it deductible, and in which period? | **Professional accounting review** |

The system records the classification and its evidence. It does not assert the statutory treatment.

---

## 8. The deferral window (CX-4)

### 8.1 What the window controls

**Operationally only:** how long an `unmatched_pending_charge` reduction stays quiet before it is escalated for investigation.

It controls nothing financial. This is the direct answer to your §7 question — **window expiry changes investigation status, never financial classification.**

### 8.2 Behaviour

| Phase | State | Visible to merchant? |
|---|---|---|
| Within window | `unmatched_pending_charge` | Not as an error; the reduction is counted as money moved |
| Window expired, still unmatched | `unmatched_pending_charge` + `investigation_due` | Surfaced as a data-quality flag |
| Evidence arrives later | `matched` or `confirmed_excess` on the evidence | Normal |

**Expiry never produces `confirmed_excess`.** If it did, the window would be a financial rule wearing an operational costume, and a slow webhook would become a merchant accusation.

### 8.3 Policy, not a fixed global timeout

A single global duration is the wrong shape, because the correct value is a property of the provider, not of us:

- P-02 verified that Shopify retries up to eight times over four hours and may stop retrying after that limit.
- A window shorter than the provider's retry envelope guarantees false investigations.
- A window far longer delays genuine reconciliation.

**Recommended shape:** a per-provider, configurable `reconciliation_window` policy object, with the effective default derived from the provider's documented retry envelope plus a margin, and a mandatory reconciliation job that closes CB to `final` after the envelope.

**No number is chosen here.** Setting a production default requires evidence we do not have:

| Evidence needed | Source |
|---|---|
| Provider retry envelope | P-02 (obtained for Shopify: 8 retries / 4 h) |
| Observed delivery latency distribution in pilot | Pilot telemetry — not yet available |
| Whether missed events are recoverable by reconciliation | P-02 confirms reconciliation is mandatory; recovery rate unmeasured |

Selecting an arbitrary timeout now would manufacture a policy from intuition, which is what you instructed against.

---

## 9. Failure behaviour

| Condition | Behaviour |
|---|---|
| CB_status unknown | Treated as `provisional_incomplete`. Never as `final` |
| Charge scope unresolvable | Reduction → `unmatched_pending_charge` |
| Duplicate identity with differing amount | **Both retained**, `duplicate_conflict` flag, investigated. Not resolved by preferring either |
| Reduction exceeds unit allocation | Spills to order scope; unit balance never negative |
| Reduction type not in the closed set | Rejected at ingestion, quarantined with evidence. Never coerced to `shipping_refund` |
| CB `final` and matched > CB | `confirmed_excess` with the cause recorded |
| Any balance computation failure | Fails closed — no figure shown |

---

## 10. MVP versus pilot

| Component | Classification |
|---|---|
| Event records with `provider_ts` + `ingestion_ts` | **MVP** |
| Derived status as a pure function (not a mutable column) | **MVP** |
| Stable source identity + duplicate suppression | **MVP** |
| `CB` with completeness status | **MVP** |
| `unmatched_pending_charge` — never auto-excess | **MVP** |
| `confirmed_excess` classification, surfaced not discarded | **MVP** |
| Separate goodwill/compensation event types | **MVP** (classification only; the merchant UI is pilot) |
| Configurable per-provider window policy object | **MVP** (object exists; default value set at pilot from evidence) |
| Investigation workflow and escalation UI | **Pilot** |
| Automated reconciliation job closing CB to `final` | **MVP** — without it CB never finalises and excess is never confirmable |

---

## 11. Open questions

| ID | Question | Needs |
|---|---|---|
| OQ-P4-1 | Which provider payload fields establish charge identity for `charge_observed`? | P-01/P-04 verification |
| OQ-P4-2 | Is a charge correction distinguishable from a new charge in provider evidence? | Provider docs |
| OQ-P4-3 | Production default for `reconciliation_window` per provider | Pilot telemetry |
| OQ-P4-4 | Is `confirmed_excess` merchant-visible, internal-only, or both? | **Founder decision** |
| OQ-P4-5 | Goodwill classification: revenue-reducing or expense? | **Professional accounting review** |
| OQ-P4-6 | Should duplicate-with-differing-amount fail closed entirely, or retain both? | **Founder decision** |

---

## 12. Recommendation

Adopt the two-layer design: immutable event records carrying both timestamps, and reconciliation status as a pure function of the event set. Adopt `CB` with explicit completeness status, and the rule that a reduction may only be `confirmed_excess` when `CB_status = final`. Keep goodwill and compensation as separate event types with no ceiling. Make the deferral window an operational, per-provider, configurable policy that never alters financial classification.

The load-bearing rule is §4.2. Everything else follows from refusing to treat an unknown balance as zero.

**This is a recommendation, not an approved rule.** D-10a and INV-26 are unchanged; C-10a.1 remains open. No test has been executed.
