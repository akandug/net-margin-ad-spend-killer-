# Round 3 — Clarification Package Index and Approval Instrument

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision clarification 2026-11-20 §8
**Operating mode:** READ-ONLY CLARIFICATION — **no implementation, schema, migration, provider connection, Supabase provisioning or deployment**
**Baseline unchanged:** D-10a = APPROVED WITH CONDITIONS · C-10a.1 = PARTIALLY SATISFIED — OPEN

---

## 1. What this round does and does not do

This round answers the eight clarification questions and packages the decisions so each can be approved individually.

It does **not** change any approval status. D-10a remains APPROVED WITH CONDITIONS, C-10a.1 remains PARTIALLY SATISFIED — OPEN, the register tally is unchanged (APPROVED 0 / AWC 4 / DEFERRED 1 / OUTSTANDING 8 / PRR 6), and **no invariant number has been allocated**.

---

## 2. Package contents

| File | Answers | Content |
|---|---|---|
| `d10a-round3-d2-07-classification-resolution.md` | §2 | D2-07 is two decisions, not a mis-filing; split into D2-07a / D2-07b with the dependency analysis |
| `d10a-round3-complete-decision-register.md` | §3 | All 16 decisions in three groups with eight columns each |
| `p1-attribution-presentation-specification.md` | §4 | Direct / pooled / unallocated / invalid at product and store level; mandatory reconciliation line |
| `p4-p5-seven-requirement-protection-mapping.md` | §5 | Each of your seven requirements mapped to its protecting invariant, with honest verification limits |
| `p7a-unallocated-shipping-revenue-display.md` | §6 | Display rules for the unallocated bucket; OQ-P7a-2 left explicitly open |
| `prop-c-clause-a-independence-assessment.md` | §7 | Clause (a) quoted; the demonstrated gap it closes; (b)/(c) mapped to D2-08/INV-12 and D2-09/D2-10/D2-11 |
| This file | §8 | The approval instrument |

---

## 3. Approval instrument — approve individually

### Group A — approvable now

| # | Decision | Approve / Amend / Defer |
|---|---|---|
| A-1 | **D2-02** Store the canonical vector string beside its digest; freeze `v1` | ☐ |
| A-2 | **D2-04** Late association evidence annotates, never reallocates | ☐ |
| A-3 | **D2-06** Two recognition states, monotone, derived from shipment events only | ☐ |
| A-4 | **D2-07a** Four reduction types, orthogonal dimensions, two separate identities | ☐ |
| A-5 | **D2-08** Immutable events; status derived from the event set, never stored | ☐ |
| A-6 | **D2-09** `CB` completeness status; only a final CB may confirm excess | ☐ |
| A-7 | **D2-10** The ceiling is `CB`, not recognised revenue | ☐ |
| A-8 | **D2-11** Excess classified, never clamped | ☐ |
| A-9 | **D2-15** Tie-break `(line_identity, unit_index)`, fail closed on non-uniqueness | ☐ |
| A-10 | **D2-16** PROP-C as one invariant, with provenance recorded | ☐ |
| A-11 | **D2-05** Versioning mandatory in MVP; correction workflow and merchant dispute at pilot | ☐ |

### Group B — mechanism now, merchant-facing claim gated

| # | Decision | Approve / Amend / Defer |
|---|---|---|
| B-1 | **D2-01** Closed value set and restricted-pair matrix *(claim gated on OQ-P1-1)* | ☐ |
| B-2 | **D2-03** `G-line`/`P` in MVP, `G-shipment` deferred, disclosure when only pooling is available | ☐ |
| B-3 | **D2-13** Window policy object and no-reclassification rule *(no numeric default)* | ☐ |

### Group C — technical treatment now, statutory classification deferred

| # | Decision | Approve / Amend / Defer |
|---|---|---|
| C-1 | **D2-07b** Record disposition values; refer cost semantics to professional review | ☐ |
| C-2 | **D2-12** Four goodwill/compensation types with no ceiling on non-refund types | ☐ |
| C-3 | **D2-14** O-C order-level treatment with the unallocated bucket | ☐ |

### Supporting decisions raised this round

| # | Decision | Recommendation | Approve / Amend / Defer |
|---|---|---|---|
| R3-A1 | Four-category presentation model with required labels | Approve | ☐ |
| R3-A2 | Contested flag travels with every pooled figure from invalid evidence | Approve | ☐ |
| R3-A3 | Mandatory reconciliation line between product and store figures | Approve | ☐ |
| R3-A4 | Unallocated bucket shown per charge scope, not per order | Approve | ☐ |
| R3-A5 | Prohibited-presentation list is binding on interface design | Approve | ☐ |
| R3-B1 | The seven refund-control requirements are binding acceptance criteria | Approve | ☐ |
| R3-B4 | No persisted reconciliation-status column may exist | Approve | ☐ |
| R3-B5 | Provider-supplied stable identity mandatory; ingestion order inadmissible | Approve | ☐ |
| R3-C2 | Per-charge-scope display of the unallocated bucket | Approve | ☐ |
| R3-C3 | Mandatory disclosure sentence for excluded unallocated revenue | Approve | ☐ |
| R3-C4 | The bucket is never described as "pending" | Approve | ☐ |
| R3-C5 | OQ-P7a-2 stays open; no accounting reading selected now | Approve | ☐ |
| R3-D1 | The §2 gap is a genuine unaddressed failure mode | Approve | ☐ |
| R3-D2 | PROP-C adopted as one invariant, three clauses | Approve | ☐ |
| R3-D3 | Clause provenance recorded: (a) new, (b)/(c) restatements | Approve | ☐ |
| R3-D4 | Allocate the next invariant number to PROP-C | Your decision | ☐ |
| R3-D5 | Does a permanently unmatched reduction stay pending, or move to a visible unresolved bucket? (OQ-PC-1) | Your decision | ☐ |
| R3-E1 | Adopt the D2-07 split into D2-07a / D2-07b | Approve | ☐ |

---

## 4. The three questions only you can answer

These are not technical, and I am not answering them for you:

1. **OQ-PC-1 (R3-D5)** — when reconciliation is exhausted and a charge never appears, may a reduction stay `unmatched_pending_charge` indefinitely? Clause (a) forbids the *silent* version, but a permanently pending reduction is visible only if something surfaces it. This determines whether PROP-C is satisfied in substance or only on paper.
2. **OQ-P7a-2 (R3-C5)** — does unallocated shipping revenue reduce product contribution or sit outside it? Both readings give the same store profit (3,700) and differ in reported shipping revenue and gross margin. Accounting review, not product design.
3. **D2-07 split (R3-E1)** — whether to adopt D2-07a / D2-07b as separate decisions or keep D2-07 as one decision with mixed dependencies.

---

## 5. Verification status

| Claim | Basis |
|---|---|
| All eight clarification sections answered | Section-to-file mapping at §2 |
| No approval status changed | Register tally unchanged; no decision block edited |
| No invariant number allocated | PROP-C referenced by name throughout |
| Numeric figures consistent across documents | 287/287/426, 12/1,188, 334/333/333, 713 vs 287, 1,200, 3,700 — arithmetic verified in the previous round, unchanged here |
| Register completeness rechecked against source | All 16 D2 IDs present. **D2-05 was missing from the first draft and has been restored**; group A is 11, total 17 rows for 16 IDs. The omission and its correction are recorded in the register file |
| **No application test executed** | No code exists in the repository |
| **No provider behaviour verified this round** | No provider documentation consulted this round; earlier P-01–P-05 findings unchanged |

**These are documentation checks, not application tests.** No calculation engine, schema or migration exists, and none is authorised by this document.

---

## 6. Boundaries confirmed

- No code, schema or migration created.
- No Supabase connection, no environment variables, no credentials requested.
- No Shopify, Meta or shipping provider connected.
- No deployment.
- No decision status altered; D-10a and C-10a.1 unchanged.
- F-01, D-09 and the affected part of INV-02 remain **open** pending genuine Shopify evidence.

**Stopping here for your individual decisions.** Implementation remains unauthorised.
