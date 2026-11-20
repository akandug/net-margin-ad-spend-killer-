# D-10a Substantive Correction Review — Founder Decision Package

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Assignment:** D-10a audit outcome and substantive correction review (founder instruction 2026-11-20, third review)
**Operating mode:** governed specification review — **NO IMPLEMENTATION**
**Branch:** appjet
**Implementation authorisation:** NOT GRANTED
**Database / Supabase authorisation:** NOT GRANTED
**Provider connection authorisation:** NOT GRANTED

---

## 1. Status at a glance

| Item | Status |
|---|---|
| **D-10a** | **APPROVED WITH CONDITIONS** — unchanged. Not implementation-ready. Not G3-ready |
| **C-10a.1** | **PARTIALLY SATISFIED — OPEN** |
| **G3** | **Blocked by D-10a.** The proviso in §5 of the register fails |
| Defects D-01, D-02, D-03 | D-02 has an editorial fix prepared; **D-01 and D-03 are substantive and unapproved** |
| New invariants proposed | INV-28, INV-29, INV-30, INV-31 — numbering requires founder approval |
| Files changed in this assignment | 1 amended (register), 4 created |
| Tests executed | **none** |
| Application code, schema, migration, Supabase, provider connection, deployment | **none** |

---

## 2. Part A — audit outcome recorded

Recorded in `docs/governance/decision-register.md` in three places:

1. **§3.5.6** — the C-10a condition table now carries a block recording **C-10a.1: PARTIALLY SATISFIED — OPEN**, with the audit's findings: the seven worked examples are arithmetically correct under their stated assumptions; the anti-balancing-plug requirement is satisfied; D-01, D-02 and D-03 remain unresolved specification defects; the shipping-refund ceiling, multi-charge reconciliation, allocation-base ambiguity and missing scenario coverage must be addressed.
2. **§5** — the G3 entry has been corrected. It previously read that D-10a "no longer blocks G3 … provided C-10a.1–C-10a.5 are enforced and the specification remains internally consistent". That proviso now **fails**, and the register says so explicitly rather than leaving a stale clearance in place.
3. **Change log** — a new row records the outcome, the three briefs prepared, and that no status changed and no test ran.

This is a status update, not a reversal of the founder's approval of D-10a. D-10a remains APPROVED WITH CONDITIONS.

---

## 3. Part B — the three substantive briefs

| Brief | Defect | File | Core proposal |
|---|---|---|---|
| **P-1** | D-01 — association basis selector unspecified and absent from calculation identity | `p1-shipping-association-selector-decision-brief.md` | **R9** basis ladder B-1/B-2/B-3, fixed at the allocation instant, immutable, recorded with its evidence, carried in the INV-27 identity, late evidence annotates and never reallocates |
| **P-3** | D-03 — R6 contradicts C-10a.5 | `p3-post-acceptance-shipping-discounts-decision-brief.md` | **R6 split** into R6.1–R6.5 plus a five-bucket partition; order-scoped discounts become order-level lines (D-13 precedent); unit-scoped discounts move buckets |
| **P-4** | Shipping-refund ceiling absent | `p4-shipping-refund-ceiling-decision-brief.md` | **R10.1–R10.7** with collectible basis `CB = S_net + Chg_add`, combined `R + Disc` cap, unit and charge ceilings, deterministic ordering, quarantine-not-reject |

### 3.1 The one finding that links all three

Each brief independently arrives at the same failure mode and the same remedy:

| Brief | Failure if the naive rule is used | Remedy |
|---|---|---|
| P-1 | Reallocating on late evidence rewrites an already-reported product figure | Immutability + annotation events |
| P-3 | Netting a post-acceptance discount into S_net and reallocating rewrites recognised revenue | Bucket movement + order-level lines |
| P-4 | Capping on recognised revenue blocks legitimate refunds for unshipped goods | Collectible basis, not recognised basis |

In every case the naive rule is the *intuitive* one, and in every case it fails by destroying or misstating something already recorded. The three remedies share one property: **derived balances are computed from an immutable record plus a dated event set**, never by editing the record. That is the same property D-10a's C-10a.2/C-10a.3 already require; these briefs extend it to the two places where D-10a did not yet apply it.

### 3.2 A shared consequence the founder should weigh once

P-1 and P-3 both move uncertainty out of the arithmetic and into **disclosure**:

- P-1 → three data-quality flags (`basis_pooled`, `basis_pooled_contested`, `basis_evidence_invalidated`)
- P-3 → an order-level line (`D_post_order`) that product-level figures do not absorb

This is cheaper to build than reallocation and honest about what is known, but it means the platform will display product-level figures that are **knowingly not the best available estimate** in some cases. That is a deliberate product position: a figure that is reproducible and labelled beats a figure that is more accurate today and different tomorrow. The founder should confirm that position, because it is a product decision, not a technical one.

---

## 4. Part C — consolidated cross-document impact matrix

Every document and section affected if all three briefs are approved. Nothing in this matrix has been applied.

| Document | Section | P-1 | P-3 | P-4 | Kind |
|---|---|---|---|---|---|
| `d10a-shipping-revenue-allocation-decision-brief.md` | §2 R1–R3 | Restate as the B-3 path | — | — | Substantive |
| | §2 R5 | Replace with R9 ladder | — | — | Substantive |
| | §2 R6 | — | Split R6.1–R6.5 | — | Substantive |
| | §2 R7 | — | — | Replace with R10.1–R10.7 | Substantive |
| | §2 R8 | — | — | Per-charge cap (was audit P-5) | Substantive |
| | §3 E1 | Basis stated explicitly | — | — | Editorial |
| | §3 E2 | — | — | — | Editorial (P-2) |
| | §3 E4 | — | Confirm it is R6.1 | — | Editorial |
| | §3 E5 | Restate as B-1 vs B-3 | — | — | Substantive |
| | §3 E6 | — | — | — | Editorial (P-8) |
| | §3 E7 | — | Cross-ref R6.4 | Add ceiling arithmetic | Substantive |
| | §5 open questions | Close the association question | — | Close the ceiling question | Substantive |
| `inv26-quantity-state-specification.md` | S2 | Release is basis-independent | Add `discounted` bucket | Transitions respect R10 | Substantive |
| | S1–S6 state table | — | Add `discounted` | — | Substantive |
| | TC-26-03 | — | — | — | Editorial (P-9 note) |
| `inv27-calculation-identity-specification.md` | §2 identity tuple | **Add `shipping_allocation_basis`** | — | — | Substantive |
| | §3 aggregation | Prohibit cross-basis aggregation | Add order-level `D_post_order` / `Rf_order` | — | Substantive |
| | §4 worked example | — | Unaffected | Unaffected | None |
| | §5 "must never" | Add cross-basis prohibition | — | — | Substantive |
| | §4 wording | — | — | — | Editorial (P-10) |
| `decision-register.md` | §3.5.6 C-10a.1 | Satisfiable only after all three applied | ← | ← | Status |
| | §3.5.6 C-10a.4 | — | — | Satisfiable after R10 | Status |
| | §5 G3 | Unblocks only when C-10a.1 satisfied | ← | ← | Status |
| `d13-payment-fee-allocation-decision-brief.md` | — | Precedent cited | Precedent cited | Precedent cited | Reference |
| `d20-*` | — | — | — | — | None |
| `d10a-c10a1-consistency-audit.md` | P-5, P-7 rows | — | — | Reclassified substantive | Correction |

### 4.1 New invariants required

| ID | Rule | Proposed in | Numbering approved? |
|---|---|---|---|
| **INV-28** | Association basis recorded, immutable, in the identity; no cross-basis aggregation | P-1 §12 | **No** |
| **INV-29** | Adjustment-event separation; no post-acceptance mutation of an allocation | P-3 §11 | **No** |
| **INV-30** | Combined post-acceptance shipping reduction ceiling, deterministically ordered | P-4 §11 | **No** |
| **INV-31** | Shipping bucket partition conservation | P-3 §11 | **No** |

These are proposals. They are not part of INV-01–INV-25, which the founder approved as required specifications. Adding to that set is a founder decision.

---

## 5. Part D — editorial change set

**File:** `d10a-editorial-change-set-proposed-diff.md` · **Applied: nothing.**

Applying the founder's own boundary test to the audit's editorial list produced a different set from the audit's:

| Item | Audit label | This review | Routed to |
|---|---|---|---|
| P-2 | Editorial | **Editorial — safe** | Change set §5 |
| P-5 | Editorial | **SUBSTANTIVE** — changes which refunds are accepted | P-4 §12, R10.3 |
| P-6 | Editorial | **Editorial — safe** | Change set §6 |
| P-7 | Editorial | **SUBSTANTIVE** — introduces the `Σbase = 0` fallback rule and the tie-break key | Change set §4 |
| P-8 | Editorial | **Editorial — safe** | Change set §7 |
| P-9 | Editorial | **Editorial — safe** (separate-scenarios wording only) | Change set §8 |
| P-10 | Editorial | **Editorial — safe** | Change set §9 |

**Five safe, two reclassified.** The reclassifications are not pedantry:

- **P-5** changes which refund events are accepted (a 700 refund against a 600 charge passes under an order-only cap and is quarantined under a per-charge cap).
- **P-7** adds a `Σbase = 0` example whose expected output (334/333/333) silently defines an equal-weight fallback rule where none exists today — `0/0` is currently undefined. Introducing a rule through an example is exactly what the founder's boundary prohibits.

### 5.1 The honest limit of the editorial set

Applying all five safe items fixes **one of three** defects:

| Defect | Fixed editorially? |
|---|---|
| D-01 — association selector | **No** — needs P-1 |
| D-02 — E2/E1 contradiction | **Yes** — P-2 |
| D-03 — R6 vs C-10a.5 | **No** — needs P-3 |

**A clean editorial pass is not progress toward G3 on its own.** C-10a.1 stays OPEN until P-1 and P-3 are decided and applied.

---

## 6. Part E — founder decisions required

### 6.1 The three substantive decisions

| # | Decision | AppJet recommendation | Alternatives offered | Blocks |
|---|---|---|---|---|
| **P-1** | Association basis selector | Adopt A-3: R9 ladder, fixed at allocation instant, immutable, in identity | A-1 pooled only; A-2 reallocate on late evidence; A-4 defer to shipment; A-5 merchant-selectable | D-01, C-10a.1, G3 |
| **P-3** | Post-acceptance discounts | Adopt R6 split + bucket model, B-3 for unit shares, C-1 for period attribution | B-1 recompute; B-2 pro-rata to units; C-2 pro-rata across periods; C-3 defer to full fulfilment | D-03, C-10a.1, G3 |
| **P-4** | Shipping-refund ceiling | Adopt R10.1–R10.7 with `CB = S_net + Chg_add` | Audit's narrower `S_net` basis (needs a separate `Chg_add` ceiling) | C-10a.1, C-10a.4, G3 |

### 6.2 The two reclassified editorial items

| # | Decision | Recommendation | Blocks |
|---|---|---|---|
| **P-5** | Per-charge refund cap where R5 applies | Adopt as R10.3 (substantive, not editorial) | C-10a.4 |
| **P-7a** | `Σbase = 0` behaviour — is charging shipping on zero-net-merchandise revenue legitimate, and is equal weighting the fallback? | Define explicitly; equal weighting is defensible but the legitimacy question is a business decision | C-10a.1 completeness |
| **P-7b** | Tie-break key when fractional parts are equal | Stable unit identity — never insertion order, line order or quantity | Reproducibility |

### 6.3 Professional review required (not founder decisions)

| ID | Question | Brief |
|---|---|---|
| OQ-P3-1 | Order-scoped discount: full deduction in effective period (C-1) or pro-rata (C-2)? Management view vs statutory view | P-3 |
| OQ-P3-2 | Is a post-fulfilment shipping credit revenue reduction or a customer refund liability? | P-3 |
| OQ-P3-3 | Does a backdated discount require restatement of a closed statutory period? (D-18) | P-3 |
| OQ-P4-1 | Is a shipping refund exceeding collected shipping ever legitimate — or should goodwill above collected shipping be a separate cost line? | P-4 |
| OQ-P4-5 | Is a refund reversal a reduction of `R` or a separate negative line? | P-4 |

### 6.4 Provider verification still required

| ID | Question | Blocks |
|---|---|---|
| OQ-P1-1 / OQ-P1-2 | Does any provider expose a genuine **line-level** shipping charge association, stable across payload revisions? If not, B-1 and B-2 are unreachable and every order is B-3 | Whether P-1's ladder is usable at all |
| OQ-P3-5 | Does the provider distinguish a shipping **discount** from a shipping **refund**? | R6.4's reclassification test |
| OQ-P4-3 | Is there a stable refund identity across re-serialisation? (T6) | INV-12 identity |
| OQ-P4-4 | Does the provider expose refund *pending* vs *completed*, and must pending count toward the ceiling? | R10 |

All four sit under **P-01 / P-02 / P-04**, which remain open. **OQ-P1-1 is the load-bearing one**: if no provider exposes line-level shipping association, P-1's recommended design still holds but B-1/B-2 are dead paths in the MVP, and that should be stated plainly to merchants rather than discovered at integration time.

### 6.5 Still open from earlier instructions — unchanged by this assignment

| Item | Status |
|---|---|
| **F-01** — Shopify refund evidence capture | **OPEN** — needs an authorised development store |
| **D-09** — refund discount basis | **OPEN** — cannot close before F-01 evidence |
| **Affected part of INV-02** | **OPEN** — same dependency |
| Supabase provisioning and live security checks | **UNAUTHORISED** — deferred by founder direction |
| OQ-20-1 key management, OQ-20-2 erasure policy | Awaiting founder decision |

---

## 7. Evidence and repository change report

### 7.1 Files examined

| File | Sections read |
|---|---|
| `decision-register.md` | §3.5.6 C-10a condition table; §5 G3 entry; change log |
| `d10a-shipping-revenue-allocation-decision-brief.md` | §2 rules R1–R10; §3 E1 base table, E2, E5, E6 |
| `inv26-quantity-state-specification.md` | §S2 body; TC-26-03 |
| `inv27-calculation-identity-specification.md` | §4 worked reconciliation |
| `d10a-c10a1-consistency-audit.md` | P-2 through P-10 defect rows; recommended sequencing |

### 7.2 Files changed

| File | Change | Kind |
|---|---|---|
| `decision-register.md` | C-10a.1 outcome block; G3 proviso corrected; change-log row | Amended |
| `p1-shipping-association-selector-decision-brief.md` | New | Created |
| `p3-post-acceptance-shipping-discounts-decision-brief.md` | New | Created |
| `p4-shipping-refund-ceiling-decision-brief.md` | New | Created |
| `d10a-editorial-change-set-proposed-diff.md` | New | Created |

**No specification rule, example, allocation amount, decision status or invariant text was modified by this assignment.** The three briefs and the change set are proposals sitting beside the specification, not edits to it.

### 7.3 What was deliberately not done

- No application code, schema, migration, seed data or test file.
- No Supabase connection, environment variable, credential or provider connection.
- No deployment.
- No decision marked approved, closed or implementation-ready.
- No editorial diff applied.
- No invariant renumbered or added to INV-01–INV-25.

---

## 8. Verification status — precise

| Claim | Status |
|---|---|
| Largest-remainder allocations in the three briefs | **Independently recomputed and matched.** All eleven figures reproduced from stated inputs, all conserve their source amount exactly |
| Basis delta B-1 − B-3 = +13 / +13 / −26 | **Verified** arithmetically; sums to zero |
| Case 6 correction delta = −100 / −100 / +200 | **Verified** arithmetically; sums to zero |
| B-2 = 415 / 415 / 170 | **Verified** as 300/300/0 plus 115/115/170 |
| INV-31 bucket conservation on P3-1, P3-2, P4-T4 | **Verified** arithmetically |
| INV-30 ceiling dispositions for T1, T2, T4, T5, T8 | **Verified** arithmetically, including the T8 boundary and the P3-5 unit-cap breach that order-level arithmetic would accept |
| T7 arrival-order determinism | **Verified** by simulating both arrival sequences under effective-date ordering — identical accepted set |
| `Σbase = 0` → 334/333/333 and tie-break → 251/250/501 | **Verified** arithmetically, but both depend on rules that are **not yet approved** (P-7a, P-7b) |
| Any invariant test executed against a database | **No.** No database exists. TC-26-03, TC-28-*, TC-29-*, TC-31-* are specifications |
| Any provider behaviour verified | **No.** All four provider questions remain open |
| Any control implemented | **No.** Every control described is a specification |

**The arithmetic is verified. Nothing else is.** Recomputation in a scratch script confirms the numbers; it does not confirm that a financial engine implements these rules, because no financial engine exists.

---

## 9. Recommendation

**The three briefs are ready for founder decision and are not ready for implementation.** Specifically:

1. **D-01 and D-03 are genuinely blocking**, and neither can be closed editorially. D-10a cannot reach G3 until P-1 and P-3 are decided.
2. **P-1's recommendation is the highest-value decision in this package.** It is the only one that changes the calculation identity tuple, and therefore the only one that, if decided late, would force rework of INV-27 and of every stored calculation artifact.
3. **P-4's recommendation is the most self-evidently correct** — the two cancellation proofs (§3.1, §3.2 of that brief) show the recognised-only ceiling is not merely conservative but wrong. It should be the easiest to approve.
4. **The founder should decide P-7a and P-7b together with P-1**, because the tie-break key is the same class of reproducibility property as the association basis, and splitting them invites inconsistency.
5. **Apply everything in one pass**, after decisions, per the change set's §12 — applying the editorial set first would leave E6 describing a method P-1 may replace.

One caution worth stating plainly: this package adds four proposed invariants and a bucket model to a specification that already carries twenty-five invariants and three unresolved defects. The design is becoming precise faster than it is becoming *smaller*. Before implementation begins, the founder may want to ask whether every one of INV-28–INV-31 earns its place in the MVP, or whether some are refinements better deferred to the pilot. I have recommended all four because each guards a distinct, demonstrated failure — but "we can specify it" is not the same as "the MVP needs it", and that judgement is the founder's.

---

## 10. Stop condition

This assignment is complete. No implementation has begun, and none should begin on the strength of this package alone — the founder's decisions on P-1, P-3, P-4, P-5 and P-7 are prerequisites.

Awaiting founder review.

---

**Prepared:** 2026-11-20 · **Branch:** appjet · **Tests executed:** none · **Implementation authorisation:** not granted · **Supabase:** not connected · **Deployment:** none
