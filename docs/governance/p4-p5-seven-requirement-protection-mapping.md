# P-4 / P-5 — Seven Refund-Control Requirements: Protection Mapping

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Responds to:** founder decision clarification 2026-11-20 §5
**Status:** CONFIRMATION OF SPECIFICATION — **every requirement is specified; none is implemented or tested**
**Affected decisions:** D2-08, D2-09, D2-10, D2-11, D2-12

---

## 0. What "confirmed" means here

Your §5 asks me to confirm seven requirements explicitly. I confirm that **all seven are stated as requirements** in the specification and that each has a designated protecting mechanism. I do **not** claim any of them has been implemented, exercised or proven.

Two honest limitations on the mapping below:

1. **No executable test exists.** The repository contains no code. The T-series test identifiers discussed in earlier review rounds are not committed as a test register, so this package cites **invariant IDs** and **specification counterexample IDs** (`CX-A`–`CX-G`) only. I am not inventing test IDs to make the table look complete.
2. **Protection is by construction, not by enforcement.** A rule that says "status is derived" protects nothing until a schema forbids the stored column. The mechanism column therefore states *what must exist*, not what does exist.

---

## 1. The seven requirements

| # | Requirement | Protected by | Mechanism that must exist | Counterexample |
|---|---|---|---|---|
| 1 | Observed financial events are immutable | **INV-19** (append-only ledger) + P-4/P-5 §2.1 | Application roles hold INSERT/SELECT only on event tables; no UPDATE/DELETE/TRUNCATE grant; a correction is a new event with a supersedes link | CX-C, CX-G |
| 2 | Reconciliation status is derived from the complete evidence set | P-4/P-5 §2.2 + **INV-19** (no mutable status column) + **PROP-C (a)** | Status computed on read from the event set; no persisted status field to drift | CX-A |
| 3 | Unmatched refunds are not automatically classified as excess | P-4/P-5 §4.2 (`CB_status = provisional_incomplete`) + D2-10 | Only a **final** `CB` may confirm an excess; an unmatched reduction holds state `unmatched_pending_charge` | **CX-A** |
| 4 | Confirmed excess is recorded, never clamped or discarded | P-4/P-5 §4.3 (classify-never-clamp) + **INV-02** (merchandise ceilings) | State `confirmed_excess` retains the full observed amount; no adjustment reduces the recorded refund | **CX-B** (100), **CX-5** (213) |
| 5 | Duplicate provider events cannot create duplicate financial movements | **INV-12** (idempotency identity) | Stable provider identity, not ingestion order: Shopify `X-Shopify-Event-Id`, Shippo `transaction_object_id`, EasyPost refund id | **CX-C** (800 vs 400) |
| 6 | Reconciliation is deterministic regardless of ingestion order | P-4/P-5 §2.2 + **PROP-C** + P-7b tie-break + **INV-27** | Status is a pure function of the event **set**; chronology lives on events (`provider_ts`), not in row order | CX-D |
| 7 | Both provider event time and ingestion time remain available | P-4/P-5 §2.1 + **INV-16** (period assignment) | Every event carries `provider_ts` **and** `ingestion_ts`; period assignment reads `provider_ts` only | CX-F |

---

## 2. The load-bearing pair

Requirements **3** and **6** are the ones most likely to fail in practice, and they fail together.

If status were stored, it would be written at ingestion. A refund arriving before its charge would be evaluated against an incomplete `CB` of zero and stored as excess — and a later charge event would not revisit it. That single design choice would break requirement 3, break requirement 6, and produce a false accusation against the merchant's customer, which is the most damaging error this control can make.

The fix is the pairing in §2.1/§2.2: **events are immutable facts, status is a derived opinion**. Because the opinion is recomputed from the whole set, a refund arriving first is simply *pending*, and becomes *matched* the moment its charge arrives — with no correction event, no mutation and no ordering assumption.

**CX-A is the proof obligation:** refund observed with no charge yet → state `unmatched_pending_charge`, excess **not** asserted. If any implementation reports a confirmed excess here, requirement 3 has failed.

---

## 3. Requirement 4 in detail, because it is counter-intuitive

Clamping looks prudent and is wrong. If the observed refund is 600 against a `CB` of 500, the correct outcome is not a recorded 500 — it is:

- refund movement **600**, fully recorded;
- `confirmed_excess` **100**, surfaced for reconciliation;
- the difference investigated against provider evidence.

Recording 500 would destroy the evidence that a discrepancy exists, and the merchant's books would not tie to their payment provider. The 100 is not an error to suppress; it is the finding. **CX-B** encodes this, and **CX-5** does the same at unit level (excess 213).

---

## 4. Requirement 5 — identity, not arrival

`INV-12` is only as good as the identity it uses. `(source, source_id, event_type)` is insufficient for a multi-component webhook, and `component_ordinal`/`occurrence` are unstable across retries and reordering — ingestion order must never form part of an identity. The specification therefore requires **provider-supplied stable identifiers** where they exist, and explicit quarantine where they do not.

**CX-C** is the test: the same 400 refund delivered twice must yield one 400 movement, not 800. If the identity is derived from arrival sequence, it yields 800.

---

## 5. What remains unverified

| Claim | Status |
|---|---|
| All seven requirements are stated | **Verified** — present in the specification text |
| Each has a designated protecting invariant | **Verified** — mapped above |
| Append-only is enforced by database privileges | **Not verified** — no schema exists; requires live PostgreSQL |
| Derived status resists a mutable-column shortcut | **Not verified** — no schema exists |
| Idempotency identity matches real provider payloads | **Partially verified** — Shopify ids documented (P-02); Shippo/EasyPost require live confirmation |
| Order-independence holds under real concurrency | **Not verified** — no engine exists; only arithmetic on specifications |

---

## 6. Decisions requested

| # | Decision | Recommendation |
|---|---|---|
| R3-B1 | Confirm the seven requirements as binding acceptance criteria for the refund control | Approve |
| R3-B2 | Confirm that only a **final** `CB` may assert a confirmed excess (requirement 3) | Approve |
| R3-B3 | Confirm classify-never-clamp, with excess surfaced not suppressed (requirement 4) | Approve |
| R3-B4 | Confirm that no persisted reconciliation-status column may exist (requirement 2) | Approve |
| R3-B5 | Confirm provider-supplied stable identity is mandatory and ingestion order is inadmissible in identity | Approve |

No test has been executed. No control is implemented.
