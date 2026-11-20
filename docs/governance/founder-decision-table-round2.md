# Founder Decision Table — Round 2 (D-10a Revised Architecture and Financial-Control Proposals)

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision round 2026-11-20 §10
**Status:** DECISION REQUEST — **nothing in this table is approved**
**Baseline unchanged:** D-10a = APPROVED WITH CONDITIONS · C-10a.1 = PARTIALLY SATISFIED — OPEN
**Evidence status:** no code, no schema, no test executed. Numeric checks below are arithmetic on specifications, not application tests.

---

## 1. How to read this table

Each row is one decision. Rows are deliberately **not** bundled: approving one does not approve its neighbour. The "AppJet recommendation" column is a recommendation and carries no authority — only your recorded decision does.

**Classification key:** `MVP` = required before any merchant sees a figure · `Pilot` = required before real merchant data at scale · `Later` = deferred with consequences documented.

---

## 2. The decision table

### D2-01 — P-1 method vector: exact allowed values

| | |
|---|---|
| **Exact rule** | Vector = per-charge tuple `(charge_identity, method, granularity, scope_ref, evidence_ref)` plus frozen `snapshot_id` and `allocation_version`. `method ∈ {D, P}`. Permitted pairs only: `D × {G-unit, G-line, G-shipment}` and `P × {none}`. `P × G-*` and `D × none` are unrepresentable and fail validity. |
| **Financial consequence** | Determines, for every order, whether product-level shipping revenue is evidenced or pooled. No effect on store-level totals — `Σ A(u) = S_net` holds under every vector (verified: 11,200 across all methods). |
| **Classification** | **MVP** (with `G-line` and `P` only; see D2-03) |
| **Evidence required** | OQ-P1-1: does any MVP provider expose line-level charge association? Pending P-01/P-02. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt the closed value set. The restricted-pairs rule is the part that matters: it makes invalid vectors *unrepresentable* rather than merely rejected. |
| **Decision requested** | Approve the closed value set and the restricted valid-pair matrix. |

### D2-02 — P-1 canonical encoding and stored digest

| | |
|---|---|
| **Exact rule** | Length-prefixed injective canonical string (`v1`), charges ordered by `charge_identity` byte comparison. Store **both** the canonical string and `SHA-256(canonical)`. Identity uses the digest; the label `A1/A2/A3` is display-only and never an identity element. |
| **Financial consequence** | Prevents two orders with identical summary labels but different vectors from sharing a calculation identity — the silent-comparison failure INV-27 exists to prevent. Worked: `[(c1,D,G-line),(c2,P,none)]` and `[(c1,P,none),(c2,D,G-line)]` both label `A2`, produce different canonical strings and different identities. |
| **Classification** | **MVP** |
| **Evidence required** | None external. Design decision. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt. Storing the preimage is what makes the identity auditable rather than merely unique — a digest alone proves nothing. |
| **Decision requested** | Approve storing the canonical string alongside the digest, and freezing the encoding version (`v1`) as part of identity governance (OQ-P1-6). |

### D2-03 — P-1 association granularity in MVP

| | |
|---|---|
| **Exact rule** | `G-line` and `P` (pooled) in MVP. `G-unit` if the provider exposes it. `G-shipment` deferred. Merchant-selectable basis not recommended at any stage. |
| **Financial consequence** | If no MVP provider exposes line-level association, every order is `A3` (fully pooled) and product-level shipping attribution cannot be claimed as evidenced. Merchant-facing copy must say so. |
| **Classification** | **MVP** for `G-line`/`P`; **Later** for `G-shipment` |
| **Evidence required** | OQ-P1-1, OQ-P1-2 (is the association stable or positional?), OQ-P1-3 |
| **Professional review** | No |
| **AppJet recommendation** | Build the mechanism for `G-line`/`P`; defer `G-shipment`. A merchant who chooses the basis chooses the number, so do not expose the choice. |
| **Decision requested** | Approve the MVP/deferred granularity split, and confirm the merchant-facing disclosure when only pooling is available. |

### D2-04 — P-1 late-arriving association evidence

| | |
|---|---|
| **Exact rule** | The vector is frozen at the acceptance snapshot. Later evidence annotates (`association_evidence_late`), never reallocates. A contradiction raises `A3_contested` and may propose a correction; it never applies one. |
| **Financial consequence** | The same order reports the same product-level figures every day. Without this, figures change with no event to explain the change. |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt, including the refusal to auto-reallocate at any stage. |
| **Decision requested** | Approve that late evidence never reallocates automatically. |

### D2-05 — P-1 correction workflow versus versioned architecture

| | |
|---|---|
| **Exact rule** | **MVP:** `allocation_version`, immutable allocation records, `supersedes` link, retained `snapshot_id`, stored digest, engine able to compute `n+1`. **Pilot:** `correction_requested` / `correction_authorised` events, merchant dispute flow. |
| **Financial consequence** | A correction creates a new calculation identity and moves product-level figures only; the store total is invariant (worked: 1,200 unchanged, product A moved by 2). |
| **Classification** | Split as stated |
| **Evidence required** | None external. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt the split. **The argument:** absence of an authorisation workflow is *safe* — nothing incorrect is produced, merchants merely cannot yet dispute. Absence of versioning is *not* safe — the only way to fix a wrong association would be to overwrite it, violating INV-19 and destroying the evidence needed to defend the earlier figures. |
| **Decision requested** | Approve the MVP/pilot split on that safety argument, and decide OQ-P1-4 (who may authorise: merchant, founder, or both). |

### D2-06 — P-3 two recognition states

| | |
|---|---|
| **Exact rule** | `recognised(u) ∈ {pending, recognised}`, derived from **shipment events only**. Monotone — never returns to `pending`. Returns, refunds and cancellations do not un-recognise. |
| **Financial consequence** | Makes the Fulfilment View computable. A collapsed state field that overwrites `shipped` with `returned` destroys the per-unit shipment timestamp the Fulfilment View requires. |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt. E-4 is the decisive case: a unit shipped 10 February and returned 20 March remains identifiable as shipped, with its date. |
| **Decision requested** | Approve the two-state model and the monotonicity rule. |

### D2-07 — P-3 typed reduction events and orthogonal dimensions

| | |
|---|---|
| **Exact rule** | Four reduction types: `cancellation_release`, `shipping_discount`, `shipping_refund`, `correction`. Three dimensions kept orthogonal: fulfilment (3) × disposition (5) × payment (6). Two separate identities: (a) `Σ effective + Σ unit reductions = CB`; (b) order-scoped reductions as separate lines. |
| **Financial consequence** | E-5: three reductions on one unit all retained with dates and reasons (100 discount, 200 refund, 126 cancellation) totalling 426. A five-bucket partition records one reason and loses two. |
| **Classification** | **MVP** (dimensions exist; not every disposition value populated) |
| **Evidence required** | None external. |
| **Professional review** | Disposition vocabulary (restocked / damaged / written off) interacts with inventory policy — see D-07, professional review open. |
| **AppJet recommendation** | Adopt. Identity (a) must exclude order-scoped reductions — adding them double-counts, the defect found during rev-2 verification. |
| **Decision requested** | Approve the four reduction types, the orthogonal dimensions, and the two-identity separation. |

### D2-08 — P-4/P-5 event records versus derived status

| | |
|---|---|
| **Exact rule** | Layer 1: immutable `charge_observed` / `reduction_observed` records carrying both `provider_ts` and `ingestion_ts`. Layer 2: reconciliation status is a **pure function of the event set**, never a mutable column. |
| **Financial consequence** | Order-independence by construction. CX-D: two refunds with identical `provider_ts` delivered in reverse order yield the same result. Under mutable last-writer state, the same evidence produces different answers on different runs. |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt. Chronology lives on the events; independence comes from the set function. Neither is sacrificed for the other. |
| **Decision requested** | Approve derived-not-stored status. |

### D2-09 — P-4/P-5 collectible balance with completeness status

| | |
|---|---|
| **Exact rule** | `CB = S_net + Σ additional_charges − Σ charge_corrections`, with `CB_status ∈ {provisional_incomplete, final}`. A reduction may be `confirmed_excess` **only** when `CB_status = final` and the scope balance is exhausted; otherwise `unmatched_pending_charge`. |
| **Financial consequence** | CX-A: a refund arriving before its charge. Naive CB = 0 → `confirmed_excess 200` (a false accusation). Specified: `provisional_incomplete` → `unmatched_pending_charge`, then `matched` once the charge is observed. |
| **Classification** | **MVP** |
| **Evidence required** | OQ-P4-1 (which fields establish charge identity), OQ-P4-2 (is a correction distinguishable from a new charge?) |
| **Professional review** | No |
| **AppJet recommendation** | Adopt. This is the load-bearing rule of the whole control: **refusing to treat an unknown balance as zero.** |
| **Decision requested** | Approve `CB` completeness status and the rule that only a final CB can confirm excess. |

### D2-10 — P-4/P-5 ceiling basis

| | |
|---|---|
| **Exact rule** | The ceiling is the collectible balance `CB`, **not** recognised shipping revenue. Unit-scoped excess spills to order scope; unit balances never go negative. |
| **Financial consequence** | CX-E: units B and C never ship; only 287 of 1,000 is recognised; the customer is refunded 713 for shipping never provided. A recognised-revenue ceiling (287) wrongly rejects it. A CB ceiling (1,000) correctly matches it. |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt. This confirms your own earlier caution: recognised revenue cannot be the ceiling without blocking legitimate refunds for unshipped goods. |
| **Decision requested** | Approve CB as the ceiling basis. |

### D2-11 — P-4/P-5 excess is classified, never clamped

| | |
|---|---|
| **Exact rule** | When matched reductions exceed CB, the difference is `confirmed_excess` with its cause recorded. Refunds are never clamped down to the balance. |
| **Financial consequence** | CX-B: CB 1,000, refund 600 matched, charge correction reduces CB to 500 → excess 100 surfaced. Clamping to 500 would silently discard 100 of real money movement. |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt. This is the same principle as unallocated ad spend: the discrepancy stays visible. |
| **Decision requested** | Approve classify-not-clamp, and decide OQ-P4-4 (is `confirmed_excess` merchant-visible, internal, or both?). |

### D2-12 — P-4/P-5 goodwill and compensation classifications

| | |
|---|---|
| **Exact rule** | Four separate event types: `shipping_refund` (ceiling applies), `delivery_goodwill`, `fulfilment_compensation`, `other_service_adjustment` (no ceiling). |
| **Financial consequence** | CX-1: free-shipping order (CB 0), 300 goodwill. Misclassified as a shipping refund → `confirmed_excess 300`, accusing the merchant of over-refunding money never collected. Correctly classified → an operating cost line of 300. Same −300 to contribution profit, different reporting meaning. |
| **Classification** | **MVP** (classification only; merchant UI is Pilot) |
| **Evidence required** | Evidence of what the payment was for, per event |
| **Professional review** | **Yes** — whether goodwill is revenue-reducing or an expense, and its period treatment (OQ-P4-5). Not decided here. |
| **AppJet recommendation** | Adopt the four types with no ceiling on the non-refund types. Record the classification; do not assert the statutory treatment. |
| **Decision requested** | Approve the four event types; refer the accounting treatment to professional review. |

### D2-13 — P-4/P-5 deferral window policy

| | |
|---|---|
| **Exact rule** | A per-provider, configurable `reconciliation_window` policy object. Expiry sets `investigation_due` only — **never** changes financial classification. No production default chosen here. |
| **Financial consequence** | If expiry could produce `confirmed_excess`, a slow webhook would become a merchant accusation. The window must exceed the provider's retry envelope — P-02 verified Shopify retries 8 times over 4 hours — or false investigations are guaranteed. |
| **Classification** | **MVP** (policy object exists; default set at Pilot from evidence) |
| **Evidence required** | OQ-P4-3: pilot delivery-latency distribution. Also whether missed events are recoverable by reconciliation — P-02 confirms reconciliation is mandatory; the recovery rate is unmeasured. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt the policy object; **do not** select a number. Choosing one now would manufacture a policy from intuition, which you instructed against. |
| **Decision requested** | Approve the policy shape and the rule that expiry never reclassifies financially. |

### D2-14 — P-7a zero allocation base

| | |
|---|---|
| **Exact rule** | When `Σ weight = 0` for a charge scope, use **O-C**: order-level treatment plus an explicit `unallocated_shipping_revenue` bucket, visible at order and store level, excluded from product margins **with disclosure**. |
| **Financial consequence** | Worked (1,000 shipping, three zero-revenue units): O-B equal-by-count gives 334/333/333 — crediting a fully discounted item with 334 of shipping revenue, making it appear contribution-positive. O-D fail-closed blocks the entire store report for one promotional order. O-A loses the money silently. O-C keeps store totals complete and the gap named. |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | **Yes** for OQ-P7a-2 (does unallocated shipping revenue reduce product-level contribution or sit outside it?) |
| **AppJet recommendation** | Adopt O-C. Reject O-B at every stage. Reject O-D as an MVP default — fail-closed belongs to integrity failures, not to a legitimate, representable state. |
| **Decision requested** | Approve O-C; confirm the bucket is per charge rather than per order (OQ-P7a-1). |

### D2-15 — P-7b deterministic tie-break key

| | |
|---|---|
| **Exact rule** | `tiebreak_key(u) = (line_identity, unit_index)`, line compared by UTF-8 byte order, then unit index numerically ascending. Fail closed if the key is not unique. |
| **Financial consequence** | Verified 287/287/426 for weights 2,700/2,700/4,000 with `S_net` 1,000. Genuine tie worked: 0.5/0.5 with a remainder of 1 → lower key receives it, identically under reversed input. 500 permutations produced byte-identical output (specification-level arithmetic check, not an application test). |
| **Classification** | **MVP** |
| **Evidence required** | OQ-P7b-1 (is `line_identity` stable across payload revisions?) |
| **Professional review** | No |
| **AppJet recommendation** | Adopt. **Shipment order is structurally impossible** — allocation occurs at acceptance, before any shipment exists, so a rule depending on shipment order cannot be evaluated at the instant it must be applied. |
| **Decision requested** | Approve the key and the fail-closed rule on non-uniqueness. |

### D2-16 — PROP-C shipping reduction conservation

| | |
|---|---|
| **Exact rule** | Three clauses within a single calculation identity: (a) **completeness** — every evidenced reduction sits in exactly one of applied-to-unit, applied-at-order, or `confirmed_excess`; (b) **no duplication** — no source identity contributes twice; (c) **no over-application** — cumulative reductions never exceed the unit allocation or CB, and excess is classified not clamped. |
| **Financial consequence** | Prevents a reduction vanishing between the provider record and the calculation, and prevents a duplicate webhook making 400 of refunds look like 800 (CX-C). |
| **Classification** | **MVP** |
| **Evidence required** | None external. |
| **Professional review** | No |
| **AppJet recommendation** | Adopt as a single invariant, with **honest provenance recorded**: clause (a) is genuinely new; clauses (b) and (c) are invariant restatements of P-4/P-5 requirements included for testability. Bundled because (a) without (c) would let "accounted for" mean "applied beyond what was collected". |
| **Decision requested** | Approve PROP-C as one new invariant, and authorise an invariant number to be allocated. |

---

## 3. Summary — what you can act on now

### 3.1 Ready for approval (no external evidence, no professional review)

D2-02 · D2-04 · D2-06 · D2-07 · D2-08 · D2-09 · D2-10 · D2-11 · D2-15 · D2-16

Ten decisions rest on internal consistency alone. Each has a worked counterexample and none depends on a provider fact or an accounting opinion.

### 3.2 Ready for approval, conditional on provider evidence

- **D2-01 / D2-03** — the value set is sound; whether `G-line` is ever reachable depends on OQ-P1-1. Approvable now as a *mechanism*, with the merchant-facing claim gated on evidence.
- **D2-13** — the policy shape is sound; the default value is not approvable without pilot telemetry.

### 3.3 Requires professional accounting review before the financial treatment is fixed

- **D2-12** — goodwill: revenue-reducing or expense, and period treatment.
- **D2-14** — whether unallocated shipping revenue reduces product contribution or sits outside it.
- **D2-07** — disposition vocabulary interacts with inventory policy (D-07).

Note the pattern: in each case the **technical treatment is approvable now** and only the statutory classification is deferred. That is the same split you applied to F-1.

### 3.4 Genuinely deferrable without compromising financial integrity

| Item | Why deferral is safe |
|---|---|
| Merchant dispute / correction-authorisation workflow | Absence produces nothing incorrect — corrections simply cannot be initiated |
| `G-shipment` granularity | Unreachable without provider support; pooling is the honest fallback |
| Merchant-facing unallocated-revenue explanation | The bucket exists and reconciles; only its presentation is deferred |
| Investigation/escalation UI | `investigation_due` is set; nothing acts on it yet |

**Deferral is not safe for:** `allocation_version` and immutability (D2-05), CB completeness (D2-09), the derived-status rule (D2-08), and PROP-C. Each of these cannot be backfilled honestly — their absence creates figures that later work would have to overwrite rather than extend.

### 3.5 Still open regardless of this table

| Open item | Blocking condition |
|---|---|
| F-01, D-09, affected part of INV-02 | Genuine Shopify refund evidence — no development store available |
| INV-16 period policy | Professional accounting review (statutory treatment of the two views) |
| D-14 return provision | Deferred; omission must be disclosed and profitability must not imply a provision |
| D-20 key management | OQ-20-1, OQ-20-2 |
| Supabase live security checks | Provisioning unauthorised; pooler, RLS and PITR remain unverified |

---

## 4. What this table does not do

- It does not approve anything. Every status above is a recommendation.
- It does not change D-10a's APPROVED WITH CONDITIONS status, or close C-10a.1.
- It does not allocate an invariant number to PROP-C.
- It does not assert that any test has passed. The arithmetic checks are on specifications.
- It does not mark D-10a implementation-ready or G3-ready.

---

## 5. Verification status

| Claim | Basis |
|---|---|
| 287 / 287 / 426 for 2,700 / 2,700 / 4,000 with 1,000 | Arithmetic verified; matches P-3 examples |
| 500 permutations produce identical allocations | Specification-level arithmetic check |
| Store total 11,200 invariant across all association methods | Verified in rev 2, re-checked |
| 1,200 unchanged under a P-1 correction | Arithmetic verified |
| CB arithmetic in CX-A through CX-G | Arithmetic verified |
| **No application test executed** | No code exists |

**These are not executed tests.** No calculation engine, schema or migration exists, and none is authorised by this document.
