# D-20 — Restore-and-Erasure Threat Model and Test Specification

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Decision:** D-20 — data retention and erasure
**Authority:** Founder direction, 2026-11-20
**Document type:** Threat model and test specification — a requirement set, not evidence of any control
**Status:** Nothing in this document is implemented, executed or verified.

> **Founder-approved scope, restated so it cannot drift.** The founder approved the **hybrid architectural direction for schema preparation only**: external subject-level key destruction or irreversible revocation, an independently protected erasure register, and identity indirection between the financial ledger and erasable identity information. This is **not** approval of a KMS vendor, a paid service, a final key-management implementation, any retention period, or production deployment. Nothing here converts that conditional architectural approval into implementation or production readiness.

---

## 1. Assets and trust boundaries

### 1.1 Assets

| ID | Asset | Property required | Erasable? |
|---|---|---|---|
| A1 | Financial event ledger | Append-only, reconstructible, never mutated | **No** — retained as financial record |
| A2 | Identity data (name, email, address, phone) | Confidential, erasable on request | **Yes** |
| A3 | Identity indirection (ledger → subject reference) | Durable, non-PII, resolvable only through A2/A4 | Reference retained; resolution destroyed |
| A4 | Subject key material | Outside database backup scope; destroyable | **Yes** — destruction is the erasure mechanism |
| A5 | Erasure register (tombstones) | Authoritative, rollback-resistant, monotonic | **No** — must outlive the data it erases |
| A6 | Erasure-matching material (§5) | Enables recognition without retention | Retained, keyed, separately protected |
| A7 | Audit records | Tamper-evident, retained | **No** |

The tension that defines this whole document: **A1 and A7 must survive; A2 must not.** Any design that stores A2 inside A1's backup scope cannot satisfy both, and any design that satisfies both must have a place to record "this subject was erased" that is not itself erasable — A5.

### 1.2 Trust boundaries

```
┌─────────────────────────────────────────────────────────────┐
│ Trust boundary 1 — external providers (Shopify, Meta, Shippo)│
│   untrusted input: webhooks, payloads, replays, delays       │
└───────────────────────────┬─────────────────────────────────┘
                            │ signature-verified ingress
┌───────────────────────────▼─────────────────────────────────┐
│ Trust boundary 2 — application (ingest, workers)             │
│   must consult A5 BEFORE writing A2                          │
└──────────┬────────────────────────────────┬─────────────────┘
           │                                │
┌──────────▼──────────────┐   ┌─────────────▼───────────────────┐
│ Trust boundary 3 — DB    │   │ Trust boundary 4 — external      │
│ ledger A1, identity A2,  │   │ A5 erasure register, A4 keys,    │
│ backup scope = {A1,A2}   │   │ A6 matching material            │
│ **restorable to an       │   │ **NOT in DB backup scope**       │
│ earlier state**          │   │ not co-restored                 │
└──────────────────────────┘   └──────────────────────────────────┘
```

**Boundary 3 is the problem.** It can be rolled backwards by a restore. Boundary 4 must therefore be the authority for erasure state, and the application must treat boundary 3's erasure-related state as **potentially stale** after any restore.

---

## 2. Actors and capability assumptions

Stated explicitly because the threat model is only meaningful against named capabilities.

| ID | Actor | Assumed capability |
|---|---|---|
| X1 | External attacker, no credentials | Can reach public endpoints, replay observed webhook bodies if a secret leaks |
| X2 | Attacker holding application runtime credentials | Full read/write on A1 and A2 through the application role; **cannot** destroy A4 keys or write A5 directly if controls hold |
| X3 | Insider with database superuser access | Can read and modify A1, A2 and the backup scope; can restore backups |
| X4 | Operator performing a restore | Legitimate action, potentially performed under pressure and without erasure awareness |
| X5 | Provider replay / delayed delivery | Re-sends a legitimate historical payload days or weeks later |
| X6 | Application defect | Missing check on one write path, or a swallowed exception |

**X3 is the honest limit.** An insider with superuser access and access to boundary 4 can defeat erasure by restoring a key backup or deleting a tombstone. This design does not claim to prevent that; it aims to make it **detectable, attributable and auditable**, and to make the *accidental* and *application-level* paths fail closed.

---

## 3. Threat register

Severity is **impact if the control is absent**, not current risk (no control exists yet).

| ID | Threat | Actor | Condition | Impact if unmitigated | Severity |
|---|---|---|---|---|---|
| T-01 | **Restore resurrection** — backup restored to a point before erasure; identity rows and their key material return together | X4, X3 | a, b | Erased subject's PII readable again | **Critical** |
| T-02 | **Key-in-backup** — key material stored in DB or DB secret facility, so restore restores the keys | — | a | Erasure impossible without re-deleting; erasure evidence lost | **Critical** |
| T-03 | **Re-establishment** — restored identity row plus a later webhook writes *new* PII for an erased subject; no read-block fires because nothing reads | X5, X6 | c, f | Erased subject's PII silently repopulated | **Critical** |
| T-04 | **Replay after erasure** — delayed webhook re-ingests an order for an erased subject | X5 | f | PII recreated, possibly without any restore involved | **High** |
| T-05 | **Write-path omission** — a new ingestion path (bulk import, manual entry, support tool) skips the register check | X6 | c | PII written for erased subject; inconsistent posture | **High** |
| T-06 | **Register unavailability treated as "not erased"** — register timeout degrades to allow | X6 | d | Erasure bypassed under load or outage | **Critical** |
| T-07 | **Register rollback** — register itself restored or reverted, so tombstones vanish | X3, X4 | b | All erasure evidence lost at once | **Critical** |
| T-08 | **Restore without reconciliation** — DB restored, app starts serving and writing immediately | X4, X6 | e | Window in which erased data is live and mutable | **High** |
| T-09 | **Incomplete erasure** — key destroyed but identity ciphertext left in a log, export, cache or replica | X6 | a, g | Erased PII persists outside the intended store | **High** |
| T-10 | **Recognition by retained PII** — to detect an erased subject, the system keeps the email/phone it was told to erase | X6 | g | Erasure is nominal; PII retained under another name | **High** |
| T-11 | **Enumeration of matching material** — keyed pseudonyms over low-entropy identifiers become reversible if the matching key leaks | X2, X3 | g | Subject identities recoverable from A6 | **Medium** |
| T-12 | **Silent partial erasure** — key destroyed, register written, but the ledger's indirection left resolvable | X6 | a, b | Read path returns "erased" while data still decryptable elsewhere | **Medium** |
| T-13 | **Double erasure / re-erasure race** — two erasure requests concurrently; one overwrites the other's tombstone | X6 | b | Audit trail wrong; unclear whether erasure completed | **Medium** |
| T-14 | **Erasure used to destroy financial records** — a request broadens from identity to ledger | X1, X2 | — | Violates A1 append-only; destroys auditability | **High** |

**T-14 deserves emphasis.** The most likely way this architecture fails commercially is not a restore — it is someone honouring an erasure request by deleting financial events. The ledger must be structurally incapable of it (INV-19) and the erasure procedure must be scoped to A2/A3 resolution only.

---

## 4. Restore scenarios

Each scenario is a test precondition in §7.

| ID | Scenario | What goes wrong without controls |
|---|---|---|
| R-01 | Full restore of DB to a timestamp **before** an erasure | A2 returns; tombstones in A5 unchanged; without reconciliation the system serves resurrected PII |
| R-02 | Full restore to a timestamp **after** an erasure | A2 stays erased; replay of A5 must be idempotent, not duplicate tombstones |
| R-03 | **Partial restore** — single table or schema restored | Inconsistent erasure state between tables; indirection may resolve where identity is gone, or vice versa |
| R-04 | Restore into a **new environment** (staging refresh from production) | Erasure state may be absent entirely; environment must inherit A5, not just A1/A2 |
| R-05 | Restore **during** an in-flight erasure | Erasure applied to pre-restore state; post-restore state may differ; needs idempotent reapplication |
| R-06 | Point-in-time recovery to **seconds before** an erasure commit | Maximum ambiguity; sequence comparison must resolve it deterministically |
| R-07 | Restore of **A5 itself** from its own backup | The authority rolls back — the failure T-07; requires monotonic sequence and independent backup policy |

### 4.1 The property that makes R-01 through R-06 tractable

A5 carries a **monotonic sequence number** per erasure event. The database records the highest A5 sequence it has applied. After any restore:

```
register_max  = max sequence present in A5 (authoritative, external)
db_applied    = highest sequence the restored DB has applied

db_applied <  register_max   →  replay erasures (db_applied, register_max]
db_applied == register_max   →  consistent
db_applied >  register_max   →  impossible: A5 has lost state → BLOCK ALL
```

The third case is the important one and the one a naive design omits: if the database claims to have applied erasure 412 but the register's maximum is 407, the **register has been rolled back**, which is T-07, and the correct response is to refuse service rather than guess.

---

## 5. Recognition without retention (founder condition g)

**The question:** after erasure, how does the system know that an incoming webhook refers to an erased subject, without keeping the erased PII?

**The answer, in three parts:**

1. **The ledger never holds PII.** It holds a **random** subject reference (not derived from email or any personal attribute — this is the approved design, and it is why hashing was rejected). Retaining a random opaque reference retains nothing about the person. This handles recognition *within* the ledger.

2. **Incoming payloads identify subjects by provider identifier, not by our reference.** So matching requires a bridge. A5 stores:

   ```
   tombstone := {
     sequence,                    -- monotonic
     match_token,                 -- see 3
     subject_ref,                 -- the random reference, retained: not PII
     erased_at,                   -- timestamp
     scope,                       -- what was erased (identity only, never ledger)
     authority,                   -- who/what authorised it
     key_destroyed_at             -- null until A4 destruction confirmed
   }
   ```

3. **`match_token` is the only derived value, and it is deliberately expensive to invert:**

   ```
   match_token = scrypt( HMAC( K_match, normalize(provider_customer_identifier) ),
                         salt = per-tombstone random,
                         N = high, r, p = tuned )
   ```

   `K_match` lives outside the backup scope, separate from A4's subject keys, and is never used for encryption. Normalisation is specified (case folding, whitespace, provider-specific identifier canonicalisation) because an unnormalised token silently fails to match and thereby resurrects data.

**Why HMAC alone is insufficient here.** Provider customer identifiers are frequently small integers. A leaked `K_match` plus a bounded numeric range makes a plain HMAC trivially enumerable — that is T-11. The scrypt layer makes each candidate guess expensive even with the key, and the per-tombstone salt prevents precomputation across subjects.

**Residual risk, stated plainly:** if `K_match` leaks *and* an attacker can afford a slow offline search over the identifier space, erased subjects become re-identifiable from A5. This is a real, bounded weakness. It is strictly better than retaining the identifier, and it is the reason `K_match` must be treated as a top-tier secret with rotation support — but it is not zero risk and must not be described as such.

**What this deliberately does not do:** it does not allow "look up whether this email was erased" as a general query. Only the ingestion path computes a token and checks for its presence; there is no endpoint that reverses a token.

---

## 6. Condition traceability

Every founder condition maps to named controls and named tests. A condition with no test is an intention.

| Founder condition | Controls | Tests | Residual risk |
|---|---|---|---|
| **a.** Key material and authoritative erasure record must not depend solely on PostgreSQL or its backup scope | A4 and A5 held in boundary 4; no DB secret facility used for A4; schema carries indirection only | TE-01, TE-02, TE-03 | X3 with access to boundary 4 |
| **b.** Register must resist rollback and not be resurrected in an earlier state by a DB restore | Monotonic sequence; append-only register; independent backup policy; `db_applied > register_max` blocks service | TE-04, TE-05, TE-06 | Restore of A5 itself (R-07) |
| **c.** All ingestion, webhook and identity-creation paths must consult the register before writing PII | Single mandatory write gate; no path may write A2 except through it; enumerated path inventory | TE-07, TE-08, TE-09 | New path added without the gate (T-05) |
| **d.** Missing or unavailable erasure evidence must fail closed | Gate treats timeout/error/absent as **blocked**; event quarantined with alert; no default-allow | TE-10, TE-11 | None intended; must be proven by test |
| **e.** Restore must trigger reconciliation before new identity writes | Startup/restore detection; reconciliation gate; replay of missing sequences; service refuses identity writes until clean | TE-12, TE-13, TE-14 | Operator overriding the gate |
| **f.** Delayed webhooks, replays and retries must not recreate erased PII | Gate applies on every write including replays; tombstone matched by token; idempotency (INV-12) is not sufficient alone | TE-15, TE-16, TE-17 | None intended |
| **g.** Erased subjects recognised without retaining erased PII | Random subject ref; keyed, salted, slow match token; no reverse lookup | TE-18, TE-19, TE-20 | Key compromise + offline search (T-11) |

---

## 7. Test specification

**Every test below is unexecuted.** No database, schema, code or key store exists. The "evidence required" column defines what must exist before a test may be marked passed — a green test with no artifact is not evidence.

### 7.1 Key material and scope (condition a)

**TE-01 — Subject key is outside the backup scope**
- *Preconditions:* A4 configured; a test subject exists with encrypted identity.
- *Steps:* Record A4 locator for the subject. Perform a full DB restore to a point while the subject exists. Attempt to resolve identity.
- *Expected:* Resolution succeeds (key survived, as intended) **and** the key was never present in the backup artifact — verified by inspecting the restore payload for key material.
- *Failure behaviour:* If key material appears in the backup payload, condition (a) is violated; the design is rejected, not patched.
- *Evidence required:* Restore payload inspection report listing key-material absence.

**TE-02 — No key material in database objects**
- *Steps:* Enumerate all database objects, extensions and secret facilities in the target environment; search for key material.
- *Expected:* Zero occurrences. Any in-DB secret facility holding A4 material fails the test.
- *Evidence required:* Schema enumeration output plus search result.

**TE-03 — Key destruction is irreversible**
- *Steps:* Destroy a subject's key. Attempt to re-derive or recover it from A5, logs, cache, replica or backup.
- *Expected:* All recovery paths fail.
- *Evidence required:* Destruction log plus recovery-attempt log.

### 7.2 Register integrity (condition b)

**TE-04 — Monotonic sequence**
- *Steps:* Apply erasures 1..n; attempt to insert a lower sequence; attempt to update an existing tombstone.
- *Expected:* Both rejected at the register layer, not merely by application code.
- *Evidence required:* Register-level rejection responses.

**TE-05 — Register survives DB restore unchanged**
- *Steps:* Snapshot register state; perform full DB restore to an earlier point; re-read register.
- *Expected:* Register state identical to snapshot; no tombstone lost.
- *Evidence required:* Before/after register dump comparison.

**TE-06 — Register rollback is detected and blocks service**
- *Steps:* Simulate register rollback so `db_applied > register_max`.
- *Expected:* System refuses identity writes **and** refuses merchant-visible reads; alarm raised; no silent reconciliation.
- *Evidence required:* Refusal response, alarm record.

### 7.3 Write-path enforcement (condition c)

**TE-07 — Path inventory is complete**
- *Steps:* Enumerate every code path that can create or update A2. For each, confirm it passes through the gate.
- *Expected:* Every path gated; the inventory is a reviewed artifact, not an assumption.
- *Evidence required:* Path inventory table with per-path gate reference and reviewer.

**TE-08 — Webhook path consults the register**
- *Steps:* Deliver a webhook for an erased subject.
- *Expected:* Gate consulted **before** any A2 write; write blocked; ledger event recorded with the subject reference only.
- *Evidence required:* Trace showing gate call preceding write.

**TE-09 — Non-webhook paths are gated**
- *Steps:* Repeat TE-08 for bulk import, manual entry and any support/admin tool.
- *Expected:* Identical blocking behaviour on every path.
- *Evidence required:* Per-path trace.

### 7.4 Fail-closed behaviour (condition d)

**TE-10 — Register timeout blocks the write**
- *Steps:* Make the register unavailable; deliver a webhook containing PII.
- *Expected:* Identity write **fails**; event quarantined; alert raised; no partial write; no default-allow.
- *Evidence required:* Quarantine record plus absence of the identity row.

**TE-11 — Register returns malformed or empty response**
- *Steps:* Return an empty set for a known-erased subject; return a malformed payload.
- *Expected:* Both treated as "unavailable" → blocked. An empty response must not be interpreted as "not erased" unless the register explicitly asserts completeness for that query.
- *Evidence required:* Blocked-write evidence for both cases.

### 7.5 Restore reconciliation (condition e)

**TE-12 — Restore to before erasure triggers replay**
- *Steps:* Execute R-01: erase subject at sequence 100; restore DB to a point before it; start the application.
- *Expected:* Reconciliation detects `db_applied < register_max`; replays erasure 100; identity unreadable **before** any identity write is accepted.
- *Evidence required:* Reconciliation log plus failed resolution attempt.

**TE-13 — Identity writes refused during reconciliation**
- *Steps:* Begin a restore reconciliation; attempt an identity write concurrently.
- *Expected:* Write refused until reconciliation reports clean.
- *Evidence required:* Refusal during window, success after.

**TE-14 — Idempotent replay**
- *Steps:* Execute R-02: restore to a point *after* erasure; reconcile.
- *Expected:* No duplicate tombstones; no new sequence consumed; state unchanged.
- *Evidence required:* Register dump comparison.

### 7.6 Replay and delayed delivery (condition f)

**TE-15 — Delayed webhook cannot resurrect**
- *Steps:* Erase a subject; replay a genuine webhook for that subject captured before erasure.
- *Expected:* Write blocked; ledger event still recorded against the subject reference (financial history preserved); no PII written.
- *Evidence required:* Blocked-write trace plus ledger event presence.

**TE-16 — Restore plus replay, combined**
- *Steps:* R-01 restore, then replay a pre-erasure webhook **before** reconciliation completes.
- *Expected:* Blocked by the reconciliation gate (condition e) rather than by the tombstone check alone — proving the two controls are independent.
- *Evidence required:* Refusal trace attributable to the reconciliation gate.

**TE-17 — Repeated retries stay blocked**
- *Steps:* Replay the same webhook repeatedly, including after reconciliation.
- *Expected:* Every attempt blocked; no cumulative side effects; no alert fatigue masking a real failure.
- *Evidence required:* Attempt log.

### 7.7 Recognition without retention (condition g)

**TE-18 — Ledger retains no PII after erasure**
- *Steps:* Erase a subject; scan the ledger and all derived stores for the erased identifiers.
- *Expected:* Zero occurrences. The random subject reference remains and is not PII.
- *Evidence required:* Scan report across ledger, logs, exports, caches, replicas.

**TE-19 — Matching works without the identifier**
- *Steps:* Erase a subject; deliver a webhook containing the identifier with different casing and whitespace; also with the provider identifier in an alternative but equivalent form.
- *Expected:* Matched and blocked in every normalised form. An unnormalised miss is a failure, because a miss equals resurrection.
- *Evidence required:* Per-variant results.

**TE-20 — No reverse lookup**
- *Steps:* Attempt to resolve a `match_token` back to an identifier through any interface, including admin tooling.
- *Expected:* No such capability exists. Recorded as a deliberate absence, not a gap.
- *Evidence required:* Interface inventory demonstrating absence.

### 7.8 Boundary discipline

**TE-21 — Erasure cannot touch the ledger**
- *Steps:* Attempt an erasure scoped to financial events.
- *Expected:* Structurally rejected (INV-19); no path exists by which an erasure request deletes or mutates ledger records.
- *Evidence required:* Rejection evidence plus privilege inventory.

**TE-22 — Concurrent erasure of the same subject**
- *Steps:* Issue two erasure requests concurrently.
- *Expected:* Both recorded; exactly one sequence consumed or two distinct sequences with deterministic ordering; no lost tombstone; idempotent outcome.
- *Evidence required:* Register dump plus both request records.

---

## 8. Evidence standard before any test may be marked passed

A test is **passed** only when all of the following exist and are linked from the decision register:

1. The executed test artifact (log, trace, dump or report) — not a description of intent.
2. The environment it ran in, stated explicitly (local, staging, or restored-restore rehearsal).
3. The identity of the control exercised, at the level it operates — register-layer rejections are not satisfied by application-layer checks.
4. For fail-closed tests, positive evidence that the write did **not** occur, not merely that an error was logged.
5. A named reviewer who is not the implementer.

**No test in this document has been executed. No control described here exists.**

---

## 9. Open questions requiring founder decision

**OQ-20-1 — Where does the erasure register live?** It must be outside the DB backup scope and independently durable. Candidate shapes: a separate managed database with its own backup policy, an object-store append-only log with versioning, or a dedicated service. **Each has a cost and an operational owner.** Selection is *not* covered by the current approval, which explicitly excludes selecting a paid service. This decision blocks schema preparation beyond the indirection columns.

**OQ-20-2 — Who may authorise an erasure, and is it reversible?** An erasure that is later found to be mistaken is unrecoverable by design. Recommended: two-person authorisation for non-automated erasures, and a delay window before key destruction where the tombstone blocks immediately but destruction is deferred. **Requires founder decision**; the delay window is a policy choice with a privacy-law dimension.

**OQ-20-3 — What does `scope` permit?** The founder's direction separates ledger from identity. This specification assumes erasure scope is **identity only**, always. If any jurisdiction requires erasure of order-level personal data inside the ledger, that must be handled by the indirection being unresolvable rather than by deleting events. **Requires legal review.**

**OQ-20-4 — Is `match_token` retained indefinitely?** It must outlive the data it protects against, otherwise erasure expires. But it is derived from personal identifiers. **Retention of A6 is itself a policy question requiring legal review** — no period is asserted here.

**OQ-20-5 — What happens to a tenant that leaves?** Tenant offboarding erases identity but retains financial records. The register must handle tenant-scoped erasure without consuming subject sequences incorrectly. Not yet specified.

---

## 10. Status

| Item | Status |
|---|---|
| Hybrid direction approved for **schema preparation** | **Approved by founder, 2026-11-20** |
| Conditions (a)–(g) | **Binding requirements** |
| Threat model T-01–T-14 | Delivered here; **not validated against a real system** |
| Restore scenarios R-01–R-07 | Delivered here; **not rehearsed** |
| Recognition design (§5) | Proposed; **residual risk acknowledged** |
| Test specification TE-01–TE-22 | Delivered here; **none executed** |
| Register location (OQ-20-1) | **OUTSTANDING — founder decision** |
| Erasure authorisation and delay (OQ-20-2) | **OUTSTANDING — founder decision** |
| Erasure scope (OQ-20-3) | **Requires legal review** |
| A6 retention (OQ-20-4) | **Requires legal review** |
| Retention periods | **OUTSTANDING — legal/tax determination; none asserted** |
| KMS vendor / paid service | **Not approved, not selected** |
| Implementation | None |
| Empirical verification | None |

---

## 11. Boundary statement

Documentation only. No application code, schema, migration, test execution, Supabase provisioning, key store, provider connection, merchant data or deployment. No vendor price is asserted. No control described in this document has been implemented, exercised or verified, and no test result is claimed. The threat model describes what could go wrong in a system that does not yet exist.
