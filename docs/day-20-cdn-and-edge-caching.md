# Day 20 — CDN and Edge Caching

**Learning objectives:** Explain what a content delivery network (CDN) does; distinguish origin servers, points of presence (PoPs), and edge servers; trace cache-hit and cache-miss requests; design cache keys and TTLs; choose between revalidation, invalidation, and purge; protect private content; and define operational SLIs for a CDN-backed application.

## 1. Start with the problem

Imagine your application is hosted in one region. A user in another country requests a 2 MB product image.

Without a CDN, the request may travel all the way to your origin region. The origin must also serve every copy of that image, even when thousands of users request the same file.

This creates two problems:

- **Latency:** network distance and extra hops can make the response slower.
- **Origin load:** repeated requests consume origin bandwidth, CPU, connections, and storage-read capacity.

A **content delivery network (CDN)** is a distributed system that serves content from locations closer to users. It caches eligible responses at edge locations and fetches from the origin when needed.

A CDN can reduce latency and origin traffic, but it does not make every request fast or automatically make private data safe to cache.

Read: [GeeksforGeeks — Designing a Content Delivery Network](https://www.geeksforgeeks.org/system-design/designing-content-delivery-network-cdn-system-design/).

## 2. Understand the main components

| Component | Responsibility | Simple mental model |
|---|---|---|
| Origin | Source of the original content | The main warehouse |
| PoP (Point of Presence) | A physical CDN site/data-center location | A regional delivery hub |
| Edge server | Serves requests at or near a PoP | The local pickup point |
| Cache | Stores reusable copies of responses | A shelf of frequently requested items |
| Request routing | Directs a user request to a suitable CDN location | Choosing a nearby delivery hub |
| Control plane | Manages configuration, routing policies, and purge operations | The management console |
| Data plane | Processes and serves user traffic | The delivery operation |

A PoP may contain multiple edge servers. The nearest PoP is not always the fastest or healthiest one: routing can consider network conditions, capacity, availability, and provider-specific policy.

A CDN is not simply a collection of copies manually placed in every country. Many CDNs fetch content on demand and use caching policies to decide what stays at each edge.

## 3. Without a CDN vs with a CDN

### Without a CDN

```text
Browser in Tanzania
       |
       | long network path
       v
Application / Origin in another region
       |
       v
Database or Object Storage
```

The origin receives the request and returns the content.

### With a CDN

```text
Browser
   |
   v
DNS / CDN request routing
   |
   v
Nearby CDN Point of Presence (PoP)
   |
   v
Edge cache
   |                    |
   | HIT                | MISS
   v                    v
Return cached       Regional cache /
response            Origin Shield
                         |
                         v
                       Origin
                         |
                         v
                  Store response at
                  eligible cache layers
                         |
                         v
                    Return response
```

The diagram is conceptual. Providers use different routing, cache tiers, and origin-fetch designs.

## 4. Cache hit and cache miss

A **cache hit** occurs when a suitable, fresh response is already available at the cache layer handling the request.

A **cache miss** occurs when the response is absent, expired, bypassed, or not eligible for caching. The CDN may fetch it from an upstream cache or the origin.

Example:

1. A user requests `/assets/logo.abc123.svg`.
2. The edge checks its cache.
3. If the object is fresh and the request matches its cache key, the edge returns it.
4. If not, the edge fetches from an upstream cache or origin.
5. If the response is eligible, it is stored for later requests.

The first request to a cold edge is often slower than later requests. A request can also miss at one edge but hit at a regional or tiered cache.

## 5. Static and dynamic content

### Static content

Examples:
- Versioned JavaScript and CSS bundles.
- Product images.
- Logos and fonts.
- Public downloads.
- Public video segments.

Static assets are often good CDN candidates, especially when their URLs change whenever their content changes.

### Dynamic content

Examples:
- Account balances.
- Personal dashboards.
- Shopping carts.
- Authenticated API responses.
- User-specific recommendations.

Dynamic content is not automatically uncacheable. Some dynamic responses can be cached safely when their representation is public or when the cache is deliberately partitioned by all relevant identity and variation dimensions. However, a shared edge cache must never mix one user's private response with another user's request.

A safe starting policy is:

- Public, versioned assets: cache for a long time.
- Frequently changing public pages: use shorter TTLs and revalidation.
- Private or user-specific responses: bypass shared caching or use a carefully designed private caching mechanism.
- Sensitive responses that must not be stored: send `Cache-Control: no-store` and configure the CDN not to override it.

## 6. TTL and HTTP cache headers

**TTL (time to live)** controls how long a cached response is considered fresh.

The origin can communicate caching policy with HTTP response headers.

Example:

```http
Cache-Control: public, max-age=31536000, immutable
```

This is appropriate only for a public asset whose URL changes when its content changes. It tells compliant caches that the response can be considered fresh for one year.

Another example:

```http
Cache-Control: public, max-age=60, s-maxage=300
```

Here, browser/shared-cache behavior can differ: `max-age` applies to general caches, while `s-maxage` specifies freshness for shared caches.

For a private response:

```http
Cache-Control: private, no-store
```

- `private`: shared caches should not store the response.
- `no-store`: caches should not store it.

Provider-specific CDN rules can override or supplement origin headers. Verify the actual deployed cache behavior; do not assume that a header alone controls every layer.

### Versioned asset names

A useful pattern is content fingerprinting:

- `app.8b1c2f.js`
- `styles.4f91aa.css`

When the file changes, its content hash and URL change. The application can safely give the new URL a long TTL, while old URLs naturally age out.

This is often safer and cheaper than purging every cached asset after every deployment.

## 7. Cache keys: what makes two requests the same?

A **cache key** identifies the representation that a cache stores.

A simple mental model is:

```text
Cache key = host + path + relevant query parameters
            + relevant representation variants
```

For example:

- `https://shop.example/images/item.jpg?size=small`
- `https://shop.example/images/item.jpg?size=large`

If the image bytes differ by the `size` parameter, those requests must not accidentally share one cache entry.

A CDN may include the host, protocol, path, query string, selected headers, or cookies in its cache key. The exact default differs by provider. Cloudflare's current documentation, for example, describes a default key that includes the full URL and other request attributes, and supports custom cache-key templates.

### Two opposite cache-key mistakes

**Mistake A — Missing a representation dimension**

Suppose a response varies by tenant or language, but the cache key ignores that dimension. One tenant or language may receive another tenant's or language's content.

This can become a data-leak vulnerability.

**Mistake B — Including irrelevant dimensions**

Suppose every analytics query parameter is included even though it does not change the response. Then requests for the same asset create many separate cache entries, reducing the hit ratio and fragmenting the cache.

### Cache-key design checklist

1. Identify every request attribute that changes the response body or security scope.
2. Include the required representation dimensions in the key, or bypass shared caching.
3. Exclude irrelevant tracking parameters only after confirming they cannot change the response.
4. Normalize URLs consistently between the cache and origin.
5. Test query-string, header, cookie, host, and language variations.
6. Inspect the provider's effective cache key and cache status in a test environment.

Do not put secrets such as raw access tokens into cache keys or logs. Do not solve private-content isolation by casually caching on an authorization header; prefer bypassing shared cache unless the provider's private-cache model is deliberately configured and tested.

Reference: [Cloudflare — Cache keys](https://developers.cloudflare.com/cache/how-to/cache-keys/).

## 8. Cache revalidation: ETag and Last-Modified

A cached object can become stale. Instead of downloading the entire object again, a cache can ask the origin whether the representation has changed.

Two common validators are:

- `ETag`: an entity tag identifying a representation version.
- `Last-Modified`: the time the origin says the representation was last modified.

Example:

```http
If-None-Match: "asset-v7"
```

If the origin confirms the representation is unchanged, it can return:

```http
HTTP/1.1 304 Not Modified
ETag: "asset-v7"
```

A 304 response tells the cache it can reuse its stored body, subject to the applicable cache rules. It avoids retransmitting the full body.

Revalidation is useful when content changes occasionally and you want freshness without always transferring the complete object.

## 9. Invalidation vs purge

These terms are sometimes used loosely, but a provider may distinguish them.

Cloudflare's current documentation describes:

| Operation | What happens | Next request |
|---|---|---|
| Purge | Matching cached content is removed | Fetches the full response from origin or an upstream cache |
| Invalidate (soft purge) | Matching content is kept but marked stale | Revalidates with the origin |

Cloudflare also documents that invalidated content can be served stale while revalidation is happening or when the origin fails, depending on the cache policy. A purge is the appropriate choice when the old cached content must not be served again.

Important operational sequence:

1. Update or remove the content at the origin.
2. Purge the narrowest matching URL, tag, hostname, or prefix.
3. Request the content again.
4. Inspect the CDN response headers and origin logs.
5. Confirm that the old representation is no longer being served.

A successful purge API response may only confirm that the provider accepted the request. Cloudflare specifically recommends verifying the subsequent request and checking cache-status headers; a request returning HTTP 200 from the purge API is not by itself proof that a particular edge object was evicted.

Avoid purging everything unless necessary. A broad purge can turn many cache hits into origin misses at once, causing an origin traffic spike or cache stampede.

References:
- [Cloudflare — Invalidate cached content](https://developers.cloudflare.com/cache/guides/invalidate-cache/)
- [Cloudflare — Purge cache](https://developers.cloudflare.com/cache/how-to/purge-cache/)

## 10. Tiered caching and Origin Shield

A CDN may have more than one cache layer:

```text
User
  |
  v
Local Edge Cache
  |
  v
Regional / Tiered Cache
  |
  v
Origin Shield (provider-specific)
  |
  v
Origin
```

A **tiered cache** allows edge locations to reuse a higher-level cache instead of each independently fetching the origin.

An **Origin Shield** is an additional cache layer offered by some providers. AWS CloudFront Origin Shield, for example, is designed to improve cache hit ratio and reduce the number of simultaneous origin requests for the same object.

Benefits can include:
- Fewer origin fetches.
- Less origin bandwidth.
- Better handling of popular objects.
- Reduced duplicate work during traffic spikes.

Trade-offs:
- More layers can complicate debugging.
- A cache miss may traverse several layers.
- Invalidation and freshness behavior must be understood across tiers.
- Additional features may have cost or regional constraints.

Reference: [AWS CloudFront — Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html).

## 11. CDN security

A CDN can add security controls, but it is not a substitute for secure application design.

### Common protections

- TLS between users and the CDN, and ideally between the CDN and origin.
- DDoS absorption and traffic distribution.
- Web application firewall (WAF) rules, where supported.
- Rate limiting and bot controls, where configured.
- Origin access restrictions so attackers cannot bypass the CDN.
- Signed URLs or cookies for restricted content.
- Request normalization and cache-poisoning defenses.
- Logging and alerting for unusual traffic.

### Cache poisoning

Cache poisoning occurs when an attacker influences a response that is stored under a cache key later used by other visitors.

Potential causes include:
- Unkeyed request headers that influence the origin response.
- Conflicting URL normalization between cache and origin.
- Unsafe query-string handling.
- Incorrect host or header handling.

### Cache deception

Cache deception can occur when a cache is tricked into storing a private response under a URL that appears to be a cacheable static resource.

Mitigations:
1. Keep private routes out of public cache rules.
2. Use explicit cache policies for public asset paths.
3. Ensure cache-key dimensions match the origin's representation logic.
4. Avoid caching authenticated or personalized responses by default.
5. Test for cache poisoning and cache deception in an authorized staging environment.
6. Restrict direct access to the origin where practical.
7. Treat CDN logs and cache keys as potentially sensitive operational data.

### Signed URLs are not a complete caching policy

A signed URL can restrict who is allowed to request an object, but it does not automatically mean every CDN configuration will keep the object private or prevent caching. Providers have different rules for signed content and cache eligibility. Verify both viewer authorization and cache behavior. For example, CloudFront supports signed URLs/cookies in cache behaviors; Google Cloud CDN documents specific cache-mode behavior for signed URLs.

References:
- [AWS CloudFront — Cache behavior settings](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html)
- [Google Cloud CDN — Caching overview](https://docs.cloud.google.com/cdn/docs/caching)

## 12. Reliability and performance metrics

A CDN can be reachable while users still experience slow or stale responses. Monitor the delivery path, not just whether the CDN API returns HTTP 200.

Useful metrics:

| Metric | Why it matters |
|---|---|
| Cache hit ratio | Indicates how often requests are served from cache |
| Origin fetch QPS | Shows how much work reaches the origin |
| Origin egress bytes | Helps explain bandwidth and cost |
| p50/p95/p99 time to first byte | Measures user-perceived response latency |
| Edge 4xx/5xx rate | Reveals errors at the delivery layer |
| Origin 4xx/5xx rate | Reveals backend failures |
| Cache status distribution | Distinguishes HIT, MISS, BYPASS, EXPIRED, and provider-specific states |
| Purge/invalidation verification | Confirms that old content stopped being served |
| PoP-level latency and errors | Reveals regional problems |
| Cache-key cardinality | Can reveal accidental cache fragmentation |
| Stale response rate | Helps track freshness and origin-failure behavior |

### Define an SLI and SLO

An **SLI** is a measured indicator of service behavior. An **SLO** is the target for that indicator.

Example objectives for a learning exercise:
- 99% of cacheable image requests complete within 200 ms at the edge under a defined test region and load.
- 99.9% of CDN requests return a successful response over a 30-day window.
- 99% of content updates are visible at the CDN within a defined freshness target.

These are example targets, not provider guarantees. Choose them from user needs and measured baselines.

Google's SRE Workbook explains how monitoring and SLO-based alerting connect service health to operational action.

References:
- [Google SRE Workbook — Monitoring](https://sre.google/workbook/monitoring/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)

## 13. Capacity estimation exercise

Assume:
- 10,000 requests per second for cacheable public assets.
- Average response size is 500 KB.
- CDN cache hit ratio is 90%.

Approximate origin request rate, assuming each miss requires one origin fetch:

```text
Origin fetch QPS = 10,000 × (1 - 0.90)
                 = 1,000 requests/second
```

Approximate origin response bandwidth, assuming each miss returns 500 KB:

```text
1,000 × 500 KB = 500,000 KB/s
               ≈ 500 MB/s (decimal)
```

This is a simplified estimate. It ignores range requests, compression, request coalescing, tiered caches, retries, cache revalidation, different object sizes, and traffic distribution.

The CDN still serves the full user-facing traffic; the hit ratio mainly reduces origin work.

## 14. Practical system design exercise

Design a CDN-backed media delivery system for a video and image platform.

Requirements:
- 1 million daily users.
- Public images and CSS/JavaScript assets.
- Private user documents.
- Product images may change during the day.
- A deployment can replace frontend assets.
- Origin bandwidth is expensive.
- A sudden traffic spike must not overload the origin.
- Private files must never be delivered to another user.

Answer these questions:

1. Draw the browser → edge → regional cache → origin request path.
2. Which content should receive a long TTL?
3. Which content should bypass shared caching?
4. How would you version frontend assets?
5. What fields belong in the cache key for localized product pages?
6. How would you prevent tenant-specific responses from being shared?
7. Explain the difference between revalidation, invalidation, and purge.
8. What would happen to origin traffic after a global purge?
9. How could an Origin Shield or tiered cache help?
10. Which SLIs, SLOs, and alerts would you define?
11. How would you verify that a private document cannot be retrieved from a shared cache?

**Stretch:** If requests average 10,000 QPS and the cache hit ratio drops from 90% to 70%, calculate the new origin fetch QPS. Explain why a small drop in hit ratio can cause a large increase in origin load at high traffic.

## 15. Interview practice

**Question: What problem does a CDN solve?**

A strong answer: a CDN serves eligible content from distributed edge locations closer to users. This reduces network distance for cache hits and offloads repeated requests from the origin. It improves performance and can improve resilience, but it does not eliminate origin dependencies or make all dynamic content cacheable.

**Question: What is the difference between a cache hit and a cache miss?**

A strong answer: a hit means a suitable fresh response is available in the cache. A miss means the cache must fetch from an upstream cache or origin, or otherwise handle the request without a stored response.

**Question: Why is cache-key design security-sensitive?**

A strong answer: if the cache key ignores a request attribute that changes the response or its authorization scope, a shared cache can serve one user's or tenant's response to another. If it includes irrelevant attributes, the hit ratio can collapse.

**Question: What is the difference between purge and revalidation?**

A strong answer: purge removes the cached representation; revalidation asks the origin whether the stored representation is still current, often using ETag or Last-Modified validators. Provider-specific invalidation may mark an object stale rather than immediately remove it.

**Question: How would you protect a private file behind a CDN?**

A strong answer: authorize the user before issuing access, use provider-supported signed access where appropriate, configure private/no-store or an explicitly isolated private cache, ensure cache keys and origin rules cannot mix users, restrict origin bypass, and test the effective cache behavior.

## Short recap

- A CDN distributes content through edge locations close to users.
- A PoP is a physical site; edge servers inside it serve requests.
- Cache hits reduce origin work; misses may fetch from an upstream cache or origin.
- TTL and HTTP headers control freshness, subject to provider rules.
- Cache keys must distinguish different representations and security scopes.
- ETag and Last-Modified enable conditional revalidation.
- Purge, invalidation, and revalidation are not always identical.
- Tiered caching and Origin Shield can reduce duplicate origin requests.
- Private content needs explicit authorization and cache isolation.
- Measure hit ratio, origin load, latency, errors, freshness, and purge outcomes.

## Learning resources

1. [GeeksforGeeks — Designing Content Delivery Network (CDN) in System Design](https://www.geeksforgeeks.org/system-design/designing-content-delivery-network-cdn-system-design/) — CDN architecture, requirements, cache modules, and scaling; page reports last updated 9 April 2026.
2. [GeeksforGeeks — Content Delivery Network in System Design](https://www.geeksforgeeks.org/system-design/what-is-content-delivery-networkcdn-in-system-design/) — origin, edge, static/dynamic content, pull/push CDN concepts; page reports last updated 25 June 2026.
3. [Cloudflare — Cache keys](https://developers.cloudflare.com/cache/how-to/cache-keys/) — cache-key components and custom key trade-offs; page reports last updated 30 October 2026.
4. [Cloudflare — Invalidate cached content](https://developers.cloudflare.com/cache/guides/invalidate-cache/) — purge vs invalidation and verification; page reports last updated 29 September 2026.
5. [Cloudflare — Default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) — origin cache headers and cacheability.
6. [Google Cloud CDN — Caching overview](https://docs.cloud.google.com/cdn/docs/caching) — cache modes, origin headers, and cache TTLs; page reports last updated 26 October 2026 UTC.
7. [AWS CloudFront — Cache behavior settings](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html) — path patterns, origins, query strings, HTTPS, and signed access.
8. [AWS CloudFront — Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) — an additional caching layer to reduce origin load.
9. [Google SRE Workbook — Monitoring](https://sre.google/workbook/monitoring/) — metrics and operational monitoring.
10. [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) — turning reliability objectives into alerts.
11. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — general resource directory; it is not a dedicated CDN course.

## Resource monitoring — checked 1 November 2026

- [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) reports **Last Updated: 5 September 2026**, matching the date recorded in the 31 October Day 19 review. No new tutorial-page date was observed.
- [GeeksforGeeks System Design category](https://www.geeksforgeeks.org/category/system-design/) currently displays an Open/Closed Principle article dated **1 October 2026** near the top of the page. The 31 October review recorded a different article dated 15 October from its opened category snapshot. The snapshots do not establish a reliable content diff or a newly published CDN-specific category change.
- [GeeksforGeeks CDN design article](https://www.geeksforgeeks.org/system-design/designing-content-delivery-network-cdn-system-design/) reports **9 April 2026**; the related [CDN overview article](https://www.geeksforgeeks.org/system-design/what-is-content-delivery-networkcdn-in-system-design/) reports **25 June 2026**. They are references for this new lesson, not proof that an earlier lesson changed.
- [Codecademy Change Log](https://www.codecademy.com/article/change-log) still shows the **16 September 2026** SQL notice with changes effective **30 September 2026**, and the earlier July 2026 Python notices. These were already recorded in the 31 October review; no newer relevant change was verified.
- [Google SRE Books](https://sre.google/books/) still labels *Site Reliability Engineering, 2nd Edition* and *AI Engineering Reliable Operations* as new. The page does not display a publication/update date for these labels, so no new change is inferred.
- [Cloudflare Cache Keys](https://developers.cloudflare.com/cache/how-to/cache-keys/) reports **Last updated 30 October 2026**. The prior Day 5 note mentioned safe key design but did not explain representation dimensions or the security consequences of omitting a varying header/tenant dimension. Day 5 is being enriched with these current documented concepts; without an earlier page snapshot, this is not described as a verified textual diff on Cloudflare's side.
- [Cloudflare Invalidate cached content](https://developers.cloudflare.com/cache/guides/invalidate-cache/) reports **Last updated 29 September 2026**. Day 19 already noted that CDN/browser caches can outlive an origin deletion, but did not distinguish purge from soft invalidation or explain how to verify the result. Day 19 is being clarified with the provider-documented distinction and verification procedure; no claim is made that the source text changed since a prior snapshot.
- [Google Cloud CDN Caching overview](https://docs.cloud.google.com/cdn/docs/caching) reports **Last updated 26 October 2026 UTC**. This is a provider-specific current reference; without an earlier page snapshot, the exact text change cannot be established.
- [AWS CloudFront cache behavior](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html) and [Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) were checked as current official technical references. The opened pages did not provide a visible update date that supports a historical change claim.

**Monitoring outcome:** The current official sources add useful detail to two existing lessons. Day 5 is enriched with CDN cache-key identity and isolation guidance. Day 19 is clarified with purge-versus-invalidation behavior and verification. The source-reported dates are recorded above, but no exact external text diff is claimed without earlier snapshots. No unrelated lessons were changed.
