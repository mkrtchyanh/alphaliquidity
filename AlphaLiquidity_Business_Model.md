# AlphaLiquidity — Business Model

**Document type:** Business model and core trading-flow specification  
**Status:** Draft v2 — updated with the decisions taken in the design review

## 1. Overview

AlphaLiquidity is a currency-asset exchange platform where funds publish liquidity and traders create and purchase individual sell offers. Assets may include fiat currencies and blockchain-based assets, such as AMD, USD, EUR, and BTC.

The platform holds assets in its own bank accounts and blockchain wallets on behalf of the exchange flow. Funds deposit assets into the platform and can later withdraw them. A completed exchange transfers the exchanged assets between fund balances, and the platform earns a fee on each exchange (section 4).

## 2. Participants and responsibilities

| Participant | Responsibilities |
|---|---|
| **Platform** | Operates trading and balance services; maintains platform bank accounts and blockchain wallets; coordinates deposits, withdrawals, exchanges, and fee accounting. |
| **Platform admin** | Defines the platform fee: global and per-client (section 4). |
| **Fund A — seller** | A fund whose trader publishes an offer to sell one currency asset for another. |
| **Fund B — buyer** | A fund whose trader selects and purchases an available sell offer. |
| **Fund admin** | Configures the fund's liquidity: Bids and Asks with an expiration (each can be disabled), blocks, price ranges, the quote convention of its pairs, trading limits and compliance checks. Initiates deposits and withdrawals. |
| **Fund trader** | Creates sell offers from the configured blocks, with a selling price inside the configured ranges, and initiates purchases on behalf of a fund. |
| **External bank / blockchain** | Transfers assets to or from the platform's bank accounts or blockchain wallets. |

A fund has a balance consisting of currency-specific accounts, for example `10 BTC`, `100,000 USD`, and `50,000 EUR`. The account currency comes from the list of currencies the platform provides.

## 3. Exchangeable assets and markets

- An **asset** is a currency block, such as AMD, USD, EUR, or BTC.
- A market is a currency pair, for example `AMD-USD`, `BTC-USD`, or `USD-EUR`.
- **Liquidity configuration:** the fund admin defines Bids and Asks for a pair, each with an expiration, and can disable them. The admin also defines the blocks (amount) and the price ranges.
- **Individual sell offers:** the fund trader creates an offer from a block with a selling price inside the configured range. A buyer purchases a block that matches the ranges. A block is indivisible: an offer is bought as a whole.
- **Offer expiration:** every offer has an expiration. When it passes, the block returns to the pool and can be published again as a new offer.
- **Quote convention** (base and quote asset, price and amount precision) is configurable by the fund admin.

## 4. Fee model

- The platform earns a **margin added to the price**, based on block size.
- The platform admin defines the margin globally and per client: a percentage with a minimum amount, a fixed amount, or similar rules.
- The fund admin and the trader see the **final price with the fee included**.
- The **buyer pays the fee on top**. One purchase creates three balance records together:
  - a debit from the buyer: price + margin;
  - a credit to the seller: the supplier price;
  - a credit to the platform fee account: the margin.
- The three records are finalized together: all `EXECUTED` or all `REJECTED`. Fees are settled **only if the exchange succeeds**.

**Policy still to define:** the fee currency, rounding and precision rules, and the fee account per currency.

## 5. Fund asset lifecycle

### 5.1 Depositing assets

1. A fund admin initiates a deposit through a supported third-party bank or blockchain transfer into a platform-controlled account or wallet.
2. The platform verifies, through the bank or blockchain integration, that the transfer has succeeded.
3. Only after confirmation does the platform credit the fund's account, through a balance record whose source is the integration payment (source type bank or crypto).

A requested or pending transfer is not a completed deposit.

### 5.2 Withdrawing assets

1. A fund admin requests a withdrawal to a fund-controlled bank account or wallet, within the balance rule of section 6.
2. The platform creates a `REQUESTED` debit record, starts the outgoing transfer from its account or wallet, and tracks the result.
3. A confirmed transfer finalizes the record as `EXECUTED`. A failed transfer finalizes it as `REJECTED`.

The external-transfer reconciliation rules are specified with the integration contract.

### 5.3 Using assets for trading

A purchase moves assets between fund accounts through balance records with the offer as their source (source type offer). The `REQUESTED` debit reserves the buyer's funds, so the same assets cannot be committed to conflicting exchanges.

## 6. Balance records and balance views

Every movement is a **debit** or **credit record** in the Balance service database.

| Field | Meaning |
|---|---|
| Status | `REQUESTED`, `EXECUTED` or `REJECTED` |
| Source id and source type | For an internal transfer the offer id with type offer. For a deposit or withdrawal the identity of the bank or crypto payment with its type. |
| Expiration | `REQUESTED` records expire on their own after a configured time-to-live. |

**Allowed status changes:** `REQUESTED → EXECUTED` and `REQUESTED → REJECTED` only. `EXECUTED` is final and never becomes `REJECTED`. A correction after execution is a new compensating record. An expired record cannot be finalized as `EXECUTED`.

| Balance view | Included records | Purpose |
|---|---|---|
| **Purchase- and withdrawal-eligible balance** | `EXECUTED` credits, plus `EXECUTED` and `REQUESTED` debits | Decides whether a fund can pay for a purchase or withdraw to a bank or wallet. Pending credits never count. |
| **Actual balance** | `EXECUTED` debits and credits only | Finalized movements. |
| **Estimated balance** | `EXECUTED` and `REQUESTED` debits and credits | Shows in-flight movements. |

Checking the available funds and creating the `REQUESTED` records for the same account are one atomic step, so concurrent purchases cannot both pass on the same funds. The debit and credit sign convention is fixed once for the platform.

## 7. Offer states and locking

### 7.1 Offer states

| State | Meaning |
|---|---|
| **Available_For_Buying** | Visible to buyers. No active lock and not expired. |
| **Available_For_Paying** | Reached only through a successful lock. Hidden from buyers while the purchase is paid. |
| **Expired** | The offer expiration passed. The block returns to the pool. |
| **Executed** | The purchase completed. Terminal. |
| **Withdrawn** | The offer was withdrawn. Terminal. |

The state is derived from the offer row and the database clock, in this order: a terminal outcome (Executed, Withdrawn); then now before `paying_until` means Available_For_Paying; then now at or after `expires_on` means Expired; otherwise Available_For_Buying. An offer that expires while being paid is not cut off: the purchase resolves first. When the paying interval ends without a successful exchange, the offer is Available_For_Buying again (or Expired, if `expires_on` has passed).

### 7.2 Lock record

A lock is a database record: `offer_id`, `version`, `holder_ref` (the purchase request id), `created_on`, `expiration`, `released_on`. The pair `(offer_id, version)` is unique. The version is the fencing token. The lock has one fixed paying interval, set by configuration. The database clock is the only time source.

### 7.3 Acquiring a lock

Every attempt, including each retry, runs in its own database transaction. In the winning attempt the lock record, the offer row update (`paying_until`, version) and the outbox row are committed together.

1. **A payment transaction already exists for the purchase:** exit with the existing result. If a payment exists, the offer is no longer available for another attempt. A repeat at that point signals an unhealthy system.
2. **An active lock belongs to another requester:** reject with `OFFER_LOCKED` and a retry time equal to the lock expiration (the end of the paying interval).
3. **An active lock belongs to the current requester:** the lock is exchanged: it is released and the next version is created with a new expiration.
4. **No lock, or the lock is expired or released:** create the next version, but only if the offer row allows it (no terminal outcome, not expired, the paying interval ended or the lock is the caller's).
5. **Concurrent inserts:** exactly one wins on the unique key. The losers retry from reading the latest record and leave with `OFFER_LOCKED`.

## 8. Purchase flow

1. A trader browses sell offers and submits a purchase request (`requestId`).
2. The offer is locked (section 7.3). The offer becomes Available_For_Paying and the freeze is broadcast to all screens.
3. **Transaction requested:** a synchronous REST call. The Balance service checks there is enough money, then creates `REQUESTED` records (section 4): buyer debit, seller credit, platform fee credit.
4. The buyer is validated against compliance rules and the fund's configured limits. On failure the records are rejected and the lock is released.
5. **Payment:** a payment transaction is inserted in a new database transaction with a timeout shorter than the record expiration. The insert is carried by CDC from the write-ahead log through Kafka.
6. **Completion:** in ONE database transaction in the Balance database the records go `REQUESTED → EXECUTED` (only if not expired) and the exchange record is inserted. If this transaction fails it rolls back as a whole.
7. **Outcome by CDC:** the sell-offer service consumes the exchange event, sets the offer Executed and releases the lock. On a rejection event it releases the lock and the offer is Available_For_Buying again.
8. If no outcome arrives, the `REQUESTED` records expire by themselves, the paying interval ends, and the offer is Available_For_Buying again by the clock.

### 8.1 Completion condition

A purchase is **not** complete because a request was accepted, a lock acquired, records created, or a payment transaction inserted. It is complete only when the records are `EXECUTED` and the exchange record exists. These two are written together. Only then are the fees settled.

### 8.2 Timeout and expiration

The lock expiration must not end before the earliest expiry of the purchase's `REQUESTED` records. Otherwise the offer could become buyable while the first payment could still be finalized.

## 9. Services, events and recovery

- **Sell offer service:** offer management (liquidity configuration, blocks, offers, fee calculation) and trading (locking, purchase flow) and the purchase itself (the Transaction Request and the payment transaction) are one service with one database.
- **Balance service:** credit and debit records, balances and exchange records, with its own database. It consumes the payment transaction sent by the Sell Offer service.
- **Stack:** MS SQL Server, WAL-based change data capture into Kafka, .NET services. State changes are published with the outbox table pattern: the outbox row is written in the same transaction as the change and CDC reads it from the log.
- **Consumers are idempotent:** CDC and Kafka can deliver an event more than once. Events carry stable identifiers (request id, offer id, exchange id, event version).
- **Recovery** replays durable changes. A failed completion leaves the records `REQUESTED`, so it is retried safely or the records expire. Changes made only by the clock (a paying interval that ended, an offer that expired) are made visible by a sweeper that emits the event.

## 10. Core business invariants

1. **Asset custody:** a deposit is credited only after the external transfer is confirmed.
2. **No overspending:** a fund cannot commit more of an asset than its eligible balance permits, including requested debits.
3. **One successful purchase per offer:** concurrent buyers cannot both complete the same offer.
4. **Reservation validity:** an expired record cannot be finalized as `EXECUTED`; an `EXECUTED` record never changes.
5. **Atomic completion:** the records become `EXECUTED` and the exchange record is inserted in one transaction.
6. **Fee only on success:** the three records of a purchase finish together; fee is settled only on success.
7. **Idempotent processing:** duplicate payment transactions, CDC events or Kafka deliveries cannot duplicate movements or exchanges.
8. **Auditable movements:** every movement and state change is traceable through durable identifiers.
9. **Concurrency-safe locks:** only the latest valid lock version authorizes a purchase, and a late message with an old version changes nothing.
10. **External reconciliation:** internal balances are reconciled with bank and blockchain outcomes.

## 11. Statuses and terminology

| Entity | Statuses / concepts | Notes |
|---|---|---|
| Balance record | `REQUESTED`, `EXECUTED`, `REJECTED`, expiration | Only `REQUESTED` can change. Only non-expired records can be finalized. |
| Offer | Available_For_Buying, Available_For_Paying, Expired, Executed, Withdrawn | Derived from the offer row and the clock. |
| Payment transaction | Inserted; success or failure outcome | Complete status list to be defined. |
| Exchange | Record inserted together with `EXECUTED` records | Business evidence of a completed exchange, propagated by CDC. |
| Offer lock | Active until expiration or release; versioned | `(offer_id, version)` is unique. |
| Deposit / withdrawal | Integration-driven; record `REQUESTED → EXECUTED` or `REJECTED` | Reconciliation rules with the integration contract. |



*This document captures the supplied business model and the decisions taken so far. It is not a substitute for API contracts, database schemas, security and compliance requirements, or runbooks.*