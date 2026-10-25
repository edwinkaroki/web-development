# SnapShare Photo App Scaling Plan

## 1. Assumptions

SnapShare is a photo-sharing app where users upload photos and view a feed containing photos from people they follow.

The estimates use the following assumptions:

- There are 10 million registered users.
- 10% of registered users are active each day.
- Each daily active user uploads one photo per day.
- Each daily active user views 50 feed pages per day.
- Each original photo averages 2 MB.
- Each photo has one thumbnail averaging 50 KB.
- One day contains 86,400 seconds.
- Peak traffic is estimated to be five times the average traffic.
- All photos are retained for one year, with no deletions or compression.
- Storage estimates use decimal units, where 1 TB equals 1,000 GB.
- These are planning estimates; actual traffic may vary throughout the day.

## 2. Traffic and Storage Calculations

### Daily active users

Daily active users = Registered users × Daily active percentage

10,000,000 × 10% = **1,000,000 daily active users**

### Uploads per second

Daily uploads = 1,000,000 users × 1 photo = 1,000,000 uploads per day.

Average uploads per second = 1,000,000 ÷ 86,400 = **11.6 uploads per second**.

### Feed views per second

Daily feed views = 1,000,000 users × 50 pages = 50,000,000 feed views per day.

Average feed views per second = 50,000,000 ÷ 86,400 = **579 feed views per second**.

Peak feed views per second = 579 × 5 = **approximately 2,895 feed views per second**.

### Photo storage per year

Original photos per year:

1,000,000 photos per day × 2 MB × 365 days = **730 TB per year**.

Thumbnails per year:

1,000,000 thumbnails per day × 50 KB × 365 days = **18.25 TB per year**.

Total photo storage per year:

730 TB + 18.25 TB = **748.25 TB per year**.

This estimate covers original photos and thumbnails only. It excludes metadata, backups, replicas, and temporary files.

## 3. Read-Heavy or Write-Heavy?

SnapShare is a **read-heavy system** because users view 50 feed pages for every photo they upload. The architecture should prioritize fast reads using a CDN, caching, and a database read replica. Uploads must still be reliable, and thumbnail processing should happen asynchronously so that users do not wait for image processing to finish.

## 4. Architecture Diagram

```text
                  +-------------------+
                  |   Mobile / Web    |
                  |      Clients      |
                  +---------+---------+
                            |
                   +--------v--------+
                   |       CDN       |
                   | Cached images   |
                   +--------+--------+
                            |
                   +--------v--------+
                   |  Load Balancer   |
                   +--------+--------+
                            |
                 +----------v----------+
                 |     App Servers     |
                 | API and feed logic |
                 +--+--------+------+--+
                    |        |      |
              +-----v---+ +--v---+  +v----------------+
              |  Cache  | |Primary| | Object Storage  |
              |  Redis  | |  DB   | | Originals and   |
              +---------+ +--+----+ | thumbnails      |
                              |      +-----------------+
                       +------v------+
                       | Read Replica|
                       +-------------+

 App Servers -------> Upload Queue -------> Thumbnail Worker
                                              |
                                              v
                                        Object Storage
```

## 5. Components and the Problems They Solve

- **Mobile/Web Clients:** Allow users to upload photos and request their personalized feeds.
- **CDN:** Delivers cached photos and thumbnails from locations closer to users, reducing latency and load on the origin servers.
- **Load Balancer:** Distributes incoming API requests across app servers so that one server does not become a bottleneck.
- **App Servers:** Handle authentication, upload requests, feed generation, and communication with the database and other services.
- **Cache (Redis):** Stores frequently accessed feed data and metadata to reduce repeated database queries.
- **Primary Database:** Stores structured information such as users, photo metadata, captions, and follow relationships, and handles writes.
- **Database Read Replica:** Handles suitable read queries to reduce the workload on the primary database.
- **Object Storage:** Stores large original photo files and generated thumbnails without filling the database with binary image data.
- **Upload Queue:** Holds thumbnail-generation jobs so that image processing does not block the user's upload request.
- **Thumbnail Worker:** Processes queued jobs, creates smaller thumbnail images, and saves them to object storage.

## 6. Photo Upload Flow

1. A user selects a photo and submits it through the mobile or web app.
2. The app server authenticates the user and validates the request, including the file type and size.
3. The server arranges for the original photo to be uploaded to object storage, ideally using a short-lived, authorized upload URL.
4. After a successful upload, the server records the photo's object key and metadata in the primary database.
5. The server adds a thumbnail-generation job to the upload queue.
6. The server confirms the upload to the user without waiting for thumbnail generation to finish.
7. A thumbnail worker retrieves the job from the queue and downloads or reads the original photo from object storage.
8. The worker creates a 50 KB thumbnail and stores it in object storage.
9. The worker updates the photo metadata to indicate that the thumbnail is ready, if the application tracks processing status.
10. The CDN can then deliver the original photo or thumbnail to users when they view it. Cached feed data should be invalidated or updated when necessary.

If a thumbnail job fails, it should be retried safely, with failed jobs eventually moved to a dead-letter queue for investigation. The system should avoid creating duplicate thumbnails when a job is retried.

## 7. Trade-Offs

### Trade-Off 1: Caching versus Freshness

Caching feed data and photo metadata improves response times and reduces database load. However, cached information can become stale when a user uploads a photo or follows someone new. The system needs cache expiration or invalidation rules, which add complexity.

### Trade-Off 2: Read Replicas versus Consistency

A database read replica increases read capacity and helps protect the primary database from heavy feed traffic. However, replication can lag behind the primary database, meaning a newly uploaded photo might not appear immediately in a feed. Important read-after-write operations may need to use the primary database.

### Trade-Off 3: Asynchronous Thumbnails versus Immediate Availability

Generating thumbnails in the background makes uploads faster and allows workers to scale independently. However, thumbnails may not be available immediately, and the system must handle queue delays, retries, and failed jobs.

### Trade-Off 4: Object Storage and CDN versus Cost and Complexity

Object storage and a CDN scale well for large image libraries and frequent image downloads. However, storage, bandwidth, CDN requests, and backups can become expensive, and the application must manage permissions, cache invalidation, and file lifecycle policies.

## Conclusion

SnapShare should separate photo files from structured database records, use object storage and a CDN for image delivery, and scale feed reads with caching and a read replica. A load-balanced group of app servers handles API requests, while a queue and thumbnail workers process images asynchronously. This architecture supports the estimated one million daily active users while allowing individual components to scale as traffic grows.