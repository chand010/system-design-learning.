# Day 11 — Message Queues and Asynchronous Processing

A **message queue** lets one part of a system hand work to another part without waiting for that work to finish. It can buffer bursts, reduce tight coupling, and let workers scale independently. It also introduces new failure modes that must be designed deliberately.

## 1. Start with a real example

An online shop receives an order. The customer should receive a quick confirmation, while several other tasks may take longer:

- send an email or SMS;
- reserve stock;
- update analytics;
- create a shipping request.

If the order API calls every service synchronously, a slow email provider could delay the checkout response. If the API publishes a durable event for background processing, it can return sooner while workers handle the follow-up tasks.

Do not move every operation to a queue automatically. Payment authorization, inventory consistency, and the customer-visible order status may have requirements that need synchronous checks or a carefully designed workflow.

## 2. Core terms

- **Producer / publisher:** creates and sends a message.
- **Message:** a unit of work or an event, often containing an ID, type, timestamp, and payload.
- **Queue:** holds work until a consumer can process it.
- **Broker:** software that accepts, stores, routes, and delivers messages.
- **Consumer / worker:** receives and processes messages.
- **Acknowledgement (ack):** tells the broker that a delivery was handled successfully.
- **Retry:** another processing attempt after a failure.
- **Dead-letter queue (DLQ):** a place for messages that cannot be processed normally and need investigation or a separate recovery process.

A broker is common but not strictly required by every queue implementation.

## 3. Synchronous vs asynchronous

### Synchronous flow

```text
Client -> Order API -> Email provider
                    -> Analytics service
                    -> Shipping service
Client waits for the required calls to finish
```

Advantages: straightforward control flow and immediate responses from dependencies.

Disadvantages: latency adds up; a slow or unavailable dependency can delay or fail the request; each service is more tightly coupled to the others.

### Asynchronous flow

```text
Client -> Order API -> Database (save order + event intent)
                         |
                         v
                    Message broker
                    /     |      \\
                   v      v       v
             Email worker Stock worker Analytics worker
```

The API publishes work and workers process it later. The client may get a response before every background task finishes. The system must expose task/order status when users need to know whether work has completed.

**Important:** A queue improves decoupling and buffering; it does not automatically make a system reliable. Publishing, persistence, retries, idempotency, monitoring, and recovery all matter.

## 4. Queue vs publish/subscribe

### Work queue / competing consumers

A message is normally handled by one worker from a group. Adding workers can increase processing capacity.

Example: each image-upload job should be processed by one worker.

### Publish/subscribe (pub/sub)

An event is published to a topic or exchange and routed to multiple independent subscribers. Each subscriber may have its own queue or subscription.

Example: an `OrderPlaced` event may trigger separate email, analytics, and fulfillment workflows.

The exact delivery and replay behavior depends on the broker and its configuration. Do not assume every queue or pub/sub product has the same semantics.

## 5. Message lifecycle

A typical lifecycle looks like this:

1. The producer creates a message with a unique message ID and a schema/version.
2. It publishes the message to the broker.
3. The broker accepts it according to its configured durability and acknowledgement rules.
4. A worker receives the message.
5. The worker validates and processes it.
6. After successful processing, the worker acknowledges the delivery.
7. If processing fails, the message is retried according to a defined policy or routed to a DLQ.

For important messages, configure durable queues/topics and persistent messages where supported, and use publisher confirms or the broker's equivalent when appropriate. Exact settings and guarantees vary by product. A successful publish call does not universally prove that the message is safely stored and will survive every failure.

## 6. Delivery guarantees and duplicates

Three common delivery models are:

| Model | Meaning | Trade-off |
|---|---|---|
| At-most-once | A message is not redelivered; it can be lost | Lower duplicate risk, but possible loss |
| At-least-once | A message is retried until acknowledged under the configured policy; duplicates can occur | Better recovery, but consumer must handle duplicates |
| Exactly-once | A scoped guarantee that a defined processing effect occurs once | Usually requires specific broker/transaction features and careful integration; not a universal end-to-end promise |

**Design for idempotency.** An idempotent operation can be repeated without applying the same business effect twice.

Example: before creating a shipment, the worker records a unique `event_id` or `order_id` in a database table with a uniqueness constraint. If the same event is delivered again, the worker detects it and avoids creating a second shipment.

A common failure sequence is:
1. The worker updates the database successfully.
2. The worker crashes before sending the acknowledgement.
3. The broker redelivers the message.
4. The worker receives the same message again.

Without idempotency, the business operation may happen twice. Acknowledgements alone cannot eliminate this failure window.

## 7. Retries, backoff, and dead-letter queues

Not all errors should be treated the same way.

- **Transient failure:** a network timeout or temporary dependency outage may succeed on a later attempt.
- **Permanent failure:** invalid data or an unsupported message schema will usually fail again until the message or code is fixed.
- **Poison message:** a message that repeatedly crashes or fails a consumer.

A sensible retry policy can include:
- a maximum number of attempts;
- exponential backoff, increasing the wait between attempts;
- random jitter to prevent many workers retrying simultaneously;
- a DLQ or equivalent quarantine after the retry limit;
- alerts and a documented process for inspecting and replaying failed messages.

Do not retry forever at high speed. It can overload a recovering dependency and prevent healthy messages from progressing. Be cautious with requeueing because it may create a hot loop.

A DLQ is not a repair by itself. Monitor it, preserve enough context to investigate, and only replay messages after correcting the underlying problem.

## 8. Ordering and concurrency

A queue may preserve some ordering, but that does not mean all work completes in that order.

With multiple workers, message A may take longer than message B, so B can finish first. Retries can also change the observed order.

If ordering matters:
- identify the ordering key, such as `order_id`;
- partition or route messages by that key where the broker supports it;
- avoid processing the same key concurrently if it would break business rules;
- accept that strict ordering can reduce parallelism.

Do not assume global ordering unless the chosen technology and configuration explicitly provide it.

## 9. Backpressure and queue growth

A queue absorbs a temporary traffic spike, but it cannot fix a permanent mismatch between incoming work and processing capacity.

If producers add 1,000 jobs/second while workers complete 700 jobs/second, the backlog grows by roughly 300 jobs/second.

Monitor:
- queue depth and oldest-message age;
- publish and consume rates;
- processing latency and failures;
- retry counts and DLQ depth;
- worker utilization and dependency latency.

Possible responses include adding workers, reducing unnecessary work, applying producer backpressure, limiting prefetch/concurrent work, or scaling the downstream dependency. Adding workers blindly can overload a database or API.

## 10. The database-and-message dual-write problem

Suppose the order API first saves an order in SQL and then publishes `OrderPlaced`.

What if the database commit succeeds but publishing fails? The order exists, but no worker is notified.

If the API publishes first and the database write then fails, workers may receive an event for an order that was never committed.

One common solution is the **transactional outbox pattern**:

1. In one database transaction, save the business record and an outbox event.
2. A separate publisher reads unsent outbox rows and publishes them.
3. After successful publication, mark the outbox row as sent.
4. Make consumers idempotent because publication or processing can still be repeated.

The outbox avoids requiring one atomic transaction across the application database and the message broker. It still needs retries, monitoring, and duplicate-safe consumers.

## 11. Security and operations

- Authenticate and authorize producers and consumers.
- Encrypt connections and protect credentials.
- Validate message schema and payload size.
- Avoid putting secrets or unnecessary personal data in messages and logs.
- Set retention, expiry, and storage limits deliberately.
- Monitor broker availability, queue age, delivery failures, and consumer lag.
- Define recovery and replay procedures before a production incident.

## 12. Practical design exercise

Design asynchronous processing for an online shop:

- After an order is saved, send a confirmation email.
- Update analytics.
- Notify a fulfillment service.
- The email provider can be unavailable for 20 minutes.
- A worker may crash after sending an email but before acknowledging the message.

Answer:

1. Which tasks should be synchronous, and which can be asynchronous? Explain the business requirements behind your choice.
2. Draw the producer, broker, queues/topics, and consumers.
3. What information belongs in the message? Include an ID and schema version.
4. How will the system retry email failures without retrying forever?
5. How will you avoid creating duplicate business effects?
6. How will you prevent an order being saved without its event being published?
7. What metrics and alerts would you configure?
8. How would you recover and replay a DLQ message safely?

### Interview structure

When asked to design an asynchronous workflow:

1. Clarify what must happen before the client receives a response.
2. Identify producers, events, queues/topics, and consumers.
3. Define persistence and delivery requirements.
4. Explain acknowledgements, retries, backoff, DLQs, and idempotency.
5. Discuss ordering, scaling, backpressure, and failure recovery.
6. Cover monitoring, security, and the database/message dual-write problem.

## Key takeaways

- Queues decouple producers and consumers and can buffer temporary bursts.
- Asynchronous processing improves responsiveness but makes completion eventual.
- At-least-once delivery means duplicate-safe consumers are important.
- Retries need backoff, limits, and a recovery path such as a DLQ.
- Queue depth and message age reveal when consumers are falling behind.
- The transactional outbox pattern helps avoid losing events between a database commit and message publication.

## Learning resources

- [GeeksforGeeks — Message Queues: System Design](https://www.geeksforgeeks.org/system-design/message-queues-system-design/)
- [GeeksforGeeks — Pub/Sub Architecture](https://www.geeksforgeeks.org/system-design/what-is-pub-sub/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials) — official beginner tutorials for queues and work distribution.
- [RabbitMQ AMQP concepts](https://www.rabbitmq.com/tutorials/amqp-concepts) — acknowledgements, delivery, and dead-letter behavior.
- [Apache Kafka — Message Delivery Semantics](https://kafka.apache.org/40/design/design/) — at-most-once, at-least-once, and exactly-once scope.
- [Google SRE Books](https://sre.google/books/) — reliability, monitoring, and operating distributed services.
- [Codecademy](https://www.codecademy.com/) — use its free learning materials where available to practise programming fundamentals.

## Source review note — 18 October 2026

The accompanying repository resource changelog records the official-source review and the limits of historical comparison. No verified material change was established that requires rewriting existing Days 1–10 notes. This new lesson uses the currently available official message-queue references; source page update dates are not treated as proof of historical content differences.
