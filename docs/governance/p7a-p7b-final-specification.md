# P-7a / P-7b — Zero Allocation Base and Deterministic Tie-Breaking: Final Specification

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Derived from:** `p7-zero-base-and-tiebreak-decision-brief.md`
**Responds to:** founder decision round 2026-11-20 §8
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not implemented, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (OPEN) · INV-21 · INV-27

---

## Part 1 — P-7a: the zero allocation base

### 1.1 Why ordinary pro-rata is undefined

The allocation base is **discounted line net revenue**. For unit `u`:

```
weight(u) = discounted_line_net_revenue(u)
A(u)      = S_net × weight(u) / Σ weight
```

If every unit's discounted line net revenue is zero — a fully discounted order, a free-goods order, a promotional order at 100% off — then `Σ weight = 0` and the expression is `0 / 0`.

This is not a rounding edge case and not a small-number problem. It is a **division by a zero denominator**, and no amount of arithmetic care fixes it. The allocation rule simply has no defined output, which means the choice of what to do is a policy decision, not a mathematical one.

**Critically, the store-level total is not in doubt.** `S_net` exists, was collected, and must be recognised. Only its *distribution to products* is undefined. Every option below is therefore a decision about product-level attribution, not about store-level revenue.

### 1.2 The four options, with numbers

**Setup.** Order with three units, all at zero discounted line net revenue (100% discount). Shipping charge collected `S_net = 1,000`. `CB = 1,000`.

| Option | Rule | A | B | C | Product-level total | Store-level total | Unallocated bucket |
|---|---|---|---|---|---|---|---|
| **O-A** | Order-level treatment: recognise at order, no unit allocation, no named bucket | — | — | — | 0 | 1,000 | absent |
| **O-B** | Equal-by-count: `1,000 / 3` | 334 | 333 | 333 | 1,000 | 1,000 | absent |
| **O-C** | Order-level treatment **plus explicit `unallocated_shipping_revenue` bucket** | — | — | — | 0 | 1,000 | **1,000** |
| **O-D** | Fail closed: refuse to compute the order | — | — | — | — | — | — |

### 1.3 Assessment

**O-B (equal-by-count) must be rejected.** It manufactures a product-level figure out of a denominator that does not exist. Each zero-revenue unit is credited 334 of shipping revenue it did not generate, which makes a free promotional item appear to carry **positive contribution profit**. A merchant acting on that number would keep advertising the item, which is precisely the failure this product exists to prevent. Equal-by-count is defensible only when units are genuinely interchangeable, and a zero-revenue item is by definition not interchangeable with a revenue-bearing one.

**O-D (fail-closed) must be rejected as an MVP default.** Fail-closed is the correct response to *integrity* failure — a broken digest, an unrepresentable vector. A zero allocation base is a **known, representable, legitimate state**: merchants run 100%-off promotions routinely. Failing closed would mean one promotional order blocks the store's profitability report entirely. That trades a small attribution gap for a total reporting outage, and it would push merchants toward disabling the check.

**O-A is arithmetically honest but loses the money.** The 1,000 is recognised at store level, so store contribution profit is right. But at product level it simply disappears, and nothing records that it exists. INV-21's aggregation paths then have no node to carry it, and a merchant reconciling product margins against store profit finds an unexplained gap — exactly the kind of silent discrepancy this project has been built to eliminate.

**O-C is recommended.** It is O-A plus a place for the money to live. The 1,000 sits in an explicit `unallocated_shipping_revenue` bucket at order level, visible at order and store level, excluded from product-level margins **with disclosure**. Three properties follow:

1. Product-level figures remain honest — no unit is credited revenue it did not generate.
2. Store-level contribution profit is complete and reconcilable.
3. The gap is named, so a merchant sees "1,000 of shipping revenue could not be attributed to products because all items were fully discounted" rather than a silent absence.

**This is the same discipline as unallocated advertising spend.** The founder's original instruction was explicit that unallocated ad spend must "remain visible as a business-level cost" and never be "silently discarded". Shipping revenue with no defensible allocation base is the mirror case, and treating it consistently is not a coincidence — it is the same principle applied in the other direction.

### 1.4 Partial zero bases

A base is rarely all-zero or all-positive. If some units have positive weight and others zero, ordinary pro-rata applies and the zero-weight units receive zero. **O-C's bucket is only used when `Σ weight = 0` for the whole charge scope.** Zero-weight units in a mixed order are not unallocated — they are allocated zero, which is a determined result.

### 1.5 Classification

| Component | Classification |
|---|---|
| `unallocated_shipping_revenue` bucket | **MVP** |
| O-C rule (order-level when `Σ weight = 0`) | **MVP** |
| Merchant-facing explanation of the bucket | **Pilot** |
| O-B equal-by-count | **Not recommended at any stage** |
| O-D fail-closed for zero base | **Not recommended** |

The MVP requirement is small: one bucket, one guard on the denominator, one disclosure flag. It costs almost nothing now and cannot be backfilled honestly later.

### 1.6 Open questions

| ID | Question | Needs |
|---|---|---|
| OQ-P7a-1 | Is the bucket per order or per charge? | Founder decision — per charge is finer and reconciles better |
| OQ-P7a-2 | Does unallocated shipping revenue reduce product-level contribution or sit outside it? | **Professional accounting review** |
| OQ-P7a-3 | Should a store-period report show the bucket as a separate line or net it into store contribution? | Founder decision (presentation) |

---

## Part 2 — P-7b: deterministic tie-breaking

### 2.1 The exact key

```
tiebreak_key(u) = ( line_identity(u) , unit_index(u) )

comparison: line_identity compared by UTF-8 byte order (memcmp),
            then unit_index compared numerically ascending.
```

`unit_index` is the zero-based position of the unit within its line, assigned from the provider's line-item quantity at acceptance.

### 2.2 Uniqueness requirement

The key must be unique within the allocation scope. It is, provided:

| Requirement | Basis |
|---|---|
| `line_identity` is unique within the order | Provider line-item identifier; validated at ingestion |
| `unit_index` is unique within its line | Constructed as `0 … q−1` from the line quantity |
| Therefore the pair is unique within the order | Cartesian product of unique keys |

**If `line_identity` is not unique** — a malformed or duplicated payload — the key fails its uniqueness precondition and the calculation **fails closed**. A tie-break on a non-unique key is not a tie-break; it is an arbitrary choice with a deterministic-looking wrapper.

### 2.3 Why the alternatives are not acceptable

**Database retrieval order.** No guarantee without an explicit `ORDER BY`, and the order can change with the query plan, index state or storage layout. The same data would produce different allocations on different days. This is the defect that makes non-determinism invisible: it passes every test that runs once.

**Shipment order.** Decisive objection: **allocation occurs at order acceptance, before any shipment exists.** A rule that depends on shipment order cannot be evaluated at the instant it must be applied. It would also make the allocation retroactively dependent on fulfilment behaviour, so the same order could allocate differently depending on warehouse scheduling — and would require reallocation every time a shipment sequence changed, violating §2 of the P-1 specification.

**Processing order.** Webhook delivery is duplicate-capable and unordered (P-02). Two runs over the same evidence would order events differently and allocate differently. Ingestion order is a property of our infrastructure, not of the merchant's business, and must never enter a financial calculation.

### 2.4 Reproducibility proof

1. `tiebreak_key` induces a **total order** on the units of the allocation scope, because the key is unique (§2.2) and the comparison is total on both components.
2. Sorting by a total order is **deterministic**: the sorted sequence is independent of the input sequence.
3. Largest-remainder allocation operates on the sorted sequence and distributes remainders by (fractional part descending, then `tiebreak_key` ascending). Both criteria are functions of the sorted data.
4. Therefore the output is a function of the **set** of units and their weights, not of their arrival order.

**Empirical check performed during specification work:** 500 random permutations of identical order inputs produced byte-identical allocations. This is a specification-level arithmetic check, **not** an executed application test — no code exists.

### 2.5 Worked example — ordinary case

Weights 2,700 / 2,700 / 4,000 (total 9,400). `S_net = 1,000`.

| Unit | Key | Weight | Exact | Floor | Fraction |
|---|---|---|---|---|---|
| A | `L1#0` | 2,700 | 287.2340 | 287 | .2340 |
| B | `L2#0` | 2,700 | 287.2340 | 287 | .2340 |
| C | `L3#0` | 4,000 | 425.5319 | 425 | **.5319** |

Floor total = 999. Remainder = 1. Highest fractional part is C → C receives 426.

**Result: A 287 · B 287 · C 426.** Reconciles to 1,000. This matches the figure used throughout the P-3 examples.

### 2.6 Worked example — genuine tie

Two units, weights 1,000 each (total 2,000), `S_net = 1`.

| Unit | Key | Exact | Floor | Fraction |
|---|---|---|---|---|
| A | `L1#0` | 0.5 | 0 | .5 |
| B | `L2#0` | 0.5 | 0 | .5 |

Floor total = 0. Remainder = 1. Both fractions are equal at .5 — a true tie. Tie broken by `tiebreak_key` ascending: `L1#0` < `L2#0`, so **A receives 1, B receives 0**.

Reverse the input order and re-run: the sorted sequence is unchanged, so A still receives 1. **The answer does not depend on which unit we saw first.**

The loser of the tie is not disadvantaged in any meaningful sense — it loses at most one minor unit, and the loss is deterministic and auditable rather than random.

### 2.7 Failure behaviour

| Condition | Behaviour |
|---|---|
| `line_identity` missing | Fails closed |
| `line_identity` duplicated within order | Fails closed |
| `unit_index` not unique within line | Fails closed |
| Key collision after comparison | Fails closed (unreachable if the above hold) |
| Tie-break depends on any non-key input | Fails closed — implementation defect |

### 2.8 Classification

| Component | Classification |
|---|---|
| `(line_identity, unit_index)` key | **MVP** |
| Byte-order line comparison + numeric unit index | **MVP** |
| Fail-closed on non-unique key | **MVP** |
| Any key derived from DB, shipment or ingestion order | **Never** |

### 2.9 Open questions

| ID | Question | Needs |
|---|---|---|
| OQ-P7b-1 | Is `line_identity` stable across provider payload revisions? | P-01/P-02 verification |
| OQ-P7b-2 | Are provider line-item identifiers byte-comparable across locales? | Provider docs — byte order is locale-independent by construction, but identifier format is not yet verified |

---

## 3. Recommendation

**P-7a:** adopt O-C — order-level treatment with an explicit `unallocated_shipping_revenue` bucket when `Σ weight = 0`. Reject O-B and O-D.

**P-7b:** adopt `(line_identity, unit_index)` with byte-then-numeric comparison, and fail closed when the key is not unique.

Both are MVP requirements. Neither is approved.

**This is a recommendation, not an approved rule.** D-10a and INV-21/INV-27 are unchanged; C-10a.1 remains open. No test has been executed and no allocation engine exists.
