# Round 3 — Complete D2-01 to D2-16 Decision Register

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision clarification 2026-11-20 §3
**Status:** DECISION REQUEST — **no decision in this table is approved**
**Baseline unchanged:** D-10a = APPROVED WITH CONDITIONS · C-10a.1 = PARTIALLY SATISFIED — OPEN

Full rule text, worked counterexamples and per-decision detail live in `founder-decision-table-round2.md` §2. This table is the approval instrument; it condenses without altering.

---

## Group A — Approvable now (11 decisions)

No external evidence, no accounting opinion. Each rests on internal consistency and has a worked counterexample.

| ID | Decision statement | Proposed rule | Financial consequence | MVP/Pilot | Prof. review | Evidence | Recommendation → approval requested |
|---|---|---|---|---|---|---|---|
| **D2-02** | Canonical encoding and stored digest of the association vector | Length-prefixed injective string `v1`, charges ordered by byte comparison; store canonical string **and** `SHA-256(canonical)` | Two orders with identical labels but different vectors cannot share an identity — prevents silent comparison | MVP | No | None | Adopt. **Approve** storing the preimage beside the digest and freezing `v1` as identity governance |
| **D2-04** | Late-arriving association evidence | Vector frozen at acceptance; late evidence annotates (`association_evidence_late`), never reallocates; contradiction raises `A3_contested` and may propose a correction | Same order reports the same product figures every day; no unexplained movement | MVP | No | None | Adopt. **Approve** that late evidence never auto-reallocates, at any stage |
| **D2-05** | Correction workflow versus versioned architecture | **MVP:** `allocation_version`, immutable allocation records, `supersedes` link, retained `snapshot_id`, stored digest, engine able to compute `n+1`. **Pilot:** `correction_requested` / `correction_authorised` events and merchant dispute flow | A correction creates a new calculation identity and moves product-level figures only; the store total is invariant (1,200 unchanged, product A moved by 2) | MVP (versioning); Pilot (workflow) | No | None external | Adopt the split. Absence of an authorisation workflow is **safe** — nothing incorrect is produced. Absence of versioning is **not** — fixing a wrong association would require overwriting it, violating INV-19 and destroying the evidence behind earlier figures. **Approve the split**; decide OQ-P1-4 (who may authorise) |
| **D2-06** | Two recognition states | `recognised(u) ∈ {pending, recognised}`, derived from shipment events only; monotone, never returns to pending | Makes the Fulfilment View computable; a collapsed state destroys the per-unit shipment timestamp | MVP | No | None | Adopt. **Approve** the two-state model and monotonicity |
| **D2-07a** | Orthogonal dimensions and typed reduction events *(structural half of D2-07)* | Four reduction types: `cancellation_release`, `shipping_discount`, `shipping_refund`, `correction`; dimensions fulfilment(3) × disposition(5) × payment(6) kept orthogonal; two separate identities | E-5: three reductions on one unit (100 + 200 + 126 = 426) all retained; a five-bucket model keeps one reason and loses two | MVP | No | None | Adopt. **Approve** the four types, orthogonality and the two-identity separation |
| **D2-08** | Immutable events versus derived status | Layer 1 immutable `charge_observed`/`reduction_observed` with `provider_ts` and `ingestion_ts`; Layer 2 status is a pure function of the event set, never a column | Order-independence by construction; CX-D gives the same answer under reversed delivery | MVP | No | None | Adopt. **Approve** derived-not-stored status |
| **D2-09** | Collectible balance with completeness status | `CB = S_net + Σ additional − Σ corrections`, `CB_status ∈ {provisional_incomplete, final}`; only a **final** CB can confirm excess | CX-A: prevents a refund arriving before its charge becoming a false `confirmed_excess 200` | MVP | No | OQ-P4-1, OQ-P4-2 | Adopt. **Approve** CB completeness and the final-only rule |
| **D2-10** | Ceiling basis | Ceiling is `CB`, not recognised shipping revenue; unit excess spills to order scope; unit balances never negative | CX-E: 713 refund for goods never shipped would be wrongly rejected by a 287 revenue ceiling | MVP | No | None | Adopt. **Approve** CB as the ceiling basis |
| **D2-11** | Excess classified, never clamped | Excess becomes `confirmed_excess` with cause recorded; refunds never reduced to fit the balance | CX-B: clamping to 500 would silently discard 100 of real money movement | MVP | No | None | Adopt. **Approve** classify-not-clamp and decide OQ-P4-4 (visibility of excess) |
| **D2-15** | Deterministic tie-break key | `(line_identity, unit_index)`, line by UTF-8 byte order then unit index ascending; fail closed if not unique | 287/287/426 verified; genuine tie 0.5/0.5 resolved identically under reversal; 500 permutations identical | MVP | No | OQ-P7b-1 | Adopt. **Approve** the key and the fail-closed rule. Shipment order is structurally impossible — allocation precedes shipment |
| **D2-16** | PROP-C shipping reduction conservation | Three clauses within one identity: (a) completeness, (b) no duplication, (c) no over-application | Prevents a reduction vanishing (a) and a duplicate webhook doubling 400 to 800 (b) | MVP | No | None | Adopt with provenance recorded. **Approve** PROP-C as one invariant; numbering is a separate decision (R3-D4) |

---

## Group B — Mechanism approvable now; merchant-facing claim gated on provider evidence (3 decisions)

The mechanism can be approved today. What cannot be approved is telling a merchant their product figures are *evidenced* until a provider is shown to expose the association.

| ID | Decision statement | Proposed rule | Financial consequence | MVP/Pilot | Prof. review | Evidence | Recommendation → approval requested |
|---|---|---|---|---|---|---|---|
| **D2-01** | Method vector allowed values | Vector = per-charge `(charge_identity, method, granularity, scope_ref, evidence_ref)` + frozen `snapshot_id`, `allocation_version`; only `D × {G-unit,G-line,G-shipment}` and `P × {none}` representable | Decides whether product shipping revenue is evidenced or pooled; store totals unaffected (11,200 under every vector) | MVP | No | **OQ-P1-1** — does any MVP provider expose line-level association? | Adopt. **Approve the value set and restricted-pair matrix now**; the *claim* of evidenced attribution waits on OQ-P1-1 |
| **D2-03** | Association granularity in MVP | `G-line` + `P` in MVP; `G-unit` if exposed; `G-shipment` deferred; no merchant-selectable basis | If no provider exposes line association, every order is fully pooled and product attribution cannot be called evidenced | MVP (`G-line`/`P`); Later (`G-shipment`) | No | OQ-P1-1, OQ-P1-2, OQ-P1-3 | Adopt. **Approve the split and the mandatory disclosure** when only pooling is available |
| **D2-13** | Deferral window policy | Per-provider configurable `reconciliation_window`; expiry sets `investigation_due` only, never reclassifies | If expiry could produce excess, a slow webhook becomes a merchant accusation; window must exceed Shopify's verified 8-retry/4-hour envelope | MVP (policy object); default at Pilot | No | **OQ-P4-3** — pilot latency distribution | Adopt the shape. **Approve the policy object and the no-reclassification rule**; **do not approve a numeric default** |

---

## Group C — Statutory classification requires professional review; technical treatment approvable now (3 decisions)

This is the same split you applied to F-1: approve the mechanism, defer the accounting label.

| ID | Decision statement | Proposed rule | Financial consequence | MVP/Pilot | Prof. review | Evidence | Recommendation → approval requested |
|---|---|---|---|---|---|---|---|
| **D2-07b** | Disposition vocabulary semantics *(vocabulary half of D2-07)* | `restocked` / `damaged` / `written off` recorded as disposition values with dates and reasons; the *cost* consequence follows D-07 inventory policy | Determines whether a written-off return retains its COGS once (the Example E defect) or is double-counted | MVP (values recorded); **semantics gated** | **Yes** — interacts with D-07 inventory valuation | None for recording | Record the vocabulary now. **Approve recording the values**; refer the cost semantics to professional review. Do not fix the inventory treatment here |
| **D2-12** | Goodwill and compensation classifications | Four separate types: `shipping_refund` (ceiling applies), `delivery_goodwill`, `fulfilment_compensation`, `other_service_adjustment` (no ceiling) | CX-1: 300 goodwill on a free-shipping order misclassified as a shipping refund → false `confirmed_excess 300`. Correctly classified → an operating cost of 300 | MVP (classification); Pilot (merchant UI) | **Yes** — revenue-reducing vs expense, and period treatment (OQ-P4-5) | Evidence of what each payment was for | Adopt the four types with no ceiling on non-refund types. **Approve the classification**; refer the accounting treatment |
| **D2-14** | Zero allocation base | **O-C**: order-level treatment plus explicit `unallocated_shipping_revenue` bucket, visible at order and store level, excluded from product margins **with disclosure** | O-B gives 334/333/333 — credits a fully discounted item with revenue and makes it appear contribution-positive. O-D blocks an entire store report for one promotional order | MVP | **Yes** — OQ-P7a-2: does unallocated revenue reduce product contribution or sit outside it? | None external | Adopt O-C; reject O-B at every stage; reject O-D as an MVP default. **Approve O-C**; leave OQ-P7a-2 open |

---

## Cross-cutting: the D2-07 resolution

D2-07 was listed in both Group A and Group C because it contains **two decisions with different dependency profiles**, not because either classification was wrong. It is now split:

- **D2-07a** — the structural model (four reduction types, orthogonal dimensions, two identities). Internal consistency only → **Group A, approvable now**.
- **D2-07b** — the *meaning* of disposition values (restocked / damaged / written off) and its interaction with D-07 inventory policy → **Group C, statutory semantics deferred**.

Full reasoning in `d10a-round3-d2-07-classification-resolution.md`. **D2-07's recorded status is unchanged pending your ruling** on whether to adopt the split.

---

## Tally

| Group | Count | Decisions |
|---|---|---|
| A — approvable now | 11 | D2-02, D2-04, D2-05, D2-06, D2-07a, D2-08, D2-09, D2-10, D2-11, D2-15, D2-16 |
| B — mechanism now, claim gated | 3 | D2-01, D2-03, D2-13 |
| C — statutory review deferred | 3 | D2-07b, D2-12, D2-14 |
| **Total** | **17** | 16 distinct IDs (D2-01 to D2-16), D2-07 counted twice as D2-07a / D2-07b |

**Nothing in this table is approved.** Approving a Group A item closes that decision; it does not close D-10a, does not satisfy C-10a.1, and does not authorise implementation.

---

## Correction recorded

D2-05 was **omitted from the first draft of this table** and restored during verification, when a completeness check showed 16 rows for 16 IDs with D2-05 absent. The omission is recorded rather than silently repaired: a decision register that drops a decision is the exact failure mode this table exists to prevent, and the check that caught it was a mechanical ID count, not a reading of the content.
