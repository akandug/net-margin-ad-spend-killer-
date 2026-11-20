# INV-27 — Calculation Identity and Cross-View Reconciliation Specification

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Invariant:** INV-27 — a financial figure is meaningless without its full calculation identity; views may never be silently aggregated or compared
**Founder direction, 2026-11-20:** **principle approved** as an architectural principle, subject to detailed specification and test design. `calculation_view` is part of the calculation identity **together with** tenant, store, reporting currency, period/timezone, calculation version and source-data boundary. Fail closed when the view is missing. Cross-view comparison only through an explicitly identified, reproducible reconciliation calculation that explains timing and quantity differences.
**Status after this document:** **principle approved; specification delivered; test design outstanding**
**Gate:** G2
**Prepared:** 2026-11-20

---

## 1. The identity tuple

A financial figure exists only as the output of a calculation over a fully specified identity. The identity has eight elements; omitting any one of them produces a number that cannot be reproduced, compared, or defended.

```
calculation_identity = (
    tenant_id,
    store_id,
    reporting_currency,
    period,
    timezone,
    calculation_version,
    source_data_boundary,
    calculation_view
)
```

| Element | Why it is in the identity | Failure if omitted |
|---|---|---|
| `tenant_id` | Isolation (INV-23). Amounts from different tenants are never comparable | Cross-tenant contamination |
| `store_id` | A merchant may run several stores with distinct currencies, catalogues and cost bases | Stores silently merged |
| `reporting_currency` | D-05. Amounts in different currencies cannot be summed | Nonsense totals |
| `period` | The reporting window | Figures compared across different windows |
| `timezone` | A period boundary is undefined without it: 2026-01-31 23:00 in one zone is February in another | Period membership flips by zone |
| `calculation_version` | INV-15. Rules change; results must be attributable to the rules that produced them | Two rule sets silently mixed |
| `source_data_boundary` | The as-of point of the source events included | **See §2 — this is the subtlest and most important element** |
| `calculation_view` | F-1. Order/Commercial and Fulfilment/Realisation measure different things | The defect this invariant exists to prevent |

### 1.1 Why `source_data_boundary` belongs here

Two runs over *the same* period, currency, store and calculation version can legitimately produce different answers, because events arrive late. A refund processed on 3 March may arrive on 5 March. A period computed on 4 March and again on 10 March will differ.

Without a boundary in the identity, both results claim to be "February contribution profit, version 3" and are irreconcilable — you cannot tell a genuine correction from a late arrival. With it, each result is a distinct, reproducible artifact, and the difference between two boundaries is itself an explainable, auditable quantity.

**Definition:** `source_data_boundary` is the maximum ingestion sequence number included in the calculation. Ingestion sequence is monotonic and assigned at ingestion time (INV-13), so it is total and stable — unlike source timestamps, which are unordered and can be backdated.

**Consequence:** a restatement is not "the same calculation run again". It is a **new identity** whose boundary is later, whose delta against the prior identity is computed explicitly, and whose delta is reported to the merchant. This is the mechanism that satisfies "no silent historical change".

---

## 2. Fail-closed rules

**INV-27.1 — No defaults.** A calculation requested without any one of the eight elements returns an error. No element may be defaulted from configuration, from the most recent prior request, or from the store record. A missing view is not "the default view"; it is an error.

**INV-27.2 — Missing view specifically.** `calculation_view` absent, null, empty or unrecognised → hard error. This is called out because it is the element most likely to be "helpfully" defaulted by a future contributor, and doing so would silently reintroduce the exact defect F-1 removed.

**INV-27.3 — No implicit coercion.** A stored figure with identity A may not be read, displayed, aggregated or compared as though it had identity B. Conversion of currency is a calculation with its own identity, not a display transform.

**INV-27.4 — Aggregation only within an identity.** Aggregation across scope (order → line → product → campaign → store → merchant) is permitted only where every contributing figure shares an identical identity apart from scope. Scope is recorded on the result but is **not** part of the identity tuple — see §7, open question OQ-1.

**INV-27.5 — Aggregation requires identical version.** Two figures with different `calculation_version` may not be summed. They may only be compared through a restatement reconciliation.

**INV-27.6 — Cross-view is never a subtraction.** `CP_commercial − CP_fulfilment` computed directly is forbidden as a merchant-visible figure. The difference is only ever presented as a reconciliation artifact with its components (§3).

**INV-27.7 — Fail closed, not degraded.** Where identity is incomplete or ambiguous, the system returns no figure and explains why. It does not return a figure with a warning attached; a warning is ignored, an absent number is not.

---

## 3. The cross-view reconciliation artifact

Where the two management views must be compared, the comparison is its own identified calculation with its own identity:

```
reconciliation_identity = (
    tenant_id,
    store_id,
    reporting_currency,
    period,
    timezone,
    calculation_version,
    source_data_boundary,
    view_pair                  -- e.g. (commercial, fulfilment)
)
```

It is **reproducible**: the same inputs and boundary produce the same artifact, including the same component amounts. It is **stored**, so a merchant-visible comparison can be re-examined later.

### 3.1 Component decomposition

Let Δ(P) = CP_commercial(P) − CP_fulfilment(P) for period P.

Δ(P) is decomposed into exactly four buckets, which must sum to Δ(P) with no residual. If they do not, the reconciliation **fails** — it does not absorb the difference.

| Bucket | Meaning | Value |
|---|---|---|
| **A — Quantity, still open** | Units ordered in P and unfulfilled at the end of P | `+ (net revenue + allocated shipping − order-date COGS)` |
| **B — Timing, shipped later** | Units shipped in P but ordered in a different period | `− (net revenue + allocated shipping − shipment-date COGS)` |
| **C — Snapshot difference** | Units ordered **and** shipped in the same P | `+ (shipment-date COGS − order-date COGS)` |
| **D — Quantity, released late** | Units ordered in an earlier period and released before shipment in P | `− (net revenue + allocated shipping − order-date COGS)` |

Bucket A is the quantity difference; B and D are timing differences; C is the valuation-snapshot difference. Together they explain the gap in terms a merchant can act on: *goods not yet shipped*, *goods shipped in a different month than ordered*, *cost changed between order and shipment*.

### 3.2 Cumulative closure — the strongest check

For an as-of point T:

```
Δ_cumulative(T)  =  Σ (shipment-date COGS − order-date COGS)  over units shipped by T
                 +  Σ (net revenue + allocated shipping − order-date COGS)
                    over units ordered by T, still unfulfilled at T, not released
```

**And over the full lifecycle, once every unit is either shipped or released:**

```
Δ_cumulative(final)  =  Σ (shipment-date COGS − order-date COGS)  over shipped units
```

**Units that never shipped contribute exactly zero in perpetuity.** They are recognised in the Commercial View and then reversed; the Fulfilment View never sees them. Any permanent non-zero contribution from a never-shipped unit is a defect, and this identity detects it. The two views are therefore not merely "different" — they differ by a quantity that is fully accounted for.

---

## 4. Worked reconciliation example

All amounts in minor units. Periods are calendar months, timezone UTC, currency fixed.

**Order O1** — ordered 2026-01-28, shipped 2026-02-03.
Two units, net revenue 2,700 each. Shipping revenue 600 (300 allocated per unit, per D-10a E1 mechanics). Order-date COGS 700/unit. Shipment-date COGS 750/unit.

**Order O2** — ordered 2026-01-29, cancelled before shipment, refund completed 2026-02-02.
One unit, net revenue 1,000, no shipping revenue. Order-date COGS 300.

### January

| View | Revenue | COGS | CP |
|---|---|---|---|
| Commercial | O1 6,000 + O2 1,000 = 7,000 | 1,400 + 300 = 1,700 | **5,300** |
| Fulfilment | 0 | 0 | **0** |

**Δ(Jan) = 5,300**

| Bucket | Amount |
|---|---|
| A — ordered in Jan, unfulfilled at Jan end (O1 4,600 + O2 700) | **+5,300** |
| B | 0 |
| C | 0 |
| D | 0 |
| **Σ** | **5,300** ✓ |

### February

| View | Revenue | COGS | CP |
|---|---|---|---|
| Commercial | O2 reversal −1,000 | O2 reversal −300 | **−700** |
| Fulfilment | O1 6,000 | 1,500 | **4,500** |

**Δ(Feb) = −700 − 4,500 = −5,200**

| Bucket | Amount |
|---|---|
| A | 0 |
| B — shipped in Feb, ordered in Jan: −(6,000 − 1,500) | **−4,500** |
| C | 0 |
| D — ordered in Jan, released in Feb: −(1,000 − 300) | **−700** |
| **Σ** | **−5,200** ✓ |

### Cumulative

| Measure | Amount |
|---|---|
| CP_commercial, Jan + Feb | 5,300 − 700 = **4,600** |
| CP_fulfilment, Jan + Feb | 0 + 4,500 = **4,500** |
| Δ_cumulative | **100** |
| Snapshot difference: (750 − 700) × 2 units | **100** ✓ |

The two views differ by exactly the cost-snapshot movement. O2 contributes nothing permanently, as required by §3.2.

**This is the test the engine must satisfy.** It is checkable arithmetic, not an assertion of correctness.

---

## 5. What the reconciliation must never do

| Anti-pattern | Why it is forbidden |
|---|---|
| Absorbing an unexplained difference into a "rounding" or "other" line | Destroys the value of the reconciliation and hides defects |
| Presenting Δ directly as "unrealised profit" or similar | Invents an economic concept that was never defined |
| Recomputing the Commercial View using shipment dates to force agreement | Destroys the very difference the two views exist to expose |
| Choosing the view that "looks right" per merchant | Makes the figure unfalsifiable |
| Comparing across views without an artifact identity | Violates INV-27.6 |

---

## 6. Required test coverage

Detailed design is outstanding; the required coverage follows from the above.

1. Each of the eight identity elements: omit it, receive an error, not a default. **Especially `calculation_view`.**
2. Same identity, same inputs → identical output (reproducibility).
3. Same period, later boundary → a *new* identity, with the delta computed and attributed to late arrivals.
4. Aggregation within an identity succeeds; aggregation across versions, currencies, tenants or views fails.
5. §4's example reproduces Δ(Jan) = 5,300, Δ(Feb) = −5,200, Δ_cumulative = 100.
6. A never-shipped unit contributes zero to Δ_cumulative; a synthetic case that makes it non-zero fails the check.
7. A deliberately mis-decomposed Δ (buckets not summing to Δ) fails the reconciliation rather than absorbing the residual.
8. A cross-view figure requested without an artifact identity is refused.

**None of these tests exists.** No database, schema or code exists.

---

## 7. Open questions

**OQ-1 — Is `scope` part of the identity?** This specification treats scope (order / line / product / campaign / store / merchant) as a reported dimension over calculations that share an identity, not as an identity element. The alternative — including scope — is stricter and makes cross-scope aggregation a reconciliation in its own right. **Recommendation: keep scope out of the tuple for the MVP**, but record it on every stored figure so the stricter model can be adopted later without data loss. Requires founder confirmation.

**OQ-2 — Boundary granularity.** This specification uses the maximum ingestion sequence included. An alternative is a timestamp cutoff, which is more human-readable but non-monotonic where ingestion is concurrent. **Recommendation: ingestion sequence**, with the corresponding timestamp recorded alongside for display.

**OQ-3 — Should a period be closable while units remain unfulfilled?** Bucket A can be arbitrarily large if orders are placed and not shipped across a period boundary. Whether a period may be *closed* with a large bucket A is a management-reporting policy question tied to D-18, not a technical one. **Flagged for founder decision.**

---

## 8. Status

| Item | Status |
|---|---|
| View is part of calculation identity | **Approved by founder** |
| Eight-element identity tuple | Proposed here |
| Fail-closed behaviour | **Required by founder**; rules proposed here |
| Reconciliation artifact and decomposition | Proposed here |
| Cumulative closure identity | Proposed here |
| Test design | **Outstanding** |
| Implementation | None |
| Empirical verification | None |

---

## 9. Boundary statement

Documentation only. No application code, schema, migration, test, Supabase provisioning, provider connection, merchant data or deployment. The arithmetic in §4 is derived from the stated inputs and reconciled in-line; it is a specification worked example, not output from an implemented engine.
