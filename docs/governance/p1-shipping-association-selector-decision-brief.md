# P-1 — Shipping Association Selector

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Decision package:** D-10a substantive correction review, Part B
**Defect addressed:** D-01 (association-basis selector unspecified and absent from calculation identity)
**Status of this brief:** PROPOSED FOR FOUNDER DECISION — **not applied, not approved, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS — remains AWC)
**Affected condition:** C-10a.1 (PARTIALLY SATISFIED — OPEN), C-10a.2, C-10a.3
**Authority:** founder instruction 2026-11-20 (third review), Part B, P-1

---

## 1. The defect

D-10a contains two allocation rules that can both apply to the same order, and nothing in the specification chooses between them:

- **R1–R3** — allocate the original shipping revenue across eligible quantities, weighted by discounted line net revenue, largest remainder resolved once across all units. This is *pooled* allocation: the shipping charge is treated as one indivisible order-level amount.
- **R5** — where a shipping charge is directly associated with specific quantities, allocate that charge over its own associated quantity set.

E5 in the D-10a brief shows the two rules producing materially different product-level answers on the same order, and then observes that "direct association is more faithful because the 400 was incurred for B's parcel specifically". That observation is correct, but it is a *comment*, not a rule. The specification never states:

1. what evidence makes R5 applicable rather than R1–R3;
2. at what moment that determination is made and when it stops being revisable;
3. what happens when association evidence arrives after the allocation has been written;
4. how a figure produced under one basis is distinguished from a figure produced under the other.

Without those four answers the same order can yield two different product-level shipping allocations depending on which importer path ran, and C-10a.1 ("the same inputs cannot yield different allocations across documents") cannot be satisfied. That is D-01.

**This is a real financial risk, not a documentation nicety.** The pooled and direct bases differ by 13 to 26 minor units per unit on a 1,000-unit order — roughly 4–6% of each unit's shipping allocation. Shipping allocation feeds product-level contribution profit, and product-level contribution profit is what the Co-Pilot gate and the pause recommendations act on. A basis that shifts with ingestion order is a basis that makes product-level profitability non-reproducible.

---

## 2. The magnitude of the difference

Test order **P1-O1**, used throughout this brief:

| Unit | Line | Gross | Line discount | Discount per unit | Net per unit |
|---|---|---|---|---|---|
| A | L1 | 3,000 | 600 (2 units) | 300 | **2,700** |
| B | L1 | 3,000 | — | 300 | **2,700** |
| C | L2 | 4,000 | — | — | **4,000** |
| | | | | Σbase | **9,400** |

Shipping: charge **S1 = 600** associated by the provider with line L1; charge **S2 = 400** associated with line L2. **S_net = 1,000.**

**Pooled basis (R1–R3), largest remainder resolved once:**

```
A: 1,000 × 2,700 / 9,400 = 287.2340 → 287
B: 1,000 × 2,700 / 9,400 = 287.2340 → 287
C: 1,000 × 4,000 / 9,400 = 425.5319 → 425
                    floors = 999, remainder 1
fractions: C .5319 highest → C takes the extra unit
```

| Unit | Pooled | Direct | Difference |
|---|---|---|---|
| A | 287 | 300 | **+13** |
| B | 287 | 300 | **+13** |
| C | 426 | 400 | **−26** |
| Σ | **1,000** | **1,000** | **0** |

Both bases conserve the total exactly. The difference is purely a mix shift between products — which is precisely why it changes product-level profitability while leaving order-level profitability correct, and precisely why it must be deterministic rather than incidental.

Direct basis arithmetic: S1 600 over A and B, weight 2,700 each, Σ 5,400 → 600 × 2,700/5,400 = 300 each, exact. S2 400 over C alone → 400, exact. No rounding arises in this case; §7 shows the case where it does.

---

## 3. Proposed rule R9 — the association basis ladder

**R9.1** The association basis is selected by a deterministic ladder evaluated against the evidence visible at the **allocation instant** (§5). Exactly one basis is selected per order.

| Basis | Selected when | Allocation method |
|---|---|---|
| **B-1 — Full direct association** | Every shipping charge on the order carries a provider-exposed association to a specific set of order lines or quantities | Each charge allocated over **its own** associated quantity set by discounted line net revenue, largest remainder applied **per charge** |
| **B-2 — Partial association** | Some charges are associated, at least one is not | Associated charges allocated as in B-1; the **unassociated charges pooled together** across all eligible quantities by R1–R3 |
| **B-3 — Pooled** | No charge carries an association | R1–R3 applied to the total shipping revenue |

**R9.2** A charge is "associated" only on the strength of a **named provider field or provider object** that links it to order lines or quantities — for example a per-line shipping price on the order line, or a shipment-level charge linked to a known quantity set. Inference is not association. In particular the following are **not** association evidence: a single charge on a single-line order (that is a degenerate pooled case, see R9.4), charge ordering, charge description text, or the merchant's own recollection.

**R9.3** B-1 requires **every** charge to be associated. One unassociated charge demotes the order to B-2. This is deliberate: it prevents a partially-evidenced order from being presented as fully evidenced.

**R9.4** On an order with exactly one eligible quantity, B-1, B-2 and B-3 all produce the same allocation (the whole S_net on that unit). The basis is still recorded, because the order may later be split or partially refunded and the recorded basis is what makes the historical figure reproducible.

**R9.5** The selected basis, the evidence that justified it, and a digest of that evidence are written **into the allocation event itself**. A basis that is not recorded did not happen.

### 3.1 B-2 worked example

P1-O1, but only S1 carries an association (to L1). S2 (400) is unassociated.

```
S1 (associated → L1):  600 over A, B  → 300 each (exact)

S2 (pooled, 400 over A,B,C by net revenue, Σbase 9,400):
  A: 400 × 2,700 / 9,400 = 114.8936 → 114
  B: 400 × 2,700 / 9,400 = 114.8936 → 114
  C: 400 × 4,000 / 9,400 = 170.2128 → 170
                  floors = 398, remainder 2
  fractions: A .8936, B .8936, C .2128 → A and B take the two extra units
  → 115 / 115 / 170, Σ 400 ✓
```

| Unit | S1 share | S2 share | Total |
|---|---|---|---|
| A | 300 | 115 | **415** |
| B | 300 | 115 | **415** |
| C | 0 | 170 | **170** |
| Σ | 600 | 400 | **1,000** ✓ |

The largest-remainder step runs **within each charge**, not across the order. Running it once across the order would let the S1 allocation absorb a rounding unit belonging to S2 and vice versa, which reintroduces the dependency on charge ordering that D-10a exists to remove.

---

## 4. Information required to select the basis

| # | Input | Required? | Source | Consequence if absent |
|---|---|---|---|---|
| I-1 | The set of shipping charges on the order, each with an immutable provider charge identity | **Required** | Order ingestion | Order cannot be allocated; fail closed, quarantine the order |
| I-2 | For each charge, the associated line or quantity set, **or** an explicit "no association exposed" marker | **Required** | Provider payload | Treated as unassociated → B-2 or B-3. Silence is never read as association |
| I-3 | Net revenue per unit (line gross less allocated discount) | **Required** | D-08 discount allocation | Order cannot be allocated; fail closed |
| I-4 | Eligibility per unit (cancelled, free) | **Required** | Order state at acceptance | Order cannot be allocated; fail closed |
| I-5 | Provider API version and the field path used for I-2 | **Required** | Integration metadata | Basis recorded as `unverified_field`; allocation still runs but carries a data-quality flag |

I-2 deserves emphasis. The distinction between "the provider says this charge has no line association" and "we did not look" is the difference between a B-3 order and an unknown order. Only the former may be allocated as B-3. **P-01/P-04 provider verification is what establishes whether I-2 can be populated at all** — until that evidence exists, B-1 and B-2 are unreachable in practice and every order is B-3, which is a legitimate but explicitly inferior state that must be disclosed to the merchant.

---

## 5. The allocation instant — when the basis becomes fixed

**R9.6** The **allocation instant** is the database transaction that commits the order-acceptance event together with its allocation events. The basis is evaluated against the evidence snapshot visible inside that transaction, and against no later evidence.

This is deliberately narrower than "when the importer ran" and narrower than "when the order was first seen". Two consequences:

- A retry, replay or re-import of the same order **cannot** change the basis, because the order-acceptance event already exists and allocation is once-only (C-10a.2).
- A late-arriving association cannot change the basis, because it did not exist at the allocation instant.

**Why the allocation instant and not "when the first shipment happens"?** Because deferring the decision to first shipment would make the allocation depend on fulfilment timing — the same defect the founder rejected when the residual-on-final-shipment rule was withdrawn. The basis must be fixed by order facts, not by logistics.

---

## 6. Immutability of the basis

**R9.7** Once written, the association basis on an allocation event is immutable. It is part of an append-only record (INV-19) and is not a field any application role may update.

**R9.8** An order receives exactly one allocation per charge. A second allocation event for the same order and charge is a duplicate and is rejected by the idempotency identity (INV-12), not merged and not averaged.

**R9.9** There is no "reallocation" operation in the system. This is a deliberate absence, not an oversight. If a reallocation were permitted, every figure derived from the superseded allocation would become underivable, and C-10a.3 ("each quantity's share is carried as a distinct identity that survives shipment, cancellation, refund and partial fulfilment without reallocation") would be violated by construction.

---

## 7. Late-arriving association evidence — the six cases

The founder's instruction is explicit: **do not assume that late-arriving evidence automatically authorises reallocation.** This section therefore treats late evidence as a *quality* input, never as an *allocation* input.

Two event types are introduced, both non-financial and both append-only:

| Event | Meaning | Financial effect |
|---|---|---|
| `association_evidence_arrived` | Association evidence for a charge now exists that did not exist at the allocation instant | **None.** Annotates the allocation; raises a data-quality flag |
| `association_evidence_invalidated` | Evidence that justified a B-1/B-2 basis is shown to be wrong | **None.** Annotates; raises a data-quality flag to the highest severity |

Neither event alters a bucket, a recognised amount, or a reported figure. They change only what the system *says about* the figure.

### Case 1 — Association available before order acceptance

- Basis: **B-1**. Evidence complete for S1 and S2.
- Allocation: A 300, B 300, C 400. Σ 1,000 ✓
- Recorded: `basis = B-1`, evidence refs for both charges, evidence digest.
- Flags: none. Product-level shipping allocation is **actual** (directly evidenced), not estimated.
- Audit trail: allocation event → two charge-level allocation records → per-unit shares, each carrying the charge identity it came from.

### Case 2 — No association available at acceptance

- Basis: **B-3**. No charge carries an association.
- Allocation: A 287, B 287, C 426. Σ 1,000 ✓
- Recorded: `basis = B-3`, evidence ref `null`, `association_absent_reason = provider_exposes_none`.
- Flags: `basis_pooled` (informational, permanent).
- Financial character: product-level shipping allocation is an **estimate produced by a deterministic rule**, not a directly evidenced cost. It must be labelled as such wherever product-level contribution profit is displayed, and the Co-Pilot data-quality gate must treat it as a reduced-confidence input.

### Case 3 — Association arrives after acceptance, before any shipment

- Basis at allocation instant: **B-3** (287/287/426). This does not change.
- Late evidence: S1 600 → L1, S2 400 → L2, which under B-1 would give 300/300/400.
- Recorded: `association_evidence_arrived` at 2026-02-01, with the evidence and digest.
- Flags: `basis_pooled` → `basis_pooled_contested`.
- Figures: **unchanged.** A 287, B 287, C 426.
- Maximum product-level deviation from the better-evidenced basis: **26** on C (426 vs 400), **13** on A and B.
- Order-level total: correct either way (1,000). Order-level contribution profit is therefore **unaffected**.

**Why not reallocate here, when nothing has shipped?** This is the case that looks most tempting, and it is the case where the temptation should be resisted. "Nothing has shipped" does not mean "nothing has been consumed". The allocation may already have been:
- included in an order-date Commercial Management View profitability report (F-1 approved this view);
- used to produce a Co-Pilot recommendation, an alert, or an emailed weekly summary;
- exported by the merchant.

Reallocating would retroactively change a figure the merchant may already have acted on, while leaving the order-level total identical — the worst combination, because the change is invisible in the total and visible only in the product mix. The disclosure path is honest; the silent-correction path is not.

### Case 4 — Association arrives after partial shipment

- Unit A ships 2026-02-03, recognising its B-3 allocation of **287**.
- Late evidence arrives 2026-02-10 (B-1 would be 300).
- Recorded: `association_evidence_arrived`; flag `basis_pooled_contested`.
- Figures: A stays 287. B 287 and C 426 remain pending, unchanged.
- If reallocation were permitted, A's recognised revenue would move 287 → 300, a **+13** change to an already-recognised figure. That requires a reversal event against a closed recognition, i.e. exactly the history-rewriting D-10a exists to prevent. Rejected.

### Case 5 — Association arrives after full shipment

- All three units shipped: 287 + 287 + 426 = 1,000 recognised.
- Late evidence arrives 2026-03-20.
- Recorded: `association_evidence_arrived`; flag raised.
- Figures: unchanged. Order-level CP unchanged. Product-level figures remain as allocated and are now **permanently** divergent from the better-evidenced basis by up to 26 on C.
- Consequence for the product: any historical product-level profitability report for P2 (unit C) overstates its shipping allocation by 26 and therefore understates its contribution profit by 26. That understatement is disclosed, not corrected.

### Case 6 — The original association is later found to be incorrect

This case is materially different from cases 3–5 and must not be conflated with them. Here the basis was **B-1** and the *evidence itself was wrong*.

- Recorded at acceptance: S1 600 → L1 (A, B); S2 400 → L2 (C). Allocation A 300, B 300, C 400.
- Later finding: the provider's association was reversed — S1 600 actually belonged to L2, S2 400 to L1.
- Correct evidence would have given: 400 over A and B → 200 each; 600 over C → 600. So A 200, B 200, C 600.
- Deviation from recorded: A **−100**, B **−100**, C **+200**. Σ 0 ✓ — again a pure mix shift.

Treatment:

| Aspect | Decision |
|---|---|
| Financial adjustment? | **None.** Allocation unchanged at 300/300/400 |
| Recorded event | `association_evidence_invalidated` with the correcting evidence, digest, source and both timestamps |
| Flag | `basis_evidence_invalidated` — highest severity of the three |
| Co-Pilot effect | **Blocks** automated recommendations for the affected products until the merchant resolves or explicitly overrides. A wrong association basis is a data-quality failure, and D-05/data-quality gating already forbids automated action on unreliable data |
| Provider restatement | If the provider later restates the association, that is captured as a further annotation. It still does not reallocate |
| Merchant remedy | The merchant may commission an **alternative-basis calculation** — a separate, explicitly labelled what-if artifact. It is never a replacement for the canonical figure (§8) |

The critical point: an invalidated basis makes the product-level figure **less trustworthy**, and the correct response to a less trustworthy figure is to *stop acting on it*, not to silently replace it with a different one. Replacing it would destroy the audit trail of what the merchant was told and when.

---

## 8. Representation in the calculation identity

**R9.10** `shipping_allocation_basis` becomes a component of the INV-27 calculation identity, alongside tenant, store, reporting currency, period and timezone, calculation version, source-data boundary and `calculation_view`.

The identity tuple becomes:

```
(tenant_id, store_id, reporting_currency, period, timezone,
 calculation_version, source_data_boundary, calculation_view,
 shipping_allocation_basis)
```

**R9.11 — Fail closed.** A calculation whose identity omits `shipping_allocation_basis` is **not produced**. There is no default, no "assume B-3", and no null-tolerant aggregation. A missing basis is a defect in the caller and must surface as an error, not as a figure.

**R9.12 — No aggregation across bases.** Two figures with different `shipping_allocation_basis` are different calculations. Summing them is prohibited by the same mechanism INV-27 already uses for `calculation_view`: the aggregation guard asserts a uniform identity tuple across all inputs and refuses otherwise.

**R9.13 — Cross-basis comparison is a first-class artifact.** Comparing a B-3 figure with a B-1 figure is permitted only through an explicitly identified reconciliation calculation that states both bases, both figures, the per-unit differences and the reason for them. The order-level difference must reconcile to zero (§2: +13 +13 −26 = 0); a non-zero order-level difference indicates a defect, not a finding.

### 8.1 How two calculations are prevented from appearing identical

Four independent mechanisms, because a single one is a single point of failure:

| # | Mechanism | What it prevents |
|---|---|---|
| M-1 | `shipping_allocation_basis` is inside the identity tuple and inside the identity digest | Two figures with different bases cannot share an identity hash |
| M-2 | The aggregation guard asserts tuple uniformity before any sum | Silent mixing of bases in a store-level or product-period total |
| M-3 | Every persisted calculation artifact records the basis in its own metadata, not only by reference | Basis becoming unrecoverable if the referenced order is archived |
| M-4 | The merchant-facing presentation must display the basis and the flag (`basis_pooled`, `basis_pooled_contested`, `basis_evidence_invalidated`) wherever product-level shipping allocation is shown | A pooled estimate being read as a directly evidenced cost |

M-4 is a product requirement, not a UI preference. INV-27's principle is that a figure the merchant cannot correctly interpret is a figure that must not be shown; the same logic applies to a figure whose basis is invisible.

---

## 9. Alternatives considered

| Option | Description | Advantages | Disadvantages | Verdict |
|---|---|---|---|---|
| **A-1** | Always pooled (R1–R3 only); drop R5 | Simplest; one basis; no selector needed; C-10a.1 trivially satisfiable | Discards genuine per-parcel evidence; on P1-O1 it misstates C by 26 and A/B by 13 when the provider did expose the association; product-level shipping becomes a pure estimate even where fact exists | Rejected — it deliberately discards evidence the founder's own E5 analysis called "more faithful" |
| **A-2** | Always direct where available, reallocate on late evidence | Most accurate product-level figures | Requires supersession of written allocations; breaks C-10a.2/C-10a.3; retroactively changes figures already reported and acted on | Rejected — see Case 3/4 |
| **A-3** | **Basis ladder B-1/B-2/B-3, fixed at the allocation instant, immutable, in identity, late evidence annotates only** | Deterministic; preserves evidence where it exists; never rewrites history; makes the basis auditable and unsummable across bases; C-10a.1 becomes satisfiable | Product-level figures can be permanently divergent from better late evidence; requires three flags and a disclosure obligation; requires the basis in the identity tuple | **Recommended** |
| **A-4** | Defer allocation until first shipment so more evidence is available | More evidence at decision time | Reintroduces fulfilment-timing dependence — the exact defect the founder rejected; unshipped orders would have no allocation and no pending balance | Rejected |
| **A-5** | Merchant-selectable basis per store | Merchant control | Non-comparable across tenants; the same order would have two legitimate answers depending on a setting; makes provider benchmarking meaningless | Rejected for MVP; a per-store default would still need A-3's machinery underneath |

**A-3 is recommended.** It is the only option that keeps the allocation immutable (C-10a.2, C-10a.3) while still using association evidence when it genuinely exists (R5's intent), and it converts the residual uncertainty into a *visible* property of the figure rather than a hidden one.

---

## 10. Consequences

### 10.1 Financial

- **Order-level contribution profit is unaffected** by the basis, in every case. Both bases conserve S_net exactly. This is the property that makes the whole design safe: a basis error can mislead about *which product* is unprofitable, never about *whether the order* was profitable.
- **Product-level contribution profit moves by up to 26 minor units per unit** on a 1,000-unit order (~4–6% of each unit's shipping allocation). Where COGS is missing or thin, a 26-unit swing can change the sign of a product's contribution profit. This is the strongest argument for the basis being in the identity rather than merely in a log.
- **Co-Pilot gating.** `basis_pooled` is a reduced-confidence signal; `basis_pooled_contested` is a stronger one; `basis_evidence_invalidated` blocks automated recommendations for the affected products outright. The system must not pause a campaign on the strength of a mix-shifted product figure without saying so.

### 10.2 Auditability

- Every allocation states its basis and the evidence digest that justified it, so a reviewer can determine *why* an order was allocated one way and not another, months later.
- Late evidence is visible as an event with a date, rather than being invisible because it was never used.
- The three flags form a monotonic severity ladder, so a merchant-facing disclosure can be generated mechanically rather than judgementally.
- The absence of a reallocation operation is itself auditable: a reviewer can confirm that no figure changed by inspecting the event log, which is a stronger guarantee than trusting that a correction was recorded properly.

### 10.3 What this design does not protect against

- **A wrong association believed at the allocation instant.** If B-1 evidence is present, internally consistent and simply false, the allocation is wrong from the start and only Case 6 applies — annotation and blocking, not correction.
- **Provider fields that do not mean what we think.** Whether a provider's line-level shipping field genuinely represents the charge for that line, rather than a pro-rata apportionment the provider itself computed, is a **provider-verification question (P-01, P-04)**. If the provider's "association" is itself pooled, B-1 inherits the provider's pooling and is not more faithful than B-3 — it is merely differently pooled. This must be established from official documentation before B-1 is treated as actual rather than estimated.
- **Merchant misinterpretation of the flag.** M-4 requires disclosure; it cannot compel comprehension.

---

## 11. Cross-document impact

| Document / section | Current state | Change required if P-1 is approved | Kind |
|---|---|---|---|
| D-10a brief §2, R1–R5 | R1–R3 pooled, R5 direct, no selector | Insert R9 ladder; state R1–R3 as the B-3 path only | Substantive |
| D-10a brief E5 | Presents both answers and calls direct "more faithful" | Restate as the B-1 vs B-3 comparison; state which basis applies and why | Substantive |
| D-10a brief §5 (open sub-questions) | Association basis listed as "the one genuinely open sub-question" | Replace with the closed R9 rule plus the remaining provider question | Substantive |
| D-10a §3.5.6 conditions | C-10a.1 open | C-10a.1 can be satisfied only after the applied and re-audited correction set | Status |
| INV-27 §2 identity tuple | tenant, store, currency, period, timezone, calculation version, boundary, view | Add `shipping_allocation_basis`; add fail-closed rule; extend aggregation guard | Substantive |
| INV-27 §5 "must never do" | Prohibits cross-view aggregation | Add cross-basis aggregation prohibition | Substantive |
| INV-26 S2 | Cancellation releases the shipping allocation | Add that release is basis-independent (the share is whatever the basis produced) | Editorial |
| INV-12 idempotency | `(source, source_id, event_type, component_ordinal, occurrence)` | Add charge-level allocation identity so a second allocation per charge is a duplicate | Substantive |
| New invariant **INV-28** | — | Proposed (§12) | New |
| D-13 fee allocation | Order-level residual precedent | Cited as the precedent for order-level (non-unit) lines; no change | Reference |

---

## 12. New invariant proposed — INV-28

**Rule.** Every shipping revenue allocation carries an explicitly recorded association basis drawn from {B-1, B-2, B-3}, fixed at the allocation instant and immutable thereafter; the basis is a component of the calculation identity; a calculation lacking a basis is not produced; and no aggregation or comparison may combine figures of differing basis except through an explicitly identified reconciliation artifact whose order-level difference reconciles to zero.

| Element | Specification |
|---|---|
| **Scope** | Every shipping revenue allocation for every order, in every calculation view |
| **Calculation basis** | Basis selected by the R9 ladder against evidence visible in the allocation transaction; recorded with evidence reference and digest |
| **Enforcement** | (i) NOT NULL constraint on the basis in the allocation record; (ii) basis inside the identity digest; (iii) aggregation guard asserting tuple uniformity; (iv) artifact metadata recording the basis independently; (v) application-level refusal to emit a calculation with an absent basis |
| **Failure behaviour** | Missing basis → calculation **not produced** (fail closed, no default). Mixed basis in an aggregation → aggregation refused, error surfaced. Late evidence → annotation event only, never a financial change. Invalidated basis → flag `basis_evidence_invalidated` and Co-Pilot recommendations blocked for affected products |
| **Test case** | TC-28-01: allocate P1-O1 with no association → expect 287/287/426, basis B-3, flag `basis_pooled`. TC-28-02: allocate the same order with both charges associated → expect 300/300/400, basis B-1, no flag. TC-28-03: attempt to sum the two results → expect refusal. TC-28-04: strip the basis from an identity tuple → expect the calculation to be refused. TC-28-05: deliver `association_evidence_arrived` after allocation → expect no bucket change and flag escalation. TC-28-06: attempt a second allocation for the same charge → expect idempotent rejection |

**Note.** INV-28 is a *proposal*. Its numbering requires founder approval, and it must not be described as adopted until the founder records it.

---

## 13. Unresolved questions

| ID | Question | Why it cannot be closed here |
|---|---|---|
| OQ-P1-1 | Does any target provider actually expose a **line-level or quantity-level shipping charge association**, as distinct from a pro-rata apportionment it computes itself? | Requires official provider documentation — **P-01** (Shopify) and **P-04** (shipping provider). If no provider exposes it, B-1 and B-2 are unreachable and every order is B-3 |
| OQ-P1-2 | Where a provider exposes an association, is it stable across payload revisions, or can it change on re-fetch? | Requires recorded payload fixtures — **F-01** and P-01 |
| OQ-P1-3 | Should `basis_pooled_contested` alone block Co-Pilot, or only reduce confidence? | Founder policy decision. This brief proposes: reduce confidence, do not block. Blocking on any late evidence would make Co-Pilot unusable in practice |
| OQ-P1-4 | Is an alternative-basis what-if calculation in scope for the MVP at all? | Founder scope decision. This brief proposes deferring it beyond the MVP; §8 R9.13 defines the shape if it is ever built |
| OQ-P1-5 | When a charge is associated to a quantity set that includes a cancelled unit, is the association still B-1-eligible? | Requires a rule decision; this brief proposes yes, with the cancelled unit's share released under the existing cancellation event |
| OQ-P1-6 | Does recording the basis in the identity tuple increase the number of distinct calculation identities enough to be operationally significant? | Requires implementation and measurement. Not measurable in a specification-only phase |

---

## 14. Recommendation, separated from the proposed decision

### 14.1 AppJet's recommendation

Adopt **R9** as specified in this brief: the B-1/B-2/B-3 ladder, basis fixed at the allocation instant, immutable, recorded with its evidence, carried in the calculation identity, with late evidence producing annotation events and data-quality flags rather than financial changes. Adopt **INV-28** as the enforcing invariant.

The recommendation rests on three properties that survive independent challenge:

1. **Order-level figures are invariant to the basis.** Both bases conserve S_net exactly (P1-O1: 287+287+426 = 300+300+400 = 1,000). A basis defect therefore cannot corrupt order-level or store-level contribution profit — it can only misattribute within the order.
2. **Reproducibility is structural, not procedural.** Because no reallocation operation exists, a historical figure cannot become underivable. That is a stronger guarantee than any correction workflow.
3. **Uncertainty becomes visible.** The three flags force the system to state whether a product-level shipping figure is evidenced, pooled, contested or invalidated, which is the only honest basis for a Co-Pilot recommendation built on it.

### 14.2 What the founder is being asked to decide

This is **not** the same thing as the recommendation. The founder is asked to choose one of:

- **Approve A-3** (the R9 ladder) as specified, including the immutability rule and the basis-in-identity requirement;
- **Approve A-3 with conditions**, naming the conditions;
- **Select A-1** (pooled only, R5 withdrawn) — simpler, but discards evidenced per-parcel costs;
- **Defer**, in which case D-01 stays open and C-10a.1 cannot be satisfied, so D-10a continues to block G3.

No part of this brief has been applied to any specification, and no test has been executed.

---

**Prepared:** 2026-11-20 · **Branch:** appjet · **Implementation authorisation:** not granted · **Supabase:** not connected · **Tests executed:** none
