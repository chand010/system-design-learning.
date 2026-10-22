# Day 15 — Resilience: Timeouts, Retries, Circuit Breakers, and Bulkheads

**Learning objectives:** Explain how remote-call failures consume resources, design safe retry policies, distinguish retries from circuit breakers, and isolate failures so one dependency does not take down the whole service.

## 1. Why resilience patterns matter

Distributed systems communicate over networks. A service can be slow, overloaded, unreachable, or return a transient error. Even if each component usually works, a dependency failure can spread through the system.

Example: the AI Image Detector calls an external provenance API. The API becomes slow. PHP requests wait while holding workers and connections. More requests accumulate, the app becomes slower, clients retry, and the extra traffic makes the dependency even less healthy. This feedback loop can become a **cascading failure**.

These patterns have different jobs:
- **Timeout:** stop waiting too long for one operation.
- **Retry:** try again when a failure is likely to be temporary.
- **Backoff and jitter:** space retries out so they do not create a new traffic spike.
- **Circuit breaker:** stop calling a dependency that is repeatedly failing.
- **Bulkhead:** limit how much capacity one dependency or workload can consume.
- **Fallback/degradation:** provide a safe reduced response when the normal path is unavailable.

No single pattern replaces capacity planning, observability, testing, or a clear product-level failure policy.

## 2. Timeouts and deadlines

A **timeout** limits how long a client waits for an operation. Without one, requests can remain stuck while consuming workers, sockets, memory, and connection-pool slots.

Set timeouts on remote calls and database operations where supported. Distinguish:
- **Connection timeout:** maximum time to establish a connection.
- **Request/read timeout:** maximum time to wait for the operation or response.
- **Overall deadline:** the total time budget for the complete user request, including all downstream work.

Choose values from measured latency and the product's user-facing objective, not by guessing. A timeout that is too long holds resources during failure; one that is too short creates false timeouts and unnecessary retries.

### Deadline budgeting example

Suppose the API has a 2-second total response budget (illustrative only). Authentication and upload metadata take 300 ms, so the remaining downstream work cannot safely be allowed to consume the full 2 seconds. The service should reserve time for response generation and propagate a smaller remaining budget to dependencies.

A downstream service should not continue expensive work indefinitely after the caller has cancelled or its deadline has expired, unless that work is intentionally designed to continue (for example, a persisted asynchronous job).

## 3. Retries: useful, but dangerous

Retry only when:
1. the failure is plausibly transient;
2. another attempt could succeed;
3. the operation is safe to repeat, or has a reliable idempotency mechanism; and
4. there is still enough time in the overall deadline.

Common retry candidates can include temporary connection failures, some timeouts, throttling responses, and selected 5xx responses. Do not blindly retry malformed requests, authorization failures, or other permanent 4xx errors. Follow the specific service's contract; status codes alone are not a complete policy.

### The duplicate-side-effect problem

A payment API might process a charge, but the response is lost. The caller sees a timeout and cannot tell whether the charge happened. Retrying as a new operation could charge twice.

Use an idempotency key for the same logical operation when the API supports it. Reuse the same key for retries of that operation; do not generate a new key for every attempt. If the outcome remains uncertain, query or reconcile the operation rather than blindly creating another one.

## 4. Exponential backoff and jitter

Immediate retries can overwhelm an already unhealthy service. **Exponential backoff** increases the delay between attempts. A simplified capped formula is:

delay = min(max_delay, base_delay × 2^(attempt_number − 1))

This is a conceptual formula, not a universal production configuration. **Jitter** adds randomness to retry timing so many clients do not all retry together.

Illustrative schedule with a 200 ms base and 2-second cap, before jitter:
- Attempt 1 fails → wait about 200 ms.
- Attempt 2 fails → wait about 400 ms.
- Attempt 3 fails → wait about 800 ms.
- Later waits increase but are capped at 2 seconds.

Also cap the number of attempts or total retry time. Retry policies should have a budget; otherwise, retries can multiply traffic during an outage.

### Avoid retry multiplication

If three layers each make up to three retries after the original attempt, one user request could cause up to 4 × 4 × 4 = 64 downstream attempts in a worst-case nested retry design. Prefer a clearly owned retry policy, and avoid retries at every layer unless their combined behavior has been deliberately designed.

## 5. Circuit breaker

A circuit breaker protects a caller from repeatedly invoking a dependency that is likely to fail. It observes outcomes and temporarily stops calls after a configured failure or slow-call threshold.

Common states:

- **CLOSED:** calls flow normally; the breaker records outcomes.
- **OPEN:** calls fail fast or use a safe fallback instead of calling the dependency.
- **HALF-OPEN:** after a wait period, a small number of trial calls test whether the dependency has recovered. Success can close the breaker; further failures reopen it.

The exact thresholds, sampling window, wait period, and treatment of exceptions must be configured for the workload. A circuit breaker does not fix the dependency, and it is not the same as a timeout or retry.

Example: if the provenance API has sustained timeouts, the breaker opens so the PHP application stops wasting workers waiting for it. After a cooling period, a limited number of probes check recovery.

A fallback must be honest. If provenance is unavailable, the app might mark the result “provenance check unavailable” and continue only if product policy allows it. It must not silently claim the image is authentic.

## 6. Bulkheads and concurrency limits

A **bulkhead** isolates resources so a slow dependency cannot consume all the capacity needed by unrelated work. The name comes from ship compartments: damage in one compartment should not flood the whole ship.

Examples:
- Separate connection pools for the database and an external API.
- Limit the number of concurrent provenance checks.
- Use a dedicated worker pool for expensive image analysis.
- Bound queues and reject or defer excess work rather than allowing unbounded growth.

A circuit breaker limits calls when failure rates are high; a bulkhead limits concurrent resource consumption. They complement each other. A circuit breaker alone does not necessarily limit the number of calls already in progress.

## 7. Combining the patterns safely

For an external provenance API:

1. Set an overall deadline for the user request or job.
2. Set a smaller timeout for the external call.
3. Retry only selected transient errors, with capped exponential backoff and jitter.
4. Use an idempotency key if the operation has side effects and the API supports it.
5. Limit concurrent calls with a bulkhead.
6. Open the circuit after a configured pattern of failures or slow calls.
7. Use a policy-approved fallback or return an explicit unavailable state.
8. Record metrics and alert on sustained failures.
9. Test recovery as well as failure.

For the AI Image Detector, the choice between blocking a user request and creating a background job depends on product requirements. If external provenance is optional and delayed results are acceptable, asynchronous processing can be safer than holding a PHP request open. If provenance is mandatory for a security decision, fail safely rather than inventing a result.

## 8. Monitoring and testing

Monitor:
- request latency percentiles and timeout rate;
- retries per logical request and retry-budget exhaustion;
- downstream error and throttling rates;
- circuit-breaker state transitions and rejected calls;
- in-flight requests, connection-pool use, and worker saturation;
- fallback rate and user-visible failures.

Test scenarios such as a slow dependency, connection refusal, throttling, lost responses after a side effect, and recovery after an outage. Verify that retry loops stop, deadlines propagate, and the system does not overload the dependency while it recovers.

Do not alert on every individual transient error. Alert on sustained user impact, exhausted budgets, unhealthy dependencies, or unusual breaker behavior.

## 9. Practical exercise

Design the call from the AI Image Detector to an external provenance API.

Write down:
1. What is the total time budget?
2. Which failures are retryable and which are permanent?
3. How many attempts are allowed, and how do backoff and jitter work?
4. How will the app handle an unknown outcome after a timeout?
5. When should the circuit open and how will recovery be tested?
6. What concurrency limit protects PHP workers and the external API?
7. What should the user see if provenance is unavailable?
8. Which metrics would show that the policy is working?

**Stretch:** Draw the call path and mark the timeout, retry decision, circuit breaker, bulkhead, and fallback. Explain what happens when the dependency is slow for ten minutes.

## 10. Interview practice

**Question:** Why are unlimited retries dangerous?

A strong answer: they increase traffic precisely when a dependency may be overloaded, hold resources longer, and can create a retry storm or cascading failure. Use bounded retries, exponential backoff, jitter, an overall deadline, and idempotency for side effects.

**Question:** What is the difference between a circuit breaker and a bulkhead?

A strong answer: the circuit breaker stops new calls when a dependency is unhealthy; the bulkhead limits how much concurrency or resource capacity that dependency can consume. Use both when needed.

## Short recap

- Timeouts bound waiting; deadlines bound the total operation.
- Retry only transient failures when repeating the operation is safe.
- Backoff and jitter spread retries; limits and retry budgets prevent storms.
- Circuit breakers fail fast during repeated failures and probe recovery.
- Bulkheads isolate resource pools and concurrency.
- Fallbacks must preserve correctness and communicate uncertainty.
- Measure and test failure behavior, not only the healthy path.

## Learning resources

1. [AWS Builders' Library — Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter) — official article; published 12 June 2026 and last modified 15 June 2026 on the current page (checked 22 October 2026).
2. [Google SRE Book — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) — free online chapter covering overload, retries, deadlines, and retry budgets.
3. [Microsoft Azure Architecture Center — Best practices for transient fault handling](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transient-faults) — official guidance on retry eligibility and circuit breakers.
4. [Resilience4j — CircuitBreaker documentation](https://resilience4j.readme.io/docs/circuitbreaker) — implementation reference for CLOSED, OPEN, and HALF_OPEN states and their configuration.
5. [Google SRE Books](https://sre.google/books/) — broader reliability, monitoring, and secure-systems material.
6. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — a directory for courses, documentation, and practice resources; not a dedicated circuit-breaker course.

**Resources checked:** 22 October 2026. Source update dates are reported only where visible; a check date does not prove that content changed that day.
