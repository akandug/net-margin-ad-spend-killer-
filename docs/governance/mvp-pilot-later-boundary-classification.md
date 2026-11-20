# MVP / Pilot / Later Boundary Classification

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Status:** PROPOSED CLASSIFICATION FOR FOUNDER APPROVAL
**Governing rule applied:** *a deferred feature must not create silent financial misstatement — if a calculation cannot be safely supported it must fail closed or be marked provisional/unallocated.*

---

## 1. MVP mandatory financial controls

These must exist before any merchant-visible profitability figure. Each is either a calculation-correctness requirement or a reproducibility requirement.

| # | Control | Why it cannot be deferred |
|---|---|---|
| M1 | Per-charge shipping association method recorded and frozen (P-1 R9′) | Without it, results are not reproducible and cannot be re-derived |
| M2 | Method **vector** in INV-27 identity; label alone insufficient | Two orders can share the summary label with different vectors; using the label permits silent comparison of incomparable figures |
| M3 | Association validity test (P-1 §4) | A present-but-invalid association produces a confidently wrong direct allocation |
| M4 | Pooled default when no valid association; **fail closed** if method absent | No silent default; the less-attributed state is the safe failure |
| M5 | Within-scope largest remainder (G1/G3 handling) | Multi-unit lines and shipment-scoped charges are not edge cases |
| M6 | Two recognition states (`pending`/`recognised`) + typed reduction events (P-3) | The five-bucket alternative loses the recognition fact on refunded units |
| M7 | Monotone `CB` (P-4 §2.1) | Without it `CB` can fall below committed reductions, producing an unexplained negative |
| M8 | Order and unit scope ceilings (P-4 §2.2) | The unit cap is the only thing catching `cancellation_release` + `shipping_refund` double-counting (T9) |
| M9 | Reality-over-ceiling disposition (P-4 §6) | Real refunds cannot be rejected; refusing to record them breaks reconciliation with the bank |
| M10 | Deferral on missing charge evidence (P-4 §5.3, CX-4) | Without it, webhook reordering produces spurious excess classifications on real data |
| M11 | Largest remainder for **every** reduction amount (CX-6) | Independent per-unit rounding breaks conservation (334 ≠ 333) |
| M12 | Conservation test (PROP-D) | The only mechanical check that nothing was absorbed |
| M13 | Zero-base rule (P-7a, O-C) | `Σ w = 0` makes pro-rata undefined; a silent divide-by-zero corrupts the order |
| M14 | Stable tie-break key + index retention on cancellation (P-7b) | Without it, the same order yields different product figures across runs |
| M15 | INV-14: no silent estimated→actual conversion | Already approved; shipping must not convert on assumption |
| M16 | Unallocated/unattributed amounts remain visible | Standing founder rule; O-C depends on it |

## 2. Pilot-required controls and exception handling

Needed before real merchants rely on the figures, but not before G2 arithmetic correctness.

| # | Control | Why it can follow G2 |
|---|---|---|
| P-1 | Correction / new allocation version workflow (P-1 §6) | Only needed once a merchant disputes a figure; the mechanism (versions, no mutation) is MVP, the *workflow* is not |
| P-2 | Late-evidence annotation + `A3_contested` flag | Affects presentation and disclosure, not the calculation |
| P-3 | `charge_pending` reconciliation with un-ingested charges | The MVP deferral (M10) is sufficient initially; full reconciliation needs real webhook behaviour |
| P-4 | `G-unit` / `G-shipment` granularity | Only if a provider exposes them (OQ-P1-1); building it before evidence is speculative |
| P-5 | Charge-scope ceiling (P-4 §2.2) | Dead code unless a provider supplies charge references (OQ-P5-1) |
| P-6 | `shipping_refund_excess` classification workflow and merchant disclosure | MVP records the excess; the review workflow is pilot |
| P-7 | Correction of a *reduction* (vs a correction of an *allocation*) | Rare; needs OQ-P3-4 resolved |

## 3. Later enhancements

| # | Item | Why later |
|---|---|---|
| L-1 | Merchant-selectable association basis | A merchant who can choose the basis can choose the number; requires disclosure design first |
| L-2 | Mass/dimension weighting for zero-base (O-B) | Data dependency; recursive zero-base problem when mass is missing |
| L-3 | Content-hash fallback for providers without line ids | Only if a provider requires it |
| L-4 | Cross-view reconciliation reporting (INV-27) | The identity rule is MVP; the merchant-facing reconciliation report is not |
| L-5 | Multi-currency per-order shipping | Existing currency rules apply; per-order multi-currency is not an MVP scenario |
| L-6 | Automatic reallocation on late evidence | **Not recommended at any stage** — destroys reproducibility |

## 4. Professional accounting, tax or provider review dependencies

None of these blocks the MVP *build*; several block specific *reported figures*.

| ID | Matter | Blocks |
|---|---|---|
| OQ-P3-1 | Order-scoped discount period assignment (effective date vs spread) | Period margin reporting |
| OQ-P3-2 | Post-fulfilment credit: revenue reduction or liability? | Shipping refund treatment in reports |
| OQ-P3-3 | Backdated discount to a closed period: restatement or open-period recognition? | Period reporting, D-18 |
| OQ-P4-1 | `shipping_refund_excess`: cost of sale or liability? | Excess reporting |
| OQ-P4-2 | Distinguishing goodwill from a shipping refund when the provider labels it shipping | Classification (CX-1) |
| OQ-P4-3 | Shipping discount exceeding charges: subsidy cost or negative revenue? | CX-7 reporting |
| OQ-P7a-3 | Whether unattributed shipping revenue belongs in product-level margin reporting | Product reporting under O-C |
| OQ-P1-1 | Whether any provider exposes line-level association | Reachability of `A1`/`A2` — **provider evidence, not accounting** |
| OQ-P5-1 | Whether any provider exposes charge references | Reachability of charge scope |
| — | Existing open items: D-06, D-07, D-09, D-14, D-17, D-18, D-20 | Unchanged by this review |

---

## 5. Founder decision summary

### 5.1 Ready for approval now

| Decision | Recommendation | Business consequence |
|---|---|---|
| **P-3 model** | Two recognition states + typed reduction events, replacing five buckets | Same net shipping revenue in every case, fewer concepts, and a refunded unit no longer loses the fact that it shipped. No merchant-visible number changes except where the old model would have hidden the recognition fact |
| **P-1 R9′** | One method per charge with granularity; vector not label in identity | Smaller than rev 1 (one decision replaces three bases). Prevents two orders with different attribution being compared as equivalent |
| **P-7b** | `(line_identity, unit_index)`, byte + integer comparison, index retained on cancellation | Product figures become reproducible across runs. Without it, the same order can produce different product figures |
| **P-4 monotonicity** | Downward charge correction is a reduction, not a `CB` decrease | Removes an unexplained negative from the control |
| **P-4 reality-over-ceiling** | Record real refunds in full; classify the excess | The ledger agrees with the bank. Rev 1's quarantine would have created a permanent unexplained difference |
| **P-4 order-insensitivity** | No ordering rule needed for the ceiling | Removes a class of non-reproducibility and one test obligation |
| **INV packaging** | One new invariant (PROP-C), three folded | Four proposed additions become one. A register with non-load-bearing entries is harder to enforce |
| **MVP boundary** | §1 mandatory, §2 pilot, §3 later | Build order is settled; nothing financial is deferred into silence |

### 5.2 Requires further evidence

| Decision | Missing evidence | Effect of deciding without it |
|---|---|---|
| **P-1 basis reachability** | OQ-P1-1/P1-2 — does any provider expose line-level association? | If none does, `A1`/`A2` are unreachable and every order is `A3`. The mechanism is still worth building (it makes pooled a recorded decision), but the merchant-facing claim must say product shipping attribution is pooled |
| **P-5 charge scope** | OQ-P5-1 — charge references in provider payloads | Building it without evidence is dead code |
| **CX-4 deferral window** | Real webhook ordering behaviour (P-02) | A window chosen without evidence will be wrong in one direction |
| **F-01 / D-09 / INV-02** | Genuine Shopify refund fixture | Unchanged. Still open, still blocking the merchandise refund basis |

### 5.3 Should be MVP requirements

M1–M16 of §1. The three most load-bearing, in business terms:

1. **M2 (vector in identity)** — without it the product can compare two merchants' or two periods' product margins that were computed on different attribution bases and present the comparison as meaningful.
2. **M9 (reality-over-ceiling)** — without it, the ledger disagrees with the bank whenever a merchant over-refunds, which is the single fastest way to lose a merchant's trust in a financial product.
3. **M10 (deferral on missing charge evidence)** — without it, ordinary webhook reordering generates false "over-refund" flags, which trains merchants to ignore alerts.

### 5.4 Deferrable without compromising integrity

- Correction workflow (P-1 §6) — provided M1–M4 make the original decision reproducible, a late correction can be added later without restating history
- `G-unit` / `G-shipment` granularity
- Charge scope
- Late-evidence flags
- Merchant-selectable basis
- Mass/dimension weighting

**Each is deferrable only because the MVP fails closed or marks the amount unattributed in its absence.** None is deferrable by silence.

---

## 6. What this review did not do

- No decision status was changed. D-10a remains APPROVED WITH CONDITIONS; C-10a.1 remains PARTIALLY SATISFIED — OPEN.
- No invariant number was allocated.
- No test was executed. All arithmetic claims are recomputations of document figures, not test runs.
- No code, schema, migration, Supabase connection, provider connection or deployment.
- The INV overlap matrix covers 14 of 27 invariants; the other 11 were not read.
