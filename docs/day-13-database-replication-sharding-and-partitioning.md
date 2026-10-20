# Day 13 — Database Replication, Sharding, and Partitioning

## Learning objectives
By the end, explain replication vs partitioning vs sharding; compare sync and async replication; choose a shard key; and discuss lag, failover, hot shards, and recovery.

## 1. Find the bottleneck first
A database may be limited by CPU, memory, storage I/O, connection capacity, query design, or write throughput. Measure query latency, query plans, read/write mix, growth, and peak load before changing architecture. Often the sensible progression is query/index tuning, vertical scaling where economical, caching where suitable, replicas for read-heavy traffic, partitioning for data management/query locality, and sharding only when a single instance cannot meet measured needs.

## 2. Replication: copy data
Replication maintains copies of data on other database nodes. A common primary-replica topology sends writes to a primary and replicates changes to one or more replicas; suitable reads may be served by replicas.

Benefits include read scaling, additional copies for recovery, and potentially improved read availability. Replication is not a backup: accidental deletion or corruption can replicate too, so maintain independent backups and test restores.

### Synchronous and asynchronous replication
- **Synchronous:** a write waits for acknowledgement from designated replicas under the configured protocol. This can reduce some data-loss risks but adds latency and can reduce write availability if required replicas cannot respond.
- **Asynchronous:** the primary acknowledges without waiting for replicas to apply the write. It often reduces write latency, but replicas can lag and failover can lose recent writes that were not replicated.

Product-specific variants exist. Understand the commit and failover guarantees instead of assuming replication means zero data loss.

### Replication lag and failover
A read from a lagging replica may show old data after a successful write. Mitigations include reading from the primary after a write, session/read-your-writes routing, waiting for a replication position when supported, or accepting staleness for noncritical views.

Failover requires health detection, controlled promotion, protection against split brain, client reconnection, and explicit recovery objectives: **RPO** (acceptable data loss) and **RTO** (acceptable recovery duration). Test the procedure. Replication alone is not a disaster-recovery plan.

## 3. Partitioning: divide a table
Partitioning divides a dataset into logical pieces, often within one database cluster. For example, an events table can be split into monthly partitions. Queries aligned with the partition key may scan fewer partitions; retention and maintenance can also be easier.

Common methods:
- **Range:** ranges of dates or IDs.
- **List:** defined categories, such as region.
- **Hash:** distribute keys using a hash function.

Partitioning does not guarantee faster queries. It helps most when query filters and data-management needs align with the partition key.

## 4. Sharding: distribute data across instances
Sharding is horizontal partitioning across separate database instances. Each shard owns part of the dataset. Replication and sharding are different: replication creates copies; sharding divides ownership. A system can use both by replicating each shard.

Sharding can increase storage and throughput capacity, but adds routing, migrations, monitoring, backup, and cross-shard query complexity. Introduce it when measurements justify the cost, not merely because traffic is growing.

## 5. Choose a shard key
A shard key determines where each record lives. Prefer a key that distributes data and traffic well and matches common access patterns.

Example for a messaging service:
- `user_id`: useful for user-scoped queries, but very active users can create hotspots depending on the routing design.
- `conversation_id`: keeps a conversation's data together, but user-wide queries may touch multiple shards.
- `created_at`: useful for time-range queries, but new writes can concentrate on the newest range.

Common routing methods:
- **Range-based:** easy range scans; risk of hot ranges.
- **Hash-based:** often distributes writes more evenly; range queries may fan out, and changing shard counts can require data movement.
- **Directory-based:** flexible key-to-shard lookup; the directory itself must be reliable and consistent.

Monitor per-shard CPU, data size, latency, traffic, and lag. A healthy cluster average can hide a single hot shard.

## 6. Cross-shard work and rebalancing
Sharding can make joins, global uniqueness, transactions, reporting, backups, and schema migrations more complex. Queries may need fan-out and result merging. Rebalancing requires a routing plan, data copy, consistency checks, controlled cutover, and a recovery plan.

## 7. Example: growing an online store
1. Start with a well-indexed database and good query patterns.
2. Add read replicas if reads dominate and the workload can tolerate replica lag.
3. Partition large tables if it helps time-based queries, retention, or maintenance.
4. Consider sharding when measured write capacity, storage, or isolation needs exceed one instance.
This is a guide, not a mandatory sequence; requirements may change the order.

## 8. Practical exercise
An event platform stores 20 million events per month. Dashboards mostly read the last 30 days. The workload is growing.
1. What would you measure before scaling?
2. Would read replicas help? What stale-read behavior must be handled?
3. Would monthly partitions help which queries?
4. When would sharding be justified?
5. What shard key fits tenant-scoped queries, and what could make it poor?
6. How would you detect lag and test failover?
7. What are your RPO and RTO?
8. How would you back up data independently of replication?

## 9. Interview structure
1. Clarify read/write mix, size, growth, latency, consistency, availability, and recovery requirements.
2. Identify the bottleneck with evidence.
3. Discuss query/index tuning and vertical scaling.
4. Explain replicas and stale-read/failover trade-offs.
5. Consider partitioning for query locality and data lifecycle.
6. Explain the shard key, routing, hot shards, cross-shard operations, and rebalancing.
7. Cover independent backups, monitoring, and recovery tests.

## Recap
- Replication copies data; partitioning divides a dataset; sharding distributes partitions across database instances.
- Asynchronous replicas can be stale; failover may lose unreplicated writes.
- Partition keys should fit query and retention patterns.
- Shard keys affect balance, hotspots, and query complexity.
- Sharding adds significant operational cost; measure first.
- Replication is not a substitute for independent backups and tested restores.

## Further reading
- [GeeksforGeeks — Database Replication in System Design](https://www.geeksforgeeks.org/system-design/database-replication-and-their-types-in-system-design/)
- [GeeksforGeeks — System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/)
- [PostgreSQL — High Availability and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [PostgreSQL — Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [MongoDB — Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [Google SRE Books](https://sre.google/books/)
- [Codecademy](https://www.codecademy.com/) (free access varies by course)
