# Day 1 — System Design Fundamentals

## Learning goals
- Explain system design in plain language.
- Identify the main parts of a typical web system.
- Distinguish functional and non-functional requirements.

## 1. What is system design?
System design is the process of deciding how software components work together to meet a set of requirements. It considers the application, data, communication between components, performance, reliability, security, and cost.

Think about a food-delivery app. A customer places an order, the app sends it to a backend service, the backend stores the order, and the restaurant and customer receive updates. System design describes those parts and how they interact.

## 2. Common building blocks
- **Client:** browser or mobile app used by a person.
- **DNS:** translates a domain name into an IP address.
- **Load balancer:** distributes incoming requests across healthy servers.
- **Application server:** runs business logic and returns responses.
- **Database:** stores durable information.
- **Cache:** keeps frequently used data for faster access.
- **Object/file storage:** stores images, videos, and documents.
- **Queue:** lets one component hand work to another asynchronously.
- **Monitoring and logs:** help detect and investigate failures.

A simple request path is:

`User → DNS → Load Balancer → Application → Database`

A real system may also use a cache, object storage, queues, and monitoring.

## 3. Functional vs non-functional requirements
**Functional requirements** describe what the system does: users can register, upload a photo, search products, or pay for an order.

**Non-functional requirements** describe how well it must work: response time, availability, security, privacy, scalability, durability, and cost.

Example: a photo-sharing service must allow users to upload and view photos (functional). It should load photos quickly, protect private images, and remain available during server failures (non-functional).

## 4. A repeatable design process
1. Clarify the problem and scope.
2. Identify users and core use cases.
3. Estimate traffic and data size.
4. Define APIs and data models.
5. Draw the high-level architecture.
6. Identify bottlenecks and failure modes.
7. Explain trade-offs and future growth.
8. Decide what to monitor and how to secure it.

## Practical exercise
Design a simple notes app. Write:
1. Three functional requirements.
2. Three non-functional requirements.
3. The main components needed.
4. What should happen if the application server restarts.

## Check your understanding
1. What is the difference between a functional and a non-functional requirement?
2. Why might a system need a cache?
3. Why is a database not the same thing as an application server?

## Suggested resources
- [GeeksforGeeks — System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/)
- [Google Cloud — Well-Architected Framework](https://cloud.google.com/architecture/framework)

## Key takeaway
Start with the problem and requirements before choosing technologies. Good system design is about justified trade-offs, not simply using many tools.
