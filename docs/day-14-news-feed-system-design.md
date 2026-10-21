# Day 14 — Design a News Feed System

## Learning objectives

By the end of this lesson, you should be able to:
- Clarify functional and non-functional requirements for a social news feed.
- Design APIs and a basic data model for users, follows, posts, and feeds.
- Compare fan-out on write, fan-out on read, and a hybrid approach.
- Explain how caching, queues, ranking, and cursor pagination fit together.
- Identify celebrity hot spots, stale feeds, duplicate work, and recovery concerns.

## 1. Clarify the requirements first

Imagine a social app where people publish posts and see posts from accounts they follow.

### Functional requirements
1. A user can publish, edit, and delete a post.
2. A user can follow or unfollow another account.
3. A user can fetch a feed of recent posts from followed accounts.
4. The feed supports pagination.
5. A user can see their own newly published post promptly.
6. Deleted or unavailable posts should not continue appearing as valid content.

For a first version, assume a reverse-chronological feed. Recommendations, comments, direct messages, video transcoding, and ads can be separate systems.

### Non-functional requirements
- Feed reads should have low latency.
- The service should remain available during ordinary component failures.
- Feed data can be eventually consistent: a followed account's post may appear a few seconds later.
- The system should scale for a read-heavy workload.
- Privacy and authorization must be enforced on every read.
- A worker failure must not permanently lose feed updates.

Clarify these assumptions in an interview. A chronological feed is much simpler than a personalized ranking system.

## 2. Define the API

Possible endpoints:

- `POST /v1/posts` — create a post.
- `DELETE /v1/posts/{post_id}` — delete a post the caller owns.
- `POST /v1/users/{user_id}/follow` — follow an account.
- `DELETE /v1/users/{user_id}/follow` — unfollow an account.
- `GET /v1/feed?cursor=...&limit=20` — retrieve a page of feed items.

Example create request:

```json
{
  "text": "Learning system design today",
  "media_ids": []
}
```

Example response:

```json
{
  "post_id": "post_8421",
  "created_at": "2026-10-21T17:00:00Z"
}
```

The timestamps and IDs here are illustrative. The service must authenticate the caller, validate the payload, enforce post ownership, and apply request limits.

## 3. Model the data

A relational starting point might contain:

### users
- `user_id` — primary key
- `display_name`
- `created_at`

### follows
- `follower_id`
- `followee_id`
- `created_at`
- Unique constraint on `(follower_id, followee_id)`
- Indexes to find both the accounts a user follows and the followers of an account.

### posts
- `post_id`
- `author_id`
- `text` or a reference to content storage
- `created_at`
- `visibility` and deletion/status fields as needed

An index such as `(author_id, created_at DESC, post_id DESC)` can support retrieving recent posts by an author. Verify index design against the actual database and query plan.

### feed_items (optional precomputed inbox)
- `viewer_id`
- `post_id`
- `created_at` or a sortable feed position
- A uniqueness rule or idempotent insertion strategy to avoid duplicate feed entries.

Keep the post itself as the source of truth. A feed entry can store a post ID rather than copying the entire post body into every follower's feed. That reduces duplicated content and makes edits/deletions easier to reflect, although reads need a hydration lookup or cache.

## 4. High-level architecture

```text
                     Publish post
Client -> API -> Post service -> Posts database
                       |
                       v
                 Outbox / Queue
                       |
                       v
                 Fan-out workers
                       |
                       v
              Feed store / feed cache

                     Read feed
Client -> Feed API -> Feed cache or feed store
                       |
                       +--> Hydrate post IDs from post cache/database
                       |
                       +--> Merge recent posts from very large accounts
                       |
                       v
                 Filter and rank
                       |
                       v
                 Cursor response
```

This is a logical architecture, not a requirement to deploy every box as a separate service. Begin with a simpler implementation and split components only when scale or team boundaries justify it.

## 5. Fan-out on write (push)

When an author publishes a post, the system finds their followers and inserts the new post ID into each follower's precomputed feed.

If an author has 500 followers, one post may generate about 500 feed-entry writes, in addition to the post write.

**Benefits**
- Feed reads are fast because much of the work is already done.
- The read path does not need to query every followed account.
- It works well for read-heavy feeds and ordinary follower counts.

**Costs**
- Publishing work grows with follower count.
- Some work is wasted for followers who never open the app.
- The queue and feed store need capacity for bursts.
- A popular account can create an enormous fan-out job.

Fan-out should usually run asynchronously. The post service can save the post and reliably record an event, then return without waiting for every follower's feed to be updated. This means a follower may see the post a little later.

## 6. Fan-out on read (pull)

With fan-out on read, the system stores a post once. When a user opens their feed, it finds the accounts they follow, fetches recent posts from those accounts, merges the results, filters them, and returns the newest page.

**Benefits**
- Publishing is cheap: store the post once.
- No feed-entry writes are needed for inactive followers.
- A post from a very popular account does not trigger millions of writes at publish time.

**Costs**
- Every feed read does more work.
- Users who follow many accounts can cause many queries and a costly merge.
- Tail latency can rise if the read path depends on many sources.

## 7. The celebrity problem: use a hybrid

Neither strategy is ideal for every author.

A common design is:
- Fan-out on write for ordinary accounts.
- Skip or limit fan-out for accounts with extremely large follower counts.
- Fetch recent posts from those large accounts during feed reads, then merge them with the precomputed feed.

The threshold should be based on measured follower distributions, write amplification, queue capacity, and latency—not a universal fixed number.

The hybrid approach moves some work to reads for popular accounts while keeping the typical feed fast. The merge must deduplicate entries and apply a consistent ordering rule.

## 8. Queue processing and correctness

A post event might look like:

```json
{
  "event_id": "evt_9002",
  "event_type": "PostPublished",
  "post_id": "post_8421",
  "author_id": "user_17",
  "created_at": "2026-10-21T17:00:00Z"
}
```

A worker reads the event, loads the follower list in batches, and adds the post ID to eligible feeds.

Important safeguards:
- Make processing idempotent; a redelivered event must not create repeated feed entries.
- Retry transient failures with bounded backoff.
- Send repeatedly failing jobs to a dead-letter queue and alert on backlog.
- Monitor queue age and processing rate.
- Consider a transactional outbox so a committed post cannot silently lose its publication event.
- Check post status and visibility during feed reads, not only when the feed entry is created.

A queue can absorb bursts but cannot fix a permanent capacity mismatch. If events arrive faster than workers process them, the backlog grows.

## 9. Cache and storage choices

A cache such as Redis can hold recent feed IDs for frequently active users. A persistent store may retain feed entries or allow rebuilding them from posts and follow relationships.

Practical rules:
- Put limits on cached feed length and use a TTL or eviction policy appropriate to the workload.
- Store IDs rather than full post bodies when that reduces duplication.
- Batch-hydrate post details instead of issuing one database query per item.
- Treat the cache as rebuildable unless the architecture explicitly makes it durable.
- Plan for a cache outage: a sudden fall-through to the database can create a traffic spike.
- Invalidate or filter deleted, private, or blocked content so stale feed entries do not bypass access rules.

Images and videos should generally live in object/blob storage and be delivered through a CDN. The feed should carry references and metadata rather than large binary files.

## 10. Ranking and pagination

Start with chronological ordering, then add ranking if the product requires it. Ranking might consider recency, relationship, quality, or explicit user preferences. Ranking introduces additional features, compute, and explainability/fairness considerations.

Use cursor pagination rather than a large offset for a live feed. An example cursor can contain the last item's timestamp and unique ID:

```text
GET /v1/feed?limit=20&before=2026-10-21T16:55:00Z,post_8421
```

The exact cursor should be opaque to clients in a production API. Use a stable tie-breaker such as `post_id` because multiple posts can share a timestamp. Cursor pagination reduces problems caused by new posts arriving above the current page, though deletions and ranking changes still need defined behavior.

## 11. Estimate the workload

Use assumptions rather than pretending to know real traffic.

Suppose:
- 10 million daily active users.
- Each opens the feed 8 times per day.
- Each publishes 1 post per day.

Then:
- Feed loads per day = 10,000,000 × 8 = 80 million.
- Posts per day = 10 million.
- Average feed-load rate = 80,000,000 / 86,400 ≈ 926 requests/second.
- Average post rate = 10,000,000 / 86,400 ≈ 116 posts/second.

If peak traffic is 4× average, the feed endpoint might need to handle roughly 3,704 requests/second under these assumptions. Real peak shape and follower distribution must be measured. Fan-out work depends on follower count, not only the number of posts.

## 12. Failure modes and monitoring

Plan for:
- A fan-out worker crashing midway through a job.
- Duplicate delivery of a post event.
- A popular author producing a large burst.
- A cache becoming unavailable.
- The feed queue falling behind.
- A post being deleted after feed entries were generated.
- A user unfollowing or blocking an account.
- A database or broker outage.

Monitor:
- Feed API p95/p99 latency and error rate.
- Cache hit rate and database fall-through traffic.
- Queue depth and oldest-event age.
- Fan-out completion latency and retry counts.
- Per-author follower count and fan-out cost.
- Feed hydration failures and duplicate-entry rate.
- Stale or unauthorized content filtered from feeds.

Define service objectives and test recovery procedures. A low average latency can hide poor p99 latency for users following many accounts.

## 13. Security and privacy

- Authenticate all write requests and enforce ownership.
- Check visibility, blocks, mutes, and deletion rules during reads.
- Do not assume that a cached feed entry grants permission to see the post.
- Rate-limit post creation and follow/unfollow operations.
- Validate text and media references.
- Avoid exposing private account data through logs or event payloads.
- Define retention and deletion behavior for feed caches and derived records.

## 14. Practical exercise

Design a feed for a service with 5 million active users. Most users follow 200 accounts, but a few accounts have 20 million followers. Feed reads are much more frequent than posts.

Answer:
1. What are your functional and non-functional requirements?
2. What APIs and database tables do you need?
3. Would you use fan-out on write, fan-out on read, or a hybrid?
4. How would you handle a post from an account with 20 million followers?
5. How would you avoid duplicate feed entries after a worker retry?
6. What happens if the queue is down after a post is saved?
7. How would you paginate a feed while new posts arrive?
8. How do you enforce deletion, privacy, and unfollow rules?
9. Which metrics would tell you that the system is falling behind?

## Interview structure

1. Clarify scope and requirements.
2. Estimate reads, writes, and fan-out work.
3. Define APIs and the data model.
4. Draw the publish and read paths.
5. Compare push, pull, and hybrid fan-out.
6. Explain caching, queues, idempotency, pagination, and ranking.
7. Cover hot accounts, privacy, failures, and monitoring.

Do not begin with a list of technologies. Explain why the architecture places work on the write path or read path, and how it behaves under skewed follower counts.

## Recap

- Fan-out on write makes reads fast but makes publishing expensive for authors with many followers.
- Fan-out on read makes publishing cheap but can make feed reads expensive.
- A hybrid handles ordinary and very popular accounts differently.
- Queues make fan-out asynchronous, so feed updates may be delayed.
- Idempotency, cursor pagination, caching limits, and privacy checks are essential.
- Measure the actual workload and tail latency before optimizing.

## Learning resources

- [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) — review scalability, caching, databases, and high availability.
- [Design a News Feed — System Design Book Notes](https://books.dwf.dev/docs/system-design/c12) — a detailed discussion of fan-out on write and read.
- [Google SRE Books](https://sre.google/books/) — service objectives, monitoring, and reliability engineering.
- [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — programming docs and learning resources; course availability varies.
- [Redis Documentation](https://redis.io/docs/latest/) — caching and data structures; choose the exact data structure based on your access pattern.

These resources are references for further study. The architecture above is a teaching design, not a claim that a particular production service uses these exact components.
