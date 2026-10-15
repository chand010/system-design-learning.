# Day 3 — Latency and Throughput

## Learning goals
- Explain latency and throughput.
- Understand averages, percentiles, and queueing.
- Identify practical ways to improve response time.

## 1. Latency
**Latency** is the time taken for an operation or request to complete. For a web request, it may include network travel, load-balancer work, application processing, database queries, and response transfer.

Example: if a user clicks “Search” and results appear 250 milliseconds later, the observed request latency is about 250 ms.

## 2. Throughput
**Throughput** is how much work a system completes per unit of time, such as requests per second, messages per second, or transactions per minute.

Latency and throughput are related but different:
- Latency: how long one request takes.
- Throughput: how many requests the system completes over time.

A system can have low latency at light load but slow down when overloaded.

## 3. Average vs percentiles
An average can hide a poor experience. Imagine 99 requests take 100 ms and one request takes 5 seconds. Most users are fast, but a small group experiences a delay.

- **p50:** half of requests finish at or below this time.
- **p95:** 95% finish at or below this time.
- **p99:** 99% finish at or below this time.

Teams often track p95 and p99 to understand tail latency. Always clarify the time window and request population when comparing metrics.

## 4. Where latency comes from
- Network distance and congestion.
- Slow application code or lock contention.
- Expensive database queries or missing indexes.
- Cache misses.
- Calls to slow third-party services.
- Queueing caused by overloaded workers.
- Large responses or inefficient serialization.

## 5. Improving latency
1. Measure first; do not guess.
2. Break down latency by component using tracing and metrics.
3. Optimize slow queries and add appropriate indexes.
4. Cache frequently requested data when freshness rules allow.
5. Reduce unnecessary network round trips.
6. Use pagination and appropriately sized responses.
7. Set timeouts and limit retries to avoid overload.
8. Move slow, non-interactive work to background queues when appropriate.

## 6. A simple queueing intuition
When a service is near its maximum capacity, even a small increase in traffic can create a large queue and sharply increase response time. Adding capacity, controlling incoming work, or reducing processing time can help. Do not treat an average response time as proof that the system will handle peak traffic.

## Practical exercise
An endpoint has p50 latency of 80 ms, p95 of 500 ms, and p99 of 2 seconds.
1. What does p95 = 500 ms mean?
2. Give three possible causes of the gap between p50 and p99.
3. What evidence would you collect before changing the architecture?

## Suggested resources
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Google Cloud — Performance and latency concepts](https://cloud.google.com/architecture)

## Key takeaway
Measure both latency and throughput, and pay attention to tail latency. Instrumentation helps you fix the real bottleneck instead of guessing.
