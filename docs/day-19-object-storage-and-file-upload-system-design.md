# Day 19 — Object Storage and File Upload System Design

**Learning objectives:** Compare block, file, and object storage; design a secure upload/download workflow; separate object bytes from relational metadata; use short-lived upload permissions; handle large uploads, integrity checks, lifecycle rules, and deletion; and explain the trade-offs in a system design interview.

## 1. Start with the problem

Imagine your application lets users upload profile pictures, PDFs, videos, or images for analysis.

A beginner implementation might save every uploaded file directly into a folder on the PHP web server. That can work for a small single-server application, but it becomes difficult when:

- multiple application servers need access to the same files;
- users upload large files over unreliable connections;
- the application needs to scale independently from storage;
- files must survive application-server replacement;
- access must be private and auditable;
- old files must be deleted according to retention rules.

A common design is to store **file bytes in object storage** and store **ownership, status, object key, checksum, and other searchable metadata in a relational database**.

Object storage is not a replacement for every database or filesystem. Choose storage based on access patterns and requirements.

## 2. Compare block, file, and object storage

| Storage type | How data is organized | Common use cases | Simple mental model |
|---|---|---|---|
| Block storage | Addressable blocks presented to a host or volume | Database volumes, virtual machines, low-level storage | A disk |
| File storage | Files and directories addressed by paths | Shared folders, home directories, shared application files | A network drive |
| Object storage | Objects with data, metadata, and unique keys, accessed through APIs | Images, videos, backups, logs, documents, data lakes | A very large API-accessed object collection |

### Block storage

A storage volume exposes addressable blocks. A filesystem or database manages how those blocks represent files and records. It is often useful for workloads requiring low-level block access and frequent small reads/writes.

### File storage

Files are organized in a hierarchy such as `/shared/reports/2026/report.pdf`. Multiple machines can mount a shared file system if the product supports that access model.

### Object storage

An object usually consists of:
- **Object data:** the bytes of the file.
- **Object key:** a unique name within a bucket or container.
- **Metadata:** descriptive attributes such as content type, size, checksum, or custom labels.
- **Version information:** where versioning is supported and enabled.

Object stores generally expose HTTP/API operations rather than a normal mounted directory interface. A key such as `uploads/2026/10/abc123.jpg` may look like a folder path, but the slash-separated prefix is usually a logical naming convention, not necessarily a real directory.

Read: [GeeksforGeeks — Block, Object, and File Storage in System Design](https://www.geeksforgeeks.org/system-design/block-object-and-file-storage-in-cloud-with-difference/).

## 3. Requirements before architecture

### Functional requirements

1. Accept uploads from authenticated users or services.
2. Support allowed file types and maximum sizes.
3. Store file bytes and associated metadata.
4. Let authorized users retrieve or delete their files.
5. Track upload and processing status.
6. Support large or resumable uploads if required.
7. Verify file integrity.
8. Apply retention and lifecycle rules.
9. Optionally scan files before making them available to other components.

### Non-functional requirements

- **Durability:** reduce the risk of losing successfully stored data.
- **Availability:** make upload and download operations available within a defined service target.
- **Security:** prevent unauthorized uploads, reads, overwrites, and deletions.
- **Scalability:** grow storage and transfer capacity without scaling every application server equally.
- **Performance:** meet upload and download latency/throughput targets.
- **Cost:** account for storage, requests, retrieval, replication, and network egress.
- **Privacy:** minimize retention and delete data according to policy.
- **Observability:** detect failed uploads, abandoned multipart uploads, unusual access, and deletion failures.

Do not confuse **durability** with **availability**. Durability concerns whether data survives; availability concerns whether a service can be accessed when needed. A storage service's actual guarantees depend on its provider, region, storage class, configuration, and contract.

## 4. High-level architecture

A secure direct-to-object-storage upload commonly looks like this:

```text
                 User / Browser
                       |
                       | 1. Request upload
                       v
                 Application API
                       |
             Authenticate + Authorize
                       |
              Validate quota and policy
                       |
                       v
              MySQL: upload session
                       |
              Generate unique object key
                       |
              Create short-lived URL
                       |
                       v
              Return upload permission
                       |
                       | 2. Upload bytes directly
                       v
                 Object Storage
                       |
                       | 3. Confirm upload
                       v
                 Application API
                       |
               Verify object metadata
               size / checksum / key
                       |
                       v
                Quarantine / Pending
                       |
                 Scan / Validate Worker
                       |
             +---------+----------+
             |                    |
             v                    v
          Rejected              Accepted
          Delete object         Mark READY
             |                    |
             v                    v
       Status = REJECTED     Authorized download
```

The application server controls identity, policy, and metadata. The client sends the large file bytes directly to object storage, so the application server does not need to proxy every byte.

This is a logical architecture. A small application can start with one API and one worker; separate services are not mandatory.

## 5. Separate object bytes from database metadata

Do not store every large file as a database row by default. A relational database is often better for ownership, workflow state, searchable metadata, and transactions; object storage is often better for large unstructured byte streams.

A simple metadata table might contain:

| Column | Purpose |
|---|---|
| `id` | Internal upload ID |
| `owner_id` | Authenticated owner or tenant |
| `object_key` | Opaque storage key |
| `original_filename` | Display name, treated as untrusted input |
| `detected_content_type` | Type determined by server-side inspection |
| `size_bytes` | Verified object size |
| `checksum_algorithm` | Algorithm used for integrity verification |
| `checksum_value` | Verified digest/checksum |
| `status` | PENDING, UPLOADED, SCANNING, READY, REJECTED, DELETED |
| `created_at` | Upload-session creation time |
| `expires_at` | Retention deadline, if applicable |
| `deleted_at` | Deletion request/completion timestamp |

Use a random or otherwise unguessable object key, for example:

`private/tenant-42/2026/10/7f1d...random-id`

Do not use a user-provided filename as the storage key. Store the original filename separately for display after sanitizing it. Avoid putting personal information in object keys because keys can appear in logs, audit records, and operational tooling.

The database should store the object key, not a permanent public URL. The application can generate an authorized download URL when needed.

## 6. Upload flow: proxy upload vs direct upload

### Option A — Proxy upload through the application

```text
Browser -> PHP API -> Object Storage
```

The PHP server receives the bytes and forwards them to storage.

**Advantages**
- The server controls the complete request body.
- Existing server-side validation can run before storage.
- Simple to understand for small files and low traffic.

**Disadvantages**
- Uses application bandwidth, memory, worker time, and connections.
- Large or slow uploads can occupy PHP workers.
- Scaling upload throughput also puts pressure on application servers.

### Option B — Direct upload with a presigned URL

```text
1. Browser -> API: ask for upload permission
2. API -> Database: create pending upload record
3. API -> Browser: return short-lived presigned URL
4. Browser -> Object Storage: upload bytes
5. Browser -> API: report completion
6. API/worker -> Object Storage: verify object
```

A **presigned URL** is a time-limited permission to perform a specific storage operation. The recipient does not need the storage provider's long-term credentials.

Important security properties:
- Treat the URL as a **bearer credential**: anyone who obtains it may use it while it remains valid, within the signed permissions.
- Keep its lifetime short and scope it to one intended object key and method.
- Do not log it, put it in analytics, or expose it in referrer-bearing pages.
- Use a new random object key for each upload attempt. Reusing a key can overwrite an existing object in some object stores.
- Do not assume a presigned URL makes a file safe. Verify the completed object on the server.
- A URL's expiry and permissions depend on the provider and the credentials/policy used to sign it.

AWS documents that a presigned S3 URL allows upload without giving the recipient AWS credentials, but it is limited by the creator's permissions; uploading to an existing key replaces that object. See [AWS — Uploading objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html).

## 7. Validate the object after upload

Never trust only the browser's filename, extension, or `Content-Type` header.

A safe flow is:

1. Authenticate the user and authorize the upload.
2. Enforce per-user and per-tenant quotas before issuing upload permission.
3. Generate an opaque object key on the server.
4. Set the expected maximum size and permitted content types in the upload policy where supported.
5. After upload, inspect the object using server-side metadata and, where necessary, read its bytes.
6. Detect the actual file type using trusted file-signature/MIME inspection.
7. Verify size and checksum.
8. Parse the file using safe libraries and enforce format-specific limits (for images, dimensions and pixel count).
9. Scan or quarantine files when the threat model requires it.
10. Only mark the object READY after all required checks pass.

A declared content type is just a claim. A file named `photo.jpg` may contain something else. Validation should be based on the actual content and the application's allowed formats.

For public downloads, use safe content types and consider `Content-Disposition: attachment` for untrusted files. Do not serve untrusted active content from the same origin as a sensitive application unless the isolation and content-handling model has been designed deliberately.

## 8. Large files and multipart uploads

For a large object, a single upload can be slow and fragile. Multipart upload divides the object into parts.

```text
Client -> API: initiate upload
API -> Storage: create multipart upload
Storage -> API: upload ID
API -> Client: upload ID + authorized part URLs

Client -> Storage: part 1
Client -> Storage: part 2
Client -> Storage: part 3
        ...
Client -> API: complete request + part receipts
API -> Storage: complete multipart upload
Storage -> API: completed object
```

Benefits:
- Parts can be uploaded in parallel.
- A failed part can be retried without retransmitting the whole file.
- Uploads can be paused and resumed if the application persists the upload ID and part information.

AWS recommends considering multipart upload for objects around 100 MB or larger; this is an AWS recommendation, not a universal threshold for every provider or workload. See [AWS — Multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html).

**Important cost and cleanup detail:** incomplete multipart uploads can leave billable parts behind. Explicitly complete or abort them, and configure a lifecycle rule to abort abandoned multipart uploads. Ordinary object-expiration rules do not necessarily remove incomplete multipart parts. See [AWS — Abort incomplete multipart uploads with a lifecycle rule](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html).

## 9. Integrity: checksums and ETags

A successful transfer should not be treated as proof that the application uploaded the intended bytes.

A checksum is calculated from data and can be compared with a checksum calculated at the destination. If the values differ, the data may have been corrupted or the wrong content may have been uploaded.

Common algorithms include CRC-based checksums and cryptographic hashes such as SHA-256. Choose an algorithm supported by the provider and appropriate to the integrity/security requirement.

**Do not assume an S3 ETag is always an MD5 hash of the complete object.** Multipart uploads and some encryption modes have different ETag behavior. Use the provider's explicit checksum features when verifying content integrity.

Read: [AWS — Checking object integrity](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html).

## 10. Download flow and authorization

A secure download should not expose the bucket or object to everyone.

```text
User -> Application API: request download for upload_id
Application API:
  1. authenticate user
  2. load metadata by upload_id
  3. verify owner / tenant / role
  4. verify status is READY
  5. issue short-lived GET URL or proxy the file
User -> Object Storage: download using temporary URL
```

The application should authorize using its database and identity model, not merely because the user knows an object key.

A random object key reduces guessing but is **not authorization**. Never treat obscurity as a substitute for access control.

Use a short-lived download URL only after the authorization check. For highly sensitive content, consider proxying downloads through an authorization layer, disabling caching, and recording access events. The right design depends on the privacy and latency requirements.

## 11. Consistency, durability, and versioning

Object-store guarantees differ by provider and operation. Do not say “object storage is eventually consistent” or “all object stores are strongly consistent” as a universal rule.

Examples from current official documentation:
- Amazon S3 documents strong read-after-write and list consistency for object operations.
- Google Cloud Storage documents strong consistency for object read-after-write, read-after-delete, metadata reads, and listing. It also documents eventual consistency for granting or revoking access, and notes that publicly cached objects can remain stale until cache lifetime expires.

References:
- [Amazon S3 — Strong consistency](https://aws.amazon.com/s3/consistency/)
- [Google Cloud Storage — Consistency](https://docs.cloud.google.com/storage/docs/consistency)

**Why this matters:** even when an object delete is strongly consistent at the storage origin, a public CDN or browser cache may still serve a cached copy until its cache policy expires. Sensitive temporary objects should not be publicly cacheable.

### Versioning

Versioning can preserve previous object versions after an overwrite or delete, helping recover from accidental changes. It also increases storage use and can conflict with a strict deletion policy.

For temporary or privacy-sensitive uploads, decide whether versioning is appropriate. If it is enabled, the deletion workflow must account for every retained version, soft-delete behavior, replicas, and backups—not just the current object key.

## 12. Lifecycle, retention, and deletion

A lifecycle policy can automatically transition or delete objects based on age, prefix, tags, or other supported conditions.

Examples:
- Delete abandoned temporary uploads after a short retention period.
- Move infrequently accessed backups to a colder storage class.
- Delete old noncurrent versions after a defined period.
- Abort incomplete multipart uploads.

Lifecycle actions are generally asynchronous. Do not promise that an object disappears at the exact second its age threshold is reached.

Google Cloud Storage documents that lifecycle actions are performed asynchronously and may lag after an object meets a rule. It also documents soft-delete retention behavior and the effect of object versioning. See [Google Cloud — Object Lifecycle Management](https://docs.cloud.google.com/storage/docs/lifecycle).

For sensitive data, use application-level deletion and provider lifecycle rules as a safety net. Verify the deletion workflow against the provider's versioning, soft-delete, replication, retention-lock, backup, and cache behavior. Do not enable an immutable retention lock on data that must be deleted immediately unless the legal and product requirements explicitly permit it.

### Temporary-file workflow

For a temporary analysis file:

1. Create an upload record with a short expiry.
2. Upload to a private quarantine prefix.
3. Validate and process the object.
4. Delete the object after processing or failure, according to policy.
5. Mark the record as deleted only after the deletion operation is confirmed.
6. Run a periodic cleanup job for expired records and orphaned objects.
7. Use a provider lifecycle rule as a backstop, not as the only deletion control.
8. Account for versions, soft-deleted objects, incomplete multipart parts, replicas, logs, and caches.

## 13. Apply this to an AI Image Detector

For an image detector that processes uploads temporarily, object storage can be a short-lived staging area rather than a permanent image library.

A suitable high-level workflow is:

1. PHP API authenticates the user and creates an upload session.
2. API issues a short-lived upload permission for a private, random object key.
3. Browser uploads directly to a quarantine location.
4. A worker checks actual file type, size, dimensions, checksum, and any required malware/content controls.
5. The analysis worker reads the image through a narrowly scoped service identity.
6. The detector records the analysis result and required metadata in the database.
7. The worker deletes the original image and verifies deletion according to the storage provider's semantics.
8. A cleanup worker reconciles expired upload sessions and orphaned objects.
9. The UI shows the analysis result without retaining the original image.

If the product's policy requires processing without persistent image storage, use an in-memory or ephemeral local staging design instead of object storage. The architecture must follow the actual retention requirement; object storage is not mandatory just because it scales.

## 14. Security checklist

- Keep buckets/containers private by default.
- Use least-privilege identities for upload, read, list, and delete operations.
- Scope presigned URLs to one method and object key, with short expiry.
- Treat signed URLs as secrets; never log them.
- Generate random server-controlled object keys.
- Enforce quotas, size limits, and request rate limits.
- Validate actual bytes, not just file extension or declared MIME type.
- Use checksums to detect transfer corruption.
- Encrypt data in transit and at rest using provider-supported controls.
- Separate quarantine from approved content.
- Prevent cross-tenant reads and deletes.
- Apply safe download headers and cache policies.
- Audit administrative changes and sensitive reads.
- Define deletion across versions, soft delete, replicas, backups, caches, and incomplete uploads.

See [AWS — Security best practices for Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html) for provider-specific controls. Do not assume the exact controls or defaults are identical on other object-storage platforms.

## 15. Capacity and cost estimation

Estimate before selecting a storage class or scaling strategy.

Illustrative example:
- 100,000 uploaded images per day.
- Average image size: 2 MB.
- Average retention: 7 days.

Daily new data:

`100,000 × 2 MB = 200,000 MB ≈ 200 GB/day`

Approximate steady-state live data after seven days, ignoring retries, versions, replication, and overhead:

`200 GB/day × 7 days ≈ 1.4 TB`

This is a rough decimal estimate. Real capacity planning must include growth, multipart leftovers, versions, soft deletes, replication, metadata, and safety headroom.

Costs can include:
- stored bytes by storage class;
- PUT, GET, LIST, and other API requests;
- retrieval and early-deletion charges for some classes;
- data transfer and CDN egress;
- replication;
- versioned and soft-deleted objects;
- incomplete multipart parts.

For temporary files, short retention and reliable cleanup are usually more important than moving the objects to an archival tier.

## 16. Monitoring and SLOs

Useful metrics include:
- upload success and failure rate;
- upload completion latency and throughput;
- bytes uploaded per tenant;
- checksum mismatch rate;
- validation and malware-scan failure rate;
- pending/quarantine object count and age;
- orphaned objects and expired upload sessions;
- incomplete multipart upload count and age;
- download authorization failures;
- delete request success and deletion lag;
- storage bytes, request counts, egress, and cost.

Possible SLO examples (illustrative only):
- 99.9% of authorized small uploads complete within a defined time under expected network conditions.
- 99% of completed uploads are validated and moved to READY or REJECTED within 30 seconds.
- 100% of expired temporary-upload records are detected by the cleanup process within a defined cleanup window.

A lifecycle policy's eventual execution time should not be confused with an application-level deletion SLO.

## 17. Practical exercise

Design a secure file-upload service for an application with:
- 500,000 registered users;
- profile images up to 5 MB;
- documents up to 100 MB;
- private user files;
- direct uploads from browsers;
- temporary image-processing files that must be deleted after analysis.

Answer:
1. Which storage type would you choose for file bytes, and why?
2. What metadata belongs in MySQL?
3. Draw the upload and download flows.
4. What does the API check before issuing a presigned URL?
5. How do you prevent one user from overwriting another user's object?
6. How do you verify file type, size, and integrity after upload?
7. When would you use multipart upload?
8. What happens if the client uploads the object but never calls the completion endpoint?
9. How do you ensure expired temporary files are cleaned up?
10. What happens if object storage confirms deletion but a CDN still has a cached copy?
11. Which metrics would you monitor?
12. How would your design change if the product must never persist the original image?

**Stretch:** Estimate seven-day storage for 100,000 daily uploads averaging 2 MB. Then explain how your estimate changes if average retention becomes 30 days or if versioning is enabled.

## 18. Interview practice

**Question: Why store files in object storage instead of MySQL?**

A strong answer: object storage is optimized for large unstructured objects and API-based access, while a relational database is useful for transactional metadata, ownership, status, and queries. The database stores an object key and metadata; the object store holds the bytes. This is a workload choice, not a universal ban on database BLOBs.

**Question: What is a presigned URL?**

A strong answer: it is a time-limited signed permission to perform a specific storage operation without giving the client long-term storage credentials. It is a bearer credential, so it must be scoped, short-lived, protected, and followed by server-side verification.

**Question: How do you prevent unauthorized file access?**

A strong answer: keep storage private, authenticate the caller, authorize access to the metadata record, then issue a short-lived download URL or proxy the object. Random object keys alone are not access control.

**Question: How do you handle large, unreliable uploads?**

A strong answer: use multipart or resumable upload where supported, retry only failed parts, verify checksums, persist upload state, and clean up incomplete uploads.

**Question: How do you delete temporary files reliably?**

A strong answer: delete through the application workflow, verify the provider's deletion semantics, reconcile orphaned objects, configure lifecycle cleanup as a backstop, and account for versioning, soft delete, replicas, backups, and caches.

## Short recap

- Block storage exposes blocks; file storage exposes paths; object storage exposes objects through APIs.
- Keep large file bytes in object storage and transactional metadata in a database when that fits the workload.
- Direct uploads with short-lived presigned URLs reduce application-server bandwidth but introduce bearer-token risks.
- Validate the completed object; do not trust filenames or client-supplied MIME types.
- Use multipart/resumable uploads for suitable large or unreliable transfers.
- Use explicit checksums; an ETag is not always a whole-object MD5.
- Provider consistency, versioning, soft delete, lifecycle, and cache behavior differ.
- Design deletion as an end-to-end workflow, especially for temporary or sensitive data.

## Learning resources

1. [GeeksforGeeks — Block, Object, and File Storage in System Design](https://www.geeksforgeeks.org/system-design/block-object-and-file-storage-in-cloud-with-difference/) — introductory comparison; page reports last updated 16 October 2026.
2. [GeeksforGeeks — System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) — broad system design roadmap.
3. [AWS — What is Amazon S3?](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) — object concepts, storage classes, consistency, and management features.
4. [AWS — Uploading objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html) — temporary upload permissions.
5. [AWS — Multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html) — large uploads, part retries, checksums, and cleanup.
6. [AWS — Checking object integrity](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html) — checksums and ETag limitations.
7. [AWS — S3 security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html) — bucket and object security.
8. [AWS — Aborting incomplete multipart uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html) — prevent abandoned parts from accumulating.
9. [Google Cloud — Cloud Storage consistency](https://docs.cloud.google.com/storage/docs/consistency) — provider-specific consistency guarantees.
10. [Google Cloud — Object Lifecycle Management](https://docs.cloud.google.com/storage/docs/lifecycle) — deletion and storage-class transitions.
11. [Microsoft Learn — Azure Blob Storage object model](https://learn.microsoft.com/en-gb/azure/storage/blobs/storage-blob-object-model) — accounts, containers, and blobs.
12. [Google SRE Books](https://sre.google/books/) — free reliability, operations, monitoring, and secure-systems references.
13. [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) — general learning resources; not a dedicated object-storage course.

## Resource monitoring — checked 31 October 2026

| Resource | Current source-reported date/status | Comparison and action |
|---|---|---|
| [GeeksforGeeks System Design Tutorial](https://www.geeksforgeeks.org/system-design/system-design-tutorial/) | Reports **5 September 2026**. | Matches the date recorded in the repository's 20 October review. No material change to existing lesson content verified. |
| [GeeksforGeeks System Design article: Block, Object, and File Storage](https://www.geeksforgeeks.org/system-design/block-object-and-file-storage-in-cloud-with-difference/) | Reports **16 October 2026**. | Used as a direct reference for Day 19. The prior review did not record this specific article, so its appearance is not claimed to be a new change since that review. |
| [Codecademy Change Log](https://www.codecademy.com/article/change-log) | Shows a **16 September 2026** notice for SQL updates effective **30 September 2026**, and an Advanced Python 3.13 update scheduled for **31 July 2026**. | These updates were already visible in the 30 October review. No new change affecting existing system-design notes was verified. |
| [Codecademy Helpful Resources](https://help.codecademy.com/hc/en-us/articles/360045725434-Helpful-Codecademy-Resources) | Reports **24 February 2026**. | Same source date as earlier reviews; no action. |
| [Google SRE Books](https://sre.google/books/) | Still labels *Site Reliability Engineering, 2nd Edition* and *AI Engineering Reliable Operations* as new; no page update date is displayed. | Same labels were recorded in the 24 October review; no verified new change. |
| [Google Cloud Storage consistency](https://docs.cloud.google.com/storage/docs/consistency) | Reports **Last updated 2026-10-30 UTC**. | This is a newly checked provider-specific source date relative to the repository's 20 October changelog. No previous snapshot is available to identify exact textual changes. Current behavior is covered as a provider-specific example in Day 19; no claim is made that a general consistency concept in Day 12 became incorrect. |
| [AWS S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html), [multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html), and [security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html) | Current official documentation checked; the opened pages do not provide a comparable last-updated date for this review. | Used for current implementation guidance. No historical change claimed. |
| [Google Cloud Object Lifecycle Management](https://docs.cloud.google.com/storage/docs/lifecycle) | Current official documentation checked; no comparable last-updated date captured. | Used for lifecycle and asynchronous deletion guidance. No historical change claimed. |

**Monitoring outcome:** No verified material source-content change was established that requires rewriting an existing lesson. Source dates and current provider behavior are recorded carefully; without earlier page snapshots, a changed or newly observed date is not treated as proof of an exact content change. The existing Day 12 consistency explanation remains accurate at its general level. No existing lesson or `RESOURCE-CHANGELOG.md` entry was changed in this review. This Day 19 note records the date checked and the references used.
