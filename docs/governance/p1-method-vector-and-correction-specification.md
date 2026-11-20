# P-1 — Method Vector and Correction Specification

**Project:** Net-Margin Ad Spend Killer — Net-Margin Ad Spend Protection Platform
**Derived from:** `p1-shipping-association-selector-decision-brief-rev2.md` (rev 2)
**Responds to:** founder decision round 2026-11-20 §2 and §3
**Status:** PROPOSED FOR FOUNDER DECISION — **not approved, not implemented, no test executed**
**Affected decision:** D-10a (APPROVED WITH CONDITIONS) · C-10a.1 (PARTIALLY SATISFIED — OPEN) · INV-27

---

## 1. Scope

This document fixes the exact allowed values of the association method vector (§2), enumerates every association shape the model must express (§3), proves the identity cannot collide (§4), and specifies the correction and versioning workflow (§6). It adds detail to rev 2; it does not change rev 2's principles.

---

## 2. The complete method vector

### 2.1 Definition

For an order `O` with accepted charges `C(O) = {c₁ … cₙ}`, the method vector is the ordered set of per-charge tuples:

| Field | Allowed values | Cardinality | Set at |
|---|---|---|---|
| `charge_identity` | provider-assigned stable charge identifier | 1 per charge | acceptance |
| `method` | `D` (direct) · `P` (pooled) | 2 | allocation instant |
| `granularity` | `G-unit` · `G-line` · `G-shipment` · `none` | 4 | allocation instant |
| `scope_ref` | the associated unit / line / shipment identity; **required** when `method = D`, **must be absent** when `method = P` | — | allocation instant |
| `evidence_ref` | identity of the payload record relied on; **required** when `method = D`, optional when `P` (records an invalid or conflicting record) | — | allocation instant |

**Allowed combinations are not the full 2 × 4 product.** The valid pairs are:

| `method` | Permitted `granularity` | Meaning |
|---|---|---|
| `D` | `G-unit` | charge associated to specific unit indices of a line |
| `D` | `G-line` | charge associated to a whole line (within-line split required) |
| `D` | `G-shipment` | charge associated to a shipment spanning one or more lines |
| `P` | `none` | charge pooled across all eligible units of the order |

`P` with any `G-*` value, and `D` with `none`, are **invalid vectors** and must fail the vector validity check (§5) — they are unrepresentable states, not permitted ones.

Two additional frozen fields accompany the vector and enter the identity:

| Field | Purpose |
|---|---|
| `snapshot_id` | the immutable payload snapshot the decision was read from |
| `allocation_version` | monotonic integer within the order, starting at 1 |

### 2.2 Canonical serialisation

The vector must be serialised **injectively** — two different vectors must never produce the same string. Naive delimiter joining is not injective, because a provider identifier may contain the delimiter.

```
canonical(v1) = "v1"
              ‖ for each charge in ascending UTF-8 byte order of charge_identity:
                  "|" ‖ decimal(len(charge_identity)) ‖ ":" ‖ charge_identity
                      ‖ ":" ‖ method
                      ‖ ":" ‖ granularity
                      ‖ ":" ‖ scope_ref_or_empty
                      ‖ ":" ‖ evidence_ref_or_empty
```

`len` is the **byte** length of the UTF-8 encoding, in ASCII decimal, with no leading zeros.

**Why the length prefix is required.** Without it, a charge identity of `c1:D` and a charge identity of `c1` with method `D` both begin `|c1:D`. With it, they are `|5:c1:D` and `|2:c1:D:` — distinguishable. The encoding is injective because every variable-length field is length-delimited and every fixed-vocabulary field is drawn from a closed set.

Charges are ordered by `charge_identity` **byte comparison**, never by payload array position, never by ingestion order. This is the same discipline as P-7b.

### 2.3 Digest and independent reconstruction

```
association_identity_digest = SHA-256( UTF-8( canonical(v1) ) )
```

Both the **canonical string** and its **digest** are stored in the allocation record. The vector itself is immutable once written.

The founder's requirement — a representation that "can be independently reconstructed from the immutable underlying vector" — is satisfied structurally, not by trust:

1. The canonical string is stored in full, so the vector can be re-derived without the digest.
2. Anyone can recompute `SHA-256(canonical)` and compare it to the stored digest.
3. If they disagree, the allocation record has been altered, and the check fails loudly.

**A digest alone would not satisfy the requirement.** A hash proves nothing on its own; it is only useful when the preimage is retained and the recomputation is possible. Retaining the canonical string is what makes the identity auditable rather than merely unique.

### 2.4 Why the digest enters the identity and the label does not

The summary label (`A1` / `A2` / `A3`) is derived for display. It is **never** an identity element. §4 shows why.

---

## 3. Every supported association shape

Nine shapes. Each row states what the snapshot contains, the resulting vector entry, and the outcome. "Frozen" means the decision cannot be revisited by later evidence.

| # | Shape | Snapshot at acceptance | Vector entry | Outcome |
|---|---|---|---|---|
| 1 | **Direct, line** | charge → line L2, valid | `(c, D, G-line, L2)` | Charge split across L2's units by largest remainder |
| 2 | **Direct, unit** | charge → units {A1, A2} of L1 | `(c, D, G-unit, L1#[0,1])` | Exact per-unit amounts |
| 3 | **Direct, shipment** | charge → shipment S1 spanning L1, L3 | `(c, D, G-shipment, S1)` | Split across the shipment's units by largest remainder over that scope |
| 4 | **Pooled** | no association record | `(c, P, none)` | Charge split across all eligible units of the order |
| 5 | **Multi-unit line** | charge → line L1 holding 4 units | `(c, D, G-line, L1)` | Not a special case — it is shape 1. The within-line largest remainder is mandatory, which is why rev 1's "direct needs no arithmetic" assumption was wrong |
| 6 | **Present but invalid** | record exists, fails §5 | `(c, P, none, evidence_ref=record)` | Pooled, reason recorded. **Never trusted** |
| 7 | **Missing** | no record, no error | `(c, P, none)` | Pooled — the safe default |
| 8 | **Conflicting** | two records disagree for one charge | `(c, P, none, evidence_ref=both)` + `association_conflict` flag | Pooled. Not resolved by preferring one record |
| 9 | **Late-arriving** | record absent at acceptance, arrives later | Vector unchanged (frozen from shape 4 or 7) | `association_evidence_late` annotation + `A3_contested` if it contradicts the frozen decision |

**Shapes 6, 8 and 9 all fail toward the less-attributed state.** This is deliberate and is the single most important behavioural rule in P-1: when the provider's own data is internally inconsistent, a confident direct allocation is worse than an honest pooled one, because the confident answer is wrong in a way nobody will question.

**Shape 9 deserves emphasis** because it is where rev 1 was silent. Late evidence does not reallocate. It annotates, and it may *propose* a correction (§6) — a proposal that requires a human authorisation event. Automatic reallocation on late evidence is not recommended at any stage: it would mean the same order reports different product-level figures on different days with no event to explain the change.

### 3.1 Quantity-level association and partial coverage

If a charge is associated to some but not all units of a line (shape 2), the remaining units are **not** allocated a share of that charge. They receive shipping revenue only from charges where they are in scope, plus any pooled charge. A unit may therefore have zero allocated shipping revenue while the order carries shipping charges — a legitimate state that must be representable rather than silently corrected.

### 3.2 Association evidence arriving after acceptance

| Stage | Permitted effect |
|---|---|
| Before acceptance | Becomes the frozen decision |
| After acceptance, before shipment | Annotation only; frozen decision stands |
| After partial shipment | Annotation only; frozen decision stands |
| After full shipment | Annotation only; frozen decision stands |
| Original found incorrect | Correction request → §6 authorisation → new allocation version |

The vector is a function of the acceptance snapshot. No later stage changes it. That is what makes `allocation_version 1` reproducible forever.

---

## 4. Identity collision — proof that labels are insufficient

### 4.1 The failure case

Two orders, each with two charges, both summarising to `A2`:

| Order | Charge | Method | Granularity | Vector |
|---|---|---|---|---|
| X | c1 | `D` | `G-line` | `[(c1,D,G-line),(c2,P,none)]` |
| X | c2 | `P` | `none` | |
| Y | c1 | `P` | `none` | `[(c1,P,none),(c2,D,G-line)]` |
| Y | c2 | `D` | `G-line` | |

Both are `A2`. Their product-level shipping distributions are not comparable. Under rev 1's proposal — the summary basis as identity element — X and Y would share an identity and could be aggregated or compared as equivalent. That is precisely the silent-comparison failure INV-27 exists to prevent.

### 4.2 The injectivity argument

Let `V` be the set of valid vectors and `canonical : V → Σ*`. §2.2's encoding is injective:

1. The leading `"v1"` fixes the encoding version.
2. Each charge contributes exactly one record, and records are separated by `|`, which cannot occur in the length-prefixed portion because the length prefix delimits it.
3. Within a record, `method` and `granularity` are drawn from closed sets containing no `:`, and `scope_ref`/`evidence_ref` are preceded by their own field boundaries.
4. Every variable-length field is length-prefixed, so no two distinct field values can produce the same byte sequence.

Therefore `canonical(V₁) = canonical(V₂) ⟹ V₁ = V₂`.

### 4.3 The digest step

`SHA-256` is collision-resistant under standard assumptions. Combining an injective encoding with a collision-resistant digest gives a collision-resistant identity over vectors.

**Residual risk, stated honestly:** SHA-256 collision resistance is an assumption about a cryptographic primitive, not a theorem about this system. The practical safeguard is not the hash — it is that the canonical string is stored, so any collision or tampering is *detectable* by recomputation rather than silently exploitable.

### 4.4 Worked demonstration

| Vector | Canonical string | Differs? |
|---|---|---|
| `[(c1,D,G-line,L1),(c2,P,none)]` | `v1\|2:c1:D:G-line:L1:\|2:c2:P:none::` | — |
| `[(c1,P,none),(c2,D,G-line,L1)]` | `v1\|2:c1:P:none::\|2:c2:D:G-line:L1:` | **Yes** |

Identical label, different canonical string, different digest, therefore different calculation identity. The label may be displayed to a merchant; it may never be substituted for the vector in the identity.

---

## 5. Vector validity check (fail-closed)

A vector is valid only if **all** hold:

| # | Test | Failure behaviour |
|---|---|---|
| V1 | Every charge in the accepted set appears exactly once | Calculation fails closed |
| V2 | Every `method` is in `{D, P}` | Fails closed |
| V3 | `method`/`granularity` is one of the four permitted pairs (§2.1) | Fails closed |
| V4 | `method = D` ⟹ `scope_ref` present and resolvable within the accepted set | Charge demoted to `P`, reason recorded |
| V5 | `method = P` ⟹ `scope_ref` absent | Fails closed (malformed record) |
| V6 | `snapshot_id` present and resolvable | Fails closed |
| V7 | Canonical string recomputes to the stored digest | Fails closed — evidence of alteration |

**V7 is the integrity test.** V1–V6 test the vector's correctness; V7 tests whether the stored record still describes the vector it claims to.

Note the asymmetry: V4 *degrades* a single charge (to the safe pooled state) and records why. Everything else fails the whole calculation closed. A bad association on one charge should not be allowed to silently corrupt one product's figures, but a malformed vector or a broken digest means the record itself is untrustworthy, and no figure computed from it should be shown.

---

## 6. Correction and versioning

### 6.1 The version chain

```
allocation_version 1  ──superseded_by──▶  allocation_version 2  ──▶  n+1 …
        │                                         │
   immutable, retained                       immutable, current
```

Each version stores: the vector, the canonical string, the digest, the `snapshot_id`, the `calculation_version`, the actor and time of creation, and `supersedes` (null for v1).

**No version is ever updated or deleted.** A correction adds a record; it does not edit one. This is the same append-only discipline as INV-19, applied to allocation rather than to financial events.

### 6.2 Evidence supporting the original version

The original version's evidence is whatever the acceptance snapshot contained, referenced by `snapshot_id` and `evidence_ref`. The snapshot is immutable and retained, so a reviewer can reconstruct *why* v1 decided as it did, even after v2 supersedes it. Without snapshot retention the correction workflow would be an argument about memory rather than about evidence.

### 6.3 Correction request and authorisation

| Step | Record | Content |
|---|---|---|
| 1 | `correction_requested` | Requester, timestamp, the charge(es) disputed, the claimed association, the evidence relied on. **Produces no financial change** |
| 2 | Investigation | Read-only review. No record mutation |
| 3 | `correction_authorised` | Authoriser, timestamp, reason, evidence reference, `allocation_version` superseded. A deliberate human act |
| 4 | `allocation_version n+1` | Computed by the engine under the new vector |

A request is not an authorisation. Splitting them is what allows a merchant to dispute a figure without being able to change it.

### 6.4 New version and new identity

The new version has:
- a new `allocation_version`;
- a new canonical string and digest;
- a new `calculation_version`, therefore a **new INV-27 calculation identity**.

It is a different calculation, not a corrected one. INV-27's guarantee — that two figures may be compared only if their identities match — therefore prevents a report from mixing v1 and v2 figures without saying so.

### 6.5 Effect on historical reports

| Question | Answer |
|---|---|
| Does a past report change? | **No.** A report is bound to the identity it was computed under. It remains reproducible as issued |
| What does a current report show? | The current version, with a disclosure that a correction occurred and when |
| Can v1 still be recomputed? | Yes — the snapshot, vector and calculation version are all retained |
| Is the change visible to the merchant? | Yes, as a disclosed correction, not as a silent revision |

### 6.6 Subsequent calculations

Later periods use the current version. They do **not** retroactively re-run v1 figures. A store-period figure spanning the correction therefore contains a mixture of versions by design, and its own identity records that fact.

### 6.7 What a correction can and cannot change

The store-level shipping total is invariant under reallocation (proved in rev 2 §8.2 and re-verified: 11,200 in every method). A correction moves the *product-level distribution* and cannot move store contribution profit.

This is worth stating plainly to merchants, because it inverts the intuition: an association correction is financially small and reputationally large. That asymmetry is exactly why step 3 is an authorised, audited act rather than a background repair.

**A correction that changes the total is not an association correction.** It is a charge or reduction correction and belongs to P-4/P-5's control.

### 6.8 Worked correction

Order P1-R2: units A (merchandise net 100, line L1) and B (9,900, line L2); charges `c1 = 1,000` → L1, `c2 = 200` → L2; `S_net = 1,200`.

| Version | Vector | A | B | Total |
|---|---|---|---|---|
| v1 | `[(c1,D,G-line,L1),(c2,P,none)]` | 1,002 | 198 | 1,200 |
| v2 | `[(c1,D,G-line,L1),(c2,D,G-line,L2)]` | 1,000 | 200 | 1,200 |

`c2` was associated to L2 all along; the v1 snapshot lacked the record. After authorisation, v2 is current. Both reconcile to 1,200 — the total did not move. Product A's reported shipping revenue moved by 2, and product B's by 2, on this deliberately extreme order; on the same order under full pooling the difference would have been 988 (rev 2 §8).

### 6.9 MVP versus pilot — the split you asked about

**Yes, the authorisation workflow can be deferred to pilot while the versioned architecture remains MVP.** The argument is a safety argument, not a convenience one:

| Component | Classification | Why |
|---|---|---|
| `allocation_version` on every allocation, starting at 1 | **MVP** | Without it, a correction is impossible without destroying history |
| Immutable allocation records, never updated | **MVP** | The append-only precondition for any later correction |
| `supersedes` link and version chain | **MVP** | Cheap now; impossible to backfill honestly later |
| `snapshot_id` retention | **MVP** | Without it, v1 is not reproducible and correction becomes memory |
| Digest + canonical string stored | **MVP** | Enables V7 and independent reconstruction |
| Engine able to compute `n+1` from a given vector | **MVP** | Small; it is the same code path with a different vector |
| `correction_requested` / `correction_authorised` events | **Pilot** | Requires roles, UI and audit surface |
| Merchant-facing dispute flow | **Pilot** | Product surface, not financial integrity |
| Automatic correction on late evidence | **Never recommended** | Destroys reproducibility |

**The safety argument:** if the versioned architecture is absent at MVP, the only way to fix a wrong association later is to overwrite it — which violates INV-19 and destroys the evidence needed to defend the earlier figures. If the authorisation workflow is absent at MVP, corrections simply cannot be initiated. Nothing incorrect is produced; merchants merely cannot yet dispute a figure. **Absence of a workflow is safe. Absence of versioning is not.** That asymmetry justifies the split.

---

## 7. MVP versus deferred granularity

| Granularity | Classification | Condition |
|---|---|---|
| `G-line` | **MVP** | The common shape; handles multi-unit lines via within-scope largest remainder |
| `G-unit` | **MVP if exposed, else deferred** | Depends on OQ-P1-1; the code path is shared with `G-line` |
| `G-shipment` | **Deferred** | Requires shipments to be identifiable before shipping occurs (OQ-P1-3), which no verified provider evidence yet supports |
| `P` (pooled) | **MVP** | The mandatory safe default |
| Merchant-selectable basis | **Not recommended** | A merchant who chooses the basis chooses the number |

**The load-bearing unknown remains OQ-P1-1.** If no MVP provider exposes a genuine line-level association, shapes 1–5 are unreachable and every order is `A3`. The mechanism should still be built, because it is what makes pooled a *recorded decision with evidence* rather than an accident — but the merchant-facing claim must then be honest: product-level shipping attribution is pooled, and no better basis is currently available.

---

## 8. Open questions

| ID | Question | Needs |
|---|---|---|
| OQ-P1-1 | Does any MVP provider expose line-level shipping-charge association? | Provider documentation (P-01/P-02) |
| OQ-P1-2 | Is that association stable across payload revisions, or positional? | Provider docs + fixture |
| OQ-P1-3 | For `G-shipment`, how are shipments identified before shipping occurs? | Provider docs |
| OQ-P1-4 | Correction authorisation: merchant-initiated, founder-approved, or both? | **Founder decision** (governance) |
| OQ-P1-5 | Is an association conflict a merchant-visible error or a silent pooled fallback? | **Founder decision** (product behaviour) |
| OQ-P1-6 | Is the digest algorithm and canonical encoding frozen as part of the identity version (`v1`)? | Founder decision (identity governance) |

---

## 9. Recommendation

Adopt the vector as specified in §2, the nine shapes in §3, the injective encoding plus stored canonical string in §2.2–2.3, and the correction-by-new-version chain in §6.

**MVP:** §6.9's mandatory rows — versioning, immutability, digest, snapshot retention, `G-line`, pooled default, validity tests V1–V7.
**Pilot:** authorisation workflow, dispute surface.
**Deferred:** `G-shipment`, merchant-selectable basis.

**This is a recommendation, not an approved rule.** D-10a's status is unchanged and C-10a.1 remains open. No invariant has been implemented or tested.
