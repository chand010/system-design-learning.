# Day 10 — Rate Limiter Design

A **rate limiter** controls how many requests a client can make in a period. It helps protect APIs and downstream services from overload, abuse, and unexpected costs.

Examples include limiting login attempts, API requests per key, or expensive operations per account. These numbers must be chosen for the actual product; there is no universal safe limit.

## 1. Requirements

Functional:
- Define a policy, such as 120 requests per minute per API key.
- Allow requests within the policy and reject or delay excess requests.
- Apply different rules by user, API key, IP, tenant, endpoint, or whole service.
- Tell clients when to retry when practical.

Non-functional:
- Add little latency to normal requests.
- Enforce policy across application instances.
- Bound state and memory use.
- Remain observable and handle dependency failures deliberately.

Clarify the identity and scope of a limit before choosing an algorithm.

## 2. Architecture

```text
Client
  |
Load Balancer / API Gateway
  |
Rate Limiter ------ Shared counter/state store
  |
  +-- over limit --> HTTP 429
  |
  +-- allowed -----> Application Service
                         |
                    Database / APIs
```

The limiter can live at an API gateway, in application middleware, or in a dedicated service. A local counter on each application server is not enough for a global limit: with four servers, a 100-request limit could accidentally become 400 if every server tracks its own counter independently. Use shared state or deliberately allocate quotas.

## 3. Common algorithms

### Fixed window
Count requests in fixed buckets, such as each calendar minute. It is simple and cheap, but a client can send its full allowance just before a boundary and again just after it.

### Sliding-window log
Store each accepted request timestamp. Remove timestamps outside the rolling period, count the remaining entries, then allow or reject. This is accurate but can consume substantial memory at high request volumes.

### Sliding-window counter
Combine the current and previous fixed-window counts, weighting the previous count by the part of its window that overlaps the rolling period. It uses much less state than a full log and reduces boundary bursts, but is an approximation.

### Token bucket
A bucket holds up to a fixed number of tokens. Tokens refill at a configured rate, and each request consumes one or more. If capacity is 10 tokens and refill is 5 tokens/second, a full bucket can allow a burst of 10 requests; after that, the long-run rate is controlled by the refill rate. This is a common general-purpose API policy.

### Leaky bucket
Incoming work is queued and processed at a steady rate. If the queue fills, new work is rejected. This smooths traffic rather than permitting the same burst behavior as a token bucket. State the exact implementation when describing a design, because terminology varies.

| Algorithm | State | Main trade-off |
|---|---|---|
| Fixed window | Counter + window | Cheap, but boundary bursts |
| Sliding log | Request timestamps | Accurate, but memory-heavy |
| Sliding counter | Two window counts | Efficient approximation |
| Token bucket | Tokens + last-refill time | Controlled bursts |
| Leaky bucket | Queue or drain state | Smooth output rate |

## 4. HTTP behavior

Use **429 Too Many Requests** when the policy rejects a request. A response can include a `Retry-After` header to tell the client how long to wait.

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 30

{"error":"rate_limit_exceeded","message":"Please retry later."}
```

The 30-second value is illustrative. Calculate it from the policy/state when possible, and teach clients not to retry in a tight loop.

## 5. Distributed implementation with Redis

Redis is often used as a shared state store so multiple app servers see the same counters. A conceptual fixed-window key might be `rate-limit:{api-key-hash}:{window-start}`.

Request flow:
1. Identify the caller.
2. Build a key from caller and policy.
3. Atomically increment the counter and set expiry.
4. Compare the count with the configured limit.
5. Continue if allowed; otherwise return 429.
6. Record metrics without logging secrets.

Redis supports `INCR` and `EXPIRE` for fixed-window counters and Lua scripts for atomic multi-step decisions. Do not assume separate commands are a safe transaction: concurrent requests and crashes can cause incorrect state or missed expiry. For a distributed token bucket, refill calculation and token deduction must also be atomic.

## 6. Capacity and reliability trade-offs

Suppose each of 50,000 active API keys has a limit of 100 requests per minute. The combined theoretical allowance is 5,000,000 requests/minute, or about 83,333 per second. This is a mathematical sum of quotas, not a traffic forecast. A global limit may still be needed to protect the service.

Ask:
- Can the shared store handle a check on every request?
- Is slight overshoot acceptable?
- Is the policy regional or global?
- What happens if Redis is slow or unavailable?

**Fail open** means allowing requests when the limiter state store fails; this can preserve availability but expose the backend to overload. **Fail closed** means rejecting requests; this protects sensitive systems but may block legitimate users. For a low-risk read endpoint, fail-open or a conservative local fallback may be acceptable. For login, OTP, or expensive payment operations, fail-closed behavior may be safer. Make the decision explicitly and monitor it.

## 7. Security and monitoring

- Combine account/API-key limits with IP and global limits where appropriate.
- Do not trust identity headers unless a trusted proxy sets them securely.
- Account for NAT and proxies; an IP-only rule can block many legitimate users together.
- Limit expensive endpoints more strictly.
- Keep key cardinality and retention bounded.
- Never put raw API keys or unnecessary personal data in keys or logs.
- Monitor allowed/rejected request rates, limiter latency, store errors, memory, and hot keys.

Google SRE practices encourage defining reliability targets and monitoring indicators, not just deploying a component and hoping it works.

## 8. Testing checklist

1. A request below the limit is allowed.
2. The request at the limit behaves as intended.
3. The next request returns 429.
4. Retry timing is useful.
5. Expiration/refill works.
6. Concurrent requests cannot all pass by racing on a counter.
7. Two application instances enforce the intended shared limit.
8. Different keys have independent state.
9. Store failure follows the documented policy.
10. Legitimate users behind the same NAT are not unfairly blocked.

## 9. Practical exercise

Design a limiter for a public API:
- 120 requests per minute per API key.
- Bursts of up to 20 requests are allowed.
- Four app instances run behind a load balancer.
- Rejected requests receive 429 and a retry hint.

Answer:
1. Which algorithm would you choose, and why?
2. What state would you store?
3. Where would you enforce the policy?
4. How would you make updates atomic?
5. What should happen if the shared store fails?
6. Which metrics and tests are essential?

### Interview structure
1. Clarify who/what is limited.
2. Establish accuracy, burst, latency, and availability requirements.
3. Compare algorithms and explain the trade-off.
4. Draw the request path and shared state.
5. Explain concurrency and distributed consistency.
6. Cover HTTP behavior, failure policy, security, metrics, and tests.

## Resources
- [GeeksforGeeks — Rate Limiting Algorithms](https://www.geeksforgeeks.org/system-design/rate-limiting-algorithms-system-design/) (reports last updated 11 August 2026).
- [GeeksforGeeks — Design a Rate Limiter API](https://www.geeksforgeeks.org/system-design/how-to-design-a-rate-limiter-api-learn-system-design/).
- [Redis official rate limiter documentation](https://redis.io/docs/latest/develop/use-cases/rate-limiter/).
- [IETF RFC 6585 — HTTP 429](https://www.rfc-editor.org/rfc/rfc6585.html).
- [Google SRE Books](https://sre.google/books/) and [Service Level Objectives](https://sre.google/sre-book/service-level-objectives/).
- [Codecademy](https://www.codecademy.com/) for programming practice; free access varies by course.

## Key takeaway

A rate limiter is a policy, algorithm, concurrency problem, HTTP behavior, and operational dependency. Choose the simplest algorithm that meets real requirements, then test bursts, concurrency, and failure.
