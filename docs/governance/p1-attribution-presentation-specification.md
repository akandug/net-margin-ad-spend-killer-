# P-1 — Product Attribution Presentation Specification

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision clarification 2026-11-20 §4
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not implemented**
**Affected decisions:** D2-01, D2-02, D2-03, D2-14 · D-10a (APPROVED WITH CONDITIONS)

---

## 1. The problem this section solves

Four different things can be true of shipping revenue on the same order, and three of them are "not evidenced". If the interface shows them alike, a merchant will read a pooled estimate as provider-attested fact — which is the failure mode the whole platform exists to prevent, applied to shipping instead of advertising.

The requirement is therefore not merely to compute four categories correctly, but to make them **structurally impossible to confuse**.

---

## 2. The four categories

| Category | Produced by | Evidenced? | Where the money sits |
|---|---|---|---|
| **Directly associated** | `method = D`, `granularity ∈ {G-unit, G-line, G-shipment}` | **Yes** — the provider record names the order, line or unit the charge belongs to | In the product's own shipping revenue |
| **Pooled** | `method = P`, `granularity = none` | **No** — no provider association exists; allocated by deterministic rule | In the product's shipping revenue, labelled as allocated |
| **Unallocated** | Any charge whose scope has `Σ weight = 0` (O-C, D2-14) | **No, and not allocable** | In an explicit bucket; in **no** product |
| **Invalid or conflicting evidence** | Association present but malformed, contradictory, or arriving after acceptance (D2-04) | **No** — evidence exists but is unusable | Degrades to **pooled**; never direct |

---

## 3. Product-level presentation

| Category | Appears in the product line? | Required label | Claim permitted to the merchant |
|---|---|---|---|
| Directly associated | Yes | `Shipping revenue — direct` | "Attributed from your shipping provider's records." |
| Pooled | Yes | `Shipping revenue — pooled allocation` | "Allocated across items by a fixed rule. Not evidenced per item." |
| Unallocated | **No** | — (order and store level only) | "Not attributable to any item. Excluded from item margins." |
| Invalid / conflicting | Yes, as pooled | `Shipping revenue — pooled allocation` **plus** a contested flag | "The provider's association data was inconsistent, so this was pooled." |

**The contested flag must travel with the figure.** Wherever a product-level shipping figure is displayed, exported or summarised, a pooled figure derived from invalid evidence carries its flag. A flag shown only on a detail screen that most merchants never open is not a disclosure.

---

## 4. Store-level presentation

All four categories roll up, and the store result is **reconciled regardless of the method vector**:

```
Σ direct + Σ pooled + Σ unallocated = S_net
```

This is the property that makes the store figure trustworthy while the product figures remain uncertain. Verified across the worked method vectors at a store total of **11,200** — the total does not move when the association method changes; only the split between the three buckets moves.

**Worked order** (shipping charges 1,000 + 200 = 1,200; products A, B, C; allocation weights 100 / 9,900):

| Vector | A | B | C | Unallocated | Total |
|---|---|---|---|---|---|
| `c1` direct to A, `c2` pooled | 1,002 | 198 | 0 | 0 | **1,200** |
| Fully pooled | 12 | 1,188 | 0 | 0 | **1,200** |
| Zero allocation base (O-C) | 0 | 0 | 0 | 1,200 | **1,200** |

Product figures move by 990 between the first two rows. The store total does not move at all. Both facts must be visible together, or the merchant will mistake stability at store level for confidence at product level.

---

## 5. Keeping pooled and unallocated distinct

Both are "not evidenced", so a naive interface collapses them. Three structural separations, each of which must hold independently:

| Separation | Pooled | Unallocated |
|---|---|---|
| **Field** | Written into `product_shipping_revenue` | Written into `unallocated_shipping_revenue`; no product field is written |
| **Arithmetic role** | Affects that product's contribution margin | Affects **no** product's margin |
| **Claim** | "Allocated by rule" | "Not attributable" |

A single separation is not enough. If the two share a field but differ by label, an export or an API consumer will merge them. If they differ by field but share a label, the merchant will merge them.

---

## 6. The reconciliation line — mandatory

If product margins exclude unallocated revenue while store contribution profit includes it, the product margins **will not add up to the store figure**, and a merchant will read that as a bug or as lost money.

Therefore any view showing both must show the bridge explicitly:

```
Product contribution profit (all items)        X
+ Unallocated shipping revenue                 Y   ← not attributable to any item
= Store contribution profit before ad spend    X + Y
```

Suppressing this line to make the numbers "tidy" would be the presentation equivalent of the balancing plug the D-10a audit rejected.

---

## 7. What must not be done

| Prohibited | Why |
|---|---|
| Showing pooled revenue as "direct" | Asserts evidence that does not exist |
| Distributing unallocated revenue to products by equal-by-count (O-B) | Credits a fully discounted item with revenue, making it appear contribution-positive — rejected in D2-14 |
| Presenting unallocated revenue as a per-product row of zero | Implies the item was considered and scored zero, rather than not attributable |
| Comparing two products attributed under different methods without disclosure | Violates INV-27 and D2-02: different vectors are different calculation identities |
| Omitting the reconciliation line | Makes correct arithmetic look like an error |
| Silently reclassifying invalid evidence as direct after a later correction | D2-04: late evidence annotates, never reallocates |

---

## 8. Fail-closed boundary

An **unrepresentable vector** (`P × G-*`, `D × none`) is not a fifth presentation category — it is an integrity failure. The calculation must refuse to produce a product-level figure rather than fall back to a default. Falling back would turn a validation defect into a silent financial estimate.

This is the distinction from §2: invalid *evidence* degrades to pooled because pooling is an honest representation of uncertainty; an invalid *vector* is a defect in our own construction and must stop the calculation.

---

## 9. Decision requested

| # | Decision | Recommendation |
|---|---|---|
| R3-A1 | Approve the four-category presentation model and the required labels in §3 | Approve |
| R3-A2 | Approve that the contested flag travels with every pooled figure derived from invalid evidence | Approve |
| R3-A3 | Approve the mandatory reconciliation line at §6 | Approve |
| R3-A4 | Confirm that unallocated revenue is shown per charge scope, not aggregated per order (OQ-P7a-1) | Your call — see §6 of the P-7a display specification |
| R3-A5 | Confirm the prohibited list at §7 is binding on interface design | Approve |

**Nothing here is implemented.** No interface exists, and no figure in this document has been produced by a running system.
