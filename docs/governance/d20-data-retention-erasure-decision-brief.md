# D-20 — Data Retention and Erasure: Founder Decision Brief

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Decision:** D-20 — Retention and erasure
**Register recommendation:** A — pseudonymise customer identity in the ledger; crypto-shred the PII key; record an erasure event
**Status:** **PROFESSIONAL REVIEW REQUIRED — unchanged.** This brief prepares the decision and the review. It does not approve it and does not change the register.
**Gates:** G0 schema / G3
**Prepared:** 2026-11-17

---

## 0. The one thing to understand before reading

D-20 is not a privacy feature bolted onto a financial system. It is a **schema constraint that must be satisfied in the first migration**, because it collides head-on with the append-only ledger:

| Requirement | Source | Demand |
|---|---|---|
| Erasure of personal data on request | Privacy law | Data must become unrecoverable |
| Financial records are never mutated or deleted | INV-19 | Rows are immutable and permanent |

Both are non-negotiable. They can both hold **only if personally identifying information never enters the ledger in the first place.** If a customer's name, email or address is written into an order or ledger row, erasure requires mutating an immutable row — so D-20 becomes unsatisfiable without violating INV-19. This is finding **F-4**, and it is the reason this brief is a G0 decision rather than a later one.

**Nothing in this brief states a retention period.** Periods are jurisdiction-specific legal and tax determinations. The design requirement is that the platform must be able to *implement* whatever period the adviser specifies — not that we guess it.

---

## 1. The three-store model

Personal information and financial evidence must live in **separate stores with different lifecycles**.

| Store | Contents | Mutability | Reaches backups? |
|---|---|---|---|
| **Ledger** (append-only) | Financial events: amounts, dates, product refs, tax, fees, shipping, ad spend, `customer_ref` | Immutable; corrections are new events | Yes — and that is correct |
| **Identity** (erasable) | `customer_ref` → encrypted PII (name, email, phone, address) + wrapped data key | Mutable; deletable | Key destruction must reach backups (see §3) |
| **Audit** (append-only) | Who did what, when: erasure events, corrections, privileged access | Immutable | Yes |

The ledger carries a **pseudonymous reference**, never the identity itself. Aggregation, reconciliation and profitability all work on the reference. Nothing financial depends on knowing who the customer was.

**Why three stores and not two columns:** erasure must be able to act on identity alone, without touching ledger or audit. Any design where erasing a customer requires updating an order row has already failed D-20.

---

## 2. Pseudonymous customer and order references

Three properties are required, and each is easy to get wrong:

1. **Random, not derived.** The reference must be a random identifier (e.g. a UUIDv4) minted per customer, per tenant. It must **not** be a hash of the email address or phone number. A hash of an email is not pseudonymisation: an attacker holding a list of email addresses can recompute every hash and re-identify the whole ledger. This is the single most common design error in this area.
2. **Tenant-scoped.** The same person shopping at two merchants must not be linkable across tenants. Cross-tenant linkage would be a tenant-isolation breach in addition to a privacy one.
3. **Stable and non-reused.** A reference is never reassigned, and never recycled after erasure — otherwise historical aggregates silently re-point to a different person.

**Order references** are separate from customer references. An order is financial evidence and is retained; a customer is a person and may be erased. An order whose customer has been erased keeps its order id and its `customer_ref`, and the `customer_ref` simply no longer resolves to anything.

**Guest orders** carry a `customer_ref` that was never linked to an identity. That is a legitimate state and must not be treated as an error.

---

## 3. PII storage, encryption and crypto-shredding

### 3.1 Mechanism

Envelope encryption with a **per-subject data key**:

```
PII plaintext
  └─ encrypted with  subject data key (DEK)        → ciphertext stored in Identity store
       └─ DEK wrapped by  tenant key-encryption key (KEK)  → stored in an external key store
            └─ KEK held in a KMS, outside the database
```

**Erasure = destroy the subject's DEK.** The ciphertext remains but is permanently unreadable. No ledger row is touched; no aggregate moves.

### 3.2 The limitation that must not be glossed over

Crypto-shredding only works if destroying the key is genuinely irreversible. Two failure modes make it reversible:

**Failure mode 1 — the DEK lives in the database.** If the wrapped DEK is stored in a Postgres table and that database is restored from a point-in-time backup, the DEK comes back and the erasure is undone. A restore silently resurrects erased personal data.

> **Architectural requirement:** the key material must live in a **key store outside the database backup scope** (a KMS), so that destroying a DEK is not something a database restore can undo. This is a hard requirement, not a preference.

**Failure mode 2 — key rotation re-wraps destroyed keys.** Rotation must skip destroyed DEKs. A rotation job that re-wraps from a backup copy re-establishes recoverability.

### 3.3 Unverified dependency — flagged honestly

Whether Supabase can provide a key store that satisfies §3.2 is **not verified**. Supabase's Vault facility stores secrets inside Postgres, which — on the reasoning above — would be restored along with the database and would therefore **not** satisfy crypto-shredding across backups. That is a design concern to be tested, not a claim about the product's behaviour.

**Consequence:** this may require an external KMS (an additional paid dependency), or a documented decision to accept that erasure reaches live data but not backups until backup expiry. Either way it is a founder decision with a cost attached, and it must be settled before the first migration. The P-05 live checks that would inform it are deferred by your direction.

### 3.4 Where PII must never appear

- **Application logs, error trackers, analytics and audit records.** A shred does not reach a log file. Log hygiene is a control, not a convention.
- **Raw provider payloads.** Shopify order and webhook payloads contain customer PII. If raw payloads are retained as reconciliation evidence, they are a **second PII store** and must either be redacted at ingestion or placed under the same erasure regime.
- **Ledger rows.** The whole point of §1.
- **Backups of the above**, which inherit the problem.

### 3.5 What crypto-shredding is not

It is **not** deletion of the ciphertext, and it does **not** guarantee legal adequacy in every regime. Some jurisdictions expect actual erasure of the data, not destruction of a key. Whether key destruction satisfies the erasure obligation is a **legal question**, not an engineering one, and it is listed in §6.

---

## 4. Retention classes

The platform must implement retention **as configuration keyed by record class and jurisdiction**, never as hardcoded constants. The values below are deliberately left to professional determination.

| Class | Contents | Retention driver | Period |
|---|---|---|---|
| **Financial events** | Ledger entries: sales, refunds, fees, shipping, ad spend | Statutory accounting and tax record-keeping | **Legal/tax determination per jurisdiction — not specified here** |
| **Audit records** | Erasure events, corrections, privileged access, corrections trail | Accountability and demonstrable compliance | **Legal determination — not specified here** |
| **Personal information** | Identity store ciphertext | Minimisation: no longer than the purpose requires; bounded above by erasure rights | **Legal determination — not specified here** |
| **Raw provider payloads** | Webhook and API captures retained as evidence | Dispute and reconciliation evidence; shortest defensible | **Commercial + legal determination** |
| **Backups** | All of the above, at rest | Inherits the longest applicable period; reached only via key destruction | **Legal determination — not specified here** |

**Design rules that follow, and that are independent of the periods chosen:**

1. Every record carries its class and the jurisdiction whose period applies.
2. The default for any newly onboarded jurisdiction must be the **longest** applicable period, not a convenient short one. Guessing short is the failure that cannot be undone.
3. Retention is enforced by a scheduled job whose runs are auditable — not by hoping.
4. A period change is a configuration event with an audit record; it never retroactively deletes evidence.

---

## 5. Honouring a deletion request without mutating the ledger

The sequence, with the failure points named:

1. **Verify authority.** Confirm the requester is entitled to request erasure for that subject. Record the verification method, not the identity evidence.
2. **Check for a retention obligation or legal hold.** If a statutory retention duty or litigation hold applies, erasure is **deferred or partially refused**, and the **legal basis is recorded**. This is a lawful outcome, not a failure.
3. **Destroy the DEK in the key store.** This is the erasure.
4. **Mark the identity record erased**, retaining the `customer_ref` so referential integrity survives.
5. **Append an erasure event to the audit store**: subject reference, timestamp, scope (which fields), legal basis, actor, request id. The event is itself retained — it is the proof the request was honoured.
6. **Purge PII from logs and exports** within a stated window. This step is only possible if §3.4 was enforced from the start.
7. **Verify.** Reads of the subject return no PII; a sample of aggregates recomputes **identically** before and after.
8. **Respond** to the requester.

**What is never done:** updating or deleting a ledger row; deleting the order; renumbering or recycling the `customer_ref`; rewriting history to pretend the customer never existed.

**What the merchant sees afterwards:** "customer identity erased on request" against the order — not a blank customer field, and not a fabricated name. The order remains fully auditable and its contribution profit is unchanged.

---

## 6. Worked example — erasure of customer C-77

Extending the example in `g0-g2-decision-preparation.md` §4.8 with the backup behaviour.

| Data | Treatment | Why |
|---|---|---|
| Name, email, phone, shipping address | **Unrecoverable** (DEK destroyed in KMS) | Not required for financial reporting |
| Payment instrument references | **Deleted** | Held by the processor, not needed here |
| Order id, dates, amounts, product ids, tax, fees | **Retained, unchanged** | Financial evidence; statutory retention |
| `customer_ref` | **Retained, now unlinkable** | Preserves order integrity and every aggregate |
| Erasure event | **Retained** | Proof the request was honoured |

**Arithmetic check.** Order contribution profit recomputes to the **same 8,063** before and after erasure, because the arithmetic never touched an identifier. The ledger hash chain is intact: no row was mutated or removed. Only the separate identity store changed, and that change is itself an event.

**Backup behaviour — the part that decides whether this works.**

| Scenario | Outcome |
|---|---|
| DEK in a KMS outside database backups (**required design**) | Restoring the database restores ciphertext with **no key**. Erasure survives the restore ✓ |
| DEK stored in a Postgres table | Restoring a pre-erasure backup **restores the key**, and the erased data becomes readable again ✗ |

The second row is why §3.2 is a hard requirement. It is also the scenario most likely to be discovered only during an incident, which is why it must be decided at G0.

---

## 7. Matters requiring professional review

**Legal — mandatory before any real merchant data is stored:**

1. For how long must order-level financial records be retained in our target jurisdictions? (Per jurisdiction; not to be assumed.)
2. May customer identity be erased on request while the financial amounts are retained, or must erasure be deferred until the retention period expires?
3. **Is key destruction (crypto-shredding) sufficient to satisfy the erasure obligation in those jurisdictions**, or is deletion of the ciphertext required?
4. How must backups be treated — is erasure reaching live data but not backups (until backup expiry) acceptable, and must that be disclosed?
5. What are our obligations as a **processor** versus the merchant as controller — who receives erasure requests, and what is our contractual duty to action them?
6. Cross-border transfer and storage-location obligations for both PII and financial records.
7. Whether an erasure request can be refused or deferred on legal-hold grounds, and what must be recorded.

**Tax:**

8. Retention periods for tax records, by jurisdiction — and whether those differ from general accounting retention.

**Accounting:** none identified. Erasure of identity does not affect any financial figure.

**Provider/technical verification — not performed:**

9. Whether the chosen platform can host key material outside the database backup scope (§3.3). Deferred with the P-05 live checks.

---

## 8. Consequences and trade-offs

**If the three-store model is adopted (recommended):**

- Erasure is a key operation, not a data migration. It is fast, auditable and cannot break financial history.
- Every aggregate stays stable — no historical total moves when a customer is erased, which is what makes the product's numbers trustworthy over time.
- Cost: an external key store is likely (see §3.3), and the ingestion boundary must strip PII from the first migration — retrofitting it later means rewriting history, which INV-19 forbids.

**If PII is allowed into ledger or order rows:**

- Erasure requires mutating immutable rows. Either INV-19 or the privacy obligation must be broken. There is no third option, and the choice would be made under time pressure during a real request.
- Every retention period then applies to the ledger itself, and the "stable aggregate" property is lost.

**Deferral is not safe here.** Unlike D-14 (return provision), D-20 cannot be deferred to a later phase: it constrains the **first migration**. Deferring the decision is effectively choosing the second option by default.

---

## 9. Recommendation

**Adopt option A**, with the constraints made explicit:

1. Three-store separation, enforced from the first migration — PII never in ledger, order or audit rows.
2. Random, tenant-scoped pseudonymous references; never a hash of an identifier.
3. Per-subject data keys, with key material in a store **outside the database backup scope**.
4. Erasure = key destruction + erasure event; ledger untouched.
5. Retention as jurisdiction-keyed configuration, defaulting to the **longest** applicable period; no invented constants.
6. Log and export hygiene as a control, so erasure is not defeated by a log file.

**Not approved by this brief.** D-20 remains **PROFESSIONAL REVIEW REQUIRED**, and the legal questions in §7 are prerequisites for G3 regardless of your decision.

---

## 10. Boundary statement

Documentation only. No code, schema, migration, test, provider connection, credential, environment variable, Supabase provisioning, live security check or deployment. No retention period is asserted: every period is marked as requiring jurisdiction-specific legal and tax determination. No claim is made about Supabase key-store or backup behaviour — §3.3 records that as an open technical question to be verified, not as established fact.
