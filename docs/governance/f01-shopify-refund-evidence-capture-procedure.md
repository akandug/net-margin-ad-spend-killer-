# F-01 — Shopify Refund Evidence-Capture Procedure

**Project:** Net-Margin Ad Spend Killer (subtitle: Net-Margin Ad Spend Protection Platform)
**Document type:** Standalone operational procedure, run by the founder in their own store
**Created:** 2026-11-17
**Executed:** **NO** — no store was connected, no payload captured, no API called
**Closes:** nothing. F-01, D-09 and the affected part of INV-02 remain **OPEN** until the captured evidence is reviewed and the treatment is confirmed.

> **Why this exists.** D-09 asks whether Shopify's refund line-item subtotal is **net or gross** of the original discount. The official reference does not say. Every other route to answering it — preference, convention, founder approval, synthetic fixture — would encode a guess as a financial policy. This procedure is the only route that produces provider evidence.

---

## 0. What this capture can and cannot settle

| Question | Settled by this capture? |
|---|---|
| Is `RefundLineItem.subtotalSet` net or gross of the original discount? | **Yes** — this is the primary purpose |
| Is the refunded tax computed on the net (discounted) amount? | **Yes**, if the store has a tax rate configured |
| Does Shopify allocate an order-level discount to lines proportionally? | **Yes** — directly relevant to D-08 and INV-01 |
| Are shipping and tax refunds separate components from merchandise? | **Yes** |
| Does a refund record prove the customer was repaid? | **No** — never. A refund object is not settlement evidence (register §4.2.2) |
| Does the payment processor actually return its variable fee? | **No** — separate provider evidence (INV-22) |
| Which accounting period a refund belongs to | **No** — policy (D-18, INV-16) |
| Tax remittance and liability roll-forward | **No** — policy and statutory (INV-03, INV-24) |
| Inventory valuation effect of a restock | **No** — policy (D-07, INV-20) |

Anything marked "No" above must not be closed on the strength of this fixture.

---

## 1. Preconditions

### 1.1 Store

- A **Shopify development store** (or a development store created under a Partner account). Never a production store.
- **No real customer data.** Create the order yourself; do not import or refund a genuine merchant order.
- Store currency noted exactly (e.g. `USD`). Record it; do not convert anything.
- **Prices configured as tax-exclusive** (product prices do not include tax). Tax-inclusive stores are a *different* evidence question and would need a separate capture. Record the actual `taxesIncluded` value regardless.

### 1.2 Tax configuration — required, with a fallback

- Configure **one manual tax rate of 8%** (Settings → Taxes and duties) and apply it to both test products and to shipping.
- **If the store cannot be made to charge tax**, run the capture anyway and record `tax components: NOT VERIFIED`. Do not simulate tax by editing numbers afterwards. A fixture without tax settles the net/gross question but leaves the refunded-tax question open.

### 1.3 Products

Two products, so that a discount allocation across more than one line is observable:

| Ref | Product | Unit price | Qty | Line gross |
|---|---|---|---|---|
| **L1** | NM Test Linen Throw | 60.00 | 2 | 120.00 |
| **L2** | NM Test Ceramic Mug | 18.00 | 2 | 36.00 |

Both physical products, both requiring shipping, inventory tracked at one location.

> These amounts deliberately mirror the canonical worked dataset in `g0-g2-decision-preparation.md`, so the captured payload can be compared directly against worked example D.

### 1.4 Discount

One **order-level discount code** `NMTEST10` = **10% off order**, no minimum, no customer eligibility, usable once. Order-level (not line-level) because `orderCreate` cannot apply line-item discounts (§3.2), and because an order-level discount is exactly the case where net-vs-gross ambiguity bites.

---

## 2. Access and permissions

### 2.1 Minimum scopes — read-only capture (recommended)

The recommended path creates the order and refund **in the Shopify admin UI** and uses the API only to *read* the result. That needs read scopes only:

| Scope | Why |
|---|---|
| `read_orders` | Order, line items, refunds, transactions |
| `read_products` | Product and variant identity for the lines |
| `read_discounts` | Discount code and its application record |
| `read_inventory` | Inventory level before/after restock |
| `read_locations` | Restock location identity |
| `read_returns` | Return object, if the refund creates one |

**`read_all_orders` is NOT required and must not be requested.** It needs Shopify approval, and `read_orders` already covers orders created within the last 60 days — the test order is created minutes before capture.

### 2.2 Getting a token

1. In the development store admin: **Settings → Apps and sales channels → Develop apps → Create an app**.
2. Configure **Admin API** scopes exactly as §2.1. Request nothing else.
3. Install the app and copy the **Admin API access token** (`shpat_…`).
4. **Treat it as a secret.** It is never committed, never pasted into this repository, never included in the fixture, never shared in chat. Revoke it immediately after the capture (§9.4).

### 2.3 Optional Path B — API-driven capture

Only if you prefer to script it. Additional scopes `write_orders` and `read_discounts`/`write_discounts`, and an **offline** access token (`orderCreate` is available only to apps using offline tokens). See §3.2 for the constraints that make this path weaker evidence.

---

## 3. API version and endpoint

### 3.1 Version

**Pin the capture to Admin GraphQL `2026-10`**, the current stable version.

- Endpoint: `https://{shop}.myshopify.com/admin/api/2026-10/graphql.json`
- Method: `POST`, header `X-Shopify-Access-Token: <token>`, `Content-Type: application/json`
- Record the version string from the URL **verbatim** in the evidence log. A payload without a pinned version is not reproducible evidence.

### 3.2 Why the admin UI is the recommended path for creating the order

Verified from the official `orderCreate` reference: it **"doesn't support applying multiple discounts, such as discounts on line items"**, it applies **one discount code** only, automatic discounts are not applied unless replicated, it is restricted to **offline tokens**, and development stores are limited to **5 new orders per minute**.

Consequence: an API-created order cannot reproduce a line-level discount, and its discount allocation may not be the allocation a real checkout produces. Since the whole point of F-01 is to observe **what Shopify actually sends for a real discounted order**, the checkout/admin path is the stronger evidence. Path B remains acceptable only for the order-level discount case, and must be labelled as such.

> The refund itself is created by Shopify's refund engine in both paths. If you use Path B, `refundCreate` requires an `@idempotent` key from API version `2026-04` onward — omitting it is rejected. If `refundCreate` rejects the transaction because no capturable payment exists, **perform the refund in the admin UI instead** and read it back; do not fabricate a transaction.

---

## 4. Step-by-step capture

### 4.1 Create the order

1. Admin → **Orders → Create order**.
2. Add L1 ×2 and L2 ×2.
3. Apply discount code `NMTEST10`.
4. Set shipping to a flat **9.00** (create a manual "NM Test Shipping" rate if needed).
5. Confirm the summary shows: merchandise 156.00, discount 15.60, shipping 9.00, tax 11.95, total 161.35. **If the totals differ, stop and record the actual figures** — do not adjust the store to force these numbers.
6. Mark as **paid** using the **Bogus Gateway** test payment (dev stores only), so a refundable transaction exists.
7. **Record the order name and ID before doing anything else.**

### 4.2 Capture the order (Query 1)

Run this **before** the refund, so the original allocations are captured unmodified. Replace `{ORDER_ID}` with the `gid://shopify/Order/…` value.

```graphql
query CaptureOrder($id: ID!) {
  order(id: $id) {
    id
    name
    legacyResourceId
    createdAt
    processedAt
    cancelledAt
    currencyCode
    presentmentCurrencyCode
    taxesIncluded
    dutiesIncluded
    displayFinancialStatus
    displayFulfillmentStatus
    subtotalPriceSet { shopMoney { amount currencyCode } }
    totalDiscountsSet { shopMoney { amount currencyCode } }
    totalTaxSet { shopMoney { amount currencyCode } }
    totalShippingPriceSet { shopMoney { amount currencyCode } }
    totalPriceSet { shopMoney { amount currencyCode } }
    cartDiscountAmountSet { shopMoney { amount currencyCode } }
    discountCodes
    discountApplications(first: 10) {
      nodes { __typename allocationMethod targetSelection }
    }
    shippingLines(first: 10) {
      nodes {
        id
        title
        originalPriceSet { shopMoney { amount currencyCode } }
        discountedPriceSet { shopMoney { amount currencyCode } }
        taxLines { title rate priceSet { shopMoney { amount currencyCode } } }
      }
    }
    lineItems(first: 20) {
      nodes {
        id
        name
        sku
        quantity
        currentQuantity
        refundableQuantity
        unfulfilledQuantity
        originalUnitPriceSet { shopMoney { amount currencyCode } }
        originalTotalSet { shopMoney { amount currencyCode } }
        totalDiscountSet { shopMoney { amount currencyCode } }
        discountedTotalSet { shopMoney { amount currencyCode } }
        discountedTotalWithCodeSet: discountedTotalSet(withCodeDiscounts: true) {
          shopMoney { amount currencyCode }
        }
        discountedUnitPriceSet { shopMoney { amount currencyCode } }
        priceAfterAllDiscountsBeforeTaxesSet { shopMoney { amount currencyCode } }
        taxLines { title rate priceSet { shopMoney { amount currencyCode } } }
        discountAllocations {
          allocatedAmountSet { shopMoney { amount currencyCode } }
          discountApplication { __typename allocationMethod targetSelection }
        }
        variant { id sku }
      }
    }
  }
}
```

Save the **complete raw JSON response** unmodified.

> If Shopify rejects a field, delete **only** that field, re-run, and record the exact error text. A removed field is a gap in the evidence and must be listed in §8.

### 4.3 Perform the partial refund

1. Open the order → **Refund**.
2. Refund **1 unit of L1 only** (of 2) — the discounted line, partially refunded.
3. Set **Restock = Restock inventory** for that unit. Note the location.
4. Refund the merchandise amount Shopify proposes. **Do not hand-edit the refund amount.** If you must edit it, record the proposed and edited values and treat the fixture as weaker evidence.
5. **Do not refund shipping.** Shipping-refund behaviour is a separate question; if you also want it, do it as a **second, separate refund** so the two are distinguishable.
6. Complete the refund. Record the timestamp.

**Optional second refund (recommended, separate):** refund the remaining 1 unit of L1 with **Restock = Do not restock**, to evidence the write-off path for INV-20. Keep it clearly separate from Refund 1.

### 4.4 Capture the refund (Query 2)

```graphql
query CaptureRefunds($id: ID!) {
  order(id: $id) {
    id
    name
    displayFinancialStatus
    totalRefundedSet { shopMoney { amount currencyCode } }
    refunds(first: 10) {
      nodes {
        id
        createdAt
        processedAt
        note
        totalRefundedSet { shopMoney { amount currencyCode } }
        refundLineItems(first: 10) {
          nodes {
            id
            quantity
            restocked
            restockType
            priceSet { shopMoney { amount currencyCode } }
            subtotalSet { shopMoney { amount currencyCode } }
            totalTaxSet { shopMoney { amount currencyCode } }
            location { id name }
            lineItem {
              id
              name
              sku
              quantity
              currentQuantity
              refundableQuantity
              originalUnitPriceSet { shopMoney { amount currencyCode } }
              totalDiscountSet { shopMoney { amount currencyCode } }
              discountedUnitPriceSet { shopMoney { amount currencyCode } }
              discountAllocations {
                allocatedAmountSet { shopMoney { amount currencyCode } }
              }
            }
          }
        }
        refundShippingLines(first: 5) {
          nodes {
            id
            amountSet { shopMoney { amount currencyCode } }
            subtotalAmountSet { shopMoney { amount currencyCode } }
            taxAmountSet { shopMoney { amount currencyCode } }
          }
        }
        transactions(first: 10) {
          nodes {
            id
            kind
            status
            gateway
            processedAt
            amountSet { shopMoney { amount currencyCode } }
            parentTransaction { id }
          }
        }
      }
    }
  }
}
```

The **only fields that settle D-09** are, per refund line: `subtotalSet`, `priceSet`, `totalTaxSet`, `quantity`, and the parent line's `totalDiscountSet` / `discountAllocations`. Everything else is corroboration.

### 4.5 Capture supporting state

- Inventory level of L1 before the refund and after (restock evidence).
- The refund's appearance in the admin UI (screenshot, or the displayed amount).
- The store's `taxesIncluded` setting and the tax rate applied.

---

## 5. Pre-stated expectation — the discriminator

**State this before running the capture.** The comparison is only a test if the expectation was recorded first.

Per-unit figures for L1, from the canonical dataset:

```
L1 line gross            120.00   (2 × 60.00)
Order discount 10%        15.60
L1 allocated discount     12.00   (proportional — TO BE VERIFIED, not assumed)
L1 net merchandise       108.00
L1 tax at 8%               8.64
L1 per-unit net           54.00
L1 per-unit tax            4.32
```

Refunding 1 of 2 units produces one of these:

| Hypothesis | `subtotalSet` | `totalTaxSet` | Cash refunded | Reading |
|---|---|---|---|---|
| **H-NET** | **54.00** | **4.32** | 58.32 | Subtotal is **net of discount**. D-09 option A holds: book the merchandise refund as-is, **no separate discount reversal** |
| **H-GROSS** | **60.00** | **4.80** | 64.80 | Subtotal is **gross**. The engine must reverse the allocated discount (6.00) as its own event, or INV-02 double-counts |
| **H-MIXED** | 60.00 | 4.32 | 64.32 | Gross merchandise with correctly netted tax — internally inconsistent. **Escalate.** Neither option applies without an explicit, documented rule |
| **H-OTHER** | anything else | — | — | Record verbatim, escalate. Do not reinterpret the numbers to fit a hypothesis |

The discriminator is **6.00 in subtotal and 0.48 in tax**. Both are large enough that rounding cannot explain them.

**Second question this capture answers (D-08 / INV-01):** whether the 15.60 order discount was allocated to lines at all, and whether the split is the proportional 12.00 / 3.60. If `discountAllocations` is empty on an order that carries an order-level discount, then line-level discount allocation must come from `discountApplications` + `cartDiscountAmountSet` instead — a material engine design change, not a detail.

**Third question:** whether `totalDiscountSet` (documented as *excluding* order-level discounts) is 0.00 for L1 despite a 12.00 order-level allocation. If so, any engine reading `totalDiscountSet` alone **silently loses the order discount** — a live double-count risk for INV-01.

---

## 6. Redaction procedure

Redact by **not requesting** sensitive fields first — the queries above deliberately omit customer, email, phone, and address.

Then, on the raw file:

| Item | Action |
|---|---|
| Customer object / name / email / phone / address | **Never requested.** If present anyway, delete the whole object and log it |
| Access token, `X-Shopify-Access-Token` | **Never written to the file.** If captured in a header log, delete before saving |
| Shop domain (`*.myshopify.com`) | Replace with `nm-test-store.example` **consistently**; keep the real mapping in your private notes, not in Git |
| Staff member, IP addresses, user agent | Delete |
| GIDs (`gid://shopify/Order/123…`) | **Keep unmodified.** They are the evidence of genuineness and carry no PII |
| Amounts, currencies, tax rates, dates, quantities, `restockType` | **Never edit.** These are the evidence |

**Rules:** redaction may only *remove* or *pseudonymise identifiers* — never alter a financial value, a date, a quantity or a status. Every redaction is written into a **redaction log** (§8) with the JSON path touched and why. A fixture with an unlogged edit is not evidence.

---

## 7. Validating that the fixture is genuine

Before the fixture is accepted, all of the following must hold:

1. **Provenance.** The file is the unmodified response body from a `POST` to `/admin/api/2026-10/graphql.json` on the dev store, captured by you, with the query text used recorded verbatim.
2. **GID shape.** `id` values match `gid://shopify/Order/…`, `gid://shopify/Refund/…`, `gid://shopify/RefundLineItem/…`, `gid://shopify/LineItem/…`.
3. **GraphQL envelope.** The response contains `data` and no `errors` array. If `errors` is present, the fixture is invalid.
4. **Cost envelope.** `extensions.cost.throttleStatus` is present — evidence the response came from Shopify's API rather than a hand-built file.
5. **Internal arithmetic.** `totalRefundedSet` equals the sum of the refund line amounts plus any shipping refund. `lineItem.refundableQuantity` decreased by exactly the refunded quantity. Quantities reconcile: original 2 − refunded 1 = 1.
6. **Cross-check against the admin UI.** The refunded amount displayed in the admin matches the payload exactly.
7. **Timestamp plausibility.** `refund.createdAt` / `processedAt` are after `order.processedAt` and consistent with your capture log.
8. **Hash.** Record a SHA-256 of the raw file at capture time, and of the redacted file after redaction, in the evidence log.
9. **Consistency of the pair.** Query 1 (pre-refund) and Query 2 (post-refund) describe the **same order GID** and the same line GIDs.

**If any check fails, the fixture is rejected.** A rejected fixture does not close D-09 and does not get "repaired" by editing values.

---

## 8. Evidence log template

Complete this alongside the fixture.

```
Capture date/time (UTC):
Captured by:
Shop (pseudonymised):            nm-test-store.example
Shop currency:                   ____        taxesIncluded:  true / false
Admin GraphQL version:           2026-10
Endpoint:                        /admin/api/2026-10/graphql.json
Order GID:                       gid://shopify/Order/________
Order name:                      #________
Refund GID(s):                   gid://shopify/Refund/________
Discount code:                   NMTEST10 (10% order-level)
Tax rate applied:                ____%
Products:                        L1 60.00 ×2 | L2 18.00 ×2
Shipping charged:                9.00
Order totals as displayed:       merch ____ disc ____ ship ____ tax ____ total ____
Refund 1: qty ____ of L1, restock = ____, cash ____
Refund 2 (optional): qty ____ of L1, restock = ____, cash ____

RAW FILE sha256:                 ____________________
REDACTED FILE sha256:            ____________________

Observed RefundLineItem.subtotalSet:  ________
Observed RefundLineItem.totalTaxSet:  ________
Observed refund cash:                 ________
Matches hypothesis:              H-NET / H-GROSS / H-MIXED / H-OTHER

Order-level discount allocated to L1?   yes / no — amount ________
Was the split proportional (12.00 / 3.60)?  yes / no — observed ________
L1 totalDiscountSet value:       ________  (0.00 despite allocation? yes / no)

Fields removed because Shopify rejected them: ____________________
Redaction log:                   ____________________
Store could not charge tax:      yes / no  (if yes, tax evidence NOT VERIFIED)
Deviations from this procedure:  ____________________
```

---

## 9. Storage and handling

### 9.1 Where the raw file lives

- **Raw capture: outside Git.** Keep it in private, access-controlled storage. It is the master evidence and must never be overwritten.
- **Redacted fixture: in Git**, at `docs/evidence/f01/`, as:
  - `shopify-refund-<YYYY-MM-DD>-order.json` (Query 1 response)
  - `shopify-refund-<YYYY-MM-DD>-refund.json` (Query 2 response)
  - `shopify-refund-<YYYY-MM-DD>-findings.md` (the §8 log plus interpretation)
- **Never committed:** access tokens, real shop domain, customer data, unredacted production payloads, screenshots showing customer details.

### 9.2 Integrity

- Record both SHA-256 hashes in the findings file.
- The committed fixture is **append-only evidence**. A correction is a **new** fixture and a new findings entry, never an edit to the existing one — the same rule the ledger follows (INV-19).

### 9.3 What the fixture may be used for

- Settling field semantics, and as a **regression input** for the calculation engine once D-09 is decided.
- It is **not** a test of the engine: it is an input, and its expected output is decided after review, not derived from itself.

### 9.4 After capture

- **Revoke the Admin API token** and delete the app, unless a later authorised phase needs it.
- Record the revocation date in the findings file.

---

## 10. What happens with the result

1. The founder supplies the redacted fixture and the completed evidence log.
2. Each of the three questions in §5 is answered as **verified**, **contradicted**, or **still unresolved**, with the exact field and value cited.
3. **D-09 is not closed by the fixture.** The fixture establishes the provider fact; closing D-09 also requires the founder's decision and, for presentation, professional accounting review.
4. If the basis is confirmed, the interim `refund_basis = 'provider_reported_unconfirmed'` assumption is superseded by a **versioned recalculation** — no history rewritten.
5. If the result is H-MIXED or H-OTHER, D-09 stays open and the engine must not be built on either option until a rule is agreed.

**Nothing in this procedure authorises code, schema, migrations, Supabase provisioning, provider connections, or deployment.**

---

## 11. Boundary statement

This document is a **procedure**. It records no provider behaviour, because none was observed. No Shopify account was created, no store was connected, no API was called, no token was requested, and no fixture exists. F-01, D-09 and the affected part of INV-02 remain **OPEN**.

The one thing this procedure *does* establish is negative and useful: the official reference describes `RefundLineItem.subtotalSet` only as "subtotal" and never states its discount basis, while several neighbouring `LineItem` fields are explicitly documented as *including* refunded and removed quantities and as *excluding* order-level discounts. Those documented quirks are exactly the traps that would silently corrupt contribution profit, and they are why the capture must be run rather than reasoned about.
