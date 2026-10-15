# Day 7 — Database Scaling

## Learning goals
- Distinguish vertical and horizontal scaling.
- Explain replication, partitioning, and sharding.
- Recognize replication lag and the trade-offs of distributing data.

## 1. Vertical scaling
Vertical scaling (scale up) gives one database server more CPU, memory, or storage. It is often the simplest first step, but hardware has limits and a single server may remain a failure point.

## 2. Horizontal scaling
Horizontal scaling adds machines and distributes work or data across them. It can increase capacity, but introduces coordination, routing, operational, and consistency complexity.

## 3. Replication
Replication maintains copies of data on multiple database servers.

- **Primary/replica:** writes typically go to the primary; replicas can serve some reads.
- **Synchronous replication:** a write may wait for replica acknowledgement, improving durability guarantees but potentially increasing latency or reducing availability during failures.
- **Asynchronous replication:** the primary does not wait for every replica; this often lowers write latency but replicas may lag.

**Replication lag** means a replica has not yet received or applied the latest changes. A user who writes data and immediately reads from a lagging replica may see old data. Systems can route read-after-write requests to the primary or use other consistency strategies.

Replication improves read capacity and can support recovery, but it does not automatically solve every write bottleneck.

## 4. Partitioning and sharding
**Partitioning** divides a large dataset into smaller logical pieces. A database may partition by date, range, or list.

**Sharding** distributes partitions across multiple database servers. A shard key determines where a record belongs.

Possible shard keys include user ID, tenant ID, or geographic region. A good shard key distributes load evenly and supports common queries. A poor key can create hot shards or force queries to contact many shards.

Sharding makes rebalancing, cross-shard transactions, uniqueness, and global queries more complicated. Avoid it until simpler options no longer meet measured requirements.

## 5. A sensible scaling sequence
1. Measure slow queries and resource use.
2. Fix schema and query problems; add appropriate indexes.
3. Improve connection pooling and workload management.
4. Cache appropriate read-heavy data.
5. Scale the database vertically if reasonable.
6. Add read replicas if read load and consistency requirements permit.
7. Partition or shard when dataset size or workload justifies the complexity.
8. Test backup, restore, failover, and recovery procedures.

## Practical exercise
An application has many reads and relatively few writes. Its primary database is healthy during writes, but read queries overload it.
1. What could read replicas help with?
2. What problem can replication lag cause?
3. Why is sharding not necessarily the first solution?
4. If you shard by country, what might happen if one country generates most of the traffic?

## Suggested resources
- [PostgreSQL Documentation — High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [GeeksforGeeks — Database Sharding](https://www.geeksforgeeks.org/dbms/database-sharding-a-system-design-concept/)

## Key takeaway
Scale in stages. Replication helps distribute reads; partitioning and sharding distribute data, but each introduces trade-offs that should be justified by measured needs.
