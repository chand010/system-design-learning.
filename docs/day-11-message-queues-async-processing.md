# Day 11 — Message Queues and Asynchronous Processing

**Goal:** Learn producers, queues, consumers, acknowledgements, retries, dead-letter queues (DLQs), delivery guarantees, and when asynchronous processing is useful.

## 1. Why use a queue?

In the AI Image Detector, checking an upload may be quick but running image detection may take several seconds. If PHP waits for detection inside every upload request, users wait and web workers remain busy.

A message queue buffers work for a separate worker. It separates when work is requested from when it is completed. It can absorb temporary traffic spikes, but it does not create unlimited processing capacity.

## 2. Components and flow

- **Producer:** creates a message, e.g. the PHP upload handler.
- **Queue/broker:** holds messages and delivers them to consumers.
- **Consumer/worker:** processes the job.
- **Acknowledgement:** confirms successful processing of a delivery.
- **Retry policy:** controls later attempts after transient failures.
- **DLQ:** isolates messages that repeatedly fail.

Architecture:

Client → PHP Upload API → Queue → Detection Worker → Database result
Client ← 202 Accepted + job ID; later checks GET /jobs/{job_id}

A work queue usually distributes each job to one worker. Pub/sub can fan an event out to multiple subscribers. Exact ordering and delivery behavior depend on the product and configuration.

## 3. Synchronous vs asynchronous

**Synchronous:** the API waits for the result. It is simple, but slow work can cause timeouts and occupy web workers.

**Asynchronous:** the API accepts and queues the job, then returns before it finishes. The client checks status or receives a notification. This improves responsiveness and lets workers scale separately, but requires job states, retries, monitoring, and handling eventual completion.

An HTTP 202 Accepted response means “accepted for processing,” not “completed.”

Use a queue when work is long-running, variable, or can safely finish later. Do not add one without considering its extra complexity.

## 4. AI Image Detector example

1. PHP validates the upload and saves it in controlled storage.
2. The app creates a job record with status queued.
3. The producer sends a small message with job ID and storage reference.
4. A worker marks the job processing, runs detection, and saves the result.
5. The worker marks the job completed or failed.
6. The client checks an authorized status endpoint.

Example message fields: message_id, type, schema_version, job_id, image_ref, correlation_id. Prefer a reference to a large image rather than embedding the image in the message. Never put passwords, API keys, or unnecessary personal data in a message. Validate the payload and referenced object in the worker.

For this project, images should only be retained when the user explicitly opts in, with the agreed seven-day retention period. Cleanup must account for stored images, metadata, and queued/retried jobs.

## 5. Delivery guarantees and duplicate work

- **At-most-once:** a message is delivered zero or one time; work may be lost.
- **At-least-once:** messages may be retried, so duplicates are possible.
- **Exactly-once:** some systems offer narrowly defined guarantees, but end-to-end exactly-once effects across a queue, database, and external APIs are difficult. Design for duplicates unless stronger end-to-end guarantees are proven.

A worker might save a result and then crash before acknowledging the message. The broker may deliver it again.

Make the handler **idempotent**: repeating the same job should not create a second result or repeat an external side effect. Use stable job IDs, database transactions or conditional updates, and unique constraints. A status check alone does not prevent concurrent duplicate processing.

## 6. Retries, DLQs, ordering, and overload

- Retry failures that may be temporary; use bounded attempts, exponential backoff, and jitter.
- Send persistently failing messages to a DLQ, alert on DLQ growth, and provide a controlled replay process.
- Do not assume global FIFO ordering. Parallel workers and retries may change completion order.
- If messages arrive faster than workers can finish them, queue depth and wait time grow. A queue buffers bursts; it cannot fix a permanent capacity deficit.
- More workers help only until CPU, database, model runtime, or another dependency becomes the bottleneck.

**Capacity example:** 120 jobs/minute = 2 jobs/second. If each worker takes 2 seconds/job, one worker handles about 0.5 jobs/second under this simplified estimate. Four workers provide a theoretical 2 jobs/second with no headroom; real capacity must be measured.

## 7. Reliability and security checklist

- Define job states: queued, processing, completed, failed.
- Plan for database writes succeeding while message publishing fails (or vice versa). Later, study the transactional outbox pattern.
- Configure acknowledgements, bounded retries, backoff, and DLQ handling.
- Make consumers safe against duplicates.
- Limit message size and pass references to large files.
- Restrict producer/consumer permissions, use TLS and suitable encryption, validate messages, and protect credentials.
- Monitor publish failures, queue depth, oldest-message age, processing time, retries, worker saturation, and DLQ size.
- Define retention/deletion for messages and referenced data.

## 8. Practical exercise

Design asynchronous detection. Answer:
1. What must happen before the API returns 202?
2. What belongs in the message, and what should stay out?
3. How does the client check status?
4. What if a worker crashes after saving the result but before acknowledgement?
5. How will you prevent duplicate results?
6. When should a message go to the DLQ?
7. Which three metrics would you alert on first?

**Interview practice:** Why use a queue rather than call a slow service directly? Explain decoupling, faster responses, buffering, independent worker scaling, and the complexity of retries, duplicates, ordering, and monitoring.

## Short recap

Producers publish; queues buffer; consumers process. Async processing improves responsiveness but adds complexity. Acknowledgements and retries aid recovery, at-least-once delivery can create duplicates, and idempotency makes redelivery safer. Monitor queue depth/age, failures, processing time, and DLQ growth.

## Learning resources

1. [GeeksforGeeks — Message Queues in System Design](https://www.geeksforgeeks.org/system-design/message-queues-system-design/) — page reports Last Updated: 21 Oct 2026 (checked 22 Oct 2026).
2. [RabbitMQ — Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms) — official acknowledgements, redelivery, and publisher confirms guide.
3. [AWS — Using Dead-Letter Queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html).
4. [Google SRE Books](https://sre.google/books/) — free online reliability resources.
5. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — course/docs/article directory, not a message-queue-specific course.

**Resources checked:** 22 October 2026. A check date alone does not prove that source content changed that day.
