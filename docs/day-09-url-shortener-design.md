# Day 9 — URL Shortener Design

A URL shortener maps a long URL to a compact code and redirects visitors to the original URL.

## Requirements
- Create a short link from a valid destination.
- Redirect visitors for a known code.
- Return not found for an unknown code.
- Keep codes unique and mappings durable.
- Keep redirects fast and limit excessive requests.

## API
- POST /api/urls creates a mapping.
- GET /{code} resolves a code and returns a redirect response.

## Storage
Store a code, destination URL, creation time, optional expiration time, and optional disabled time. Add a database UNIQUE constraint to the code field. An application-side check alone is not sufficient because concurrent requests can race.

## Code generation
Base62 uses uppercase letters, lowercase letters, and digits. Seven characters provide about 3.52 trillion combinations. A unique ID encoded as Base62 avoids random collisions if ID allocation itself is unique. Random codes are also possible, but collisions must be detected and retried safely.

## Scaling
Start with one application service and a relational database. Add a cache for frequently accessed mappings only when measurements justify it. If analytics are needed, process click events asynchronously so they do not delay redirects.

## Exercise
Draw the create and redirect flows. Choose a code-generation strategy, explain the unique constraint, and describe what happens if the cache or database is unavailable.

## Resources
- [GeeksforGeeks URL Shortener System Design](https://www.geeksforgeeks.org/system-design/system-design-url-shortening-service/)
- [PostgreSQL Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [Google SRE Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)

## Key takeaway
Design around requirements, unique mappings, low-latency redirects, and clear failure behavior. State assumptions before choosing technologies.
