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

## 5. Request Flows

### Uploading a Photo

1. The client sends an authenticated upload request.
2. The application validates the file type, size, and user's permissions.
3. The original image is uploaded to object storage, preferably through a controlled upload URL when appropriate.
4. The database records the photo metadata and object key.
5. A processing job is added to the message queue.
6. The API returns the upload status.
7. A worker generates thumbnails and resized variants, stores them in object storage, and updates processing status.
8. The CDN delivers eligible image variants to viewers.

The implementation should handle partial failures. For example, if the object upload succeeds but the database write fails, a cleanup process should detect and remove orphaned objects.

### Viewing the Feed

1. The client requests its feed.
2. The load balancer forwards the request to an application server.
3. The application authenticates the user and checks Redis for cached feed data.
4. On a cache miss, the application retrieves the relevant metadata from the database or an appropriate read replica.
5. The application returns feed metadata and authorized image URLs.
6. The CDN serves cached image variants where possible.

Private photos must use suitable authorization and caching rules to prevent unauthorized access.

## 6. Handling Traffic Spikes

A fivefold peak multiplier gives approximately 116 uploads per second and 1,157 feed views per second under the assumptions above.

To handle these peaks:

- Scale application server instances horizontally behind the load balancer.
- Use direct-to-object-storage uploads where appropriate to avoid routing large file bodies through application servers.
- Place resizing jobs on a queue instead of processing every image synchronously.
- Scale workers according to queue depth and processing latency.
- Use Redis and the CDN to reduce repeated database queries and image transfers.
- Apply upload size limits, rate limiting, timeouts, and backpressure to protect the system.
- Monitor load balancer connections, application worker saturation, database connections, cache hit rate, queue age, and storage errors.

The queue absorbs temporary bursts, but it does not create unlimited capacity. If incoming jobs permanently exceed worker throughput, the backlog will continue to grow.

## 7. Reliability and Security

- Run multiple application server instances across failure domains where supported.
- Use database backups, replication, and a tested restoration and failover plan.
- Configure object storage durability, appropriate access policies, and recovery controls.
- Authenticate users and enforce photo ownership checks.
- Validate file formats and sizes; do not trust client-supplied content types alone.
- Use HTTPS and short-lived authorized upload or download URLs where appropriate.
- Retry transient failures safely and make background jobs idempotent to prevent duplicate processing.
- Monitor error rates, latency, queue backlog, database load, and storage failures.

A read replica helps scale reads but is not a substitute for a backup.

## 8. Architectural Trade-Offs

### Object Storage vs. Database BLOBs

Object storage handles large image files efficiently and keeps the relational database focused on structured data. However, it introduces coordination between the database and object store, so the application needs cleanup and recovery procedures for partial failures.

### Synchronous vs. Asynchronous Image Processing

Synchronous resizing can return a completed image immediately, but it increases request latency and consumes application resources. Asynchronous workers keep uploads responsive and absorb bursts, but resized variants may not be ready immediately.

### Caching vs. Data Freshness

Redis and the CDN reduce repeated work and improve response times. However, cached feeds or images can become stale. Cache invalidation, versioned image URLs, and appropriate expiration policies help manage freshness.

### Read Replicas vs. Consistency

Read replicas increase read capacity but may lag behind the primary database. Operations that must immediately reflect a newly uploaded photo should use a suitable consistency strategy rather than assuming every replica is instantly up to date.

### Durability vs. Cost

Replicas, backups, and multiple image variants improve resilience and user experience but increase storage costs. Lifecycle policies, retention rules, and appropriate image formats can reduce costs without sacrificing required recovery guarantees.

## 9. Conclusion

SnapShare's design separates request handling, metadata storage, and image processing. The CDN and Redis reduce repeated read work, the load balancer distributes requests across stateless servers, object storage holds image binaries, and background workers process resizing jobs asynchronously.

The capacity estimates show that peak upload traffic and feed-read traffic impose different demands on the system. Scaling decisions should be validated through monitoring and load testing, with particular attention to upload concurrency, database connection capacity, queue throughput, and annual storage growth.