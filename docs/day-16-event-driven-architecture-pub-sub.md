# Day 16 — Event-Driven Architecture and Publish/Subscribe

**Learning objectives:** Explain event-driven architecture (EDA), distinguish events from commands and requests, describe publishers/topics/subscriptions/consumers, compare work queues with pub/sub, and design a basic asynchronous event flow with reliability and observability.

## 1. Start with a real example

Imagine the AI Image Detector accepts an image and saves a record saying that analysis is requested. Several things may then need to happen:

- run the image analysis;
- check external provenance when configured and approved;
- update the result status;
- notify the user;
- record operational metrics.

If the upload request calls every downstream component synchronously, a slow notification provider could delay the upload response. An event-driven design lets the application record the accepted work and publish an event so independent consumers can react.

Do not make every action asynchronous automatically. The application must first enforce upload authorization, validation, privacy rules, and any checks required before accepting the request. Asynchronous processing is appropriate only when the product can tolerate work completing later.

## 2. What is event-driven architecture?

**Event-driven architecture (EDA)** is an architectural style in which components communicate by producing and reacting to events that describe something that happened.

An event is a fact, usually expressed in the past tense:
- `ImageUploadAccepted`
- `AnalysisRequested`
- `AnalysisCompleted`
- `ProvenanceCheckUnavailable`

An event should describe a meaningful occurrence, not ask another component to do something. For example, `ImageUploaded` describes a fact; `AnalyzeThisImage` is closer to a command telling a worker what to do.

EDA often uses a message broker, event bus, queue, topic, or event stream, but these are implementation choices rather than the definition of EDA itself.

### Request-response vs event-driven flow

**Direct request-response**

```text
Client -> PHP API -> Analysis service
                 <- Result
Client waits for the analysis call to finish
```

This is useful when the caller needs an immediate answer and the operation fits within the request's time budget.

**Event-driven flow**

```text
Client -> PHP API -> Database
                       |
                       v
                Event broker/topic
                  /      |       \
                 v       v        v
             Analysis  Status   Notification
              worker   updater     worker
```

The API can return an accepted/pending response after the request and event intent have been durably recorded according to the design. Consumers process their work independently. This improves decoupling and burst handling, but the result may not be ready immediately.

## 3. Core components

- **Event producer / publisher:** component that emits an event.
- **Event:** record of something that happened, often with an ID, type, timestamp, schema version, and payload.
- **Broker / event bus:** accepts and routes messages according to the selected technology and configuration.
- **Topic:** named category or stream to which publishers send messages.
- **Subscription:** a consumer's configured view or delivery path for messages from a topic.
- **Consumer / subscriber:** application or worker that receives and processes events.
- **Event handler:** code that implements the response to a received event.
- **Acknowledgement:** signal that a delivery was processed, with exact behavior defined by the broker.

Product terminology varies. A Kafka topic, a RabbitMQ exchange and queue, and a Google Cloud Pub/Sub topic and subscription are not identical features, even though they can implement related messaging patterns.

## 4. Publish/subscribe (pub/sub)

**Pub/sub** is a messaging pattern where publishers send messages to a topic or channel without needing to know each subscriber. Multiple independent subscriptions can receive the same event.

Example: after an order is placed, separate consumers may update analytics, send a confirmation, and start fulfillment. The order service does not need a direct call to each consumer.

```text
                  OrderPlaced
                       |
                 Publisher/API
                       |
                    Topic
                 /     |      \
                v      v       v
          Email sub  Analytics  Fulfillment
                     sub        sub
                |      |          |
                v      v          v
             Worker  Worker     Worker
```

Each subscription can usually be scaled independently, subject to the chosen broker's semantics and limits. If the email consumer is slow, analytics may still progress through its own subscription.

### Fan-out

**Fan-out** means one published event is delivered to multiple interested subscribers. It is useful when one business fact has several independent consequences.

Do not confuse fan-out with broadcasting every message to every worker in one competing-consumer group. A queue/work-queue pattern usually distributes each work item to one worker in the group, whereas pub/sub lets each independent subscription receive the event. Some products combine both ideas: workers can compete within each subscription.

## 5. Queue vs pub/sub vs event stream

These are related but not interchangeable terms.

| Pattern | Typical use | Typical delivery shape |
|---|---|---|
| Work queue / competing consumers | Process each job once as work, such as one image-analysis job | One worker in a consumer group handles a delivery; retries may cause redelivery |
| Pub/sub | Notify multiple independent systems about a fact | Each independent subscription can receive the event |
| Event stream / log | Retain ordered records for consumers to read or replay | Consumers may track offsets/positions; retention and replay depend on the platform |

A broker may support several patterns. Select based on required persistence, replay, ordering, throughput, fan-out, operations, and cost—not just the product name.

## 6. Design an event contract

A message should be clear, versioned, and safe to process. For example:

```json
{
  "event_id": "evt_01J9EXAMPLE",
  "event_type": "AnalysisRequested",
  "schema_version": 1,
  "occurred_at": "2026-10-23T16:00:00Z",
  "image_id": "img_8421",
  "request_id": "req_3190"
}
```

All IDs and timestamps above are illustrative.

Good event contracts should:
- use a unique event ID;
- define a stable event type and schema version;
- include only data consumers need;
- avoid embedding secrets or unnecessary personal data;
- define timestamp and identifier formats;
- document whether a field is optional and how changes remain compatible.

For privacy, consider sending an opaque image ID rather than the image bytes or a long-lived public URL. The consumer can fetch the authorized object from storage using a controlled access path. In the AI Image Detector project, external image processing must follow the established privacy decision: temporary processing and deletion after analysis, with no assumption that an external provider is approved until reviewed.

## 7. Delivery guarantees, duplicates, and idempotency

Messaging systems commonly offer different delivery behaviors. The exact guarantee depends on the broker, configuration, acknowledgement rules, and application integration.

- **At-most-once:** a message may be lost, but it is not redelivered under the defined delivery behavior.
- **At-least-once:** a message can be redelivered, so duplicates are possible.
- **Exactly-once:** a scoped guarantee that depends on the specific platform and processing boundary. Do not assume it means every external side effect happens once end-to-end.

A common failure sequence is:
1. A consumer receives an event.
2. It updates a database successfully.
3. It crashes before acknowledging the message.
4. The broker redelivers the event.
5. The consumer tries to apply the update again.

Design consumers to be **idempotent**: processing the same logical event again should not repeat the business effect. A common approach is to store the event ID in a processed-events table with a uniqueness constraint, and apply the business update in the same database transaction where possible. A simple in-memory “already seen” set is not enough for a multi-instance production service.

## 8. Ordering and eventual consistency

Do not assume all events are delivered or completed in global order. Multiple consumers can process messages concurrently, retries can delay one event, and different brokers provide different ordering scopes.

If order matters:
- define the business ordering key, such as `image_id` or `order_id`;
- choose a broker feature and configuration that supports the needed ordering scope;
- prevent conflicting work for the same key where necessary;
- test what happens when an earlier event fails;
- accept that stronger ordering may reduce parallelism.

Event-driven workflows often produce **eventual consistency**. For example, the upload record may exist while the analysis status remains `pending`; later, a consumer updates it to `completed` or `unavailable`. The UI should represent these states honestly rather than implying that processing has already finished.

## 9. Reliable publishing: the dual-write problem

Suppose the PHP API writes an upload record to MySQL and then publishes `AnalysisRequested`.

- If the database commit succeeds but publishing fails, the image record exists but no worker receives the event.
- If the event is published first and the database write then fails, a consumer may receive an event for a record that does not exist.

A common solution is the **transactional outbox pattern**:

1. In one MySQL transaction, save the upload/analysis record and an outbox row.
2. A publisher process reads unsent outbox rows and publishes the events.
3. After confirmed publication, mark the outbox rows as sent.
4. Make consumers idempotent because the publisher can crash after publishing but before marking the row sent.

The outbox avoids requiring a single atomic transaction spanning MySQL and the broker. It still requires retries, monitoring, and duplicate-safe consumers. The broker's publish confirmation and durability semantics must be checked for the selected product.

## 10. Failures, retries, and dead-letter handling

Define what happens when a consumer cannot process an event:
- Retry transient failures with a bounded policy and backoff.
- Do not retry permanent schema or validation errors indefinitely.
- After the configured retry limit, quarantine the event in a dead-letter queue (DLQ) or equivalent mechanism.
- Alert on DLQ growth and old messages.
- Keep enough diagnostic context to investigate without logging secrets or unnecessary personal data.
- Document how an operator can fix the cause and replay a message safely.

A queue can absorb a temporary burst, but it cannot fix a permanent capacity mismatch. If messages arrive faster than consumers process them, backlog and event age grow. Monitor publish rate, processing rate, queue depth, oldest-message age, redelivery count, and consumer failures.

## 11. Security and privacy

- Authenticate publishers and consumers; authorize access to each topic, subscription, or queue.
- Validate message schema, size, and expected event type.
- Treat message payloads as untrusted input.
- Protect credentials and encrypt network traffic.
- Avoid placing image bytes, secrets, or unnecessary personal data in messages.
- Apply retention and deletion rules to broker storage, logs, retries, and dead-letter queues.
- Use correlation IDs for tracing, but do not use them as authorization.
- Prevent unauthorized consumers from retrieving private images or results.

## 12. When should you use EDA?

Good candidates:
- work that can complete asynchronously;
- one business event that triggers several independent actions;
- traffic bursts that can be buffered;
- integration between independently deployed components;
- workflows where consumers need independent scaling.

Use direct request-response when the caller needs an immediate result, the interaction is simple, or a broker would add unjustified operational complexity. Hybrid systems are common: synchronous validation and acceptance, followed by asynchronous processing.

## 13. Practical exercise — AI Image Detector

Design the event flow after a user uploads an image.

Requirements:
- Validate the upload and save its metadata.
- Process analysis asynchronously.
- Update the result status.
- Allow notifications to be added later without changing the upload API.
- Keep image processing temporary and delete the image after analysis according to the project policy.
- Do not claim a provenance result when the external check is unavailable.

Answer:
1. Which checks must finish before the API returns, and which can run asynchronously?
2. What event type and fields would you publish?
3. Which consumers or subscriptions do you need?
4. What happens if MySQL saves the record but publishing fails?
5. How will a consumer handle duplicate delivery?
6. Does the UI need to handle eventual consistency? Which statuses should it show?
7. What happens when the analysis worker is down for 20 minutes?
8. What metrics and alerts would you add?
9. How will you prevent unauthorized consumers from retrieving an image?

**Stretch:** Draw the architecture and mark the database transaction, outbox publisher, broker, consumers, acknowledgements, retry path, and DLQ.

## 14. Interview practice

**Question:** What is the difference between event-driven architecture and pub/sub?

A strong answer: EDA is an architectural style in which components react to events; pub/sub is one messaging pattern commonly used to deliver events to multiple independent subscribers. EDA can also use queues, streams, and other patterns.

**Question:** Why can an event-driven system process the same event more than once?

A strong answer: a consumer may complete its side effect and crash before acknowledging the message, so the broker redelivers it. The consumer should be idempotent and use durable deduplication where appropriate.

**Question:** How do you avoid losing an event after saving a database record?

A strong answer: write the business record and an outbox event in the same database transaction, then publish the outbox asynchronously. Make consumers idempotent because publishing may be repeated.

## Short recap

- EDA is an architectural style; pub/sub is a messaging pattern often used within it.
- Events describe facts; commands request actions.
- Queues commonly distribute work among workers; pub/sub allows independent subscriptions to receive the same event.
- Delivery, ordering, persistence, and replay guarantees vary by product and configuration.
- At-least-once delivery requires duplicate-safe consumers.
- The transactional outbox reduces the risk of losing an event between a database commit and publication.
- Monitor backlog and event age, and design for failure, security, privacy, and recovery.

## Learning resources

1. [GeeksforGeeks — Event-Driven Architecture](https://www.geeksforgeeks.org/system-design/event-driven-architecture-system-design/) — currently reports “Last Updated: 21 Oct, 2026” (checked 23 October 2026); use for an introductory overview.
2. [GeeksforGeeks — Pub/Sub Architecture](https://www.geeksforgeeks.org/system-design/what-is-pub-sub/) — beginner-friendly overview of publishers, topics, subscriptions, and brokers.
3. [Google Cloud — What is Pub/Sub?](https://docs.cloud.google.com/pubsub/docs/overview) — official documentation on asynchronous messaging and service decoupling.
4. [Google Cloud — Subscription overview](https://docs.cloud.google.com/pubsub/docs/subscription-overview) — official details on subscriptions, delivery, and ordering. The service's defaults and guarantees are product-specific; do not generalize them to every broker.
5. [Apache Kafka — Design documentation](https://kafka.apache.org/documentation/#design) — official reference for Kafka's log, delivery semantics, and related design choices.
6. [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials) — official hands-on tutorials for queues and pub/sub-style messaging.
7. [Google SRE Books](https://sre.google/books/) — free reliability and operations reading; the books page currently marks “Site Reliability Engineering, 2nd Edition” and “AI Engineering Reliable Operations” as new, but shows no page update date.
8. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — resource directory, last updated 24 February 2026 on the page; it is not a dedicated EDA course.

**Resources checked:** 23 October 2026. The opened GeeksforGeeks System Design Tutorial page still reports “Last Updated: 5 Sep, 2026,” matching the repository's 20 October review; search snippets can show a conflicting older date, so no tutorial-page content change is inferred from that discrepancy. The EDA article reports 21 October 2026, but without an earlier article snapshot its exact text changes cannot be verified. No existing lesson was changed on the basis of an unverified source diff.
