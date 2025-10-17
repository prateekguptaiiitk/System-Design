![image.png](https://assets.leetcode.com/users/images/0c4bec88-05db-48d8-b6bf-f7336b6601f0_1760682499.9387467.png)

**Functional Requirements** - 
- Upload, download, and delete files/folders
- Share files/folders with permissions
- Sync changes across devices
- Support offline editing and changes be synced when online

**Non-functional Requirements** - 
- High durability (data never lost)
- High availability and reliability
- Scalability (millions of users, billions of files)
- Secure file access and transfer
- Efficient storage usage

**Capacity Estimation** - 
- Let’s assume that we have 500M total users, and 100M daily active users (DAU).
- Let’s assume that on average, each user connects from three different devices.
- On average if a user has 200 files/photos, we will have 100 billion total files.
- Let’s assume that average file size is 100KB, this would give us ten petabytes of total storage. 100B * 100KB => 10PB
- Let’s also assume that we will have one million active connections per minute. 

**API Design** -
- To upload a file
```
POST /files
Body: {
  file, 
  fileMetadata
}
``` 
- To download a file
```
GET /files/{fileId} -> File & Metadata
Body: {
  file, 
  fileMetadata
}
```
- To share a file
```
POST /files/{fileId}/share -> File & Metadata
Body: {
  users: [], // users to share the file with,
  permissions: []
}
```
- For clients to query for changes to files on the remote server
```
GET /files/{fileId}/changes -> FileMetadata[] 
```

**High Level Architecture** – 
- **Client (Mobile/Web)**: End users uploading, downloading, sharing and syncing files.
- **API Gateway**:  Entry point for all client-server communication. Handles authentication, routing, throttling, and monitoring.
- **Cache**: Holds hot files/chunks
- **Load Balancer**: Distributes incoming connections to app servers
- **App Servers**: Handle upload, download, sync, etc.
- **Message Queue / Pub-Sub (Kafka/Redis)**: Async update of metadata and file services and trigger notification service
- **Database**: Stores file metadata
- **Block Storage**: Saves files in object storage/CDN
- **Chunker**: Splits files into smaller pieces called chunks and reconstructs files from those chunks. By detecting and transferring only the modified parts of files during edits, the chunker reduces both bandwidth usage and synchronization time when saving to cloud storage.
- **Watcher**: Monitors the local workspace for user actions—such as creation, deletion, or updates to files and folders—and notifies the indexer of these changes. It also listens for updates from other clients, received via the synchronization service, to keep the local workspace in sync.
- **Indexer**: Processes events from the watcher and updates the local metadata database with information about modified file chunks. After successfully uploading or downloading these chunks to the cloud storage, the indexer communicates with the synchronization service to broadcast changes to other clients and update the remote metadata database.
- **Internal Metadata Database**: Keeps track of all the files, chunks, their versions, and their location in the file system. 

**Workflows (Step by Step)** –

**Flow-1: File Upload Workflow**

- Client's watcher detects local file change and notifies indexer.
- Indexer splits the file, identifies changed chunks.
- Client requests upload permission and presigned URLs from the file service (checks with metadata service).
- Client uploads chunks directly to object storage (e.g., S3).
- Storage notifies file service of successful upload.
- File service triggers async update to metadata service to store file/chunk info.
- Sync service notifies other clients via notification/sync service.
- Subscribed clients get notified and update their local state with the new file/chunk metadata.

**Flow-2: File Download/View Workflow**
- Client requests file info from file service. File service gets file data from metadata service.
- Metadata service checks access permissions.
- Client receives presigned URL from file service.
- Client downloads file or chunks directly from object storage using presigned URL.

**Flow-3: File Sharing Workflow**
- Client requests to share a file/folder (specifies users/permissions)
- Sharing service sends async request to metadata service to check permissions and update access control list for the file in metadata DB
- On successful update, notification service informs target recipients of new access.
- Recipients' devices update local views with shared file info..

**Flow-4: File Sync Across Devices Workflow**
- Client’s watcher monitors local folder.
- Change triggers chunking, upload, and metadata updates, as in the upload flow.
- Sync service check permissions and validates changes -> queues change request to central request queue (kafka) -> triggers file service and metadata service for updating remote view -> response queues that correspond to individual subscribed clients deliver update messages to -> notification service that broadcast file change notifications via WebSocket/long-polling/push.
- Subscribers are notified and download changed chunks.

**Flow-4: File Delete Workflow**
- Client deletes file/folder locally.
- Client notifies sync service and updates metadata service.
- Metadata service marks file/folder as deleted.
- Notification service broadcasts delete to other devices.

**Component Deep Dive** -

**File Storage**  -
- For metadata, a NoSQL database like DynamoDB works well because the metadata is loosely structured and the main query pattern is by user—though a SQL database like PostgreSQL would also suffice. 
- Storing files directly on backend servers is simple but doesn't scale or provide reliability. Blob Storage services (like S3) are the scalable, reliable industry solution. 
- The best file upload pattern is to let the client request a presigned URL from your backend, then upload directly to blob storage (like S3), avoiding double-upload (s3 and backend storage), decreasing backend complexity and avoid backend storage bloat. 
- Store the metadata with status "uploading" when generating the URL, then on successful upload (verified by S3 notification), the backend marks metadata as "uploaded." 
- This model is scalable, reliable, and avoids many issues found in naive backend-only storage approaches.
- Downloading files through the backend is inefficient because it duplicates data transfer and increases latency and costs.
- The optimal approach is to use presigned URLs that allow clients to download files directly from blob storage, bypassing the backend for the file transfer. 
- To further improve download speed for users across diverse locations, a CDN (Content Delivery Network) can cache popular files closer to users. 
- The CDN serves the file via a time-limited, secure presigned URL. Strategic use of CDN cache control and invalidation ensures only frequently accessed files are cached, optimizing both cost and performance.

**Sharing Files** - 
- **Basic Approach**: Each file’s metadata contains a sharelist — a list of user IDs who can access it. While easy to implement, it is inefficient for listing all files shared with a user since it requires scanning all files’ sharelist.
- **Improved Cached Mapping**: 
    - Maintain an additional mapping of users to the files shared with them, such as a key-value cache:
user1: ["fileId1", "fileId2"]
    - This allows quick lookups but requires synchronizing both mappings (the file’s sharelist and the user's shared file list). A transaction-based update can keep these consistent.
- **Fully Normalized Table Approach**: Create a dedicated SharedFiles table mapping userId to fileId:
![Screenshot 2025-10-17 113208.png](https://assets.leetcode.com/users/images/71cd8b46-a662-4a6f-965f-77eeebc1be4a_1760680955.4569824.png)

**Synchronizing Files** - 
- The Synchronization Service is central to file sync architecture: it processes file updates from clients, validates permissions and changes, checks metadata consistency, broadcasts notifications to subscribers, and syncs local client databases with the authoritative remote metadata.
- Efficient synchronization is achieved by using chunk-based differencing—clients and servers only transmit changed file parts, verified using chunk hashes, which minimizes bandwidth and storage usage.
- The architecture employs a scalable messaging middleware (like queues), which supports asynchronous, high-throughput communication. 
- Clients use a global Request Queue to submit updates; the Synchronization Service takes updates from this queue and processes metadata changes. 
- Per-client Response Queues deliver update notifications to each subscribed client, enabling real-time sync and efficient offline update handling. 
- This design ensures scalable, reliable, and bandwidth-efficient synchronization across all user devices.

![image.png](https://assets.leetcode.com/users/images/40e7fc15-710e-45d5-b5aa-8e0df44f5bee_1760682745.3836727.png)

**Data Deduplication** - 
Data deduplication eliminates duplicate copies of data to optimize storage and reduce network usage.
- **Post-process deduplication**: Stores new chunks first, then scans for duplicates later; fast for clients but temporarily wastes bandwidth and storage during upload.
- **In-line deduplication**: Calculates chunk hashes in real-time; if a duplicate is found, stores only a reference and skips the full chunk upload, optimizing both bandwidth and storage immediately but may add client-side processing delay.Data deduplication is used to eliminate duplicate data, saving storage and reducing data transfer.​
- **Post-process deduplication**: Data is stored first, and duplication is analyzed later. This avoids client delays but temporarily wastes some resources.

**Metadata Partitioning** - 
To scale the metadata database, partitioning is required:
- **Vertical Partitioning**: Stores related tables (e.g., user data vs. file/chunk data) on different servers. This is simple but doesn’t solve scale issues for massive tables and creates challenges for performing joins across databases.
- **Range-Based Partitioning**: Partitions large tables based on a range value (like the first letter of the file path). This can lead to unbalanced partitions if data is skewed.
- **Hash-Based Partitioning**: Uses a hash of an attribute (like file ID) to distribute data evenly across partitions. This is more balanced, but can still create hotspots, which consistent hashing can help resolve.​

The most scalable approach for billions of records is hash-based partitioning, optionally enhanced with consistent hashing to avoid overloaded partitions.

**Storing Large Files** - 
When dealing with large file uploads, two key user experience considerations are crucial:
- **Progress Indicator**: Users should see the upload progress in real-time, keeping them informed of how much has completed and how much remains.
- **Resumable Uploads**: Users must be able to pause and resume uploads, allowing interrupted uploads (due to connection loss or browser closure) to resume instead of restarting from scratch.

**Challenges with Large File Uploads via Single POST**
- Web servers and browsers have timeout and payload size limits (often <2GB for servers, <10MB for some API gateways).
- Large file uploads can be very slow (e.g., 50GB over 100 Mbps can take over an hour).
- Network interruptions can cause full restarts and poor user experience.
- Users get no feedback during long uploads without chunked progress.

**Chunking Approach**
- Break files into smaller 5-10 MB chunks client-side before uploading.
- Upload chunks individually, enabling tracking and reporting upload progress per chunk.
- Maintain a chunks field in the server-side file metadata, recording each chunk's status (uploaded, uploading, not uploaded).
- Clients query metadata on existing chunks during resume to upload only missing chunks.

**Synchronization of Chunk State**
- **Client-driven orchestration**: Client sends chunk uploads to S3, then PATCHes backend metadata to update chunk status.
- **Risk**: Client-trusted status can be faked, causing inconsistent metadata.
- **Better**: Server-side verification using chunk ETags validated against S3 ListParts API to ensure chunk authenticity.

**File and Chunk Identification**
- A fingerprint is a mathematical calculation that generates a unique hash value based on the content of the file. This hash value, often created using cryptographic hash functions like SHA-256.
- Use a fingerprint (hash) of the entire file as the file ID to uniquely identify the file.
- Use fingerprints for each chunk to track which parts are uploaded.
- This allows efficient detection of resumed uploads and avoids duplicating chunk uploads.

**Overall Upload Flow**
- Client chunks file and calculates fingerprints.
- Client checks for existing file metadata using file fingerprint for resume capability.
- If new, client requests presigned upload URLs from backend and backend stores metadata with chunk list, all marked "not-uploaded".
- Client uploads chunks to S3 using presigned URLs.
- After each upload, client sends chunk status and ETag to backend; backend verifies against S3 before updating metadata.
- Once all chunks are uploaded, backend marks file as "uploaded".
- Client UI reflects progress throughout.
- This approach balances user experience, efficiency, and data integrity in large file uploads.

**Security** - 
- **Encryption in Transit**: Use HTTPS to encrypt data moving between clients and servers, protecting against eavesdropping during transfer. This is standard and supported by modern browsers.
- **Encryption at Rest**: Enable server-side encryption in storage like Amazon S3. Files are encrypted using unique keys and stored encrypted. Keys are stored separately to prevent unauthorized decryption even if someone gains file access. Options include SSE-S3 (Amazon-managed keys), SSE-KMS (AWS Key Management Service), and SSE-C (customer-provided keys).
- **Access Control**: File metadata includes ACLs (like share lists) to restrict file access to authorized users only.
- **Preventing Unauthorized Access via Shared Links**: Use signed URLs to generate temporary, secure download links valid only for a limited time (e.g., 5 minutes). These URLs include encrypted signatures tied to URL, expiry time, and optionally IP restrictions.
- **Integration with CDN**: Signed URLs also work with CDNs (e.g., CloudFront), ensuring cached file delivery remains secure and time-limited.
- **Validation**: When a user requests a file, the CDN or storage validates the signed URL’s signature and expiry before serving the content.
This layered approach safeguards files both in transit and at rest, enforces strict access control, and mitigates risks from leaked download links.

**Cache** - 
- To handle hot files and chunks efficiently in block storage, a cache layer like Memcached can be used to store whole chunks indexed by their IDs or hashes. Since memory is limited (e.g., 144GB per server can store roughly 36,000 chunks), employing a Least Recently Used (LRU) cache eviction policy is suitable. LRU discards the chunk that has been accessed least recently when the cache is full, ensuring that frequently accessed chunks stay in cache to optimize performance.
- Similarly, a cache layer can be applied to the Metadata Database to improve metadata retrieval times and reduce load on the primary DB.
- Memcached is a distributed in-memory cache using hashing to distribute data across nodes, fast lookups, and LRU eviction, widely used to accelerate storage and metadata access in large-scale systems.​
