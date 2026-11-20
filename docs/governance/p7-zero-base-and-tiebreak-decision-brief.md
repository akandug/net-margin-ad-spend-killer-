# P-7a / P-7b — Zero Allocation Base and Tie-Breaking

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not applied, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (OPEN) · INV-27

---

## 1. Why this brief is separate

Rev 1 introduced both rules inside worked examples, which is exactly the failure mode you warned about: *"Do not introduce a new financial allocation rule through a worked example alone."* A worked example can make a rule look inevitable by choosing inputs that flatter it. These are allocation rules that change reported product profitability, so they are decided here, on their own, before any example that depends on them.

---

# P-7a — Zero allocation base

## 2. Why ordinary pro-rata allocation is undefined

D-10a allocates `S_net` across units by net revenue weight:

```
A(u) = S_net × w(u) / Σ w
```

Largest-remainder arithmetic requires `Σ w > 0` as a denominator. Net revenue weights are non-negative, so `Σ w = 0` implies every `w(u) = 0`. Every term is then `0/0`.

This is not a rounding inconvenience. It is the absence of a proportion: with all weights equal and all equal to zero, there is **no ratio to preserve**. Any split is a convention imposed by the engine, not a proportion implied by the data. Calling it "pro-rata" would be false.

**Is it reachable?** Yes — 100%-discounted lines, free promotional items and zero-price gifts, any of which may still carry a shipping charge. Uncommon, but real, and silently dividing by zero or defaulting to zero would corrupt the whole order's product-level figures.

## 3. Options compared

| Option | Rule | Advantages | Disadvantages |
|---|---|---|---|
| **O-A** equal-by-count | each unit receives `S_net / n` | Deterministic, trivial, no data dependency. Defensible as the limit of proportionality among equals | Attributes shipping **revenue** to a product that generated none. A free gift appears to earn `S_net/n` of shipping revenue, inflating its contribution profit — the precise figure an ad-spend product is used to judge |
| **O-B** value-independent weighting (mass, dimensions) | weight by physical attribute | Arguably more "fair" for freight | Requires data not reliably available from any MVP provider; when mass is missing it creates a *second* zero-base problem, recursively. Adds a provider dependency for a rare case |
| **O-C** order-level treatment | hold `S_net` as an order-level line; no unit attribution | Honest: no defensible basis, so no pretence. Amount stays visible, is not discarded, and store-level totals are unaffected. Small, contained branch | Product-level figures for that order exclude shipping revenue entirely; requires a recognition-timing rule |
| **O-D** fail-closed | refuse product-level calculation for the order; mark provisional | Strictest; no misleading number can be produced | Blocks store-level reporting unnecessarily — the store total is unaffected by the attribution choice, so refusing it punishes the merchant for a product-level problem |

## 4. Recommendation — O-C, order-level treatment

Reasoning, in order of weight:

1. **O-A is arithmetically defensible but commercially misleading.** It gives a zero-revenue SKU a positive shipping revenue share. For a product whose entire purpose is deciding whether to keep advertising a SKU, inflating that SKU's contribution profit is worse than declining to attribute. O-A is the natural mathematical limit, but the limit is not the right report.
2. **O-C fails closed at the level where attribution is unsupported, and nowhere else.** The store total is invariant under any attribution choice (proved in the P-1 brief §8.2), so there is no integrity reason to suppress it. O-D suppresses it anyway.
3. **The amount remains visible.** O-C satisfies your standing rule that a deferred feature must not create silent misstatement: the shipping revenue is reported, explicitly labelled as unattributed, not dropped.
4. **O-B depends on data we cannot obtain.** Deferring it costs nothing today.

**Recognition timing for O-C:** the order-level shipping revenue is recognised when the order reaches a terminal fulfilment state — every accepted unit shipped or cancelled. Before that, the portion relating to undelivered goods is unknown, and recognising early would front-load revenue that may later be released. This is simple and deterministic.

**O-D as your alternative:** if you would rather show nothing than an unattributed number for those orders, O-D is coherent and I would implement it. It is a product judgement, not a technical one — which is why it is your decision rather than my recommendation.

## 5. MVP versus pilot

| Item | Classification |
|---|---|
| O-C branch + terminal-state recognition | **MVP** — a small branch; without it the order's figures are wrong or undefined |
| O-D as an alternative | Founder choice, same effort |
| O-A | **Not recommended** |
| O-B mass/dimension weighting | **Later**, only if a provider exposes reliable attributes |

## 6. Unresolved

| ID | Question | Needs |
|---|---|---|
| OQ-P7a-1 | O-C or O-D? | **Founder decision** |
| OQ-P7a-2 | Is unattributed shipping revenue disclosed per order or only in aggregate? | Founder decision (presentation) |
| OQ-P7a-3 | Does order-level unattributed shipping revenue belong in product-level margin reporting at all, or only at store level? | **Professional accounting review** (interacts with D-18 period policy) |

---

# P-7b — Tie-breaking in largest remainder

## 7. When it is needed

Largest remainder allocates floors, then distributes the remaining units to the largest fractional remainders. When two or more remainders are **equal**, something must choose. Without a stated key the result depends on iteration order — which varies with database retrieval order, hash ordering, or processing sequence.

Rev 1 used an unspecified key. That is a reproducibility defect: the same order could produce different product-level shipping revenue across runs, and INV-27's promise of reproducible calculations would be false in a way that is extremely hard to diagnose.

## 8. The key

**`(line_identity, unit_index)`**, compared as:

1. `line_identity` — the provider's line-item identifier, compared as ascending **UTF-8 byte sequence**
2. `unit_index` — integer `0 … q−1` within the line, compared **numerically**

### 8.1 Why `line_identity` is stable

Provider line-item identifiers are assigned by the provider, unique within the order, and unchanged across payload revisions. They are not positional array indices. This is the same requirement as P-1's validity test §4 item 4.

If a provider genuinely exposes no stable line identifier, the fallback is a content hash of the line's immutable fields — but this must be recorded as a data-quality degradation, because two genuinely identical lines would collide and require a positional discriminator, which is exactly what we are trying to avoid.

### 8.2 Why `unit_index` is not an arbitrary sequence

`unit_index` is assigned **once, at order acceptance, from the accepted quantity** of the line. It is:

- **not** shipment order — shipping unit 2 before unit 1 does not renumber them
- **not** database retrieval order — we sort explicitly before allocating
- **not** processing sequence — it derives from quantity at acceptance, not from when anything was processed

### 8.3 Cancellation must not renumber

When a unit is cancelled or refunded, **its index is retained**. Its allocation is not removed and the surviving units are not renumbered; the cancelled unit's allocation is zeroed by a `cancellation_release` reduction event (P-3). Renumbering would shift every subsequent unit's allocation, silently changing figures that had already been reported.

### 8.4 Comparison discipline

`line_identity` must be compared as bytes, not as a locale-collated string. `unit_index` must be compared as an integer, not as text — otherwise `"10"` sorts before `"2"`. Both are easy to get wrong in a way that only shows up on orders with ten or more units of a line.

### 8.5 Scope

The tie-break is applied to the **full unit set of the allocation scope** — the whole order, or the whole associated scope for a `D` charge — not to whatever subset is currently being processed. A partial-scope sort produces a different result from a full-scope sort, so the scope must be fixed before sorting.

## 9. Why this key satisfies the requirement

The key is a pure function of data fixed at order acceptance. It cannot depend on:

| Artefact | Why it cannot affect the key |
|---|---|
| Database retrieval order | Sorting is explicit and total |
| Shipment order | `unit_index` is assigned at acceptance, not at shipment |
| Processing sequence | No field derives from processing time |
| Array position in the payload | `line_identity` is provider-assigned, not positional |

It is also a **total** order: `line_identity` is unique per line, and `unit_index` is unique within a line, so no two units can share a key. Total order plus a pure function of immutable data gives determinism under any input permutation.

## 10. Verification performed

A throwaway script allocated a 1,000 charge across units with equal fractional remainders under **500 randomised input permutations** and confirmed an identical result each time. A second check confirmed that integer comparison of `unit_index` differs from text comparison on a 12-unit line, demonstrating that the comparison discipline in §8.4 is not decorative.

This is **arithmetic and determinism verification of the rule as stated**. It is not a specification test, not an implementation, and it does not make P-7b approved.

## 11. MVP versus pilot

| Item | Classification |
|---|---|
| Stable key + total order | **MVP** — without it, reproducibility is not achievable |
| Cancellation-preserves-index | **MVP** — otherwise reported figures shift |
| Byte and integer comparison discipline | **MVP** |
| Content-hash fallback for providers without line ids | Pilot, only if a provider requires it |

## 12. Unresolved

| ID | Question | Needs |
|---|---|---|
| OQ-P7b-1 | Do all MVP providers expose stable line-item identifiers? | Provider documentation (P-01/P-02) |
| OQ-P7b-2 | Should the key be exposed to merchants for audit, or internal only? | Founder decision |

---

## 13. Recommendation

**P-7a:** adopt **O-C** (order-level treatment, recognised at terminal fulfilment state). Reject O-A. Keep O-D available if you prefer to show nothing rather than an unattributed figure.

**P-7b:** adopt **`(line_identity, unit_index)`** with byte and integer comparison, full-scope sorting, and index retention on cancellation.

**These are recommendations, not approved rules.** No worked example may rely on either until you approve them — and if you change either, the examples that depend on them must be recomputed rather than patched.
