# Day 5 — Caching Fundamentals

## Learning goals
- Explain how a cache works.
- Understand cache hits, misses, TTL, and invalidation.
- Identify when caching helps and when it creates risks.

## 1. What is a cache?
A cache stores a copy of data that is expensive or slow to retrieve, so future requests can be served faster. Caches can exist in a browser, CDN, application process, shared cache such as Redis, or database.

Example: a product page is requested thousands of times. Instead of querying the database for every request, the application can temporarily cache product details.

## 2. Cache hit and cache miss
- **Cache hit:** requested data is present in the cache.
- **Cache miss:** data is absent or expired, so the application loads it from the source and may cache it.

Typical cache-aside flow:
1. Check the cache.
2. If found, return the cached value.
3. If missing, query the database.
4. Store the result with a suitable expiry.
5. Return the result.

## 3. TTL and invalidation
**TTL (time to live)** is how long a cache entry remains valid before it expires.

Invalidation means removing or refreshing cached data when it is no longer safe to use. Short TTLs reduce the duration of stale data but can increase database load. Event-based invalidation can improve freshness but is more complex.

Ask: How stale can this data safely be? A product description may tolerate a short delay; an account balance or authorization decision needs much more careful treatment.

## 4. Cache design risks
- **Stale data:** the cache is older than the source.
- **Cache stampede:** many requests miss or expire at once and overwhelm the database.
- **Hot key:** one key receives a disproportionate share of requests.
- **Unbounded growth:** the cache consumes too much memory.
- **Sensitive data exposure:** private information is cached or shared incorrectly.
- **Cache poisoning:** untrusted input influences cached content or keys.

Mitigations can include TTLs with jitter, request coalescing, bounded memory, access controls, safe key design, and explicit invalidation.

## 5. Cache-key design and representation isolation

A **cache key** identifies the particular response representation stored by a cache. It must distinguish requests when the returned content or security scope differs.

For an application cache, a key might look like:

```text
product-list:v3:tenant-42:locale-sw:category-shoes:sort-price
```

For a CDN, the key may be based on the URL (scheme, host, path, and query string) plus selected request headers or cookies. The exact defaults differ by provider.

Two mistakes are dangerous:

- **Omitting a relevant dimension:** if a response varies by tenant, language, device, or another request attribute but the key ignores it, users may receive the wrong representation. If the omitted dimension is an authorization boundary, this can become a cross-user data leak.
- **Including irrelevant dimensions:** tracking parameters or headers that do not change the response can create many cache entries and reduce the hit ratio.

Design checklist:
1. Identify every request attribute that changes the response body or access scope.
2. Include required representation dimensions, or bypass a shared cache for private content.
3. Exclude irrelevant parameters only after confirming they cannot change the response.
4. Normalize URLs consistently between cache and origin.
5. Test query-string, header, cookie, host, and locale variations.
6. Inspect the provider's effective cache key and cache-status headers.

Do not put raw access tokens or secrets in cache keys or logs. Do not casually solve private-response isolation by caching on an authorization header. A safe starting point is to bypass shared caching for user-specific responses unless a deliberate private-cache design has been tested.

Reference: [Cloudflare — Cache keys](https://developers.cloudflare.com/cache/how-to/cache-keys/) (source page reports last updated 30 October 2026; checked 1 November 2026).

## 6. Tiny Python example
```python
cache = {}

def get_user_name(user_id, database):
    key = f"user-name:{user_id}"

    if key in cache:
        return cache[key]  # cache hit

    name = database.get_user_name(user_id)  # cache miss
    if name is not None:
        cache[key] = name
    return name
```

This illustrates the idea only. A Python dictionary is local to one process, has no expiry in this example, and is not shared across servers. Production systems need a deliberate expiration and invalidation strategy.

## Practical exercise
A news homepage receives 10,000 requests per minute for the same top stories. The stories update every 60 seconds.
1. Would a short-lived cache help? Why?
2. What TTL might you test first?
3. What happens if every cache entry expires simultaneously?
4. Which data should not be placed in a public shared cache?

## Suggested resources
- [Redis Documentation](https://redis.io/docs/latest/)
- [GeeksforGeeks — Caching in System Design](https://www.geeksforgeeks.org/system-design/caching-system-design-concept-for-beginners/)

## Key takeaway
Caching can reduce latency and backend load, but freshness, invalidation, capacity, cache-key identity, and privacy must be designed deliberately.
