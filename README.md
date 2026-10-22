# AlphaLiquidity: Architecture and Requirements

## What this is

AlphaLiquidity is an institutional digital-asset trading platform. Hedge funds (tenants) publish **All-or-None block sell offers**. Other funds buy a whole block or nothing. The platform earns a margin on block size, paid by the buyer on top of the supplier price.

The system is built for these needs:

- **Real-time:** when one buyer clicks Purchase, the offer freezes on every screen of the currency pair within 150 ms (SignalR with a Redis backplane). It unfreezes if the purchase fails.
- **Money correctness:** an offer is sold once, funds are never double-spent, and every balance change is a record that can be traced.
- **Multi-tenant, regulated:** tenants are isolated, and compliance checks (an external Compliance System, replicated locally) run on every trade.

This is a design package, not code. It holds the requirements, the system design (subsystems and flows) and the software design (components, protocols).

**Stack:** .NET, MS SQL Server, Kafka, Redis, Kubernetes. **Single Region** - one datacenter.

## Scope of the design

The design covers only the few critical flows, where the money and the consistency risks are. Everything else (reports, user administration, onboarding, notifications and similar) is outside this package.

| Flow | What it covers | Diagrams |
|---|---|---|
| Offer publishing | Authorization, validation, asset reservation in Balance, offer creation, confirmation, offer becomes Available_For_Buying | `System/Subsystem/SellOffer/Management/seq_sell_offer_publishing`, `System/Subsystem/Balance/seq_balance_asset_reservation` |
| Offer releasing | Withdrawal and expiry, block back to the pool, assets released in Balance | `System/Subsystem/SellOffer/Management/seq_expired_offer`, `System/Subsystem/SellOffer/Trading/seq_trading_expire_sell` |
| Trading | Purchase click, lock, Transaction Request, balance reservation, compliance, payment, completion or failure, and recovery | `System/Subsystem/SellOffer/Trading/seq_trading_basic`, `seq_trading_recovery_*`, `System/Subsystem/Balance/seq_balance_offer_payment`, `Software/Components/SellOffer/Trading/Locking/` |


## Subsystems

| Subsystem | Responsibility |
|---|---|
| Sell Offer | One service: liquidity configuration (Bids, Asks, blocks, price ranges), offers, fee margin, locking, Transaction Request, payment transaction, liquidity depth |
| Balance | Credit and debit records, derived balances, exchange records, funding |
| Real-time Sync | SignalR hub for clients: freeze, unfreeze, removal, snapshots |
| Compliance Gateway | Local replica of party status and limits from the external Compliance System |
| Tenant Management | Tenants, users, roles, credentials, notifications |
| Audit | Append-only history of every state change, off the trade path |

## How consistency works

The platform is **eventually consistent between services and strongly consistent inside each one**. It does not use distributed transactions. Correctness comes from five mechanisms working together.

**1. One owner per piece of state, one database transaction per change.**
Sell Offer owns offers, locks and requests. Balance owns the credit and debit records and the exchange records. Each service changes its own state in a single local transaction, and writes the event for the others into an **outbox table** in the same commit.

**2. Locks as immutable records, with database constraints.**
A lock is a row `(offer id, version)`, unique on that pair. Concurrent buyers insert the next version, and the unique constraint picks exactly one winner. The version is a **fencing token**, so a late or stale outcome carrying an old token is ignored. Each attempt and each retry runs in its own transaction. The offer row is updated in the same transaction (a version that grows by one on every change).

**3. State derived from time, with expirations.**
The offer state is computed from the offer row and the database clock (`expires_on`, `paying_until`, terminal outcome). A balance record in REQUESTED carries its own expiration. A lock, a paying interval or a record that runs out does not need a job to clean it up: after the expiry it is simply ignored. A crash cannot strand an offer or funds. Sweepers exist only to emit events for changes that the clock made.

**4. A big business transaction split into a sync step and async steps.**
A purchase is not one transaction. It is a chain, each step a small local transaction:

**5. Idempotency everywhere.**
Every command carries a request id, and every event consumer is idempotent by event id. CDC and Kafka may deliver twice, so replays never create a second effect.

**Balance rules that protect against double spending**
- Eligible balance = EXECUTED credits + EXECUTED and REQUESTED debits, so a pending purchase already reduces what can be spent.
- The check and the creation of the records are atomic per account.

**Client path and recovery**
- SignalR with Redis serves clients only. Kafka carries CDC sync and the audit log, and the freeze path does not depend on it.
- Clients detect a missed update by the offer version and resync from a snapshot.
- Failures resolve by themselves: records expire, the offer becomes buyable by the clock, and a sweeper marks the request Failed and raises an alert.

## Where to find things

```
AlphaLiquidity_Business_Model.md      business model: participants, fees, balance records, offer states, flows, invariants
network_topology.mermaid              network segments and allowed flows
System/                               system design (what and why)
  c4_1_system_overview.mermaid
  Requirements/                       functional.md (FR), non_functional.md (NFR)
  Subsystem/
    c4_2_subsystems.mermaid
    Audit/ Balance/ Compliance/Gateway/ RealTimeSync/ TenantManagement/    c4_1_*_overview diagrams
    Balance/                          seq_balance_asset_reservation, seq_balance_offer_payment
    SellOffer/
      c4_1_sell_offer_overview.mermaid, state_sell_offer.mermaid
      Management/                     seq_sell_offer_publishing,
      Trading/                        seq_trading_basic, seq_trading_expire_sell,
                                      seq_trading_recovery_client_resync, seq_trading_recovery_unknown_outcome
Software/                             software design (how)
  c4_2_overview.mermaid               containers
  Requirements/sr_sell_offers.md      SW-SO-01..31
  Components/
    Balance/ Compliance/ RealtimeHub/ TenantManagement/    c4_3_* component diagrams
    Balance/                          copies of the two Balance sequences
    SellOffer/
      Trading/c4_3_trading_service.mermaid, Locking/ ( seq_locking, state_lock)
```

### Helpful links
ChatGpt
https://chatgpt.com/share/6accd71b-1a5c-83ed-b958-9227b76c0fdc