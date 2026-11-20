# D-20 — Key Management and Erasure-Restore Specification

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Decision:** D-20 — data retention and erasure, specifically the key-management approach that makes erasure survive a backup restore
**Founder direction, 2026-11-20:** the **architectural principles are approved** — separation of the immutable financial ledger, erasable identity data and audit records; random pseudonymous references rather than email hashes; and the requirement that **subject-level encryption-key destruction or irreversible revocation must survive restoration of an older PostgreSQL backup**. Final retention and key-management policy remain **open**.
**Required before implementation:** key-management comparison, restore-and-erasure threat scenario, restore protection method, durable erasure reconciliation mechanism, cost and operational assessment. **Do not select a paid KMS product without presenting the alternatives.**
**Status after this document:** **principles approved; key-management selection and retention policy OUTSTANDING**
**Gate:** G1 (security foundation) / G2
**Prepared:** 2026-11-20

---

## 1. The requirement, stated precisely

The naive erasure story is: delete the personal data. That story is broken by backups, and it is broken in a way that is silent — nobody notices, because the erasure appeared to succeed.

The approved principle is therefore stronger and more specific:

> Once a subject's personal data has been erased, **no restoration of any database backup may make that data readable again**, and the system must be able to **demonstrate** that this holds.

Three things follow, and they drive everything below.

**1.1 — The erasure record must live outside the restored scope.** Anything stored only in the PostgreSQL cluster — including encrypted key material — is restored along with the data it was meant to protect. This is not a vendor defect; it is what a backup is for. The erasure state must therefore reside somewhere a database restore cannot reach.

**1.2 — Erasure is not only a read block.** A restored backup also restores the *identity rows*. If erasure is implemented only as "refuse to decrypt", the restored system may treat the subject as a normal active customer and begin writing **new** personal data on the next webhook. Erasure must leave a durable **tombstone** that prevents re-establishment, not merely a decryption barrier.

**1.3 — Verification must be continuous, not one-off.** A control that is only checked at build time cannot detect a restore. It has to be re-established automatically after every restore and re-verified on a schedule.

---

## 2. Key-management options

Six viable structures. Each is described by where the key material lives, because that determines whether a database restore can reach it.

### K1 — Subject keys stored in the application database

A per-subject data key row inside the same PostgreSQL cluster, encrypted at rest by the storage layer. Erasure deletes the row.

**Restore verdict: fails.** The restore returns both the key and the ciphertext. Erasure is silently reversed.

### K2 — In-database secret management (e.g. Supabase Vault)

The key material is held in the same PostgreSQL cluster through an in-database secrets facility.

**Restore verdict: fails, for the same structural reason.** In-database key material is inside the database backup scope, so a point-in-time restore reinstates it. This is an inference from where the keys live, not a measured result — **the specific facility's backup and root-key behaviour has not been verified against official documentation in this project, and must be before it is relied on or ruled out.** It is listed here because it is the option a reader will ask about, and because it looks like a solution while sharing K1's failure mode.

### K3 — Per-subject key in an external cloud KMS

Each subject's key is created and destroyed in a managed external key service. The database stores only a key reference.

**Restore verdict: passes.** The restored reference points at a key the KMS no longer has; unwrap fails; the ciphertext in the restored database is inert.

**Two costs, both material:**

- **Erasure is not immediate.** Managed key services place a mandatory waiting period on key destruction (commonly measured in days, and cancellable during that window). Until the period elapses, an administrator can still recover the key. This is a real property that must be disclosed to whoever reviews the erasure policy, and it means "erased" has a **latency** that the product must state honestly.
- **Cost scales with subjects.** A per-key monthly charge multiplied by every subject holding personal data is structurally hostile to a per-merchant SaaS price point at any meaningful volume. See §6.

### K4 — External key store outside the backup scope, immediate hard delete

A small dedicated store — not the application database — holding wrapped subject keys, configured so that deletion is immediate and permanent.

**Restore verdict: passes, conditionally.** It passes *only if* the key store's own backup policy does not resurrect deleted records. A key store with point-in-time recovery defeats erasure for the length of its recovery window. This condition is the whole option.

**Trade-off:** you now operate a small stateful service. It must be durable, patched and available, and its loss means all personal data becomes unreadable. It must **never** live on the application container's local disk: on the current hosting entitlement there is no persistent disk, and container restarts would destroy it. Any design that puts keys next to the app is a design that loses keys.

### K5 — Erasure register outside the backup scope, keys retained

Erasure adds the subject to an append-only register held outside the database backup scope. The application consults the register before decrypting and refuses for erased subjects. Keys are not destroyed.

**Restore verdict: passes for access.** After a restore the application still refuses, because the register was not restored.

**The limitation must be stated plainly, not buried:** the ciphertext remains **decryptable in principle** by anyone holding the key-encryption key with direct database access, bypassing the application. This is **access revocation, not cryptographic erasure.** Whether access revocation satisfies a legal erasure obligation is a legal question, not an engineering one, and this document does not answer it. K5 is cheap, immediate and weak.

### K6 — Hybrid: register for immediacy, key destruction for finality

K5's register blocks decryption immediately and unconditionally; key destruction under K3 or K4 then provides cryptographic finality once it completes.

**Restore verdict: passes on both counts.** Immediate blocking, eventual cryptographic death, and the two mechanisms corroborate each other — if the register says erased but the key is still present, that is a detectable inconsistency.

**Cost:** K4's operational burden, plus the engineering to keep the two mechanisms in agreement.

### 2.1 Comparison

| | Survives DB restore | Immediate | Cost at scale | Ops burden | Strength if bypassed | Key-store outage |
|---|---|---|---|---|---|---|
| **K1** in-DB | ✗ | ✓ | negligible | low | — | n/a |
| **K2** in-DB vault | ✗ | ✓ | low | low | — | n/a |
| **K3** per-subject KMS | ✓ | ✗ (waiting period) | **high** | low | strong | PII unreadable |
| **K4** external store | ✓ *conditional* | ✓ | low | **medium-high** | strong | PII unreadable |
| **K5** register only | ✓ | ✓ | negligible | low | **weak** | PII blocked |
| **K6** hybrid | ✓ | ✓ | low–medium | medium | strong | PII blocked |

---

## 3. Restore-and-erasure threat scenario

**Day −2, 2026-03-01** — routine backup taken. Subject **S-77** has identity data encrypted under subject key DEK-77.

**Day 0, 2026-03-03** — S-77 exercises erasure. Personal data erased; the financial ledger is untouched, as approved. An audit event records the erasure.

**Day 4, 2026-03-07** — a faulty migration corrupts data. An operator restores the 2026-03-01 backup.

| Option | Outcome after restore | Detected? |
|---|---|---|
| **K1 / K2** | DEK-77 returns; ciphertext returns; personal data fully readable again. Erasure reversed. | **No — silently** |
| **K3** | Reference returns, key is gone or pending deletion → unwrap fails → ciphertext inert. **Hazard:** if deletion was still inside its waiting period and an administrator cancels it, data becomes readable again. | Only if cancellation is monitored |
| **K4** | Key store was never restored and still shows the key deleted → unwrap fails. The restored reference points at nothing, which must be read as *erased*, not as corruption. | Yes, if replay runs |
| **K5** | Ciphertext and keys return; register still lists S-77 → application refuses. **But** direct database access with the key-encryption key still decrypts. | Yes, at the application |
| **K6** | Blocked immediately by the register; cryptographically dead once destruction completes. | Yes, on both layers |

### 3.1 The failure nobody would notice

There is a second, subtler failure that no option in §2 prevents on its own, and it is the one most likely to occur in practice.

After the restore, S-77's identity row exists again. The next inbound webhook for that customer finds an identity record and writes fresh personal data against it. **The subject has been un-erased by an ordinary business event, and no erasure control fired**, because every control was about *reading* the old data, not about *preventing new* data.

The mitigation is a **tombstone**: the erasure record must be consulted on every *write* of personal data, not only on reads, and must cause the write to be refused or the subject to be re-erased immediately. This must be part of the design from the start, because retrofitting it means the product has been silently retaining data it claimed to have deleted.

---

## 4. Method: ensuring restored backups cannot resurrect erased data

Eight controls. The first two are structural; the rest are enforcement and evidence.

**M1 — Erasure register outside the backup scope.** An append-only register of erased subject references, held where a PostgreSQL restore cannot reach it. It is the authority on who is erased. It carries no personal data itself — only pseudonymous references and timestamps.

**M2 — Key state outside the backup scope.** Subject keys live under K3, K4 or K6 — never in the application database (which rules out K1 and K2 outright). The register and the key state must both be outside the restored scope, or the whole mechanism collapses.

**M3 — Startup gate, fail-closed.** On start, the application verifies that the register is reachable and internally consistent. If it cannot, **personal-data reads fail closed** with an explicit error. It does not serve degraded results, and it does not "assume nothing is erased". An unavailable register is indistinguishable from an empty one, and treating it as empty is how erasure gets silently undone.

**M4 — Restore is a gated operation, not a routine one.** Every restore writes a `restore_marker`. While `restore_marker` is newer than `last_erasure_replay_at`, the application refuses to serve personal data. The runbook requires an erasure replay before traffic resumes. A restore that skips the replay is visible in the data, not merely in a process document.

**M5 — Erasure replay.** For every subject in the register, verify the key is absent or unwrappable. If a key has reappeared — the exact signature of a restore — destroy it again and record a **restore remediation** audit event naming the subject and the restore it followed. This converts an invisible failure into a recorded one.

**M6 — Continuous verification.** A scheduled job samples erased subjects and attempts decryption. **Any success is a Sev-1 alert.** A full sweep runs on a slower cadence. This is the control that makes the guarantee evidence rather than assertion: the system proves, repeatedly, that erased data cannot be read.

**M7 — Tombstone on write.** As §3.1: every write of personal data checks the register first. An erased subject may not accumulate new personal data. This closes the re-establishment path.

**M8 — Audit without personal data.** Every erasure, replay, remediation and refusal is recorded in an append-only audit log containing only pseudonymous references, timestamps, actor and outcome. The audit log must not become a copy of the data it is meant to protect.

### 4.1 What this design does not claim

- It does not make the financial ledger erasable. The ledger is append-only by design (INV-19); erasure applies to identity data, and the ledger retains only pseudonymous references.
- It does not prove that data is unrecoverable from storage media, from a storage provider's own internal replicas, or from any backup taken *before* the mechanism existed.
- It does not resolve the legal question of whether access revocation (K5) counts as erasure.
- **None of it is implemented or tested.** No register, key store, gate, replay or verification job exists.

---

## 5. Durable erasure reconciliation mechanism

The mechanism that keeps erasure true over time, independent of any single restore.

### 5.1 Subject erasure lifecycle

| State | Meaning | Advance on |
|---|---|---|
| `requested` | Erasure accepted, recorded in the register | Immediate |
| `blocked` | Reads refused; tombstone active | Immediate, same transaction as `requested` |
| `key_destroyed` | Subject key destroyed or scheduled for destruction | Key store confirmation |
| `verified` | A decryption attempt has been observed to fail | Verification job |
| `closed` | Verified and past the key store's destruction latency | Scheduled |

The distinction between `key_destroyed` and `verified` matters: scheduling a deletion is not the same as observing that decryption fails, and only the second is evidence.

### 5.2 Reconciliation states

For each erased subject, the reconciler holds exactly one:

| State | Meaning | Response |
|---|---|---|
| `consistent` | Key absent or unwrappable; reads blocked | None |
| `key_present` | Key exists and unwraps — **erasure defeated** | Destroy again, record remediation, Sev-1 |
| `unverifiable` | Register or key store unreachable | Fail closed on reads, alert |

`unverifiable` must never be treated as `consistent`. This single rule prevents the most likely real-world regression.

### 5.3 Triggers

- Application startup (M3).
- Immediately after any restore (M4, M5).
- Scheduled full sweep, and sampled continuous checks (M6).
- Any `key_present` result escalates until it reaches `consistent`.

### 5.4 Reporting

The reconciler reports, per period: subjects erased, subjects verified, subjects in `key_present` (should be zero), subjects `unverifiable`, and the age of the oldest `key_destroyed` subject that has not reached `verified`. That last figure is the honest measure of erasure latency, and it is the number to disclose if the product ever claims a deletion timeframe.

---

## 6. Cost and operational assessment

Actual vendor prices are **not asserted here** and require confirmation against current provider pricing. What can be assessed is the **cost structure**, which is what determines whether an option is viable at all.

### 6.1 Cost structure

| Option | Driver | Shape |
|---|---|---|
| K1 / K2 | storage only | negligible — and non-compliant, so irrelevant |
| K3 | **price per key per month × subjects holding personal data** | **linear in customer count** |
| K4 | one small store + operator time | near-flat; burden is operational, not financial |
| K5 | register storage | negligible |
| K6 | K4's costs | near-flat |

### 6.2 Why K3's shape is the problem

Let `p` be the provider's monthly charge per managed key and `n` the number of subjects with personal data. K3 costs approximately `n × p` per month.

| Subjects `n` | Monthly key cost |
|---|---|
| 1,000 | 1,000 `p` |
| 10,000 | 10,000 `p` |
| 100,000 | 100,000 `p` |

A merchant-facing subscription price is roughly independent of how many *customers* that merchant has. K3 makes cost proportional to a number the merchant controls and we do not. At pilot volume it is trivial; at success it is the largest line in the cost base, and it grows exactly when the business is working. **This is a structural objection, not a pricing quibble**, and it is the reason per-subject KMS keys should not be the default even though they are technically the cleanest.

Storage for K4 or K5 is roughly a hundred bytes per subject, so at 100,000 subjects the data is on the order of tens of megabytes — negligible. The cost of K4 is **operational**: a durable small service to run, patch, monitor and keep backed up with a deletion policy that does not resurrect data. On a 1 vCPU / 1 GB deployment with no persistent disk, that service must be external and is therefore a dependency the product cannot run without.

### 6.3 Non-financial costs

| Cost | Note |
|---|---|
| Erasure latency | K3's destruction waiting period is a **product** statement, not an internal detail, if any deletion timeframe is ever published |
| Single point of failure | K4/K6: key-store unavailability makes personal data unreadable — correct behaviour, but it is an outage the product now has |
| Restore discipline | M4 adds a mandatory manual gate to every restore. Under incident pressure this is the step that gets skipped, which is why it is enforced in data rather than in a runbook |
| Engineering | Replay, verification, tombstone and reconciler must be built before real merchant data is accepted |

---

## 7. Recommendation

**Recommended: K6 hybrid** — register-based blocking for immediacy (K5) plus external key destruction for finality (K4's store, or K3 if per-subject KMS pricing is ever acceptable).

Reasons:

1. It is the only option that both blocks immediately **and** becomes cryptographically dead, so it does not depend on a legal judgement about whether access revocation is enough.
2. Its cost is near-flat in customer count, which keeps the business model intact as volume grows.
3. The two mechanisms corroborate each other: register-says-erased with key-still-present is a detectable, alertable inconsistency rather than a silent failure.
4. It survives the §3 restore scenario on both layers, and survives §3.1's re-establishment path through the tombstone.

**If K6 is too much for the pilot**, the minimum defensible position is **K5 with the limitation disclosed** and legal review confirming that access revocation satisfies the applicable obligation. That is a conscious acceptance of a weakness, not an equivalent design.

**K1 and K2 should not be used under any circumstances** for data that must be erasable: they fail the approved principle by construction.

**What must not happen:** selecting a paid KMS product before this comparison is reviewed. The recommendation above is a direction, not a purchase.

---

## 8. Open questions

**OQ-4 — Is access revocation sufficient?** Legal, not technical. Determines whether K5 alone is defensible.

**OQ-5 — What is the acceptable erasure latency?** A regulatory or contractual figure. Drives whether K3's waiting period is acceptable and whether K4's immediate deletion is required.

**OQ-6 — Retention periods remain unset.** Per the founder's direction, no periods are invented here. Retention must be set by jurisdiction and by applicable legal, tax and accounting requirements, and data minimisation must be applied rather than retaining everything for the longest financial-record period. **This blocks the retention policy, not the key-management architecture.**

**OQ-7 — Who operates the key store?** K4/K6 place a durable service in our operational scope. Whether that is acceptable at pilot stage, or whether a managed equivalent is preferred, is a founder decision with a cost consequence.

**OQ-8 — Backups taken before the mechanism existed.** If any backup predates the register, it can resurrect erased data. The policy for such backups (expire them, or accept and disclose) needs a decision.

---

## 9. Status

| Item | Status |
|---|---|
| Ledger / identity / audit separation | **Approved by founder** |
| Random pseudonymous references | **Approved by founder** |
| Erasure must survive backup restore | **Approved by founder** |
| Key-management options compared | Delivered here |
| Restore threat scenario | Delivered here |
| Restore protection method (M1–M8) | Proposed here |
| Erasure reconciliation mechanism | Proposed here |
| Cost and operational assessment | Delivered here — **prices require confirmation** |
| Key-management selection | **OUTSTANDING — founder decision** |
| Retention periods | **OUTSTANDING — legal/tax determination** |
| Implementation | None |
| Empirical verification | None |

---

## 10. Boundary statement

Documentation only. No application code, schema, migration, test, Supabase provisioning, key store, provider connection, merchant data or deployment. No vendor price is asserted. No claim is made that any control in this document has been implemented or verified.
