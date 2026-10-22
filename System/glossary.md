#### Liquidity depth
Shows how much of an asset is available to trade at each price level.

#### Block
the indivisible quantity of a base asset (for example 20 BTC) contained in a single [Sell offer](#Sell-Offer).<br/>
It can be bought only in full or not at all (All-or-None) and cannot be split, merged or partially filled.

#### Atomic lock
The operation in the Sell Offer subsystem that moves an offer from Available to Locked_For_Pre_Processing and records the holder, in one indivisible step.<br/> Of any number of concurrent attempts, exactly one wins, and the others fail with a lock failure code.

#### Atomic Release
The atomic operation that returns a locked offer to Available, ends the holder's claim and publishes the change. It is used for every outcome that is not a successful purchase.

#### Sell Offer
A seller's All-or-None listing to sell one block of a base asset in a currency pair at a stated price.<br/>
It is the **tradable entity** on the platform and the object that carries the state, the Purchase button and the [Atomic lock](#atomic-lock).

#### Party
A legal or organizational participant with accounts in the ledger (a fund, the platform itself, a liquidity provider)

#### Ledger Account
The record that holds the balance of one currency or asset for one owner (a Party) inside the platform's books.<br/>
Ledger entries debit or credit it, and its balance is the sum of those entries.<br/>
It is the unit that "accounts per currency, including BTC" refers to.

#### Fund
An institutional customer (for example a [hedge fund](https://en.wikipedia.org/wiki/Hedge_fund)) 
that has an approved account on the platform and trades through its [Trader](#Trader)

#### Trader
A person who acts on behalf of a [Party](#Party) and holds the permissions.

#### Fund Admin
A user of a tenant (hedge fund) who holds the administrative role for that tenant.<br/>
The Fund Admin acts on behalf of the Fund (the Party) to run its account on the platform:
* users
* credentials
* funding and block offers.<br/>
It is a role inside one tenant, not a platform-level role.

#### Platform
The operator of **AlphaLiquidity**, acting as a [Party](#Party) in its own right with its own [ledger accounts](#ledger-account).

#### Seller
The Party that publishes a [Sell offer](#Sell-Offer) and delivers its Block against payment. 
The Seller is fixed for the life of the offer.

#### Buyer
The Party that purchases a [Sell offer](#Sell-Offer) in full by submitting a [Transaction Request](#Transaction-Request). 
The Buyer pays the total value and receives the [Block](#Block).

#### Sell Offer State
Enumeration: Available, Locked_For_Pre_Processing, Locked_For_Processing, Executed

#### Ledger Entry
An immutable, timestamped record of a single movement of value (a debit or a credit) on an account in the platform's books.<br/>
A successful purchase is recorded as a set of balanced ledger entries that together show who paid what and who received what.

#### Sell Offer Buying Request
The process entity created when a buyer's click acquires the lock on a [Sell offer](#Sell-Offer). It carries the lifecycle, the lock, the TTLs and the failure code until a final outcome.

#### Transaction Request
A buyer's attempt to purchase a [Sell offer](#Sell-Offer), created when the buyer acquires the lock. 
It carries the lifecycle, TTLs, lock token and failure code, and ends as either succeeded or failed.

#### Payment Transaction
An immutable record of an executed purchase, created atomically when a [Transaction Request](#Transaction-Request) succeeds. 
It is never changed or deleted, refers to [Ledger Entry](#Ledger-Entry).

####  Real-time Sync
The subsystem that delivers offer state changes to every trader viewing a currency pair, so that every screen shows the same truth within the latency target.<br/> 
It carries state and never decides it.<br/> 
The source of truth stays in the Sell Offer (Trading) subsystem, and the UI is a projection of what Real-time Sync delivers.

#### Audit 
An internal subsystem that keeps an append-only, tamper-evident history of every state change, decision and administrative action in AlphaLiquidity.<br/>
It receives audit events from the other subsystems, asynchronously, and can reconstruct the full history of any offer or purchase.<br/>
It supports retention and exports records for investigations and regulator requests.<br/>
It only records history and never decides anything, so it never blocks a trade.

#### Operations alerts
the channel through which the platform warns Platform Operators when something needs attention.
Examples are an offer frozen with an unknown outcome, a stale reference price, a compliance alert relayed from the Compliance System, or a missed latency target.
Subsystems raise alerts as events, and operators receive them by paging, chat or email.
It is one-way and asynchronous, so it never affects trading.

####  Settlement and Custody
An external system, is the part of the landscape that holds real assets and moves them in and out of the platform: 
  * banks and payment rails for fiat
  * blockchain networks and wallets for crypto.<br/> 
The platform's ledger records who owns what inside AlphaLiquidity, and Settlement and Custody makes that match the real world.


#### Identity Provider
An external system that authenticates users and confirms who they are, using standards such as SSO/OIDC and MFA.

#### Compliance System
An external system that makes and records compliance decisions for the platform:
* **KYC** and onboarding checks
* **AML** and sanctions screening,
* limits
* trade surveillance
* case handling and regulatory reporting.<br/>
It is the source of truth for a [Party's](#Party) compliance status.<br/>
**AlphaLiquidity** talks to it only through the Compliance Gateway, which keeps a local copy of status and limits so trades never wait on it.

#### Accounting / Finance Reporting
An external system that keeps the company's books and produces financial and regulatory reports.<br/>
**AlphaLiquidity** sends it journals and balance snapshots from the ledger, including spread revenue, so the books match the platform's balances.<br/>
It is a one-way, asynchronous consumer and never affects trading.

#### Notification Service
An external service that delivers messages to people and systems outside the platform, such as email and webhooks.<br/>
**AlphaLiquidity** sends it events like offer published, purchase succeeded or failed, funding confirmed and account suspended, based on each tenant's notification settings.<br/>
It is one-way and asynchronous, so it never affects trading.