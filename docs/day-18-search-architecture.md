# Day 18 — Search Architecture

**Learning objectives:** Explain how a search system ingests documents, builds an index, answers queries, ranks results, handles updates, scales across shards, and protects private data.

## 1. Start with the problem

Imagine an online shop has 2 million products. A customer searches for “wireless noise cancelling headphones.” A naive application could scan every product title and description on every request. That becomes expensive as the catalogue and query rate grow.

A search system prepares data before users ask questions. It builds structures that make likely queries faster, then retrieves and ranks matching documents.

A search engine is not just a database with a search box. It has an **indexing path** that prepares documents and a **query path** that retrieves relevant results.

## 2. Define requirements first

### Functional requirements
1. Index documents such as products, articles, files, or support tickets.
2. Search selected fields such as title, description, category, and tags.
3. Support exact filters such as category, price range, status, and date.
4. Rank results by relevance, with optional business rules such as popularity or availability.
5. Support pagination, sorting, autocomplete, and typo tolerance where needed.
6. Add, update, and delete documents from the searchable corpus.
7. Enforce access rules so users only see documents they are allowed to see.

### Non-functional requirements
- **Latency:** queries should return within a defined target, including at peak load.
- **Throughput:** the system should support expected queries per second (QPS).
- **Freshness:** define how long an update may take to become searchable.
- **Availability:** searches should continue through expected node failures.
- **Relevance:** useful results should appear near the top.
- **Scalability:** indexing and query capacity should grow with the workload.
- **Cost:** storage, memory, compute, replicas, and re-indexing must be affordable.
- **Security:** authorization must be enforced for every query and result path.

Write measurable targets where possible. “Fast search” is not a requirement; “p95 query latency below 250 ms at 500 QPS” is a testable example, not a universal target.

## 3. High-level architecture

A common design separates ingestion from query serving:

\`\`\`text
                 Source Systems
              (DB, files, APIs, events)
                       |
                       v
              Ingestion / Connectors
                       |
                       v
            Validate, Extract, Normalize
                       |
                       v
                 Durable Queue
                       |
                       v
                 Index Workers
                       |
                       v
              Search Index / Shards
                       ^
                       |
User -> Search API -> Query Parser / Planner
                       |
              Authorization + Filters
                       |
                 Query Execution
                       |
                Rank / Merge Results
                       |
                  Response + Cache
\`\`\`

The **indexing path** prepares and updates searchable data. The **query path** answers user searches. They share the index but have different workloads and scaling needs.

For a small application, a relational database's full-text search may be enough. A dedicated search engine such as OpenSearch or Elasticsearch becomes useful when you need richer text analysis, relevance tuning, aggregations, distributed indexing, or independent query scaling. Do not introduce a search cluster just because the application has a search box.

## 4. What is an inverted index?

An inverted index maps a term to the documents that contain it. It is the central data structure behind many full-text search engines.

Suppose we have three documents:

- D1: “blue wireless headphones”
- D2: “wireless keyboard”
- D3: “blue running shoes”

A simplified inverted index is:

| Term | Documents |
|---|---|
| blue | D1, D3 |
| wireless | D1, D2 |
| headphones | D1 |
| keyboard | D2 |
| running | D3 |
| shoes | D3 |

For the query “wireless headphones”, the engine looks up the postings for both terms and combines them according to the query semantics. It does not need to scan every original document from scratch.

A postings list can contain more than document IDs. It may store term frequency, positions, or other information that supports phrase matching and scoring.

**Trade-off:** an inverted index speeds up reads but consumes storage and requires work when documents are inserted, updated, or deleted. Indexing is not free.

Read: [GeeksforGeeks — Inverted Index](https://www.geeksforgeeks.org/dbms/inverted-index/).

## 5. Text analysis: turning text into searchable terms

Before indexing, text is usually analyzed.

A simplified analysis pipeline:

1. **Character filters:** normalize or remove selected characters.
2. **Tokenizer:** split text into tokens.
3. **Token filters:** lowercase, remove selected stop words, stem words, or apply synonyms.

Example input:

“THE Wireless Headphones — RUNNING!”

A simple analyzer might produce:

“wireless, headphones, running”

The exact output depends on the configured analyzer and language.

The query text must be analyzed compatibly with the indexed text. If the index stores lowercased tokens but the query is processed differently, expected matches may be missed. OpenSearch documents separate index-time and search-time analyzers, and recommends compatible analysis in most cases.

Useful features:
- **Stemming:** maps related word forms to a common stem, where appropriate.
- **Stop words:** optionally excludes very common words, depending on the language and search needs.
- **Synonyms:** maps terms such as “laptop” and “notebook” when the product vocabulary supports it.
- **Edge n-grams:** can support prefix matching and autocomplete, at the cost of a larger index.
- **Language analyzers:** handle language-specific tokenization and morphology.

Do not apply stemming or stop-word removal blindly. Names, product codes, short queries, and languages with different word boundaries may need different treatment.

References:
- [OpenSearch — Index analyzers](https://docs.opensearch.org/latest/analyzers/index-analyzers/)
- [OpenSearch — Search analyzers](https://docs.opensearch.org/latest/analyzers/search-analyzers/)
- [OpenSearch — Text analysis](https://docs.opensearch.org/latest/analyzers/)

## 6. Query processing and relevance ranking

A basic query path can be described as:

1. Validate and normalize the incoming query.
2. Authenticate the caller and determine allowed document scope.
3. Parse query terms, filters, sort options, and pagination.
4. Search the index for candidate documents.
5. Score and rank candidates.
6. Apply safe business rules and return a bounded result set.

### What is relevance?

A search engine may find hundreds of matching documents. It must decide which ones should appear first.

A common lexical ranking algorithm is **BM25**. At a beginner level, it considers:
- how often a query term appears in a document (term frequency);
- how rare that term is across the corpus (inverse document frequency);
- document length, so very long documents are not rewarded simply for containing many words.

BM25 is a useful baseline, not a perfect measure of user intent. A product search may also need structured filters, popularity, stock availability, freshness, personalization, or curated boosts.

Do not confuse relevance with sorting. Sorting by price or date is deterministic; relevance ranking estimates how well a document matches the query.

References:
- [OpenSearch — Keyword search and BM25](https://docs.opensearch.org/latest/search-plugins/keyword-search/)
- [OpenSearch — Explain API](https://docs.opensearch.org/latest/api-reference/search-apis/explain/)

### Example query

A simplified OpenSearch request might look like this:

\`\`\`json
GET products/_search
{
  "query": {
    "match": {
      "description": {
        "query": "wireless noise cancelling headphones"
      }
    }
  },
  "size": 10
}
\`\`\`

This is an educational example. Production queries should explicitly define fields, filters, authorization scope, timeouts, pagination limits, and the expected response shape.

## 7. Indexing, updates, and search freshness

Search indexes are usually derived data. The primary business record may live in MySQL or PostgreSQL, while the search engine maintains a representation optimized for retrieval.

A common indexing workflow is:

1. A business record is created or updated.
2. An event or change-data-capture process records the change.
3. A durable queue buffers indexing work.
4. An index worker transforms the record into a search document.
5. The worker writes the document to the search index.
6. The document becomes searchable after the engine's refresh behavior makes the change visible.

This is often eventually consistent: the primary database may show a new value before search results do.

OpenSearch uses a near-real-time model. With its usual refresh behavior, an indexed document is not necessarily searchable immediately. Forcing an immediate refresh can increase resource cost, so choose freshness targets deliberately.

Define a **freshness SLI**, for example: “99% of product updates become searchable within five seconds.” Measure it rather than assuming that a successful indexing API response means the user can already search the change.

Reference: [OpenSearch — Document API refresh behavior](https://docs.opensearch.org/latest/api-reference/document-apis/).

### What about deletes?

A deleted or access-restricted record must stop appearing in search. Design the deletion/update event path, retries, and reconciliation carefully. For sensitive or tenant-private records, authorization should also be checked at query or fetch time; do not rely only on eventual index cleanup to prevent data exposure.

## 8. Distributed search: shards and replicas

As the corpus grows, one machine may not have enough storage, memory, or CPU.

A search index can be divided into **shards**. Each shard holds part of the index. A coordinator sends a query to the relevant shard copies, receives partial results, and merges them into one response.

A **replica** is an additional copy of a shard. Replicas can improve availability and provide more copies that can serve search requests, depending on the engine and configuration.

Simplified example:

\`\`\`text
                 Search Coordinator
                    /          \\
                   v            v
              Shard 1        Shard 2
              /     \\        /     \\
          Primary  Replica  Primary  Replica
\`\`\`

Important trade-offs:
- More primary shards can distribute data and query work, but each shard adds overhead.
- Replicas use additional storage and resources, but can improve resilience and read capacity.
- Queries spanning many shards require coordination and result merging.
- Poor routing or skewed data can create hot shards.
- Adding nodes does not automatically fix an inefficient query or an overloaded shard.

OpenSearch documents routing search requests to primary or replica shard copies and using adaptive replica selection to consider prior latency and queue conditions.

References:
- [GeeksforGeeks — Designing Distributed Search System](https://www.geeksforgeeks.org/system-design/designing-distributed-search-system/)
- [OpenSearch — Search shard routing](https://docs.opensearch.org/latest/search-plugins/searching-data/search-shard-routing/)
- [OpenSearch — Index operations](https://docs.opensearch.org/latest/im-plugin/index-operations/)

## 9. Caching, autocomplete, and pagination

### Caching
Popular queries can be expensive and repetitive. A cache may store results for a short time.

A cache key should include every factor that changes the result, such as:
- normalized query;
- filters and sort;
- locale;
- index or catalogue version;
- tenant or authorization scope.

Never share cached private results across users or tenants merely because the query text is the same. Cache invalidation and freshness must match the product's requirements.

### Autocomplete
Autocomplete can use:
- prefix indexes or edge n-grams;
- a trie or another prefix data structure;
- a curated suggestion service;
- popularity signals.

Autocomplete needs its own latency and abuse controls because users may send a request after every keystroke.

### Pagination
Offset pagination is simple for small result sets, but deep offsets can become expensive. Search engines often provide cursor-like mechanisms such as search-after for stable traversal. Use a deterministic tie-breaker and understand how concurrent index changes affect pagination.

## 10. Search quality and evaluation

A technically fast search system can still be a bad product if the results are irrelevant.

Create a small evaluation set:
- query;
- documents judged relevant;
- expected ordering or relevance labels;
- important filters and edge cases.

Measure quality with suitable ranking metrics such as Precision@k, Recall@k, or NDCG@k. Review zero-result queries, reformulations, click behavior, and user feedback. Do not optimize only for clicks: a click can be accidental or reflect poor ranking.

A practical iteration loop:
1. Collect representative queries.
2. Label relevant results.
3. Run the current ranking.
4. Compare metrics and inspect failures.
5. Adjust analyzers, fields, boosts, synonyms, or ranking features.
6. Re-run the evaluation set and check latency/cost regressions.

## 11. Security and privacy

Search can become a data-exposure path if authorization is overlooked.

- Authenticate every caller.
- Enforce tenant and document-level authorization in the query path.
- Avoid trusting tenant IDs supplied by an untrusted client without checking them against the authenticated identity.
- Validate query size, filters, sort fields, and pagination limits.
- Restrict access to index-management and administrative APIs.
- Encrypt network traffic and protect credentials.
- Minimize sensitive fields stored in the search index.
- Propagate deletes and access changes reliably.
- Treat snippets and autocomplete suggestions as data disclosures too.
- Log query metadata carefully; search terms may contain personal or confidential information.

For multi-tenant systems, test that a user from tenant A cannot discover tenant B's records through search results, counts, facets, suggestions, cached responses, or error messages.

## 12. Capacity and operational metrics

Estimate workload before selecting shard counts or cluster size.

Useful estimates:
- documents indexed per day;
- average source-document size;
- index-size multiplier measured from a representative sample;
- update/delete rate;
- peak search QPS;
- average and p95/p99 query latency;
- replica count and retention period;
- expected growth and re-indexing headroom.

Do not assume index size equals source JSON size. An index stores analyzed terms, metadata, doc values, and other structures; measure a representative sample.

Monitor:
- query rate and latency percentiles;
- timeout and failed-shard rates;
- zero-result rate and relevance metrics;
- indexing throughput and indexing failures;
- source-to-search freshness lag;
- shard size and shard skew;
- CPU, memory, disk, and merge pressure;
- cache hit rate;
- queue depth and age of oldest indexing event;
- rejected requests and thread-pool queues.

A healthy API response can still contain partial results if some shards fail. Inspect the engine's timeout and shard-failure indicators, and define whether partial results are acceptable for each use case.

## 13. Practical exercise

Design product search for an online shop with 2 million products.

Requirements:
- Search by product name and description.
- Filter by category, price, and availability.
- Sort by price.
- Show the ten most relevant results first.
- Product updates should become searchable within five seconds for 99% of updates.
- Search must never reveal products belonging to another merchant.
- Traffic is 300 queries/second on average, with bursts to 1,000 QPS.

Answer:
1. What are the functional and non-functional requirements?
2. Would you start with PostgreSQL full-text search or a dedicated search engine? Explain the trade-off.
3. Draw the ingestion and query paths.
4. What fields would you index as analyzed text versus exact-match keywords?
5. Explain how an inverted index helps the query.
6. How would you rank results?
7. How would you propagate updates and deletes?
8. How would you enforce merchant isolation?
9. What would you cache, and what must be part of the cache key?
10. Which metrics and alerts would you define?
11. How would you test the five-second freshness target?

**Stretch:** Assume a representative test shows that the search index occupies 1.6 times the source-data size. Estimate the raw index storage for 2 million products if average source data is 2 KB per product. Then explain why replicas, merge headroom, snapshots, and growth require additional capacity. The 1.6 multiplier is an exercise assumption, not a universal property of search engines.

## 14. Interview practice

**Question: Why use an inverted index instead of scanning every document?**

A strong answer: an inverted index maps terms to postings lists of documents containing them. This avoids scanning the entire corpus for each term query, trading additional index storage and update work for faster retrieval.

**Question: What is the difference between a shard and a replica?**

A strong answer: shards divide the index into partitions; replicas are additional copies of shards that can improve availability and, depending on the engine, search capacity.

**Question: Why might a newly updated product not appear immediately in search?**

A strong answer: the search index is derived and may be near-real-time rather than immediately consistent. The indexing pipeline and refresh interval introduce freshness lag.

**Question: How do you prevent cross-tenant data leakage?**

A strong answer: derive tenant scope from authenticated identity, enforce it in every query and result path, scope caches correctly, and test results, facets, suggestions, and snippets for isolation.

## Short recap

- Search has separate indexing and query paths.
- An inverted index maps terms to documents.
- Analyzers turn text into terms; query analysis must be compatible.
- Ranking orders candidates by relevance; BM25 is a common baseline.
- Search freshness is a measurable consistency requirement.
- Shards partition an index; replicas copy shards.
- Caches must include filters and authorization scope.
- Search results, facets, snippets, and suggestions all require authorization.
- Measure relevance, latency, freshness, indexing lag, and shard health.

## Learning resources

1. [GeeksforGeeks — Designing Distributed Search System](https://www.geeksforgeeks.org/system-design/designing-distributed-search-system/) — architecture, ingestion, indexing, sharding, and distributed query coordination; page reports last updated 18 October 2025.
2. [GeeksforGeeks — Inverted Index](https://www.geeksforgeeks.org/dbms/inverted-index/) — beginner-friendly explanation and example; page reports last updated 18 April 2026.
3. [OpenSearch — Index analyzers](https://docs.opensearch.org/latest/analyzers/index-analyzers/) — index-time text analysis.
4. [OpenSearch — Search analyzers](https://docs.opensearch.org/latest/analyzers/search-analyzers/) — query-time text analysis and autocomplete example.
5. [OpenSearch — Keyword search and BM25](https://docs.opensearch.org/latest/search-plugins/keyword-search/) — relevance scoring.
6. [OpenSearch — Search shard routing](https://docs.opensearch.org/latest/search-plugins/searching-data/search-shard-routing/) — primary/replica routing and adaptive replica selection.
7. [OpenSearch — Document API refresh behavior](https://docs.opensearch.org/latest/api-reference/document-apis/) — freshness/performance trade-offs.
8. [OpenSearch — Search your data](https://docs.opensearch.org/latest/getting-started/search-data/) — query basics and response fields.
9. [Google SRE Books](https://sre.google/books/) — use reliability and monitoring chapters to define search SLIs, SLOs, and operational practices.
10. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — a general directory for learning resources, not a dedicated search-architecture course.

## Resource monitoring — checked 30 October 2026

- [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) currently reports **5 September 2026**, matching the 24 October repository review. No new source-reported update date was observed.
- [GeeksforGeeks System Design category](https://www.geeksforgeeks.org/category/system-design/) currently shows an article dated **15 October 2026** on event-driven architecture; that item was already recorded in the 23 October review. No newer relevant category entry was verified in the opened page.
- [GeeksforGeeks — Designing Distributed Search System](https://www.geeksforgeeks.org/system-design/designing-distributed-search-system/) reports **18 October 2025**. [Inverted Index](https://www.geeksforgeeks.org/dbms/inverted-index/) reports **18 April 2026**. These are references for this new lesson, not evidence of changes to earlier lesson notes.
- [Codecademy Change Log](https://www.codecademy.com/article/change-log) reports a **16 September 2026** notice for SQL content changes effective **30 September 2026**, including refreshed datasets and workspace changes in the Analyze Data with SQL path. It also reports a **14 July 2026** notice for beginner Python updates. Day 6's current database note does not depend on the affected Codecademy SQL path, and no other existing lesson dependency was verified; no lesson edit was justified.
- [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) still reports **24 February 2026**.
- [Google SRE Books](https://sre.google/books/) still labels *Site Reliability Engineering, 2nd Edition* and *AI Engineering Reliable Operations* as new, as recorded in the 24 October review. No page update date is displayed, so no new change is inferred.
- Current OpenSearch documentation was checked for analyzers, BM25, refresh behavior, shard routing, and query responses. The opened documentation pages did not expose a source-reported update date that establishes a change since the previous review.

**Monitoring outcome:** No verified material change affecting an existing system-design lesson was detected. The Codecademy SQL/Python updates were reviewed against the repository's current Day 6 note and do not justify a system-design note change. The resource changelog is intentionally left untouched; the date and findings for this new lesson are recorded here.
