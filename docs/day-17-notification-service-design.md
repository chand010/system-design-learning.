# Day 17 — Notification Service Design

**Learning objectives:** Gather requirements, draw a notification architecture, model notification state, respect user preferences and consent, design reliable multi-channel delivery, handle retries and provider quotas, and monitor end-to-end outcomes.

## 1. Start with a real problem

Imagine an online shop needs to notify a customer when:
- an order is accepted;
- a payment succeeds;
- a parcel is dispatched;
- a password reset is requested.

The shop may send email, SMS, push notifications, or in-app messages. Calling each provider directly from the order API creates tight coupling: a slow SMS provider could delay checkout, and adding a new channel would require changing several business services.

A **notification service** centralizes notification policy and delivery. Business services request a notification or publish a business event; the notification system validates it, checks preferences and consent, selects a channel, queues delivery, calls a provider, and records status.

Do not confuse "the provider accepted the request" with "the person read the message." Delivery is a lifecycle, not a single boolean.

## 2. Define requirements before drawing boxes

### Functional requirements

1. Accept notification requests from trusted application services.
2. Support channels such as email, SMS, push, and in-app notifications.
3. Use templates with variables and localization.
4. Respect opt-outs, user preferences, quiet hours, and legal/consent requirements.
5. Prioritize time-sensitive messages over low-priority campaigns.
6. Retry temporary failures and track delivery status.
7. Prevent duplicate sends when the same logical request is submitted more than once.
8. Support scheduling, cancellation where feasible, and bulk campaigns with safeguards.

### Non-functional requirements

- **Availability:** business operations should not fail simply because a non-critical notification provider is down.
- **Latency:** critical messages need a defined end-to-end delivery objective; bulk campaigns can tolerate longer delays.
- **Durability:** accepted requests should not disappear silently.
- **Scalability:** queues and workers should handle bursts.
- **Privacy and security:** only authorized services can request messages, and sensitive content is minimized.
- **Cost control:** provider quotas, per-tenant limits, and campaign budgets are enforced.
- **Observability:** operators can distinguish queued, attempted, accepted, delivered, failed, and suppressed messages.

Write down which guarantees matter for each notification type. An order receipt, promotional message, and password-reset code do not necessarily need the same priority or retry policy.

## 3. High-level architecture

```text
Business Services
 (orders, accounts, billing)
          |
          | authenticated request or business event
          v
   Notification API
          |
          v
 Authentication + Validation
          |
          v
 Preferences / Consent / Policy
          |
          v
 Template + Localization
          |
          v
 Durable notification record + outbox
          |
          v
       Message Broker
       /     |      \
      v      v       v
 Critical  Standard  Bulk queues
   worker   worker   workers
      |       |        |
      +-------+--------+
              |
              v
       Channel adapters
       /      |       \
    Email     SMS     Push
   provider provider provider
       |       |       |
       +-------+-------+
              |
              v
       Provider callbacks
              |
              v
       Status/metrics store
```

This is a logical design, not a requirement to deploy every box as a separate microservice. A beginner-sized application can start as a modular service plus a durable queue and worker. Split components when scale, team boundaries, or reliability requirements justify the operational cost.

### Responsibilities

- **Notification API:** authenticates the caller, validates the request, applies request-level quotas, and returns a notification ID.
- **Preference/policy check:** determines whether the user can receive this category on the requested channel now. Critical transactional messages still need a documented policy; they must not automatically bypass every preference or legal restriction.
- **Template renderer:** selects a versioned template and safely substitutes variables.
- **Durable store/outbox:** records the request and the intent to enqueue it atomically when using the transactional outbox pattern.
- **Broker/queues:** buffer work and separate priority classes.
- **Workers:** claim jobs, re-check time-sensitive policy if required, enforce provider limits, call the provider, and record outcomes.
- **Channel adapter:** presents a stable internal interface while handling provider-specific request and response formats.
- **Callback handler:** verifies provider callbacks, deduplicates them, and updates message status.
- **Status store:** supports support teams, delivery metrics, and safe investigation.

## 4. Request and API design

A possible request is:

```json
{
  "idempotency_key": "order-8421-confirmation-v1",
  "recipient_id": "user-294",
  "notification_type": "order_confirmation",
  "channels": ["email"],
  "template_id": "order-confirmation",
  "template_version": 3,
  "data": {
    "order_id": "order-8421"
  },
  "priority": "normal"
}
```

All identifiers are illustrative. Prefer an internal recipient ID over accepting an arbitrary email address or phone number from any caller. The service can resolve contact details through an authorized user-data path.

A successful API response might be `202 Accepted` with a `notification_id` and a status URL. This means the request was accepted for processing, not that it was delivered.

Example response:

```json
{
  "notification_id": "ntf-10081",
  "status": "queued",
  "status_url": "/notifications/ntf-10081"
}
```

Use an idempotency key scoped to the authenticated caller and the intended logical notification. Store it with a uniqueness constraint. If the same request is retried after a timeout, return the existing notification record rather than creating a second logical send. Define what happens if a caller reuses a key with a different payload; rejecting it is often safer than silently changing the original operation.

Avoid putting full message bodies, passwords, access tokens, or unnecessary personal data in queue messages and logs. A message can often carry a notification ID and retrieve the approved content through an authorized path.

## 5. Data model and status lifecycle

A minimal relational model might include:

- **notifications:** ID, tenant/caller, recipient ID, notification type, priority, status, created time, scheduled time, expiry time, idempotency key, and correlation ID.
- **notification_attempts:** notification ID, channel, provider, attempt number, provider message ID, attempt time, result category, and retry-after time.
- **preferences:** recipient ID, notification category, channel, opt-in/opt-out state, quiet hours, and update time.
- **templates:** template ID, version, locale, channel, content, and publication status.
- **outbox:** event ID, event payload or reference, created time, and publication status.
- **processed_callbacks:** provider callback/event ID and processing status, protected by a unique constraint where the provider supplies a stable ID.

Keep only the data needed for delivery, support, auditing, and legal requirements. Define retention and deletion rules for message bodies, provider payloads, callbacks, and logs.

A useful lifecycle is:

```text
ACCEPTED -> QUEUED -> SENDING -> PROVIDER_ACCEPTED
                 |       |              |
                 |       +-> RETRY_WAIT |
                 |                      +-> DELIVERED (if confirmed)
                 +-> SUPPRESSED
                         |
                         +-> FAILED / EXPIRED / CANCELLED
```

The exact states should be defined carefully. For example, a provider's `accepted` or `sent` status does not necessarily prove that the recipient's device or mailbox received the message. Provider callbacks can arrive late, more than once, or out of order, so status updates should be idempotent and must not move a terminal state backwards because of an old callback.

## 6. Preferences, consent, priority, and quiet hours

Preferences are business rules, not merely a user-interface setting.

Before sending:
1. Identify the notification category (transactional, security, or marketing).
2. Check the user's current channel preference and consent/opt-out state.
3. Apply quiet hours and local-time rules where applicable.
4. Confirm that the caller is authorized to send this category.
5. Re-check policy at dispatch time if a queued notification could become ineligible while waiting.

A notification can become stale. A delivery worker should check its expiry time and suppress obsolete messages instead of sending them hours later.

Separate priority classes when necessary. For example, a password-reset message may need faster processing than a bulk promotional campaign. Do not let an unbounded bulk queue starve time-sensitive traffic. Apply per-tenant and per-recipient rate limits so one customer or campaign cannot overwhelm the system or spam recipients.

## 7. Provider adapters and rate limits

Define a provider-neutral interface such as:

- `send(message)`
- `get_status(provider_message_id)` when supported
- `verify_callback(request)`
- `classify_error(response)`

Each adapter maps provider responses into a small internal set of outcomes, for example:
- accepted by provider;
- retryable transient error;
- throttled / quota exceeded;
- permanent invalid-recipient or request error;
- unknown outcome (request may have succeeded, but response was lost).

Provider limits are real design constraints. They may be scoped by account, region, sender, destination, or channel. Use a rate limiter and bounded concurrency per provider, not just a global limit. On throttling, honor provider-specific retry guidance when supplied; otherwise use bounded exponential backoff with jitter. Do not retry a permanent invalid-address or invalid-payload error indefinitely.

A timeout after submitting a send request creates uncertainty: the provider may have accepted the message even though your worker did not receive the response. If the provider supports an idempotency token, use it. If not, use the provider's message lookup/callback capabilities and define a cautious reconciliation policy; do not assume that blindly repeating the request guarantees one send.

## 8. Retries, dead letters, and duplicate safety

Use a bounded retry policy:
- classify errors before retrying;
- set a maximum attempt count and total retry window;
- apply backoff and jitter;
- honor a valid provider `Retry-After` or equivalent instruction;
- expire messages that are no longer useful;
- move exhausted or repeatedly failing messages to a dead-letter queue (DLQ) or quarantine store;
- alert on DLQ growth and old messages;
- provide a controlled, auditable replay process.

At-least-once queue delivery means a job can run more than once. Use a stable notification ID and attempt IDs, unique constraints, and conditional state transitions. Idempotency can prevent duplicate logical records, but it cannot guarantee that an external provider sends exactly one message unless the provider offers a suitable guarantee.

Separate two ideas:
- **Request deduplication:** prevent the same logical request from creating multiple notification jobs.
- **Delivery deduplication:** prevent retries or duplicate workers from causing duplicate provider sends, to the extent the provider API supports it.

## 9. Delivery callbacks and status tracking

A provider callback is another external request. Treat it as untrusted until authenticated.

- Validate provider signatures using the provider's documented SDK or verification method.
- Validate the expected message/provider IDs.
- Deduplicate repeated callback events.
- Expect callbacks to arrive out of order.
- Store enough information to investigate delivery without retaining unnecessary message content.
- Return the provider's required success response promptly, then process heavier work asynchronously if appropriate.

Track the full lifecycle: accepted by your API, queued, attempted, accepted by provider, delivered/undelivered when the channel provides that status, opened/read when available, suppressed, expired, and failed. These are not equivalent outcomes. For example, email provider acceptance does not guarantee that a recipient's inbox displays the email.

## 10. Scaling and reliability

Start by measuring the workload.

Illustrative capacity exercise: if the system receives 600,000 notification requests per hour, the average arrival rate is:

`600,000 / 3,600 ≈ 167 requests/second`

This average hides bursts. A campaign might create a much higher short-lived peak, and one request may fan out into multiple channel deliveries. Estimate work per channel and expected retries, then size queues, worker concurrency, database connections, and provider limits accordingly.

Key design choices:
- scale workers horizontally while ensuring one job cannot be processed concurrently without protection;
- isolate critical, normal, and bulk queues;
- bound queue growth and apply backpressure;
- use a database index that matches the worker's pending-job query;
- partition high-volume tables only when measurements justify it;
- use a transactional outbox to avoid losing the intent to enqueue after a database commit;
- define behavior when the queue or provider is unavailable;
- protect the notification API from abuse with authentication, quotas, and rate limits.

A queue absorbs bursts; it does not create infinite capacity. If arrivals continuously exceed processing capacity, the backlog and message age will grow. Define a maximum acceptable delay and a policy for pausing, shedding, or expiring low-priority traffic.

## 11. Observability and SLOs

Measure outcomes, not just whether the API returned HTTP 200.

Useful metrics:
- API acceptance rate and latency;
- queue depth and age of oldest queued message;
- enqueue-to-provider-attempt latency;
- provider acceptance, delivery, bounce, failure, and throttling rates;
- retries per notification and DLQ size;
- suppression and opt-out rates;
- per-channel and per-tenant volume/cost;
- callback verification failures;
- percentage of critical notifications delivered within the defined target, where delivery status is available.

Define service-level indicators (SLIs) and objectives (SLOs) that match the business need. For example, a team might define an objective that 99% of eligible password-reset notifications are accepted by a provider within 10 seconds. That is an illustrative target and is not the same as promising delivery to the user's device within 10 seconds.

Use correlation IDs across API, database/outbox, queue, worker, and provider attempt logs. Avoid logging full message content or contact details unnecessarily.

## 12. Security and privacy checklist

- Authenticate and authorize calling services.
- Enforce tenant isolation and per-tenant quotas.
- Validate notification type, template variables, payload size, and recipient references.
- Prevent template injection and unsafe HTML content.
- Keep provider credentials in a secrets manager or protected runtime configuration.
- Verify webhook signatures and defend against replay.
- Encrypt data in transit and protect stored contact information.
- Apply opt-out, consent, retention, and deletion requirements.
- Restrict access to message bodies, delivery logs, and dead-letter queues.
- Avoid putting secrets or unnecessary personal data into queue messages.
- Audit administrative changes to templates, provider settings, and user preferences.

## 13. Practical exercise

Design a notification service for an online shop with email, SMS, and push notifications.

Requirements:
- An order-confirmation request must return quickly.
- Password-reset messages should have high priority.
- Promotional messages must respect opt-outs and quiet hours.
- A provider can throttle requests or become unavailable.
- Duplicate API requests and repeated callbacks must not corrupt status.
- Operators need to know whether messages are queued, delivered, failed, or suppressed.

Answer:
1. What are the functional and non-functional requirements?
2. Draw the architecture and data flow.
3. Which tables and unique constraints are required?
4. What should the API return when a notification is queued?
5. How do you prioritize password-reset messages over bulk campaigns?
6. How do you handle provider throttling and timeouts?
7. What happens if the provider accepted a message but your worker timed out?
8. How do you authenticate and deduplicate delivery callbacks?
9. Which metrics and alerts would you define?
10. What personal data would you avoid placing in messages, logs, and queues?

**Stretch:** Estimate capacity for 600,000 requests per hour, then explain how a 10x campaign burst affects queue depth, provider quotas, and delivery latency.

## 14. Interview practice

**Question: Why should notification sending usually be asynchronous?**

A strong answer: sending is external I/O and can be slow or throttled. A durable queue lets the business API acknowledge accepted work quickly, isolate provider failures, and scale workers independently. The API must still persist the request reliably and expose a truthful status.

**Question: How do you prevent duplicate notifications?**

A strong answer: use an idempotency key for request deduplication, durable unique constraints, safe state transitions, and provider idempotency support when available. Delivery callbacks must also be authenticated and deduplicated. End-to-end exactly-once sending cannot be assumed.

**Question: What happens when an SMS provider starts throttling?**

A strong answer: classify the provider error, stop sending above the applicable quota, honor retry guidance, use bounded backoff with jitter, and monitor queue age and provider error rates. Keep critical traffic isolated from bulk traffic.

**Question: Does a provider's HTTP success mean the user received the notification?**

A strong answer: no. It may mean the provider accepted the request. Delivery status depends on the channel and provider callbacks or status APIs. The system should represent each state accurately.

## Short recap

- Gather requirements before selecting components.
- Separate API acceptance from external delivery.
- Use durable jobs, request idempotency, and safe worker state transitions.
- Check preferences and consent; prioritize critical messages and limit bulk traffic.
- Rate-limit per provider and retry transient failures with bounded backoff.
- Treat timeouts as uncertain outcomes and callbacks as untrusted, possibly duplicated and out of order.
- Monitor queue age, delivery outcomes, retries, DLQs, cost, and critical-message SLOs.
- Minimize personal data and define retention and deletion rules.

## Learning resources

1. [GeeksforGeeks — Design Notification Services](https://www.geeksforgeeks.org/system-design/design-notification-services-system-design/) — introductory architecture, requirements, priority, preferences, and bulk notifications; page reports last updated 18 September 2023.
2. [AWS — Managing Amazon SES sending limits](https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas.html) — official email quotas, rate limits, and throttling considerations.
3. [AWS — Amazon SES delivery problems](https://docs.aws.amazon.com/ses/latest/dg/troubleshoot-delivery.html) — distinguishing delay, bounce, and delivery problems.
4. [Twilio — Track outbound message status](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status) — status callbacks and message lifecycle. The page notes that callbacks may arrive out of order and recommends documented signature validation.
5. [Twilio — Best practices for delivery-status logging](https://www.twilio.com/docs/messaging/guides/outbound-message-logging) — storing provider IDs and status history.
6. [Firebase — Understanding Cloud Messaging delivery](https://firebase.google.com/docs/cloud-messaging/understand-delivery) — delivery metrics and their limitations.
7. [Google SRE Books](https://sre.google/books/) — free reliability and operations references, including the Site Reliability Workbook and Building Secure & Reliable Systems.
8. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — general resource directory, not a dedicated notification-service course.

## Resource monitoring — checked 24 October 2026

- [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) currently reports **5 September 2026**, matching the 20 October repository review. A search result showed an older date, but opening the page confirmed 5 September; no content change is inferred from the conflicting snippet.
- [GeeksforGeeks Design Notification Services](https://www.geeksforgeeks.org/system-design/design-notification-services-system-design/) reports **18 September 2023**. Its core topics align with this lesson, but no earlier saved snapshot was available to verify a substantive change.
- [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) reports **24 February 2026**, unchanged from the repository's previous review.
- [Google SRE Books](https://sre.google/books/) currently marks “Site Reliability Engineering, 2nd Edition” and “AI Engineering Reliable Operations” as new; the page does not show an update date, and the previous repository review already recorded these labels.
- AWS SES, Twilio messaging, and Firebase Cloud Messaging documentation were checked for current technical guidance on quotas, status callbacks, delivery outcomes, and monitoring. The opened pages do not establish a new source update date or a verified change from the prior review.

**Monitoring outcome:** No verified material source change affecting an existing lesson was detected. The resource changelog was intentionally left unchanged; source-date discrepancies and current documentation are not evidence of a substantive historical change by themselves.
