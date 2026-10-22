# Additional Concepts to Learn

The original 111-concept list is the core curriculum, not a limit. This page tracks useful concepts that are not explicitly listed, so we can add them without removing or renumbering the original topics.

## How additions are selected

Add a concept when it:
- fills a prerequisite or important knowledge gap;
- is commonly used in real production systems;
- helps explain a trade-off, failure mode, or security risk; or
- is needed for one of our practical projects.

We will prioritize additions by dependency and practical value. Each topic should include a plain-language explanation, a real-world example, trade-offs, a practical exercise, and interview questions. Where relevant, connect it to the existing AI Image Detector project. Lessons should be published to this repository and verified before being taught in chat.

## Additional concepts backlog

| Concept | Why it matters | Related original topics |
|---|---|---|
| System design fundamentals: functional/non-functional requirements and trade-offs | Establishes the decision-making process for designing systems | 1–111 |
| Capacity estimation | Estimates requests/sec, storage, bandwidth, and compute needs | 45–47, 88, 91–94 |
| API design principles | Makes interfaces predictable, maintainable, and safe to evolve | 5, 14, 101–103 |
| Database normalization | Reduces duplication and update anomalies in relational data | 25–32, 81–82 |
| Database transactions and ACID | Explains atomicity, consistency, isolation, and durability | 18, 36–40, 81 |
| Object storage | Supports scalable storage of uploads, media, and backups | 2, 48–50, 83–84 |
| Consensus algorithms | Explains how distributed nodes agree on shared state | 33–38, 95 |
| Quorum reads and writes | Helps reason about replication, latency, and consistency | 29, 32–35, 95 |
| Load shedding | Protects critical work during overload by rejecting lower-priority work | 1, 10–16, 44–45 |
| Bulkheads | Isolates failures so one dependency or workload does not exhaust all resources | 10–12, 28, 44 |
| Zero-downtime deployments | Reduces user-visible disruption during releases and migrations | 51–57, 81–82 |
| Threat modeling | Identifies assets, trust boundaries, threats, and mitigations early | 67–80 |
| Software supply-chain security | Secures dependencies, build artifacts, images, and CI/CD pipelines | 6–8, 67 |
| Data retention, deletion, and privacy | Helps manage sensitive data, legal obligations, storage cost, and user trust | 58–68, 83–84 |
| Multi-tenancy and tenant isolation | Helps serve multiple customers safely on shared infrastructure | 5, 25–32, 67–68 |
| Search architecture | Explains indexing and retrieval systems such as Elasticsearch/OpenSearch | 2, 25–28, 45–47 |
| Architecture Decision Records (ADRs) | Captures why major technical decisions were made and what trade-offs were accepted | All topics |
| RTO and RPO | Defines recovery-time and data-loss objectives for disaster recovery | 83–86 |
| Hot partitions and hot keys | Explains uneven load that can overwhelm a shard, partition, or cache key | 2, 29–32, 38, 45 |
| Rate-limit algorithm details | Adds fixed-window, sliding-window, token-bucket, and leaky-bucket choices to rate-limiter design | 1 |

## Rate limiting: focused additions

The Day 10 rate-limiter lesson already introduces common algorithms. We will reinforce these as subtopics rather than duplicate the existing lesson:

- **Token bucket:** allows bounded bursts while controlling the long-run request rate.
- **Leaky bucket:** smooths output or processing rate; exact behavior depends on the implementation.
- **Fixed window:** simple counter per interval, with boundary-burst risk.
- **Sliding-window log/counter:** measures requests over a rolling period, with accuracy/state trade-offs.
- **Distributed rate limiting:** shares or coordinates policy state across multiple application instances.
- **Atomic updates and failure policy:** prevents race conditions and explicitly defines behavior if the shared store fails.

## Status

This is a prioritized backlog, not a claim that every topic has already been taught. We will mark concepts as planned, in progress, or completed as lessons are published and studied.
