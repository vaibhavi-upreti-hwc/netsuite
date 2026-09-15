# NetSuite PRs — organized by real business sequence

## Segment 1 — Order Sync (Sales to Cash)

- `mantle-netsuite-connector` **#400** (open issue, the master ticket) — "Sales order sync to NetSuite through the REST record API, chosen by a rule group" — https://github.com/hotwax/mantle-netsuite-connector/issues/400
- **#398** (open) — "create#NetSuiteSalesOrder and create#NetSuiteCustomer" — https://github.com/hotwax/mantle-netsuite-connector/pull/398 — **(you already studied this one)**
- **#399** (open) — "Order sync to NetSuite through the REST batch API: the rules, customers, sales orders, customer deposits and refunds" — https://github.com/hotwax/mantle-netsuite-connector/pull/399 — **big one: this single PR is actually the shared engine behind Segment 1 AND Segment 3 below**
- `gorjana-maarg` **#333** (open) — "sales order sync through the NetSuite REST record API" (gorjana's own rules/data for choosing which orders go first) — https://github.com/hotwax/gorjana-maarg/pull/333

**Reading material (Anil said read 3 times):**
- https://www.thewayhow.com/learn-something/netsuite-rest-web-services-record-update-performance

---

## Segment 2 — Returns (Return to Cash)

**Major PRs (Anil named these explicitly):**
- `gorjana-maarg` **#305** (open) — "The NetSuite return lifecycle: services, views, and the orchestrator" — https://github.com/hotwax/gorjana-maarg/pull/305
- `mantle-netsuite-connector` **#389** (open) — "RMA creation services: from a sales order, and standalone" — https://github.com/hotwax/mantle-netsuite-connector/pull/389 — **(you already studied this one)**

**Smaller supporting PRs, same area (stacked around #389):**
- **#379** (open) — "Add the leading slash to the transform endpoints" — https://github.com/hotwax/mantle-netsuite-connector/pull/379
- **#380** (open) — "Keep NetSuite's error message when a transform is rejected" — https://github.com/hotwax/mantle-netsuite-connector/pull/380
- **#388** (open) — "Index NetSuiteReturnItemHistory by return line" — https://github.com/hotwax/mantle-netsuite-connector/pull/388
- **#384** (closed, NOT merged) — early draft of RMA-from-sales-order, superseded by #389 — skip, historical only

**NetSuite-side restlets for credit memo / refund (returns money to customer):**
- `hotwax-gorjana-netsuite` **#128** (merged) — "Add HC_RL_CreateCreditMemoFromRma restlet" — https://github.com/hotwax/hotwax-gorjana-netsuite/pull/128
- `hotwax-gorjana-netsuite` **#129** (merged) — "Restlet: credit memo to customer refund by one method for one amount" — https://github.com/hotwax/hotwax-gorjana-netsuite/pull/129

**Reference / business context (not code, read for understanding):**
- `gorjana-maarg` **#309** (open issue) — "POC: does an unnamed line get received when we transform an RMA to an item receipt?" — https://github.com/hotwax/gorjana-maarg/issues/309
- Concept: **What is a Credit Memo** — decreases what a customer owes you; reverses billed revenue; debits Sales Income, credits Accounts Receivable; no cash moves.
- Concept: **which returns get pushed to NetSuite, and when** — a return only pushes at `RETURN_REQUESTED`/`RETURN_ACCEPTED` if its destination facility is (or is under) a `DISTRIBUTION_CENTER`; everything else waits until `RETURN_RECEIVED`. (Chinmay's answer: warehouse-bound returns push earlier; AfterShip-style returns push only once physically received.)
- Claude artifact — store credit / credit memo workflow proposal: https://claude.ai/code/artifact/d89f9828-362a-48fe-b6b6-e297593c6424
- Claude artifact (context not yet confirmed, check when you get here): https://claude.ai/code/artifact/c46af29b-e475-4ad6-8ce7-48b90d314503
- Open question Anil asked Chinmay, worth understanding: *"When creating RMA from a NetSuite Sales Order, we pass the items list ourselves instead of letting NetSuite auto-create lines by transforming the order — why?"*

---

## Segment 3 — Deposits & Refunds — builds on Segments 1 + 2, STUDY LAST

**Merged already (foundation, quick read):**
- **#401** (merged) — "NetSuiteCustomField: the account's custom field definitions... and the sync" — https://github.com/hotwax/mantle-netsuite-connector/pull/401
- **#402** (merged) — "get#NetSuiteCustomFieldsForRecordType" — https://github.com/hotwax/mantle-netsuite-connector/pull/402

**Open, currently active:**
- `oms` **#1039** (open) — "OrderPaymentPreference.sourceOrderPaymentPreferenceId: the payment an exchange payment moves" — https://github.com/hotwax/oms/pull/1039
- `gorjana-maarg` **#341** (open) — "Exchange linking names the real payment in sourceOrderPaymentPreferenceId, and a migration to fill it" — https://github.com/hotwax/gorjana-maarg/pull/341
- `gorjana-maarg` **#340** (open) — "NetSuiteCustomField data: the customer deposit carries the Shopify order id" — https://github.com/hotwax/gorjana-maarg/pull/340
- `hotwax-gorjana-netsuite` **#131** (open) — "set the payment method on REST-made customer deposits" (already deployed to sandbox) — https://github.com/hotwax/hotwax-gorjana-netsuite/pull/131
- `hotwax-gorjana-netsuite` **#132** (open) — "Batch runs from a File Cabinet file: a runner that calls a handler RESTlet per line, the HC Batch Run record, the refund-from-deposit RESTlet" — https://github.com/hotwax/hotwax-gorjana-netsuite/pull/132

**Closed, not merged (checked — likely superseded, not wasted time to skip):**
- **#403** (closed, NOT merged) — "Customer deposits to NetSuite in batches: the exchange difference" — its scope looks folded into #399 / `gorjana-maarg`#342 below

**⚠️ Needs a 1-line confirm from Anil before you rely on it:**
- `gorjana-maarg` **#342** (open) — "Customers, sales orders and customer deposits to NetSuite in batches: the due views, the prepare, the rule groups, the jobs" — https://github.com/hotwax/gorjana-maarg/pull/342 — this looks like a **newer, consolidated** gorjana-side PR that may replace/extend #333, #340 and #341 into one. I haven't read its body in detail yet — ask Anil whether this is now the one to follow instead of the three separate ones above.

---

## Not part of this flow — set aside, different topic

- `mantle-shopify-connector` #736, #737, #738 — Shopify return-reason work, unrelated to NetSuite; you already noted Anil said this is superseded/not relevant now.

---

## Anil's mindset — keep this in mind throughout, not just once

- "I don't know what I don't know. I am here to learn. I am here to learn and do whatever it takes." — the attitude he expects from anyone he works with.
- He wants understanding of the **business process** (supply chain, accounting), not just working code.
- He is rewriting the whole NetSuite integration over time — expect the workflow you're testing today to keep changing; be ready to retest.
- Code currently specific to Gorjana is gradually moving into the shared `mantle-netsuite-connector` — so what you learn in gorjana-maarg today will show up as shared/reusable code later.
