# Day 4 — Performance and Bottlenecks

## Learning goals
- Recognize common bottlenecks.
- Understand CPU, memory, disk, network, and database limits.
- Follow a measurement-first troubleshooting process.

## 1. What is a bottleneck?
A bottleneck is the component that limits the performance or capacity of the overall system. Adding capacity somewhere else may not help if the constrained component remains unchanged.

Example: four application servers may still be slow if every request waits on one overloaded database.

## 2. Common bottlenecks
- **CPU:** expensive calculations, inefficient loops, too much serialization.
- **Memory:** memory leaks, oversized caches, too many in-memory objects.
- **Disk/storage:** slow reads/writes, saturated I/O, inefficient file operations.
- **Network:** limited bandwidth, high latency, packet loss.
- **Database:** missing indexes, lock contention, too many connections, slow queries.
- **External dependency:** a third-party API is slow or unavailable.
- **Concurrency:** too few workers, thread contention, shared locks, exhausted connection pools.

## 3. A systematic troubleshooting process
1. Define the symptom: errors, slow responses, high cost, or reduced throughput.
2. Check when it started and what changed.
3. Review metrics: request rate, errors, latency percentiles, CPU, memory, disk, network, database connections.
4. Use logs and distributed traces to find where time is spent.
5. Form one testable hypothesis.
6. Make a controlled change and compare before/after measurements.
7. Document the result and add monitoring or a regression test.

## 4. Useful signals
- High CPU plus slow requests may indicate compute saturation, but verify with profiling.
- Rising memory over time may indicate a leak or an unbounded cache.
- High database query latency may point to query plans, locks, storage, or connection exhaustion.
- Low CPU with slow requests can mean the app is waiting on a network, database, lock, or queue.
- Increasing queue depth means work is arriving faster than it is being completed.

A single metric rarely proves the cause. Correlate several signals over the same time window.

## 5. Safe performance improvements
- Fix inefficient queries and inspect query plans.
- Add caching only where correctness and freshness allow.
- Batch repeated operations where useful.
- Apply timeouts and bounded retries to dependencies.
- Limit payload sizes and paginate large results.
- Use load tests that resemble expected real traffic.
- Keep capacity headroom and test failure scenarios.

Avoid blindly increasing timeouts, retrying every failure, or adding servers without identifying the limiting component.

## Practical exercise
An API's p95 latency increases from 200 ms to 1.8 seconds during peak hours. CPU is 35%, memory is stable, and database query latency rises from 40 ms to 1.4 seconds.
1. What is your leading hypothesis?
2. Which database metrics and query evidence would you inspect?
3. Why would adding more application servers not necessarily solve the issue?

## Suggested resources
- [Google SRE Book — Identifying and Managing Risks](https://sre.google/sre-book/monitoring-distributed-systems/)
- [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

## Key takeaway
Observe, form a hypothesis, test, and measure again. Performance engineering is an evidence-based process.
