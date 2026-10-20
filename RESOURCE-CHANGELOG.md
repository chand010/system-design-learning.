# Learning Resource Change Log

This file records meaningful changes to external learning resources that may affect the lessons in this repository.

## Monitoring rules

- Check official source pages where possible.
- Record the date checked, source-reported update date (if available), and the affected lesson.
- Compare with this log and the repository notes; do not call an item a change unless evidence supports that conclusion.
- Update lesson notes only when a verified change materially affects the explanation, recommendation, example, or technical accuracy.
- Ignore cosmetic edits and unrelated articles.
- Use a clear Git commit message for actual note updates. If no material change is found, leave the notes untouched.

## Baseline review — 15 October 2026

This is the initial baseline; there was no earlier resource-change log in this repository to compare against, so the dates below are source metadata, not proof that a change happened today.

| Resource | Source-reported update / status | Relevance | Action |
|---|---|---|---|
| [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) | Page reported “Last Updated: 17 Apr, 2026” in the baseline check | Main overview and topic roadmap | Recorded as baseline; no specific material change to Days 1–8 established from that first check |
| [GeeksforGeeks System Design category](https://www.geeksforgeeks.org/category/system-design/) | Category includes individually dated articles; dates vary by article | New topics and examples that may enrich future lessons | Monitor relevant articles; do not treat every new article as a change to existing notes |
| [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) | Page reported updated 24 February 2026 | Course, docs, articles, and roadmap discovery | Recorded as baseline; no specific material change to Days 1–8 established from this first check |
| [Codecademy Updates archive](https://www.codecademy.com/resources/blog/category/updates) | Official updates archive for courses and learning paths | Course/resource changes | Monitor for changes relevant to this learning path |
| [Google SRE Books](https://sre.google/books/) | Reference collection; no single page update date recorded in baseline | Reliability, monitoring, and operational design | Monitor relevant chapters and guidance |
| [PostgreSQL Documentation](https://www.postgresql.org/docs/current/) | “Current” documentation can evolve as PostgreSQL versions change | Database lessons and practical examples | Check version-sensitive claims before revising database notes |

## Review — 17 October 2026

| Resource | Current source-reported update / status | Summary compared with baseline | Affected lesson(s) | Action |
|---|---|---|---|---|
| [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) | Current page reports “Last Updated: 5 Sep, 2026” | The displayed date differs from the 17 Apr 2026 date captured in the baseline. Because no saved page snapshot is available, this confirms a metadata difference but not which text changed. | No material correction to Days 1–9 verified | No existing lesson notes changed; continue monitoring |
| [GeeksforGeeks Rate Limiting Algorithms](https://www.geeksforgeeks.org/system-design/rate-limiting-algorithms-system-design/) | Reports “Last Updated: 11 Aug, 2026” | Relevant overview of fixed window, sliding window, token bucket, and leaky bucket; added as a resource for the new rate-limiter lesson. This date is source metadata, not proof of a change since the baseline. | Day 10 | Used as a learning reference; no claim of historical text changes |
| [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) | Still reports updated 24 February 2026 | No source-date change from the baseline was observed. | None identified | No existing lesson changes |
| [Google SRE Books](https://sre.google/books/) | Current page labels “Site Reliability Engineering, 2nd Edition” as new and describes updates on current SRE practices, cloud reliability, and AI; no page update date displayed | The baseline did not preserve edition-level page details, so it is not possible to verify when this presentation appeared. Treat it as current resource information, not a confirmed change since the baseline. | Reliability topics in future lessons; no correction to Days 1–9 verified | Included as a current resource; no existing lesson changes |
| [Redis official rate limiter documentation](https://redis.io/docs/latest/develop/use-cases/rate-limiter/) | Current documentation explains shared counters and atomic Lua-based decisions; no source update date displayed | Useful current technical reference for distributed rate limiting; no historical comparison is available. | Day 10 | Added as a resource for the new lesson |
| [IETF RFC 6585](https://www.rfc-editor.org/rfc/rfc6585.html) | Standards-track specification for HTTP 429; published April 2012 | Defines 429 Too Many Requests and permits a Retry-After header. This is a technical reference, not a newly changed source. | Day 10 | Added as a protocol reference |

### Important limitation

The baseline establishes a starting point only. Without earlier snapshots or revision histories, a changed displayed date does not establish the exact content differences. On this review, no verified source change was found that materially requires rewriting existing Days 1–9 lesson explanations. The README was updated to index the newly published Day 10 lesson, and this log records the source review.


## Review — 20 October 2026

Official pages checked: [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) currently reports “Last Updated: 5 Sep, 2026”, matching the 17 October check but differing from the 15 October baseline's recorded 17 Apr date. No saved snapshots are available to establish why the displayed dates differ or what exact text changed. No material lesson correction was verified.

[GeeksforGeeks Database Replication](https://www.geeksforgeeks.org/system-design/database-replication-and-their-types-in-system-design/) reports 17 Apr 2026 and was used as a Day 13 reference, not treated as proof of a new change. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) still reports 24 Feb 2026. [Google SRE Books](https://sre.google/books/) currently labels both “Site Reliability Engineering, 2nd Edition” and “AI Engineering Reliable Operations” as new; without an earlier snapshot, the timing of the latter's appearance cannot be verified. The current [PostgreSQL partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html) and [replication](https://www.postgresql.org/docs/current/high-availability.html) pages document PostgreSQL 18 and link a PostgreSQL 19 RC 1 announcement dated 15 Oct 2026. The current [MongoDB sharding manual](https://www.mongodb.com/docs/manual/sharding/) covers shard keys, routing, and balancing.

Affected lessons: Day 13 uses these current database references. No verified external change required rewriting Days 1–13. The Day 12 update was a completeness repair because its previous repository file contained only an introduction, not a response to an external source change.
