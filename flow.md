# NetSuite Business Model — Complete Flow Diagrams

*This file has two master diagrams: Flow 1 (Sales Order — forward, Order-to-Cash) and Flow 2 (Return & Exchange — reverse, Return-to-Cash). Every term from your notes is placed exactly where it happens in the business process, with the accounting nature of each step labeled (posting vs non-posting, GL impact, revenue/AR/inventory effect).*

---

## Legend — Accounting Nature of Each Box

| Color / Style | Meaning |
|---|---|
| 🟦 Blue box | Standard processing step |
| 🟥 Red diamond | Decision point |
| 🟩 Green box | Financial transaction that **posts to the General Ledger** (has real accounting/GL impact) |
| 🟨 Yellow box | Internal / non-customer-facing inventory step |
| ⬜ Grey box | Non-posting transaction (record only, no GL impact yet) |

---

## Flow 1: Sales Order — Complete (Order-to-Cash / O2C)

```mermaid
flowchart TD

    A["Customer places order<br/>via OMS<br/>(OMS = source of truth for the order)"]:::grey
    B["Webhook fires: 'Created' event<br/>Real-time ping from OMS to NetSuite"]:::grey
    C["Sales Order (SO) created in NetSuite<br/>NON-POSTING transaction<br/>No GL / revenue impact yet"]:::grey
    D["Avalara calculates Sales Tax<br/>Dynamic, geography-based tax engine<br/>(tax value attached to SO)"]:::grey
    E{"Approval Event<br/>Does the order pass approval<br/>workflow / manager check?"}:::decision
    E1["Order held<br/>Waiting for approval logic to clear"]:::grey
    F["Item Fulfillment (IF) created<br/>Warehouse picks, packs, ships<br/>Inventory reduced (Asset -> COGS begins)"]:::posting
    G["Invoice (INV) generated<br/>POSTING transaction<br/>Triggers Revenue Recognition<br/>Debits Accounts Receivable, Credits Revenue"]:::posting
    H{"Reconciliation Check:<br/>Does OMS total match<br/>NetSuite Invoice total?"}:::decision
    H1["Flag for reconciliation<br/>Investigate mismatch before closing"]:::grey
    I["Order-to-Cash Complete<br/>Revenue booked, AR balance created,<br/>awaiting customer payment"]:::posting

    A --> B --> C --> D --> E
    E -- "No / On Hold" --> E1 --> E
    E -- "Yes, Approved" --> F --> G --> H
    H -- "No, mismatch" --> H1
    H -- "Yes, matches" --> I

    classDef grey fill:#f5f5f5,stroke:#999,color:#333;
    classDef posting fill:#d5e8d4,stroke:#82b366,color:#1b3a1b;
    classDef decision fill:#f8cecc,stroke:#b85450,color:#5c1a1a;
```

**Reading this flow (in accounting words):**
- **Sales Order** = a commitment record only — nothing hits your books yet.
- **Item Fulfillment** = physical inventory starts moving; this is where the seed of Cost of Goods Sold (COGS) begins, though it isn't formally expensed until invoicing in most NetSuite setups.
- **Invoice** = the real accounting moment — this is what actually creates **Revenue** and **Accounts Receivable (AR)**. Before this, nothing has "happened" financially.
- **OMS-vs-NetSuite total match** is a control check — it exists because the OMS is the system of record for what the customer actually agreed to pay, and NetSuite must reflect exactly that, not a different number from RMS.

---

## Flow 2: Return & Exchange — Complete (Return-to-Cash / R2C)

This is the big one. It has **four branches** hanging off one entry gate:
- **Branch A** — Standard physical return (RMA path)
- **Branch B** — Reason Lost / Reason Warranty (no physical item, no RMA)
- **Branch C** — Exchange Order (its own financial hold logic)
- **Branch D** — Floor Damaged / Internal Inventory (not customer-facing at all — shown as its own zone, but connected wherever a real return can feed into it)

```mermaid
flowchart TD

    START(["Customer initiates a return<br/>or internal damage is discovered"]):::grey
    ELIG{"Eligible Return?<br/>(within policy days,<br/>condition acceptable)"}:::decision
    REJECT["Return Rejected<br/>No RMA, No Credit Memo"]:::grey

    START --> ELIG
    ELIG -- "No" --> REJECT
    ELIG -- "Yes" --> REASON{"What triggered this credit?"}:::decision

    %% ================= BRANCH A: STANDARD RETURN =================
    subgraph BranchA["BRANCH A — Standard Physical Return"]
        direction TB
        A1["RMA (Return Merchandise Authorization)<br/>created in NetSuite<br/>Tracking document for the return lifecycle"]:::grey
        A2["Customer ships item back"]:::grey
        A3["Item Receipt (IR) created<br/>Warehouse logs physical inventory<br/>back into NetSuite stock<br/>(customer-facing receiving)"]:::posting
        A4{"Item condition on receiving?"}:::decision
        A5["Item is sellable<br/>Returns to normal inventory location"]:::posting
        A6["Item is damaged<br/>--> routes to BRANCH D<br/>(Floor Damaged / Repair flow)"]:::yellow
        A7["Rule Check:<br/>Credit amount must match<br/>OMS total — NOT the RMS total"]:::decision
        A8["Credit Memo (CM) generated<br/>POSTING transaction<br/>Debits Revenue/Returns, Credits AR<br/>(clears the customer balance)"]:::posting

        A1 --> A2 --> A3 --> A4
        A4 -- "Sellable" --> A5 --> A7
        A4 -- "Damaged" --> A6 --> A7
        A7 --> A8
    end

    %% ================= BRANCH B: LOST / WARRANTY =================
    subgraph BranchB["BRANCH B — Reason Lost / Reason Warranty (No RMA)"]
        direction TB
        B1["Reason Code = Lost<br/>OR Reason Code = Warranty"]:::grey
        B2["RMA and Item Receipt are SKIPPED entirely<br/>No physical inventory expected back"]:::yellow
        B3["Credit Memo (CM) issued directly<br/>POSTING transaction<br/>Same GL impact as Branch A's CM,<br/>but with NO linked Item Receipt"]:::posting

        B1 --> B2 --> B3
    end

    %% ================= REFUND SUB-PROCESS (shared by A & B) =================
    subgraph REFUND["Refund Decision (shared logic for Branch A and Branch B)"]
        direction TB
        R1{"Refund Method Chosen?"}:::decision
        R2["Cash-to-Cash / Original Payment<br/>Refund MUST return to original<br/>payment method (Visa -> Visa)<br/>Fraud-prevention rule"]:::grey
        R3["Customer Refund transaction<br/>POSTING transaction<br/>Debits Credit Memo balance,<br/>Credits Cash/Bank"]:::posting
        R4["Store Credit chosen instead of cash"]:::grey
        R5["Special Invoice created<br/>ONLY to represent the store credit balance<br/>POSTING transaction (non-cash)"]:::posting
        R6["That Invoice is applied back<br/>against the Credit Memo<br/>Net effect: balance held as usable credit"]:::posting

        R1 -- "Cash Back" --> R2 --> R3
        R1 -- "Store Credit" --> R4 --> R5 --> R6
    end

    %% ================= BRANCH C: EXCHANGE =================
    subgraph BranchC["BRANCH C — Exchange Order"]
        direction TB
        C1["Exchange requested<br/>Customer wants a different item,<br/>not a refund"]:::grey
        C2["Credit Memo generated FIRST<br/>for the balance amount of returned item<br/>POSTING transaction"]:::posting
        C3{"Has Credit Memo cleared<br/>the balance?"}:::decision
        C4["Exchange Order HELD<br/>NOT synced to NetSuite yet<br/>(custom middleware hold — not native NetSuite)"]:::yellow
        C5["Exchange Order releases,<br/>syncs to NetSuite as a<br/>new Sales Order"]:::grey
        C6{"New item price vs<br/>Credit Memo balance?"}:::decision
        C7["Equal:<br/>No extra payment needed<br/>New Sales Order proceeds<br/>into Flow 1 (fulfillment + invoice)"]:::posting
        C8["New item costs MORE:<br/>Customer Payment created<br/>against new Invoice for the delta<br/>POSTING transaction"]:::posting
        C9["New item costs LESS:<br/>Residual Customer Refund<br/>OR residual Store Credit issued<br/>(re-enters Refund sub-process)"]:::posting

        C1 --> C2 --> C3
        C3 -- "No, not yet cleared" --> C4 --> C3
        C3 -- "Yes, cleared" --> C5 --> C6
        C6 -- "Equal" --> C7
        C6 -- "New > Old" --> C8
        C6 -- "New < Old" --> C9
    end

    %% ================= BRANCH D: FLOOR DAMAGE / INTERNAL INVENTORY =================
    subgraph BranchD["BRANCH D — Floor Damaged / Internal Inventory (Non-Customer-Facing)"]
        direction TB
        D0["Entry Point 1:<br/>Internal discovery<br/>(warehouse/floor staff finds damage)<br/>NO customer involved"]:::yellow
        D0b["Entry Point 2:<br/>Fed from Branch A<br/>(returned item found damaged<br/>on Item Receipt)"]:::yellow
        D1["Floor Damaged identified<br/>Internal loss — no RMA,<br/>no customer transaction of any kind"]:::yellow
        D2["Transfer Order (TO) created<br/>Moves inventory internally:<br/>Sellable Location --> virtual 'Repair' Bucket"]:::yellow
        D3["Shipment Receipt logged<br/>Receiving INTO the Repair bucket via the TO<br/>(distinct from Item Receipt —<br/>this is internal/vendor-style receiving,<br/>not customer-facing)"]:::yellow
        D4{"Is item repairable?"}:::decision
        D5["Item repaired<br/>Transfer Order moves it BACK<br/>to sellable location"]:::yellow
        D6["Not repairable:<br/>Inventory Write-off<br/>POSTING transaction<br/>Debits Loss/Write-off Expense,<br/>Credits Inventory Asset"]:::posting
        D7["HARD RULE:<br/>Floor Damaged items must NEVER<br/>be routed through Hotwax<br/>(Hotwax = external warranty system only)"]:::decision

        D0 --> D1
        D0b --> D1
        D1 --> D2 --> D3 --> D4
        D4 -- "Yes" --> D5
        D4 -- "No" --> D6
        D1 -.-> D7
    end

    %% ================= MAIN ROUTING =================
    REASON -- "Standard physical return" --> A1
    REASON -- "Reason: Lost" --> B1
    REASON -- "Reason: Warranty" --> B1
    REASON -- "Exchange requested" --> C1

    A8 --> R1
    B3 --> R1

    A6 -.-> D0b

    R3 --> DONE1["Return-to-Cash Complete"]:::posting
    R6 --> DONE1
    C7 --> DONE2["Exchange Complete<br/>(feeds into Flow 1 for new item)"]:::posting
    C8 --> DONE2
    C9 --> DONE2

    classDef grey fill:#f5f5f5,stroke:#999,color:#333;
    classDef posting fill:#d5e8d4,stroke:#82b366,color:#1b3a1b;
    classDef decision fill:#f8cecc,stroke:#b85450,color:#5c1a1a;
    classDef yellow fill:#fff2cc,stroke:#d6b656,color:#5c4a00;
```

---

## Reading Branch D "Beautifully" — Why It's Drawn This Way

Branch D is intentionally isolated from the customer-facing branches because, in accounting terms, **it is a completely different kind of event**:

| | Branch A / B / C (Customer Returns) | Branch D (Floor Damaged) |
|---|---|---|
| Who is involved | Customer + Company | Company only (internal) |
| Triggering document | RMA or direct Credit Memo | Transfer Order (TO) |
| Customer balance affected? | Yes — AR/Credit Memo | No — no customer record touched at all |
| GL impact | Revenue reversal, AR reduction | Inventory write-off or repair-in-progress (no revenue impact) |
| Receiving document used | **Item Receipt** (customer return) | **Shipment Receipt** (internal/TO receiving) |

That's why the diagram gives Branch D **two separate entry points**:
1. A standalone trigger (staff finds damage on the floor — nothing to do with any customer).
2. A dotted-line feed from Branch A (a customer's returned item turns out to be damaged when received) — this is the only bridge between the customer world and the internal world, and it's drawn as a **dotted line** on purpose, to show it's a conditional handoff, not a normal step in sequence.

The **hard rule** — Floor Damaged must never touch Hotwax — is drawn as a decision-style box directly under the trigger, because it's a boundary check your team explicitly called out, not just a note.

---

## Every Term — Where It Lives (Quick Index)

| Term | Flow | Branch |
|---|---|---|
| OMS | Flow 1 | Entry point |
| NetSuite | Both | Core system throughout |
| RMS | — | Explicitly excluded from totals-matching check |
| Hotwax | Flow 2 | Branch D (explicitly excluded) |
| Avalara | Flow 1 | Tax step |
| Webhook | Flow 1 | Order creation trigger |
| Sales Order (SO) | Flow 1 | Core step |
| Item Fulfillment (IF) | Flow 1 | Core step |
| Invoice (INV) | Flow 1 / Branch C | Core step / exchange pricing |
| RMA | Flow 2 | Branch A only |
| Item Receipt (IR) | Flow 2 | Branch A only |
| Credit Memo (CM) | Flow 2 | Branch A, B, C |
| Customer Refund | Flow 2 | Refund sub-process |
| Transfer Order (TO) | Flow 2 | Branch D |
| Eligible Return | Flow 2 | Entry gate |
| Cash-to-Cash / Original Payment | Flow 2 | Refund sub-process |
| Store Credit Refund | Flow 2 | Refund sub-process |
| Exchange Order | Flow 2 | Branch C |
| Floor Damaged | Flow 2 | Branch D |
| Reason Lost / Reason Warranty | Flow 2 | Branch B |
| Approval Event | Flow 1 | Gate before fulfillment |
| Shipment Receipt | Flow 2 | Branch D only |

Every term from your three source documents is placed above — nothing left out.