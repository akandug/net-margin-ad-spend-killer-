# D-10a Condition C-10a.1 — Seven-Example Consistency Audit

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Decision audited:** D-10a — shipping revenue allocation, approved with conditions 2026-11-20
**Condition audited:** C-10a.1 — "the complete specification and all worked examples are internally consistent; the same inputs cannot yield different allocations across documents"
**Mode:** Strict read-only specification audit
**Date:** 2026-11-20
**Status:** No document other than this report was created or modified. No test was executed. No code, schema, migration or integration exists.

---

## 1. Audit scope and exact material examined

| # | File | Sections examined | Lines |
|---|---|---|---|
| 1 | `docs/governance/d10a-shipping-revenue-allocation-decision-brief.md` | All — §1 previous proposal, §2 rules R1–R8, §3 examples E1–E7, §4 guarantees, §5 open question, §6 cost side, §7 status, §8 boundary | 1–238 |
| 2 | `docs/governance/decision-register.md` | §3.5.6 D-10a approval record and conditions C-10a.1–C-10a.5; §5 gate lists | 244–275, gate block |
| 3 | `docs/governance/inv26-quantity-state-specification.md` | Quantity dimensions, state partition, scenarios S1–S2, cancellation handling, fail-closed rules FC-26-1–FC-26-7 | targeted |
| 4 | `docs/governance/inv27-calculation-identity-specification.md` | §2 identity tuple, §3.1 bucket decomposition and sign convention, §3.2 cumulative closure, §4 worked reconciliation | 86–170 |
| 5 | `docs/governance/d20-*` | Not relevant to allocation arithmetic; examined only for retention interaction with the refund-event ledger | — |

**Not examined, and why:** no shipping *cost* specification exists yet, so the revenue/cost interface is auditable only in the direction D-10a §6 states.

**Reproduction:** every calculation below was recomputed from the rules in §2 of the brief using exact rational arithmetic (`fractions.Fraction`), independently of the figures printed in the document. The script is not committed: this is a read-only audit and the repository holds specifications only.

---

## 2. Method

For each example I extracted (a) the ordered quantity set, (b) each unit's net revenue base, (c) the shipping revenue source `S_net`, (d) the events and their dates. I then applied rules R1–R8 mechanically:

1. Build the base set per R2 (net of discount).
2. Compute each unit's exact share `S_net × base_i / Σbase`.
3. Floor every share.
4. Distribute `S_net − Σfloors` to the largest fractional parts, ties broken by unit identifier ascending (R3).
5. Check `Σ allocations = S_net` exactly (R4, R8).

A figure is **verified** only if my independent result equals the document's. Where the document states an invariant, I tested whether the example's own numbers satisfy it.

---

## 3. The seven examples, individually assessed

### E1 — Multiple shipments, pooled allocation

**Inputs as stated:** three units — A (line L1, 1 unit), B (line L1, 1 unit), C (line L2, 1 unit). Net revenue 2,700 / 2,700 / 4,000. `Σbase = 9,400`. `S_net = 1,000`. Shipments: A on 2026-02-05, B on 2026-02-18, C on 2026-02-25.

**Independent computation:**

| Unit | Exact share `1000 × base / 9400` | Floor | Fractional part | Allocation |
|---|---|---|---|---|
| A | 287.234043 | 287 | 0.234043 (11/47) | **287** |
| B | 287.234043 | 287 | 0.234043 (11/47) | **287** |
| C | 425.531915 | 425 | 0.531915 (25/47) | **426** |

Σfloors = 999; remainder = 1; largest fractional part is C (0.5319) → C receives it. **Σ = 1,000 = S_net. Reconciles exactly.**

**Result: verified.** Matches the document. Recognition per shipment: 02-05 → 287, 02-18 → 287, 02-25 → 426; cumulative 287 / 574 / 1,000.

**Note on the tie:** A and B are tied at 0.234043, but the remainder is assigned to C, so **the tie-break rule does not decide anything in E1**.

### E2 — Partial cancellation (continues E1)

**Inputs as stated:** unit B cancelled 2026-02-10 before shipment; its 287 is released; recognition is 713.

**Independent computation:** `1,000 − 287 = 713`. **Σ recognised + released = 713 + 287 = 1,000 = S_net. R8 holds.**

The contrast case (reallocating 1,000 over the remaining base 2,700 + 4,000 = 6,700) is also verified: A 402.985 → floor 402 (frac 0.985), C 597.015 → floor 597 (frac 0.015), floors 999, remainder 1 → A. **A = 403, C = 597, Σ = 1,000.** The document's rejection of reallocation is arithmetically justified — the difference is A −116, C +171, i.e. material.

**Result: arithmetic verified, but the example is defective in sequence — see D-02.** E1 ships B on 2026-02-18 and recognises 1,000; E2 places B's cancellation on 2026-02-10 and releases 287. Under the literal reading "continues E1", the order has recognised 1,000 **and** released 287 → `1,000 + 287 = 1,287 > S_net 1,000`, which violates R8 and C-10a.4. E2 is only valid as a **fork** from E1 at the moment of acceptance, which the document does not say.

### E3 — Free shipping

**Inputs as stated:** shipping charged to the customer is 0.

**Independent computation:** `S_net = 0` → every unit's exact share is 0, floors 0, remainder 0, **Σ allocations = 0 = S_net. Reconciles trivially.**

**Result: verified.** The allocation rule is silent and correct. The substantive content of E3 is a *cost-side* fact — free shipping means shipping cost with no shipping revenue, so contribution profit is reduced by the full cost — which belongs to §6 and is not an allocation question. No defect.

### E4 — Shipping discount

**Inputs as stated:** shipping charged 600, discount 500, `S_net = 1,000` after discounting, base unchanged.

**Independent computation:** identical base (2,700 / 2,700 / 4,000) and identical `S_net` to E1 → **allocations identical: 287 / 287 / 426, Σ = 1,000.**

**Result: verified**, and the identity with E1 is the point of the example: a discounted shipping charge allocates exactly as an undiscounted charge of the same net amount. Correct.

### E5 — Multiple shipping charges, direct association

**Inputs as stated:** charge 1 of 600 associated with A and C; charge 2 of 400 associated with B.

**Independent computation — charge 1** (`Σbase = 6,700`):

| Unit | Exact share | Floor | Fraction | Allocation |
|---|---|---|---|---|
| A | 241.791045 | 241 | 0.791045 (53/67) | **242** |
| C | 358.208955 | 358 | 0.208955 (14/67) | **358** |

Σfloors = 599; remainder 1 → A (0.791). **Σ = 600 ✓**

**Independent computation — charge 2** (`Σbase = 2,700`): B = `400 × 2,700 / 2,700 = 400.000000` exactly, floor 400, remainder 0. **B = 400 ✓**

**Differences versus E1:** A `242 − 287 = −45`; B `400 − 287 = +113`; C `358 − 426 = −68`. **Σ differences = 0**; E5 total 1,000 = E1 total 1,000. Both documents' claim of zero-sum difference is verified.

**Result: verified.** This example is also the strongest evidence for defect **D-01**: the same order and the same `S_net` produce two different allocations (287/287/426 pooled vs 242/400/358 directly associated).

### E6 — Incomplete fulfilment at period end

**Inputs as stated:** A and C shipped (287 + 426 = 713), B unshipped, 287 pending.

**Independent computation:** `713 + 287 = 1,000 = S_net. R8 holds**; the pending balance is exactly B's original allocation, not a plug.

**Result: verified.** One wording defect: the text describes "the 287 difference between the views" as if it were the whole view difference for the order. Under INV-27 §3.1, `Δ = CP_commercial − CP_fulfilment`, and 287 is the **shipping-revenue component** of Δ; Δ also contains revenue and COGS timing terms. In an example that isolates shipping only, the two coincide, but the phrasing invites a reader to treat 287 as the total reconciliation. See **D-09**.

### E7 — Shipping refund

**Inputs as stated:** `S_net = 1,000` fully recognised; a shipping refund of 500 is issued.

**Independent computation:** attributable portion `= min(500, 426) = 426`; order-level residual `= 500 − 426 = 74`; recognised net `= 1,000 − 426 = 574`; total refund effect `= 426 + 74 = 500` ✓.

**Result: verified.** No double count: the 426 reverses the unit's allocation once and the 74 is treated as order-level, so `Σ` remains 1,000 gross and 574 net. The example does **not** establish a ceiling on the 74 — see **D-04**.

---

## 4. Cross-example consistency matrix

Checks C1–C10 as specified by the founder. ✓ = satisfied and verified; ✗ = defect; G = gap (no example exercises the rule); n/a = not applicable to that scenario.

| Check | E1 | E2 | E3 | E4 | E5 | E6 | E7 |
|---|---|---|---|---|---|---|---|
| C1 Inputs independently reconstructable | ✓ | ✗ D-02 | ✓ | ✓ | ✓ | ✓ | ✓ |
| C2 Allocation base and discount treatment | ✓ | ✓ | n/a | ✓ | ✓ | ✓ | n/a |
| C3 Allocated once at acceptance | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| C4 Allocation stays attached to original quantity | ✓ | ✓ | n/a | ✓ | ✓ | ✓ | ✓ |
| C5 Partial / multiple shipment treatment | ✓ | ✓ | n/a | n/a | ✓ | ✓ | n/a |
| C6 Unshipped allocations remain pending | ✓ | ✓ | n/a | n/a | ✓ | ✓ | ✓ |
| C7 Cancellation / refund / partial fulfilment | n/a | ✓ | n/a | n/a | n/a | ✓ | ✓ |
| C8 Post-acceptance change as explicit event | n/a | ✓ | n/a | **G** | n/a | ✓ | ✓ |
| C9 No residual forced onto final shipment | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| C10 Rounding reconciles exactly to source | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

**Coverage gaps made explicit by this matrix:**

- **G-1:** No example exercises a **post-acceptance shipping discount** or a **post-acceptance shipping-charge change**. E4's discount is captured at order time, so C8 is untested for the discount case — and R6 asserts it is "not a later event", which conflicts with C-10a.5 (D-03).
- **G-2:** No example exercises the **tie-break rule decisively**. E1 contains a tie between A and B, but the remainder goes to C, so the rule never determines an outcome (D-08).
- **G-3:** No example exercises the **`Σbase = 0` degenerate case** in R7 (D-07).
- **G-4:** No example exercises **partial refund of shipping** (a refund smaller than a unit's allocation), nor **repeated** shipping refunds (D-04).

---

## 5. Cross-document consistency matrix

| # | Interface | D-10a states | Counterpart states | Verdict |
|---|---|---|---|---|
| X1 | Allocation base ↔ INV-27 revenue | Base is net revenue per unit; allocated shipping is revenue in the Fulfilment View | INV-27 §3.1 buckets add "net revenue + allocated shipping" | **Consistent** ✓ |
| X2 | INV-27 worked example shipping figures | E1 method: largest remainder over net revenue | §4 uses 600 over two units → 300 each | **Consistent** — `600 × 2,700/5,400 = 300` exactly, remainder 0. Arithmetic correct for its own inputs. Cross-reference wording loose (D-11) |
| X3 | INV-27 sign convention ↔ E6 | E6: recognised 1,000 vs 713 → 287 pending | `Δ = CP_commercial − CP_fulfilment`; bucket A `= +(net revenue + allocated shipping − order-date COGS)` | **Consistent** ✓ — 287 is + in Δ under the C−F convention. Wording risk only (D-09) |
| X4 | INV-27 cumulative closure ↔ D-10a R8 | `recognised + released + refunded + pending = S_net` | "Units that never shipped contribute exactly zero in perpetuity" | **Consistent** ✓ — a released unit's allocation returns to 0 contribution, matching R8 |
| X5 | INV-26 cancellation ↔ D-10a E2 | E2 cancels **unit B** on 2026-02-10 | INV-26 S2 cancels **unit C** on 2026-02-10 | **Inconsistent narrative** (D-10) — same date, different unit |
| X6 | INV-26 quantity dimensions ↔ D-10a pending | Pending is a balance, not a state | INV-26 separates fulfilment / disposition / payment and refuses a single collapsed state | **Consistent** ✓ — D-10a keeps shipping recognition as a balance, which INV-26 permits |
| X7 | Register C-10a.2/C-10a.5 ↔ D-10a R6 | R6: allocate once, never recomputed | C-10a.5: every post-acceptance change is an explicit event | **Consistent for cancellation/refund; in tension for shipping discount** (D-03) |
| X8 | Register C-10a.1 ↔ D-10a §5 | §5 leaves pooled-vs-direct association open | C-10a.1: same inputs must not yield different allocations | **Not satisfied** — the selector is unspecified and absent from INV-27's identity (D-01) |
| X9 | INV-02 refund ceiling ↔ D-10a shipping refunds | E7 refunds 500 against 426 attributable | INV-02 caps merchandise refunds at line level | **Gap** — no cumulative shipping-refund ceiling (D-04) |
| X10 | R8 order-level ↔ R5 multiple charges | R8 stated at order level only | R5 permits several charges per order | **Incomplete** — per-charge balance not required (D-05) |

**Duplicated rules:** none found. R4 and R8 state the same identity at allocation time and over lifecycle respectively; they are complementary, not duplicated.

**Ambiguous terminology found:** "gross" in the E1 base table (D-06); "the 287 difference between the views" (D-09); "per D-10a E1 mechanics" (D-11).

---

## 6. Defect register

Severity: **High** = can produce a wrong or non-reproducible merchant-visible figure; **Medium** = can produce an inconsistent implementation or mask a defect; **Low** = clarity/coverage only.

| ID | Severity | Defect | Exact location | Evidence |
|---|---|---|---|---|
| **D-01** | **High** | The **pooled-versus-direct-association selector is unspecified and absent from calculation identity.** R5 permits direct association "where the provider exposes the link", but no rule states what input selects R1/R2 over R5, and INV-27's identity tuple (tenant, store, currency, period/timezone, calculation version, source-data boundary, calculation_view) has no slot for it. Two runs over identical data can therefore produce 287/287/426 or 242/400/358 with **no identity difference** — a direct breach of INV-27's reproducibility rule and of C-10a.1. Related: R6 forbids recomputation after acceptance, so a link discovered *after* acceptance has no lawful treatment — reallocating violates C-10a.2, not reallocating preserves an allocation the evidence contradicts. **No rule exists for either branch.** | Brief §2 R5, R6; INV-27 §2 | E1 vs E5 verified allocations differ; identity tuple contains no selector |
| **D-02** | **Medium-High** | **E2 cannot be a continuation of E1.** E1 ships B on 2026-02-18 and recognises 1,000. E2 cancels B on 2026-02-10 and releases 287. Read literally, the order recognises 1,000 **and** releases 287 → 1,287 > `S_net` 1,000, violating R8 and C-10a.4. Valid only as a fork at acceptance, which the heading "continues E1" contradicts. | Brief §3, E1 timeline and E2 heading | `1,000 + 287 = 1,287 ≠ 1,000`; dates 02-10 precede 02-18 |
| **D-03** | **Medium-High** | **R6 contradicts C-10a.5 on shipping discounts.** R6 states a shipping discount is "captured at order time and netted before allocation — not a later event". C-10a.5 requires that *every* change after acceptance, expressly including a shipping discount, produce an explicit event. R6 is correct only for an order-time discount; no rule covers a post-acceptance one, and no example exercises it. | Brief §2 R6; Register §3.5.6 C-10a.5 | Rule texts conflict on their face; gap G-1 |
| **D-04** | **Medium-High** | **No cumulative shipping-refund ceiling.** INV-02 caps merchandise refunds at line level; D-10a imposes no bound on shipping refunds. Verified: three refunds of 500 against `S_net` 1,000 drive recognised to **−500** with nothing blocking. The 74 order-level residual in E7 is unbounded and repeatable. | Brief §3 E7; no ceiling rule in §2 | Recomputed: 1,000 → 500 → 0 → **−500** |
| **D-05** | **Medium** | **R8 is stated only at order level.** With R5 multiple charges, one charge can over-recognise and another under-recognise while the order-level sum still balances, masking the defect. A per-charge identity is required. | Brief §2 R8, R5 | R8 text; E5 is the only multi-charge example and checks aggregate only |
| **D-06** | **Medium** | **The E1 base table is materially ambiguous.** "Gross 3,000" for line L1 (2 units) and "Line discount 600 (2 units)" can be read per line or per unit. The per-line reading gives `Σbase = 6,400` and allocations **188 / 187 / 625** — not 287/287/426. `Σbase = 9,400` disambiguates, but the table should state per-unit values. | Brief §3 E1 base table | Recomputed alternative: 188 / 187 / 625, Σ = 1,000 |
| **D-07** | **Low-Medium** | **R7's `Σbase = 0` rule has no worked example and sits in tension with R2.** Equal-by-unit-count contradicts R2's proportionality principle and has an economic question attached (is equal split right when all merchandise is free?). Verified the mechanism works: 1,000 over three equal bases → 334 / 333 / 333. | Brief §2 R7 | Recomputed; no example |
| **D-08** | **Low** | **The tie-break rule is never decisive in any example.** E1 has A and B tied at 0.234043, but the remainder goes to C. Rule coverage is zero. | Brief §2 R3; §3 E1 | Fractional parts recomputed |
| **D-09** | **Low** | **E6 wording overstates scope.** "The 287 difference between the views" is the shipping component of Δ; INV-27 defines Δ to include revenue and COGS terms too. | Brief §3 E6; INV-27 §3.1 | Bucket A formula |
| **D-10** | **Low** | **INV-26 S2 and D-10a E2 describe the same scenario with different units.** Both cancel on 2026-02-10; INV-26 cancels C, D-10a cancels B. | INV-26 S2; Brief §3 E2 | Text comparison |
| **D-11** | **Low** | **Loose cross-reference** "per D-10a E1 mechanics" in INV-27 §4 — E1's figures are not used; the method is. Correct for its own inputs. | INV-27 §4 | `600 × 2700/5400 = 300` exactly |

**No arithmetic error was found in any of the seven examples.** Every stated total reconciles exactly to its source under the stated rule. The defects are of specification completeness, internal contradiction and reproducibility — not of the presented figures.

---

## 7. Unresolved professional-review questions

| ID | Question | Why it needs a professional | Affects |
|---|---|---|---|
| PR-10a-1 | Is shipping revenue a **separate performance obligation** from the goods, or part of a single combined obligation? | Determines whether allocation across units is even the right model, or whether shipping is recognised on a different basis | D-10a principle; R1 |
| PR-10a-2 | Does **shipping tax** belong in the allocation base, and is it revenue at all? | Tax must not be treated as merchant revenue (INV-03) | Base definition; R2 |
| PR-10a-3 | Treatment where shipping is **charged but the goods never ship** (cancelled after payment, or lost) | Whether the retained shipping charge is revenue or a liability | R6; E2; D-04 |
| PR-10a-4 | **FX basis** for an allocation made in one period and recognised in another | Allocation is fixed at acceptance; rates move | R3; INV-27 period identity |
| PR-10a-5 | Is an **equal-by-count** split (R7, `Σbase = 0`) an acceptable measure when all merchandise is free? | Presentation choice with revenue-recognition implications | D-07 |

These are carried forward from the D-10a approval record; none is resolved by this audit.

---

## 8. C-10a.1 assessment

### **PARTIALLY SATISFIED**

**Satisfied in part — what is verified:**

1. All seven examples reconstruct **exactly** under the stated rule. No figure in the brief is arithmetically wrong.
2. Rounding is resolved once, at allocation, and every example reconciles to `S_net` with no plug (C10 ✓ in all seven).
3. No example forces an unexplained residual onto the final shipment (C9 ✓ in all seven) — the specific defect that caused the previous proposal to be rejected is genuinely fixed.
4. Quantity-level allocations remain attached to their original units through cancellation, partial fulfilment and refund (C4, C6, C7 ✓).
5. Cross-document arithmetic with INV-27 holds, including the cumulative-closure identity: Δ(Jan) 5,300 + Δ(Feb) −5,200 = 100 = 2 × (750 − 700) ✓.

**Not satisfied — why the condition cannot be marked met:**

1. **D-01 is decisive.** C-10a.1 requires that "the same inputs cannot yield different allocations across documents". E1 and E5 demonstrate that the same order and the same `S_net` *do* yield different allocations, and the input that determines which applies is neither specified nor captured in calculation identity. Until the selector is defined and made part of identity, the condition is not met.
2. **D-02 breaks R8 under a literal reading.** An internally consistent specification cannot contain an example whose stated sequence violates its own balance invariant.
3. **D-03 is a direct contradiction** between an allocation rule (R6) and a founder-approved condition (C-10a.5).
4. **Three scenario classes required by the founder are absent** — post-acceptance shipping discount, post-acceptance charge change, and repeated/partial shipping refunds (gaps G-1, G-4), plus two uncovered rules (G-2 tie-break, G-3 degenerate base).

**Consequence for D-10a:** D-10a remains **APPROVED WITH CONDITIONS**. C-10a.1 is **open**, so the condition precedent in the register is not discharged. This does **not** withdraw the approval of the allocation *principle*, and it does not move D-10a to APPROVED in full. No gate status changes.

---

## 9. Proposed corrections — for founder review only

Presented, not applied. No document has been modified.

| # | Resolves | Proposed change | Exact target |
|---|---|---|---|
| P-1 | D-01 | Add rule **R9 — association-evidence selector**: the allocation basis (pooled R1/R2 vs direct R5) is determined by the association evidence present at acceptance, recorded as an explicit input. Add `shipping_association_basis` to INV-27's calculation identity. Define the late-arriving-link treatment as an explicit **adjustment event** carrying the prior allocation, not a reallocation. | Brief §2; INV-27 §2 |
| P-2 | D-02 | Relabel E2 as **"E2 — Partial cancellation (alternative branch from E1 at acceptance)"** and state that E1's shipments do not occur in this branch. Alternatively keep the continuation and correct the cancellation date to precede 2026-02-05. | Brief §3 E2 heading and opening |
| P-3 | D-03 | Amend R6 to: "A shipping discount captured **at order time** is netted before allocation and is not a later event. A shipping discount granted **after acceptance** is an explicit adjustment event under C-10a.5 and does not modify the original allocation." Add a post-acceptance discount example. | Brief §2 R6; §3 new example |
| P-4 | D-04 | Add rule **R10 — shipping refund ceiling**: cumulative shipping refunds for an order may not exceed `S_net`; the attributable portion is bounded by the unit's allocation and the order-level residual by `S_net − Σattributable refunded`. Excess is a **fail-closed** condition, not a negative balance. | Brief §2 |
| P-5 | D-05 | Restate R8 per charge where R5 applies, with the order-level identity as the aggregate check. | Brief §2 R8 |
| P-6 | D-06 | Replace the E1 base table with explicit per-unit values: A net 2,700; B net 2,700; C net 4,000; `Σbase = 9,400`. Remove "gross" or qualify it per unit. | Brief §3 E1 table |
| P-7 | D-07, D-08 | Add two examples: a `Σbase = 0` order (three units, `S_net = 1,000` → 334/333/333), and a case where the tie-break is decisive (two units with equal fractional parts and a remainder of 1). | Brief §3 |
| P-8 | D-09 | Reword E6 to "the 287 **shipping-revenue component** of the difference between the views". | Brief §3 E6 |
| P-9 | D-10 | Align INV-26 S2 and D-10a E2 to the same cancelled unit, or state explicitly that they are separate scenarios. | INV-26 S2 or Brief §3 E2 |
| P-10 | D-11 | Change "per D-10a E1 mechanics" to "using the D-10a largest-remainder method". | INV-27 §4 |

**Recommended sequencing:** P-1, P-3 and P-4 are substantive and should be founder-reviewed before the specification is treated as closed. P-2, P-6–P-10 are editorial and can be applied in one pass afterwards. P-5 depends on P-1.

---

## 10. Evidence and boundary statement

**Verified by computation in this audit:** every allocation in E1–E7; the contrast case in E2; the zero-sum difference in E5; the balance identities in E2, E6 and E7; the alternative per-line reading of the E1 base table; the degenerate case in R7; the tie-break coverage question; the repeated-refund negative balance in D-04; and the INV-27 worked example's closure (5,300 / −5,200 / 100).

**Not verified, and not claimed:** no test has been executed against any system; no code, schema, migration or database exists; no provider behaviour is established; no merchant data has been touched. These are specification arithmetic checks performed on documents, not execution evidence.

**Boundaries observed:** no approved decision was modified; the allocation principle is unchanged; no test was marked executed; no implementation, schema or migration was created; no provider was connected; Supabase was not provisioned; no repository permission or deployment setting was changed; D-10a is **not** claimed implementation-ready.

**Repository change:** this report is the only file added. No existing document was altered.

---

## 11. Status summary

| Item | Status |
|---|---|
| C-10a.1 | **PARTIALLY SATISFIED — open** |
| Arithmetic of E1–E7 | **Verified correct** |
| Internal consistency of specification | **Not achieved** — 3 substantive defects (D-01, D-02, D-03) |
| Cross-document consistency | **Mostly consistent** — 1 high defect, 2 gaps |
| D-10a overall | **APPROVED WITH CONDITIONS** — C-10a.1 undischarged; C-10a.2–C-10a.5 unaffected by this audit except as noted in D-01/D-03 |
| Implementation readiness | **Not claimed** |
| Tests executed | **None** |
