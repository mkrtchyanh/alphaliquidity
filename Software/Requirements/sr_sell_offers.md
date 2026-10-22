# Sell Offer Service: Software Requirements

Software requirements of the Sell Offer service. Sell Offer Management and Sell Offer (Trading) are one service with one database (MS SQL Server). Balance is a separate service with its own database. Internal state changes travel by outbox table, WAL-based CDC and Kafka. Clients are notified through SignalR. Each requirement traces to a system requirement.

## State and intake

### SW-SO-01
Keep on each offer row `expires_on` (offer expiration), `paying_until` (end of the paying interval, empty until the first lock), a terminal outcome (Executed or Withdrawn, or none) and a per-offer version that increases by exactly 1 on every change. Derive the offer state from them and the database clock, in this order: a terminal outcome means Executed or Withdrawn; otherwise now < `paying_until` means Available_For_Paying; otherwise now >= `expires_on` means Expired; otherwise Available_For_Buying. An offer is visible to buyers only while Available_For_Buying and becomes Available_For_Paying only through a successful lock. An offer past `expires_on` is not cut off while it is Available_For_Paying. A paying interval that ends needs no write: the offer is Available_For_Buying again (or Expired) by the clock alone.
#### Traces to
FR-02, FR-03, NFR-03
#### Verification
After any sequence of operations and clock values the derived state follows the state model. An Available_For_Paying offer always has an unreleased latest lock record.

### SW-SO-02
Create an offer from a configured block (SW-SO-19) with a selling price inside the configured range. The offer carries the block's expiration as `expires_on` and starts as Available_For_Buying at version 1. The same offerId with the same content is an idempotent replay. The same offerId with different content is rejected. A block is indivisible: an offer is sold whole.
#### Traces to
FR-16, FR-01, FR-02
#### Verification
A replay returns the original result and creates no second offer. A price outside the range, a disabled or expired block, or a pair that is not permitted is refused.

## Locking

### SW-SO-03
Grant a lock by inserting an immutable lock record (offerId, version, holder_ref = requestId, created_on, expiration) that is unique on offerId + version. Every lock attempt, including each retry, runs in its own database transaction. In the winning attempt the same transaction updates the offer row (`paying_until`, version + 1) and writes the outbox row. The update applies only where there is no terminal outcome, `expires_on` has not passed and the offer is free (no active lock) or the lock is the caller's. If it changes no row, roll back the whole attempt, including the lock record, and refuse the request.
#### Traces to
FR-04, FR-08, NFR-04
#### Verification
With N concurrent lock requests on one offer, exactly one succeeds. Repeat under load. A crash at any point leaves either the lock record, the offer update and the outbox row all committed, or none. A lock is never committed on an offer that is not Available_For_Buying (or held by the caller).

### SW-SO-04
Return a result for every lock attempt. Success gives requestId, the fencing token (the lock version) and the expiration. Failure gives `OFFER_LOCKED` with `retryAfter` = the current `paying_until`, or `OFFER_NOT_AVAILABLE` (offer terminal or Expired).
#### Traces to
FR-04, FR-18
#### Verification
Each outcome has the right code, and `retryAfter` equals `paying_until` of the offer.

### SW-SO-05
Before locking, ask Balance whether a payment transaction already exists for the requestId. If it exists, exit with that result and change no lock. Otherwise decide by the unique constraint: of concurrent inserts for the same offerId and version exactly one wins. Each loser discards its transaction, starts a new one from reading the latest lock record, finds the winner's active lock and returns `OFFER_LOCKED`. When the caller owns the active lock (same requestId), exchange it: release it and create the next version with a new expiration. No queue and no waiting. After a configured attempt limit, a lock attempt still contended returns `OFFER_LOCKED`, never an error.
#### Traces to
NFR-05, FR-08
#### Verification
A later attempt never waits on an earlier one and never overrides it. The owner repeating its request gets a new lock version while no payment exists. A repeat after a payment exists creates no new lock and returns the existing result. Under contention every loser ends in `OFFER_LOCKED` and none surfaces a database error.

## Transitions

### SW-SO-06
Complete and release are driven by Balance outcome events (`ExchangeCompleted`, `PaymentRejected`) that arrive through CDC and Kafka. Apply them only if the event's fencing token equals the version of the latest lock record, otherwise ignore it as `STALE_FENCING_TOKEN` with no change. Complete sets the terminal outcome Executed (version + 1) and `released_on`. Release sets `released_on` once and `paying_until` to now, so the offer is Available_For_Buying again at once. Each is one transaction with its outbox row. Releasing an already released lock is a no-op.
#### Traces to
FR-05, FR-06, FR-07, NFR-03
#### Verification
A late message with an old token changes nothing. A duplicate event changes nothing. After an exchange the offer is Executed exactly once.

### SW-SO-07
Expiry is derived: an offer is Expired once `expires_on` has passed and it is not Available_For_Paying. New locks are refused with `OFFER_NOT_AVAILABLE`. A sweeper finds newly Expired offers, emits the event once and returns the block to the pool, so a new offer can be created from it. Withdraw is a conditional update of the offer row (terminal outcome Withdrawn, version + 1) allowed only while the offer is Available_For_Buying, so it is refused with `OFFER_LOCKED` while a purchase is paid, and racing a lock has exactly one winner. Executed, Withdrawn and Expired are terminal.
#### Traces to
FR-16
#### Verification
A paying offer is never withdrawn mid-payment. An offer past expiration accepts no new lock, and its block is returned to the pool once. A terminal offer accepts no further transition.

### SW-SO-08
Use one fixed paying interval for the whole purchase, from configuration (per pair, with defaults). `paying_until` and the lock expiration = database time at lock + that interval. The interval must not end before the earliest expiry of the purchase's REQUESTED balance records. Use the database clock only, never an application server clock, for `paying_until`, `expires_on` and every state decision.
#### Traces to
FR-05, FR-07
#### Verification
`paying_until` matches the configured interval. Skewing an application server clock changes no state decision. The configuration check refuses an interval shorter than the record expiry.

## Recovery

### SW-SO-09
Trading never waits for recovery: a purchase that gets no outcome ends by the clock, because Balance's REQUESTED records expire by themselves and cannot be finalized later. A sweeper sets `released_on` on locks that ended without a release, and emits the event for changes made by the clock alone (an ended paying interval, an expired offer). Events that are consumed again after a restart or a replay change nothing.
#### Traces to
NFR-15, NFR-03
#### Verification
Kill the service mid-purchase: the offer ends Executed or Available_For_Buying, never sold twice and never frozen without end. A lock with no outcome gets its `released_on` and its event from the sweeper.

## Events

### SW-SO-10
Write each offer or lock change and its event in the same commit: the lock record, the offer row update and the outbox row. CDC reads the outbox table from the write-ahead log and publishes to Kafka, keyed by offerId so events of one offer stay in order, at least once. No event exists for an uncommitted change, and no committed change lacks an event.
#### Traces to
NFR-03, NFR-16
#### Verification
Fault injection between commit and publish never loses or invents an event. Events of one offer arrive in version order, and a gap is detectable by the offer version.

### SW-SO-11
Internal subsystems consume events from Kafka in the internal form, idempotently by event ID. Clients (buyers and sellers) are notified through SignalR (Real-time Sync) in the public form: state and version only, no identities, no failure reason. Only the traders viewing the affected pair receive the event. The way the hub is fed from the commit is open item 5.
#### Traces to
NFR-01, FR-04, FR-09, FR-18
#### Verification
The public form contains no buyer or supplier identity. A client viewing pair A receives no event for pair B. Replaying an event changes nothing.

## Liquidity depth and snapshots

### SW-SO-12
Derive liquidity depth per pair and price level from Available_For_Buying offers and publish changes.
#### Traces to
FR-01
#### Verification
Depth equals the sum of Available_For_Buying blocks per level after any change.

### SW-SO-13
Serve a consistent snapshot of a pair's offers and depth with their versions, so a client can continue with live events with no gap.
#### Traces to
NFR-03
#### Verification
A snapshot plus later events equals the live state.

## Quality

### SW-SO-14
Lock decision time and commit-to-notification time stay within the budget set in the latency budget (target values TBD).
#### Traces to
NFR-01
#### Verification
Measured at p99 under the target load.

### SW-SO-15
Contention exists only between operations on the same offer. Different offers never block one another, and the service scales horizontally (throughput target per hot pair TBD).
#### Traces to
NFR-04, NFR-06
#### Verification
Load test with many offers and one hot offer.

### SW-SO-16
Committed offer state and lock records are durable. No loss of a committed change (RPO 0), and recovery restores locks with their versions and `paying_until`.
#### Traces to
NFR-03, NFR-15
#### Verification
Restart or fail over during load: no change is lost.

### SW-SO-17
Emit an audit event for every offer change and every lock record change (acquire, exchange, release, interval ended), with offerId, requestId, lock version, actor, old and new state and timestamps, through the outbox. Expose metrics for lock contention (insert retries after unique violation), lock exchanges and paying intervals that ended without an outcome.
#### Traces to
NFR-16
#### Verification
Any offer's history can be reconstructed from the audit events.

### SW-SO-18
Accept calls only from authenticated services and authorized users. Never expose buyer, supplier or tenant identities outside the internal form.
#### Traces to
NFR-13, NFR-14
#### Verification
Calls without a valid identity or permission are rejected.

## Liquidity configuration and offers

### SW-SO-19
Let a fund admin define, per currency pair, Bids and Asks with an expiration, blocks (amount) with price ranges, the pair's quote convention (base and quote asset, price and amount precision), trading limits and compliance checks. Each Bid and Ask can be disabled. Expiry of a block returns it to the pool. Only a fund admin of the fund can change the fund's configuration.
#### Traces to
FR-01, FR-16, FR-17
#### Verification
A fund admin creates, changes and disables a configuration. A trader or another fund's admin is refused. A disabled or expired configuration creates no new offer.

### SW-SO-20
Let a fund trader create offers from the fund's blocks (SW-SO-02), withdraw them (SW-SO-07) and see the fund's offers with their state. A purchase is allowed only if the offer matches the buying fund's configured ranges and limits (matching rule: open item 3).
#### Traces to
FR-03, FR-16
#### Verification
A trader creates and withdraws offers for permitted pairs and is refused for pairs that are not permitted. A purchase outside the buyer's ranges is refused with a failure code.

### SW-SO-21
Price an offer with a platform margin based on block size. The platform admin defines the margin globally and per client: a percentage with a minimum amount, a fixed amount, or similar rules. Store the price breakdown (supplier price, margin, final price) with the offer. Show the final price, fee included, to the fund admin, the trader and buyers.
#### Traces to
FR-16, FR-12
#### Verification
The shown price equals supplier price plus margin by the effective rule (per client over global). Changing a rule does not change the breakdown of existing offers.

## Purchase orchestration

### SW-SO-22
After a lock is granted, create the Transaction Request in status Pre_Processing (see SW-SO-25), then send Balance a synchronous "transaction requested" call with requestId (idempotency key), offerId, buyer, seller, supplier price and margin. Balance checks there is enough money and creates the REQUESTED records: buyer debit (price + margin), seller credit and platform fee credit. A rejection (for example `INSUFFICIENT_BALANCE`) ends the request Failed, releases the lock (SW-SO-06) and returns the failure code to the buyer.
#### Traces to
FR-05, FR-07, FR-12, FR-18
#### Verification
Repeating the call with the same requestId creates no second set of records. A rejection leaves the offer Available_For_Buying and the buyer with exactly one failure code.

### SW-SO-23
After the records exist, validate the buyer against Compliance Gateway (status, limits, sanctions flag) and the fund's configured limits. On failure, ask Balance to reject the records, end the request Failed, release the lock and return one failure code. On success, insert the payment transaction locally in a new database transaction with its outbox row (SW-SO-26). CDC and Kafka carry it to Balance.
#### Traces to
FR-05, FR-07, FR-18
#### Verification
A failed check rejects the records and frees the offer. A passed check inserts exactly one payment transaction per requestId.

### SW-SO-24
Treat the purchase as complete only on the `ExchangeCompleted` event: Balance has moved the records to EXECUTED and inserted the exchange record (the immutable Transaction) in one transaction. An accepted request, a lock, REQUESTED records or an inserted payment transaction are not completion. On completion the offer becomes Executed, the lock is released and the request Succeeded in one transaction. The offer disappears from all screens and is shown in the Executed Transactions log. On `PaymentRejected` the request ends Failed and the lock is released (fencing token checked, a stale token is ignored).
#### Traces to
FR-06, NFR-03
#### Verification
No offer is Executed before its `ExchangeCompleted` event. After completion the offer appears once in the Executed Transactions log.

## Transaction Request and payment transaction

### SW-SO-25
Keep the Transaction Request in the Sell Offer database. Create it from a lock grant (requestId, offerId, buyer Party and Trader, price snapshot, fencing token) in status Pre_Processing. Creation is idempotent by requestId. Statuses are Pre_Processing, Processing, Succeeded and Failed, with a version that grows on every change. Succeeded and Failed are terminal and immutable. At most one request is active per offer.
#### Traces to
FR-04, FR-05, FR-06, FR-07, NFR-04, NFR-16
#### Verification
A replayed grant returns the original request. No transition out of a terminal status is accepted.

### SW-SO-26
Fix the terms at creation (pair, volume, unit price, margin, total value, pricing breakdown, supplier reference). Run validation in this order and stop at the first failure: self-trade block (local rule, buyer and supplier are the same Party), Balance check with REQUESTED records, then compliance lookup. Pass each failure code through unchanged from its owner. On success move to Processing and insert the payment transaction (status Processing, new database transaction, outbox row, timeout shorter than the record expiry).
#### Traces to
FR-02, FR-05, FR-07, FR-08, FR-18, NFR-08
#### Verification
Each failure ends the request at the right step and later steps do not run. A later change to the offer never alters an existing request. The code seen by the buyer equals the code of the failing step.

### SW-SO-27
Never insert the payment transaction after the lock deadline or the record expiry has passed. If the request has no outcome by then it is marked Failed by the sweeper (`PROCESSING_TIMEOUT` or `VALIDATION_TIMEOUT`), the event is emitted and an alert is raised. The records expire by themselves.
#### Traces to
NFR-04, NFR-15
#### Verification
After a crash mid-processing the offer ends Executed or Available_For_Buying, never both. A late payment transaction is refused.

### SW-SO-28
Resume requests found incomplete after a restart without waiting for the clock. Every command and event handler is idempotent: a replay returns the original result and causes no new effect.
#### Traces to
NFR-04, NFR-15
#### Verification
Replaying any message sequence leaves the same final state. A killed instance's requests are resolved within a bounded time.

### SW-SO-29
Deliver the buyer's own outcome, including the failure code, through the client channel (SignalR). Failure reasons go only to the buyer, never to other traders. Start external settlement asynchronously after execution through Balance. Settlement never delays the request.
#### Traces to
FR-12, FR-18, NFR-14
#### Verification
The buyer receives exactly one outcome per request. The request completes without waiting for settlement.

### SW-SO-30
Serve a tenant's own requests and Transactions, scoped to that tenant. Staff and auditors may read across tenants with authorization. Provide the executed-trade record for the "Executed Transactions" archive log. Which parts are public and anonymized is an open decision.
#### Traces to
FR-06, NFR-13, NFR-14
#### Verification
One tenant can never read another tenant's data. An executed purchase appears in the archive log once.

### SW-SO-31
Record each Transaction Request status change with requestId, actor, old and new status, failure code, the compliance status version used and timestamps. Write the status change, the offer change and the outbox row in one commit.
#### Traces to
NFR-03, NFR-16
#### Verification
The decision path of any request can be reconstructed. No lost or invented events under fault injection.
