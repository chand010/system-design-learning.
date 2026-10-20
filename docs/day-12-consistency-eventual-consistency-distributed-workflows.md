# Day 12 — Consistency, Eventual Consistency, and Distributed Workflows

## Learning objectives
Understand strong vs eventual consistency, replication lag, business invariants, sagas, idempotency, and safe recovery from partial failure.

## 1. Consistency
A distributed system may store copies of data on multiple nodes. After a write, a replica may not yet have applied the change.

**Strong consistency** must be defined by the system's contract; for this lesson, a covered read observes the latest successfully committed write. **Eventual consistency** permits temporary disagreement and expects replicas to converge if updates stop and replication continues successfully. It does not promise a specific maximum delay unless separately documented.

Example: a profile update succeeds on the primary, but a read replica briefly returns the previous picture.

## 2. Replication lag and read-your-writes
Replication lag is the delay between a change being committed and being applied or visible elsewhere. Network delay, heavy writes, or an overloaded replica can increase it.

Options include reading from the primary immediately after a write, routing a user's follow-up reads consistently, waiting for a replication position when supported, or accepting stale data for noncritical features such as analytics. Check the chosen database's guarantees; replicas, indexes, and streams may behave differently.

Useful properties:
- **Read-after-write:** a client sees its own successful write in covered subsequent reads.
- **Monotonic reads:** a client does not later see an older value after seeing a newer one, within the guarantee.
- **Causal consistency:** related operations are observed in cause-and-effect order.

## 3. CAP theorem
The classic CAP trade-off concerns a network partition:
- **Consistency:** a strong single-copy-style guarantee.
- **Availability:** every request to a non-failing node receives a response.
- **Partition tolerance:** the system operates despite communication failures between nodes.

During a partition, a distributed system cannot guarantee both CAP consistency and CAP availability for every operation. This does not mean every database permanently “chooses two”; behavior depends on topology, operation, and failure mode.

## 4. Business invariants
An invariant is a rule that must remain true: a reservation must not exceed available inventory; an order must not ship before required payment conditions; a repeated request must not create the same business effect twice.

Enforce invariants at the authoritative operation using transactions, constraints, atomic updates, version checks, idempotency keys, or a state machine. UI checks alone are insufficient. If two customers reserve the last seat concurrently, the final database operation must coordinate them so only a valid reservation succeeds.

## 5. Distributed workflows and sagas
A workflow can span order, inventory, payment, shipping, and notification services. Each step may succeed or fail independently.

A **saga** coordinates local transactions and may use compensating actions when a later step fails:
1. Create a pending order.
2. Reserve inventory.
3. Authorize payment.
4. Request shipment.
5. Mark the order complete.

If payment fails after inventory is reserved, a compensation may release inventory and mark the order failed. Compensation is not a time machine: a refund is a new transaction, and an email cannot be unsent.

- **Choreography:** services react to events; simple at first but potentially difficult to trace at scale.
- **Orchestration:** a coordinator directs steps and records workflow state; clearer control but the coordinator must also be operated reliably.

## 6. Idempotency and uncertain outcomes
A payment request may succeed even if its response is lost. Retrying as a new operation can charge the customer twice. An **idempotency key** identifies retries of the same logical operation. Follow the provider's rules, check operation status when possible, and reconcile uncertain outcomes before creating a new operation.

Message consumers also need duplicate handling. Record a unique event or operation ID with a database uniqueness constraint so redelivery does not repeat the business effect.

## 7. Transactional outbox
A dual-write failure occurs when the application saves a record and separately publishes an event:
- Database commit succeeds, publication fails: the event is lost.
- Publication succeeds, database transaction fails: consumers may see an event for nonexistent data.

With a **transactional outbox**, one database transaction saves both the business record and an outbox event. A publisher later sends pending events and marks them sent. Consumers must still be idempotent because the publisher may crash after publishing but before marking the event sent.

## 8. Monitoring and recovery
Track replication lag, event delivery delay, outbox backlog, workflow completion time, retries, failed compensations, orders stuck in pending states, and unresolved payment outcomes. Define alerts and safe recovery steps. A stuck order may need status reconciliation—not a blind retry of a payment request.

## 9. Practical exercise
Design checkout with these requirements: no duplicate charge for one logical operation; inventory must not remain reserved forever; shipping can be down for 20 minutes; the order page can show “Processing.”

Answer:
1. Which operations require strong coordination, and which can tolerate eventual consistency?
2. Draw the order-to-shipment workflow and failure paths.
3. What if payment succeeds but its response times out?
4. How will retries avoid duplicate effects?
5. What if payment fails after inventory reservation?
6. How can the workflow resume after restart?
7. What metrics and alerts would you use?
8. How would you reconcile uncertain outcomes?

## Interview structure
Clarify invariants and acceptable staleness; identify authoritative data owners; choose guarantees per operation; draw success and failure paths; explain idempotency, compensation, and outbox; then cover monitoring and recovery.

## Recap
- Strong and eventual consistency are different guarantees.
- Replication lag can hide a recent write.
- Enforce business invariants at authoritative operations.
- Sagas model multi-step workflows and compensation.
- Idempotency helps retries and duplicate delivery remain safe.
- An outbox reduces the risk of losing events between database commit and publication.

## Further reading
- [GeeksforGeeks — System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/)
- [DynamoDB — Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [AWS — Saga Patterns](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-patterns.html)
- [AWS — Transactional Outbox](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [Google SRE Books](https://sre.google/books/)
- [Codecademy](https://www.codecademy.com/)

## Source review note — 20 October 2026
No verified material source change requiring revision of these concepts was established in today's review. See RESOURCE-CHANGELOG.md for the source comparison and its historical-comparison limitations.
