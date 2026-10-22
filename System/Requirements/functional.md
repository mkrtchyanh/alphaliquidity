# System Requirements: Functional


### FR-01
The dashboard displays individual sell offers and liquidity depth per currency pair (for example AMD-USD, BTC-USD, USD-EUR).

#### Verification
Open the dashboard for each supported pair. Individual offers and depth are shown and match the offers held by the system.

### FR-02
Each sell offer is executed All-or-None: in its full volume or not at all. No partial fills.

#### Verification
Try to buy part of an offer: not possible. Every executed purchase moves the full volume of the offer.

### FR-03
Each Available_For_Buying sell offer has a "Purchase" button for eligible buyers.

#### Verification
Every Available_For_Buying offer shows an enabled Purchase button to an eligible buyer.

### FR-04
Instant Freeze: when a buyer clicks Purchase, the offer's button is disabled via SignalR for all other traders viewing that currency pair, worldwide.

#### Verification
With two clients viewing a pair, a click on one disables the button on the other within the latency target (NFR-01).

### FR-05
Processing Phase: the button stays disabled while the multi-server backend validates the balance, creates the ledger entry and finalizes the transaction.

#### Verification
Throughout validation and processing the button stays disabled on all screens, including after reconnects.

### FR-06
Success: the offer disappears from all screens and moves to the "Executed Transactions" archive log.

#### Verification
After a successful purchase the offer is gone from every screen and appears exactly once in the Executed Transactions log.

### FR-07
Failure (for example insufficient buyer margin or validation errors): the button is re-enabled on all screens and the offer can be purchased again.
#### Traces to
Brief (3)
#### Verification
After any failure the button is enabled on all screens and a new purchase can start.

### FR-08
Only one buyer can hold an offer at a time. Concurrent attempts have exactly one winner.

#### Verification
N simultaneous clicks on one offer produce exactly one lock and one winner. The others receive a lock failure.

### FR-09
State changes go only to traders viewing the affected currency pair.

#### Verification
A client viewing pair A receives no events for pair B.

### FR-10
Each Party has one account per supported currency, fiat and crypto (including BTC).
#### Traces to
System analysis (accounts per currency)
#### Verification
Every Party has an account for each supported currency, created as decided (at onboarding or on first use).

### FR-11
Balances change only through ledger entries (debit or credit). No direct edits.
#### Traces to
System analysis (accounts per currency)
#### Verification
No interface allows editing a balance. Every balance change has a matching ledger entry.

### FR-12
On execution, the buyer's account is debited the price plus the platform margin, the seller's account is credited the supplier price and the platform fee account is credited the margin. These are credit and debit records that go from REQUESTED to EXECUTED, and an exchange record is stored with them.
#### Traces to
System analysis (accounts per currency)
#### Verification
After an execution the expected records are EXECUTED and the exchange record exists, and balances changed as expected on all sides.

### FR-13
The system exposes an account's eligible balance for validation: EXECUTED credits plus EXECUTED and REQUESTED debits. Transfers to a bank use EXECUTED records only.
#### Traces to
System analysis (accounts per currency)
#### Verification
The eligible balance read equals EXECUTED credits plus EXECUTED and REQUESTED debits.

### FR-14
When a purchase is requested, REQUESTED debit records reserve the buyer's funds. They end EXECUTED or REJECTED, or expire by themselves.
#### Traces to
System analysis (accounts per currency)
#### Verification
Creating REQUESTED records lowers the eligible balance. Rejecting or expiring them restores it.

### FR-15
A tenant can request deposits and withdrawals and see balances per currency.
#### Traces to
System analysis (Tenant Management)
#### Verification
A tenant submits a deposit or withdrawal request, sees its status and sees the updated balance.

### FR-16
A tenant can create and withdraw block offers for permitted pairs.
#### Traces to
System analysis (Tenant Management, Sell Offer)
#### Verification
A tenant creates, views and withdraws offers for permitted pairs, and is refused for pairs that are not permitted.

### FR-17
A tenant manages its users, roles and API credentials.
#### Traces to
System analysis (Tenant Management)
#### Verification
A Fund Admin adds a user, assigns a role and issues credentials. A user without the role cannot do these actions.

### FR-18
Failure feedback: when a purchase fails, the buyer receives a failure code. Other traders see only that the offer became Available_For_Buying again.
#### Traces to
System analysis (failure flows, failure codes)
#### Verification
A failed purchase gives the buyer exactly one failure code. Other clients see only the Available_For_Buying state.

### FR-19
Balance reservation creates balance reservation record associated with offer and credit/debit records atomically.
#### Traces to
System analysis (Sell Offer, Balance)
#### Verification
Trader buys an offer - prepay-phase(pre-processing)

### FR-20
Expired,Rejected payment: release balance record creation and updates credit/debit records to REJECTED atomically.
#### Traces to
System analysis (Sell Offer, Balance)
#### Verification
Simulate Balance Service not-healthy case, just after balance reservation was done.

### FR-21
A purchase is considered done only after marking corresponding credit/debit records as EXECUTED. 
#### Traces to
System analysis (Sell Offer, Balance)
#### Verification
Offer becomes EXECUTED only after corresponding balance change

### FR-20
If credit/debit records associated with this offer were expired the records are marked as REJECTED and offer is returned to Available_For_Buying state.
#### Traces to
System analysis (Sell Offer, Balance)
#### Verification
Simulate Balance Service not-healthy case, just after balance reservation was done
