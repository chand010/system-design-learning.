# Day 12 — Consistency, Eventual Consistency, and Distributed Workflows

Distributed systems replicate data and split business processes across services. This can improve scale and resilience, but raises two questions: when will replicas show a write, and how should a workflow recover when only some steps succeed?

This lesson covers strong and eventual consistency, replication lag, business invariants, sagas, idempotency, and workflow monitoring.
