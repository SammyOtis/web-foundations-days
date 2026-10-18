# SnapShare System Architecture & Scaling Plan

This document outlines the system architecture and capacity planning for SnapShare, a photo-sharing application handling user uploads and social feed generation.

## 1. Assumptions & Baseline Metrics

Using an approximation of $1\text{ day} \approx 100,000\text{ seconds}$ (exact: 86,400 seconds):

- **Total Registered Users:** 10,000,000
- **Daily Active Users (DAU):** 10% of 10,000,000 = **1,000,000 DAU**
- **User Activity:**
  - 1 photo upload per DAU per day
  - 50 feed views per DAU per day
- **File Sizes:**
  - Original Photo: 2 MB
  - Compressed Thumbnail: 50 KB (0.05 MB)
  - Total Storage per Upload: 2.05 MB

## 2. Traffic & Storage Estimates

### Uploads per Second

- **Total Daily Uploads:** $1,000,000\text{ DAU} \times 1\text{ upload} = 1,000,000\text{ uploads/day}$
- **Average Uploads/Sec:** $\frac{1,000,000\text{ uploads}}{100,000\text{ seconds}} =$ **10 uploads/sec**
- **Peak Uploads/Sec (5× multiplier):** $10 \times 5 =$ **50 uploads/sec**

### Feed Views per Second

- **Total Daily Feed Views:** $1,000,000\text{ DAU} \times 50\text{ views} = 50,000,000\text{ feed views/day}$
- **Average Feed Views/Sec:** $\frac{50,000,000\text{ views}}{100,000\text{ seconds}} =$ **500 views/sec**
- **Peak Feed Views/Sec (5× multiplier):** $500 \times 5 =$ **2,500 views/sec**

### Storage Estimates

- **Daily Storage Added:** $1,000,000\text{ uploads} \times 2.05\text{ MB} = 2,050,000\text{ MB} = 2.05\text{ TB/day}$
- **Yearly Storage Added:** $2.05\text{ TB/day} \times 365\text{ days} =$ **748.25 TB/year** (approximately **750 TB/year**)

## 3. Read vs. Write System Characteristics

SnapShare is heavily **read-heavy**, with a read-to-write ratio of **50:1** (50 feed views for every 1 photo upload).

**Design Implications:**
Because reads vastly outnumber writes, the system must prioritize caching strategies (CDN for media assets, Redis/Memcached for feed data) and database read-replicas to serve high query volumes without bottlenecking primary database writes.

## 4. Media Storage Strategy

Photos must **not** be stored directly inside relational databases as BLOBs because large binary files bloat database size, corrupt cache efficiency, slow down database backups/restores, and consume expensive database connections during file transfers.

Instead, raw image files and thumbnails are stored in **Cloud Object Storage** (such as AWS S3 or Google Cloud Storage), while the relational database only stores metadata records containing string URLs/URIs pointing to those stored files.

## 5. System Architecture Diagram

                               +-------------------+
                               |    Client App     |
                               +---------+---------+
                                         |
                                         v
                               +-------------------+
                               |        CDN        |
                               +---------+---------+
                                         |
                                         v
                               +-------------------+
                               |   Load Balancer   |
                               +---------+---------+
                                         |
                       +-----------------+-----------------+
                       |                                   |
                       v                                   v
             +-------------------+               +-------------------+
             |    App Server 1   |               |    App Server 2   |
             +----+----+---------+               +----+----+---------+
                  |    |                              |    |
       +----------+    +--------------+    +----------+    +--------------+
       |                              |    |                              |
       v                              v    v                              v

+--------------+ +------------------+ +------------------+
| Redis Cache | | Object Storage | | Message Queue |
+--------------+ | (Photos/Thumbs) | +--------+---------+
| +------------------+ |
v v
+--------------+ +------------------+
| Primary DB | | Worker Service |
| (Writes) | | (Thumbnails) |
+------+-------+ +------------------+
|
v
+--------------+
| Read Replica |
| (Reads) |
+--------------+

## 6. Component Responsibilities

CDN (Content Delivery Network): Caches static photo and thumbnail files at edge locations globally to reduce download latency for users.

Load Balancer: Distributes incoming HTTP traffic across multiple application servers to prevent any single server from becoming overwhelmed.

App Servers: Executes core business logic, handles API authentication, processes feed queries, and validates uploads.

Redis Cache: Stores pre-computed user feeds and frequent database queries in memory to deliver low-latency read performance.

Primary Database (SQL): Handles transactional write operations such as user accounts, follow relationships, and photo metadata.

Database Read Replica: Replicates data from the primary database to serve read-only feed metadata queries concurrently.

Object Storage (S3): Provides scalable, low-cost blob storage specifically designed for binary media files like raw photos and thumbnails.

Message Queue (RabbitMQ/SQS): Buffers asynchronous tasks like image processing jobs so app servers can respond to upload requests immediately.

Worker Service: Consumes jobs from the message queue in the background to resize uploaded images and write compressed thumbnails back to object storage.

## 7. Step-by-Step Photo Upload Flow

Upload Initiation: The client sends an HTTP POST request containing the image file and user metadata to the Load Balancer, which routes it to an active App Server.

Direct Storage: The App Server streams the raw photo file directly into Object Storage and receives a storage object key/URL.

Queue Notification: The App Server publishes an asynchronous event job (containing the photo ID and Object Storage key) to the Message Queue.

Immediate Client Response: The App Server writes the initial metadata entry (status: "processing") to the Primary Database and returns a 202 Accepted response to the client application.

Background Thumbnail Processing: A Worker Service picks up the job from the Message Queue, downloads the original photo from Object Storage, generates a 50 KB thumbnail, and uploads the thumbnail back to Object Storage.

State & Feed Update: The Worker updates the photo metadata record in the Primary Database (status: "ready", adding the thumbnail URL) and invalidates or appends the new photo ID to relevant follower feed caches in Redis.

## 8. Architectural Trade-Offs

Asynchronous Processing vs. Real-Time Thumbnail Availability:

Trade-off: Offloading thumbnail generation to a background worker queue ensures fast API upload response times for users, but introduces eventual consistency where a newly uploaded photo's thumbnail might not appear in a follower's feed for a few seconds.

Push vs. Pull Feed Model (Fan-out on Write vs. Fan-out on Read):

Trade-off: Pre-computing feeds and pushing photo IDs directly into Redis caches upon upload makes feed reads extremely fast (critical for a 50:1 read-heavy system), but significantly increases background write/processing overhead when high-profile users with millions of followers upload a photo.
