# System Requirements: Non-Functional

System-level non-functional requirements of AlphaLiquidity. NFR-01 to NFR-08 come from the brief. NFR-09 onward come from the system analysis. The verification lines for these requirements are proposed acceptance checks.

### NFR-01
Latency: disable and enable state reaches traders' viewports in under 150 ms across the multi-instance tier.
#### Traces to
Brief
#### Verification
Measure commit to client receive across instances and regions. The percentile, regions and measuring point are open decisions.

### NFR-02
Transport: real-time synchronization uses SignalR.
#### Traces to
Brief
#### Verification
State updates reach clients over SignalR connections.

### NFR-03
Consistency: zero state mismatch between backend state and what any trader sees.
#### Traces to
Brief
#### Verification
After any sequence of operations, including reconnects and lost events, every client converges to the server state.

### NFR-04
Concurrency control: race-free locking under heavy load, no double execution.
#### Traces to
Brief
#### Verification
Concurrency and load tests show no double execution and no lost lock.

### NFR-05
Fairness: all traders are treated equally and the first valid attempt wins.
#### Traces to
Brief (fairness)
#### Verification
Of competing clicks, the one committed first wins in every run, regardless of region or tenant.

### NFR-06
Scalability: multi-instance, multi-server tier handling high volume under heavy load.
#### Traces to
Brief (heavy load)
#### Verification
A load test at the target volume across several instances meets latency and correctness targets.

### NFR-07
Global reach: all traders worldwide receive the synchronized state.
#### Traces to

#### Verification
Clients in several regions receive state changes within the latency target.

### NFR-08
Integrity: an offer is marked Executed only after Balance has moved the records to EXECUTED and stored the exchange record in one transaction.

#### Verification
No offer reaches Executed without an ExchangeCompleted event.

### NFR-09
Finalizing the credit and debit records and storing the exchange record is atomic and idempotent. Every consumer is idempotent by event id.
#### Traces to
System analysis (Balance Crediting)
#### Verification
Replaying a payment transaction creates no second exchange record. Replayed events leave the same final state.

### NFR-10
Client accounts cannot go negative.
#### Traces to
System analysis (Balance Crediting)
#### Verification
A posting that would overdraw an account is rejected.

### NFR-11
Any balance can be recomputed from its entries, and every entry is immutable and auditable.
#### Traces to
System analysis (Balance Crediting)
#### Verification
Recomputing a balance from its entries equals the stored balance. Entries cannot be modified or deleted.

### NFR-12
Amounts use fixed precision per currency with a defined rounding rule.
#### Traces to
System analysis (Balance Crediting)
#### Verification
Every operation keeps the currency's precision and the defined rounding.

### NFR-13
Tenant data is isolated. No tenant can see another tenant's balances, offers or history.
#### Traces to
System analysis (Tenant Management)
#### Verification
A user of tenant A cannot read any data of tenant B through any interface.

### NFR-14
Tenant actions are authorized per role and audited.
#### Traces to
System analysis (Tenant Management)
#### Verification
Actions without the required role are rejected. Every action leaves an audit record.

### NFR-15
Recoverability: after the failure of any instance or component, every offer ends in a correct state: executed exactly once, or Available_For_Buying again. It is never sold twice and never left frozen without an alert. Recovery needs no manual action except for flagged cases.
#### Traces to
System analysis (recovery flows)
#### Verification
Kill instances during processing and locking. Every offer ends Executed once or Available_For_Buying. Unresolved cases raise an alert.

### NFR-16
Auditability of state changes: every state change of an offer and of a Transaction Request is recorded with who, when, and the old and new state. The history of any offer or purchase can be reconstructed.
#### Traces to
System analysis (recovery flows, audit)
#### Verification
For any offer or purchase, the full state history can be reconstructed from the records.

# System Requirements: Infrastructure and Network

Infrastructure and network requirements of AlphaLiquidity. The deployment is one datacenter. The verification lines are proposed acceptance checks.

### INF-01
Deploy in a single datacenter. Every stateful component (database, Kafka, Redis) has a replica in the same datacenter. The replicas cover hardware failure and recovery only. Loss of the datacenter is outside this design.
#### Traces to
NFR-15
#### Verification
Disable one host of each stateful component. The platform recovers as defined in INF-08 to INF-11.

### INF-02
Place the components in separate network segments: Kubernetes cluster nodes, CI/CD nodes, docker registry, DMZ (reverse proxy), DB nodes, Kafka nodes (Kafka Connect and Debezium share the broker hosts), Redis nodes, quorum (witness), monitoring, backup, and egress proxy. Traffic between segments is denied unless a rule below allows it.
#### Traces to
NFR-14
#### Verification
A connection test between every pair of segments shows only the allowed flows.

### INF-03
The DMZ holds the reverse proxy. It terminates TLS and forwards HTTP, HTTPS and WebSocket traffic to the Kubernetes ingress. Its WebSocket idle timeout, and that of the ingress, is longer than the SignalR keep-alive interval.
#### Traces to
NFR-01, NFR-14
#### Verification
An idle SignalR connection survives longer than the keep-alive interval through the proxy and the ingress.

### INF-04
The CI/CD nodes reach the docker registry only through a VPN and an SSH bastion, for pushing images. Nothing else can push to the registry.
#### Traces to
NFR-14
#### Verification
A push from any other segment is refused.

### INF-05
The docker registry accepts outbound traffic only. The Kubernetes cluster nodes may pull images. No other inbound traffic is open.
#### Traces to
NFR-14
#### Verification
Pull works from the cluster nodes. Any other inbound connection is refused.

### INF-06
The DB nodes accept connections from the Kubernetes cluster and from the Kafka Connect nodes (CDC reads). Their outbound traffic is limited to replication, backups and CDC reads.
#### Traces to
NFR-14, NFR-08
#### Verification
A connection from any other segment is refused. An outbound connection from a DB node to any other destination is refused.

### INF-07
The Redis nodes serve the SignalR backplane for the SignalR hub. The Kafka nodes serve CDC synchronization between services and the audit log only. The freeze and unfreeze path to clients does not depend on Kafka.
#### Traces to
NFR-01, FR-04
#### Verification
With Kafka stopped, freeze and unfreeze still reach clients within the latency target.

### INF-08
The database has a synchronous replica with automatic failover and an asynchronous replica with manual promotion. A witness node provides the third vote so that failover is safe. When the synchronous replica is lost, writes stop (RPO 0). They resume when the replica returns or an operator decides.
#### Traces to
NFR-11, NFR-15
#### Verification
Stop the primary: the synchronous replica becomes primary and no committed transaction is lost. Stop the synchronous replica: writes stop and an alert is raised.

### INF-09
Kafka runs two brokers with `min.insync.replicas=2`, producers use acks=all, and leader election is automatic. The Kafka controller quorum has three voters: both brokers and the witness. When one broker is down, publishing pauses and outbox rows wait in the database until it returns.
#### Traces to
NFR-03, NFR-15
#### Verification
Stop one broker: no event is lost or duplicated in effect, and the backlog drains after it returns.

### INF-10
Redis has a primary and a replica for fault tolerance only. Sentinel promotes the replica automatically, using the witness as the third vote. Messages lost during a failover are accepted. Clients recover by the version gap check and a snapshot resync.
#### Traces to
NFR-15
#### Verification
Stop the Redis primary: the replica is promoted and clients converge to the correct state.

### INF-11
Kafka Connect and Debezium run on the same hosts as the Kafka brokers, in the Kafka segment, and read CDC from the database primary. After a database failover the connector reconnects to the new primary and resumes from its stored position without losing or duplicating outbox rows in effect. A tested runbook exists.
#### Traces to
NFR-03, NFR-15
#### Verification
Fail over the database during load. Every outbox row reaches Kafka and consumers show no duplicate effect.

### INF-12
The witness node holds no business data and runs on a separate physical host from the data nodes. Only the DB, Kafka and Redis nodes can reach it on the cluster ports. Its loss raises an alert.
#### Traces to
NFR-15
#### Verification
Stop the witness: an alert is raised and no service stops. Check that it shares no host with a data node.

### INF-13
Every call from the platform to an external system (Compliance System, settlement and custody, identity provider, notification service) goes through one egress proxy with a destination allowlist and a connection log.
#### Traces to
NFR-14
#### Verification
A direct outbound connection from the cluster to any external address is refused. Every external call appears in the proxy log.

### INF-14
A monitoring segment, outside the Kubernetes cluster, holds metrics, logs, traces, dashboards, alerting and the synthetic latency probes. All segments send telemetry to it. Alerts leave only through the egress proxy.
#### Traces to
NFR-01, NFR-16
#### Verification
With the cluster down, the monitoring segment still shows the last state and raises the alert.

### INF-15
A backup segment on separate hardware receives the full and log backups of the databases and the backups of the audit store. It accepts traffic only from those sources. Redis has no backup. Kafka has no backup beyond its replica.
#### Traces to
NFR-11, NFR-15
#### Verification
Restore a database from the backup segment to a test point in time.

## Open items

1. Off-site copy of the backups and a second site later.
2. Who is on call for the manual promotion of the asynchronous DB replica, and the runbook.
3. Capacity of the segments and of the witness node.