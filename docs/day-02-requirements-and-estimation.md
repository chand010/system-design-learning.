# Day 2 — Requirements and Capacity Estimation

## Learning goals
- Ask clarifying questions before designing.
- Estimate requests per second and storage needs.
- State assumptions clearly.

## 1. Clarify the scope
Before drawing an architecture, ask:
- Who uses the system?
- What are the most important user actions?
- How many users are active per day?
- Are requests mostly reads or writes?
- What response time and availability are expected?
- Is data private, regulated, or required to be retained?
- What is out of scope for the first version?

Do not assume every possible feature must be built immediately. Identify the core use cases first.

## 2. Back-of-the-envelope estimates
Estimates are approximations that help reveal the scale of a problem. State your assumptions and keep units consistent.

### Requests per second (RPS)
If a service receives 86,400,000 requests per day:

`Average RPS = daily requests / 86,400 seconds`

`86,400,000 / 86,400 = 1,000 average requests/second`

Peak traffic may be several times higher than average. If the peak multiplier is 3, plan for roughly 3,000 RPS, then validate with measurements and a safety margin.

### Storage
Suppose 100,000 users each upload 2 photos per day, and each photo averages 2 MB.

`Daily storage = users × photos/user/day × average size`

`100,000 × 2 × 2 MB = 400,000 MB ≈ 400 GB/day`

This excludes replicas, thumbnails, metadata, backups, and compression. If retained for 30 days without deletion, raw uploads alone could approach 12 TB.

### Bandwidth
If 1,000 responses/second each transfer 100 KB, outbound traffic is about 100 MB/s before protocol overhead and caching effects. Clarify whether sizes use decimal or binary units for precise estimates.

## 3. Read/write ratio
A feed might serve many more reads than writes. That may justify caching and read replicas. A payment ledger may require careful durable writes and consistency rather than aggressive caching. The workload determines the design.

## 4. Record assumptions
A useful estimate note looks like this:
- 1 million daily active users
- 20 requests per active user per day
- 80% reads, 20% writes
- peak traffic is 4× average
- each stored record is 1 KB before indexes and replication

These are assumptions, not facts. Update them when real usage data becomes available.

## Practical exercise
Estimate a simple image-sharing app:
1. 20,000 daily active users.
2. Each uploads 3 images/day.
3. Average image size is 1.5 MB.
4. Estimate daily raw storage and 30-day raw storage.
5. Assume 10 requests per user per day; estimate average RPS and peak RPS using a 5× peak multiplier.

## Answers to check yourself
- Daily raw images: 20,000 × 3 × 1.5 MB = 90,000 MB ≈ 90 GB/day.
- 30 days: about 2.7 TB before replicas, thumbnails, metadata, and backups.
- Daily requests: 20,000 × 10 = 200,000; average RPS ≈ 2.3; 5× peak ≈ 11.6 RPS.

## Suggested resources
- [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [GeeksforGeeks — System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/)

## Key takeaway
Estimation is not about getting a perfect number. It is about exposing assumptions and discovering which parts of the design may need special attention.
