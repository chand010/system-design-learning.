# Day 8 — Load Balancing and High Availability

## Learning goals
- Explain how a load balancer distributes requests.
- Compare common balancing algorithms.
- Understand health checks, redundancy, and stateless application servers.

## 1. What is a load balancer?
A load balancer receives requests and distributes them across backend servers. It can improve capacity and resilience, provided the backends and other dependencies are healthy.

Example:

`Clients → Load Balancer → App Server A / App Server B / App Server C`

The load balancer may terminate TLS, route by host or path, enforce limits, and perform health checks, depending on its type and configuration.

## 2. Common algorithms
- **Round robin:** sends requests to each server in turn.
- **Weighted round robin:** sends more requests to servers assigned higher weights.
- **Least connections:** prefers the server with fewer active connections.
- **Least response time:** prefers servers based on observed response times and/or active connections, depending on implementation.
- **Hash-based routing:** maps a key (such as a user ID) to a backend; can help affinity but may cause uneven load.

No algorithm is best for every workload. Request duration, server capacity, connection reuse, and session behavior matter.

## 3. Health checks
A health check helps the balancer avoid sending traffic to an unhealthy backend. A basic liveness check asks whether the process is running; a readiness check asks whether it is ready to handle requests.

Health checks must be designed carefully. A check that depends on every downstream service may remove all backends during a shared dependency outage. Configure thresholds and recovery behavior to avoid unnecessary flapping.

## 4. Layer 4 vs Layer 7
- **Layer 4 (transport):** routes based on connection-level information such as IP addresses and TCP/UDP ports.
- **Layer 7 (application):** understands protocols such as HTTP and can route based on host, path, headers, or other request details.

Layer 7 routing is more flexible but requires understanding the application protocol and may add processing overhead.

## 5. High availability
A load balancer does not guarantee high availability if it is itself a single point of failure. Production designs often use a managed or redundant load-balancing service, multiple application instances across failure domains, health checks, and tested recovery procedures.

Application servers are easier to scale when they are **stateless**: any instance can handle any request without relying on local session data. Store shared sessions in a suitable shared store or use secure tokens where appropriate. Keep durable business data in durable storage, not only in an instance's local filesystem.

## 6. Capacity estimate
If one application instance safely handles 200 requests/second and expected peak traffic is 700 requests/second, the theoretical minimum is `ceil(700 / 200) = 4` instances. That leaves little room for a failure or traffic spikes. With one instance unavailable, three instances can handle 600 RPS, so the design needs more capacity or a lower peak target. Validate estimates with load tests.

## Practical exercise
Design a load-balanced API:
1. Draw the client, load balancer, three app servers, and database.
2. Choose a balancing algorithm and explain why.
3. Define what counts as a ready backend.
4. Explain what happens when one server fails.
5. Explain how users can keep their session if their next request reaches another server.

## Suggested resources
- [NGINX — What Is a Load Balancer?](https://www.nginx.com/resources/glossary/load-balancing/)
- [AWS — Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html)
- [GeeksforGeeks — Load Balancer in System Design](https://www.geeksforgeeks.org/system-design/load-balancer-in-system-design/)

## Key takeaway
Load balancing distributes traffic, while high availability also requires redundancy, health checks, stateless design, and resilient dependencies.
