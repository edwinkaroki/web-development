# Day 7 Assignment: Scaling a Photo-Sharing App — SnapShare

## 1. Overview

SnapShare is a photo-sharing application that allows users to upload photos, view feeds, and interact with shared content. The system must support 1 million daily active users while remaining performant, reliable, and cost-effective.

## 2. Capacity Estimation

### Assumptions

- Daily active users: 1,000,000
- Average uploads per user per day: 2
- Average feed views per user per day: 20
- Peak traffic multiplier: 5× average traffic
- Average original image size: 5 MB
- Seconds per day: 86,400

These are planning assumptions, not measured production statistics.

### Daily Uploads and Peak Upload Rate

Daily uploads:

1,000,000 × 2 = **2,000,000 uploads/day**

Average uploads per second:

2,000,000 ÷ 86,400 = **23.15 uploads/second**

Peak uploads per second:

23.15 × 5 = **approximately 116 uploads/second**

The system should plan for bursts of at least 116 uploads per second under these assumptions. The load balancer must distribute requests, application servers must handle concurrent uploads, and the background queue must absorb bursts of image-processing jobs.

### Feed Reads

Daily feed views:

1,000,000 × 20 = **20,000,000 feed views/day**

Average feed views per second:

20,000,000 ÷ 86,400 = **approximately 231 views/second**

Peak feed views per second:

231 × 5 = **approximately 1,157 views/second**

Feed traffic is more frequent than upload traffic in this example, so caching and read scaling are important.

### Annual Original-Image Storage

Daily original-image storage:

2,000,000 × 5 MB = **10 TB/day**

Annual original-image storage:

10 TB × 365 = **3,650 TB, or approximately 3.65 PB/year**

This estimates decimal storage for original images only. It excludes resized variants, metadata, replication, backups, and storage overhead. Actual storage requirements depend on image compression, retention, and upload patterns.

## 3. High-Level Architecture

```text
                       Users / Mobile App
                               |
                         DNS and CDN
                               |
                         Load Balancer
                               |
                  +------------+------------+
                  |                         |
             App Server 1              App Server 2
                  |                         |
                  +------------+------------+
                               |
                  +------------+------------+
                  |                         |
              Redis Cache              Message Queue
                  |                         |
                  |                    Image Workers
                  |                         |
                  +------------+------------+
                               |
                      Relational Database
                               |
                         Read Replica

          App Servers / Workers
                    |
              Object Storage
          (Originals and Variants)
```

### Component Responsibilities

- **DNS:** Directs clients to SnapShare's service endpoint.
- **CDN:** Delivers cached images and static assets from locations closer to users.
- **Load balancer:** Distributes requests across healthy application servers and performs health checks.
- **Application servers:** Authenticate users, validate uploads, enforce permissions, create metadata records, and generate authorized access to images.
- **Redis cache:** Stores frequently accessed feed data and metadata to reduce repeated database reads.
- **Relational database:** Stores user accounts, photo metadata, ownership, object keys, and processing status.
- **Object storage:** Stores original photos and resized variants without placing large image binaries in ordinary database rows.
- **Message queue:** Buffers image-processing jobs during traffic spikes.
- **Image workers:** Generate thumbnails and resized versions asynchronously.
- **Read replica:** Handles suitable read queries to reduce load on the primary database.

Application servers are stateless, so requests can be handled by any healthy instance. Sessions and durable application data are stored in shared services rather than in individual servers.

## 4. Image Storage Strategy

SnapShare should store image files in object storage and keep lightweight references in its relational database.

For example, a photo record may contain:

- `photo_id`
- `user_id`
- `original_object_key`
- `thumbnail_object_key`
- `file_size_bytes`
- `width`
- `height`
- `created_at`
- `processing_status`

An object key might look like `photos/user_42/photo_1001/original.jpg`. It identifies the object; it does not contain the image itself.

### Why Not Store Image Binaries Directly in the Database?

**Buffer pool memory pressure:** Relational databases use buffer pools to cache database pages. Large binary values can increase I/O and compete with frequently accessed metadata for memory, depending on the database engine and storage design. Separating images allows the database cache to focus on users, photo records, and feed queries.

**Indexing and query overhead:** Indexes are most useful for compact structured fields such as user IDs, timestamps, and photo IDs. Image bytes are not useful as ordinary index keys, and large binary values can increase row-management and storage overhead. Keeping binaries outside the database lets queries work primarily with lightweight metadata.

**Backup size and recovery time:** Including all image binaries in relational database backups can substantially increase backup size and restoration time. Keeping image objects separate allows the database and object storage to use appropriate backup, durability, and lifecycle policies independently.

The application must still coordinate the two systems. Failed uploads, deleted database records, or orphaned objects should be handled using retries, cleanup jobs, and reconciliation.

## 5. Step-by-Step Photo Upload Flow

SnapShare separates photo storage from image processing so that users can upload photos without waiting for every thumbnail to finish generating.

1. **Client initiates upload:** The user selects a photo. The client sends an authenticated request to the application server to initiate the upload.
2. **Authentication and validation:** The application verifies the user's identity, checks upload permissions, and validates the file size and allowed image formats.
3. **Upload to object storage:** The application issues a short-lived, authorized upload URL. The client uploads the original image directly to object storage, reducing bandwidth and processing pressure on the application servers.
4. **Create the database record:** After the upload succeeds, the application creates a photo record in the relational database containing the photo ID, user ID, object key, dimensions, file size, and processing status.
5. **Enqueue a background job:** The application publishes a thumbnail-generation job to the message queue. The job contains the photo ID and object key, not the entire image binary.
6. **Process the image:** A background worker consumes the job, retrieves the original image from object storage, generates the required thumbnail and resized variants, and saves them back to object storage.
7. **Update processing status:** The worker updates the database record to indicate that processing has completed and records the object keys for the generated variants. If processing fails, the job can be retried safely, with repeated attempts prevented from creating duplicate results.
8. **Deliver the photo:** The application returns photo metadata and authorized image URLs to the client. The CDN can serve eligible resized images to viewers, reducing repeated requests to the origin storage.

### Failure Handling

The database and object store do not normally share a single atomic transaction. If an upload succeeds but creating the database record fails, the system should retry or clean up the orphaned object. If publishing a queue job fails after the database write, a transactional outbox or retry mechanism can ensure the job is eventually published. Failed jobs should be retried with limits and moved to a dead-letter queue for investigation when necessary.

The photo record should distinguish between states such as `uploading`, `processing`, `completed`, and `failed`, so clients can understand the current status.

## 6. Architectural Trade-Offs

### Trade-Off 1: Synchronous vs. Asynchronous Thumbnail Generation

**Synchronous processing** generates thumbnails during the original upload request. Its advantage is that the resized images are ready before the request completes. However, image processing increases response time and consumes application resources. During peak upload traffic, this can exhaust application workers and reduce the number of requests the system can handle.

**Asynchronous processing** stores the original image first and sends a job to a message queue. Background workers generate thumbnails independently. This keeps uploads responsive and allows worker capacity to scale separately. The disadvantage is that thumbnails may not be available immediately, and the queue requires monitoring and retry handling.

**Decision:** SnapShare uses asynchronous thumbnail generation because it separates expensive image processing from the user-facing upload request and handles bursts more effectively.

### Trade-Off 2: Primary Database vs. Read Replicas

The primary database handles writes and provides the authoritative record of each photo. Read replicas can serve feed and metadata queries, reducing read pressure on the primary database.

However, replication is often asynchronous. A newly uploaded photo may exist in the primary database before it appears on a replica. A user who immediately refreshes their feed could temporarily fail to see the photo.

**Decision:** Use the primary database for writes and read replicas for suitable high-volume reads. For read-after-write operations, read from the primary database or use a consistency strategy that ensures the new record is visible before relying on a replica.

### Trade-Off 3: Caching vs. Freshness

Redis and the CDN reduce repeated database queries and image transfers. This improves latency and reduces infrastructure load. However, cached feeds or image metadata may become stale after a photo is uploaded, updated, or deleted.

**Decision:** Invalidate affected feed-cache entries after writes, use appropriate expiration policies, and use versioned image URLs for updated image variants. Private images must have suitable authorization and caching rules.

### Trade-Off 4: Object Storage vs. Database BLOBs

Storing image binaries in object storage keeps the relational database focused on compact, searchable metadata. This reduces pressure on database memory, query performance, and database backups.

The trade-off is that metadata and image files live in separate systems. Partial failures can create missing references or orphaned files.

**Decision:** Store image binaries in object storage and references in the relational database. Use retries, reconciliation jobs, and cleanup policies to handle partial failures.

## 7. Conclusion

SnapShare uses stateless application servers behind a load balancer, a CDN for image delivery, Redis for frequently accessed data, a relational database for metadata, object storage for image binaries, and a queue with background workers for thumbnail generation.

The capacity estimates indicate approximately 116 peak uploads per second under the stated assumptions. The architecture addresses that load by separating upload handling from image processing and scaling application servers and workers independently.

The design balances responsiveness, consistency, reliability, and cost. Its effectiveness should be validated through load testing and monitoring of upload latency, application-server saturation, queue backlog, database performance, and storage growth.