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


## Review — 1 November 2026

Date checked: **1 November 2026**. The review used the current official pages and compared their displayed dates and concepts with the existing lesson notes and earlier entries in this file. A source-reported update date is recorded as metadata; without a saved earlier page snapshot, it is not treated as proof of the exact text that changed.

| Source | Source-reported update / current finding | Affected lesson | Summary and action |
|---|---|---|---|
| [Cloudflare — Cache keys](https://developers.cloudflare.com/cache/how-to/cache-keys/) | **Last updated 30 October 2026** | Day 5 — Caching Fundamentals | The current page explains cache-key identity using URL components and selected request attributes, and describes the trade-offs of custom keys. Day 5 previously mentioned “safe key design” but did not explain response-variation dimensions or the risk of omitting a tenant/authorization dimension. **Action:** added a cache-key design and representation-isolation section, with provider-specific guidance. No exact external text diff is claimed because no prior snapshot was preserved. |
| [Cloudflare — Invalidate cached content](https://developers.cloudflare.com/cache/guides/invalidate-cache/) | **Last updated 29 September 2026** | Day 19 — Object Storage and File Upload System Design | The current page distinguishes purge (remove cached content) from invalidation/soft purge (mark stale and revalidate), documents stale-serving behavior, and provides verification steps. Day 19 already noted that CDN/browser caches can outlive origin deletion but did not explain these distinct operations or verification. **Action:** clarified the note and added provider-specific verification guidance. No exact external text diff is claimed without a prior snapshot. |
| [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) | **Last Updated: 5 September 2026**, matching the 31 October Day 19 review | Course roadmap | No newer source-reported date observed. **Action:** no lesson edit. |
| [GeeksforGeeks System Design category](https://www.geeksforgeeks.org/category/system-design/) | Current opened page displays an Open/Closed Principle article dated **1 October 2026** near the top. The 31 October note recorded a different article dated 15 October from its opened category snapshot. | Course roadmap | The differing snapshots do not establish a reliable content diff or a new relevant category item. **Action:** no lesson edit; retain uncertainty. |
| [GeeksforGeeks — Designing a CDN](https://www.geeksforgeeks.org/system-design/designing-content-delivery-network-cdn-system-design/) | **Last Updated: 9 April 2026** | Day 20 — CDN and Edge Caching | Used as a new-lesson learning reference for CDN components, requirements, and architecture; not treated as a change to earlier lessons. |
| [GeeksforGeeks — CDN in System Design](https://www.geeksforgeeks.org/system-design/what-is-content-delivery-networkcdn-in-system-design/) | **Last Updated: 25 June 2026** | Day 20 — CDN and Edge Caching | Used as a new-lesson reference for origin/edge, static and dynamic content, and pull/push CDN concepts. |
| [Codecademy Change Log](https://www.codecademy.com/article/change-log) | Still shows the **16 September 2026** SQL notice (effective 30 September 2026) and July 2026 Python notices, as recorded on 31 October | No affected system-design lesson identified | No newer relevant change verified. **Action:** no lesson edit. |
| [Google SRE Books](https://sre.google/books/) | Still labels *Site Reliability Engineering, 2nd Edition* and *AI Engineering Reliable Operations* as new; no page update date displayed | Day 20 — CDN and Edge Caching | Used current SRE Workbook monitoring and SLO alerting chapters as operational references. The “New!” labels have no displayed date, so no new publication/change is inferred. |
| [Google Cloud CDN — Caching overview](https://docs.cloud.google.com/cdn/docs/caching) | **Last updated 26 October 2026 UTC** | Day 20 — CDN and Edge Caching | Used as a current provider-specific reference for cache modes and origin headers. No earlier snapshot is available to establish the exact textual change. |
| [AWS CloudFront — Cache behavior settings](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html) and [Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) | Current official pages checked; no visible page update date captured | Day 20 — CDN and Edge Caching | Used as current technical references for cache behaviors, signed access, and origin shielding. No historical change claimed. |

**Outcome:** Day 5 and Day 19 received focused, source-grounded clarifications. No unrelated lesson was changed. The repository notes do not claim that a displayed “Last updated” date proves a particular text diff when no earlier snapshot exists.
