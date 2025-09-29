![image.png](https://assets.leetcode.com/users/images/db51fb6c-2688-45dc-801b-6cc1d61f38da_1759170130.8916526.png)

**Functional Requirements** - 
- 1:1 and group messaging (threaded conversations)
- Online/offline status
- Message delivery/read receipts
- Media (images, videos, voice notes) sharing
- Message persistence (sync across devices)
- Push notifications for offline users

**Non-functional Requirements** - 
- High scalability (millions of users, billions of messages)
- Low latency (real-time delivery)
- High availability and reliability
- Message ordering and consistency
- End-to-end encryption/security

**Capacity Estimation** - 
- Estimate 500 million DAU
- Each user sends 20 messages/day
- Total messages per day = 500 x 20 = 10 billion messages/day
- Typical message payload: 500 bytes (text). Total text message data/day = 10 billion/day x 500 bytes = 5 TB/day
- If 5% of messages include media (avg size 200 KB) = 10 billion/day x 5% x 200 KB= 100 TB/day.
- Message history for 1 year = 5 TB/day x 365 = 1825 TB ≈ 1.8 PB
- Media storage for 1 year = 100 TB/day x 365 = 36500 TB ≈ 36.5 PB
- Peak online users: ~100 million concurrent connections
- Peak throughput = 10 billion / 86400 ~ 115740 messages/sec at peak load

**API Design** -
- ```POST /send``` - send message
- ```GET /messages?since={timestamp}``` - fetch chat since a particular time
- ```POST /upload``` - upload and send media files
- ```POST /group/create, POST /group/send``` - group actions
- ```GET /presence``` - get status

**High Level Architecture** – 
- **Client (Mobile/Web):** Maintains a persistent connection (WebSocket/long polling) with servers
- **Load Balancer:** Distributes incoming connections to app servers
- **App Servers (Chat/Presence):** Handle messaging, status, and group logic
- **Message Queue / Pub-Sub (Kafka/Redis):** Decouple delivery across multiple servers
- **Presence Service:** Tracks users' online/offline status
- **Push Notification Service:** Delivers notifications to offline users
- **Database:** Stores messages, user profiles, group info, unread status, media URLs
- **Media Storage:** Saves shared files in blob storage/CDN, referenced by messages

**Workflows (Step by Step)** –
**Flow 1: 1-1 Message**
- Sender sends message (via WebSocket) to Chat Service.
- Chat server -> sends async event to Message Storage Service to persist to DB -> sends in real-time to all recipients (via sockets)
- If any recipient is offline, the message sent to Notification Service -> adds the message to their inbox/unread table
- Delivery/read receipts update status notified to sender
- Presence changes trigger notifications or status updates

**Flow 2: Group Message**
- Sender sends message (via WebSocket) to Chat Service.
- Chat Service verifies it as group chat -> sends to Group Service.
- Chat Service -> sends async event to Message Storage Service to persist to DB -> sends in real-time to all recipients (via sockets)
- If any recipient is offline, the message sent to Notification Service -> adds the message to their inbox/unread table
- Delivery/read receipts update status notified to sender
- Presence changes trigger notifications or status updates

**Flow 3: Media Sharing**
- Media uploaded to blob storage/CDN by Media Service.
- Message contains a reference to a media URL, served directly to recipients.
- Secure URLs (presigned) for private access.

**Component Deep Dive** -
**Messages Handling** -

1. **Pull Model (Polling)**
- Clients regularly ask for new messages from server.
- Simple but inefficient: frequent empty responses waste bandwidth and increase server load.
- Introduces latency proportional to polling interval.

2. **Push Model (Persistent Connection)**
- Clients establish persistent connections (long polling or WebSockets) with the server.
- Server pushes new messages instantly on these connections.
- Minimises latency and redundant resource use.

**Long Polling -**
- Client sends request not expecting server to respond immediately.
- Server does not send empty response, instead holds it open until new data or timeout.
- When data arrives, server responds and client sends a new long poll request.
- Improved over polling, but still overhead from repeated requests and reconnections.

**Connection Management by Server** -
- Server maintains a hash map: UserID → connection/session object (WebSocket or long poll).
- When a new message arrives for a user, server looks up the connection and sends message immediately.
- Requires handling connection lifecycle: timeouts, disconnections, reconnections.

**Real-World Practice** - 
- WebSockets are preferred now for chat apps due to:
    - Truly full-duplex, low-overhead persistent connections.
    - Real-time bi-directional communication without repeated HTTP headers.
- Long Polling is fallback or legacy solution.

**Message Sequencing** - 
The problem of message sequencing in chat systems arises because relying solely on server timestamps cannot guarantee that both users see the messages in the same logical order in a conversation. Here’s a brief explanation:
- Two users (User-1 and User-2) might send messages to each other around the same time.
- The server receives messages at different times (e.g., User-1's message M1 at time T1, User-2's message M2 at time T2 where T2 > T1).
- Due to network and processing delays, User-1 might see messages in order M1 then M2, while User-2 sees M2 then M1.
- This difference causes inconsistent conversation views between users.

To solve this, systems assign a sequence number per client for messages, indicating the order in which messages were sent from that user's perspective. Each client orders messages by these sequence numbers, ensuring a consistent message order across all devices for that user. While different users may see messages interleaved differently, the ordering is stable and logical per user.

**Additional Technique** - Lamport timestamps or Vector clocks can be used in more complex scenarios for distributed systems ensuring causal ordering.

**Database Choices** - 
The choice of wide-column store databases like HBase for chat message storage is a very good fit considering the requirements.

**Why HBase (Wide-Column Store)?**
- **High write throughput for small updates:**
HBase buffers writes in memory and flushes them in batches, avoiding the overhead of many small row writes typical in RDBMS or document stores.
- **Efficient sequential reads:**
Data is stored sorted by key, enabling fast range scans (e.g., fetching messages in chronological order for a conversation).
- **Scalable and distributed:**
Designed to scale horizontally, handling billions of rows and massive throughput, running on Hadoop clusters.
- **Flexible schema:**
Can store variable-length messages and metadata in columns, suited for chat messages with varying-sized content and properties.
- **Dealing with small, frequent updates:**
By storing message streams keyed by conversation ID + timestamp or sequence, HBase efficiently appends and scans chat history.

**Why not RDBMS or typical NoSQL?**
- RDBMS like MySQL are costly for high-volume small writes and have limited horizontal scaling ability.
- Document stores (MongoDB) have higher write/read cost per document, less optimised for range queries by clustered keys.

**Managing User Status** - 
The proposed approach is an effective optimisation for managing presence notifications in a large-scale chat system:
1. **Initial Status Pull:** When the client app opens, it fetches the current online/offline status of all friends. This bulk pull avoids constant real-time pushes for every status on app startup.
2. **Delayed Online Broadcast:** When a user comes online, the server broadcasts this status update after a short delay. This prevents rapid online/offline fluctuations (like quick app toggling) from causing excessive network traffic.
3. **Viewport-based Pulling:** Clients pull presence status only for users currently visible on their screen (in chat windows or contact lists). This targeted pulling reduces unnecessary updates and bandwidth usage.
4. **On-demand Status Fetch:** When a user opens a new chat, the client fetches the latest presence status of the chat participant to provide fresh information.

**Data Partitioning** - 

**Shard by User ID (or Conversation ID)**
- Partition users’ data based on user ID hash or range, so all data related to a user (messages, metadata, unread counts) resides on the same shard.
- For 1-to-1 chats, messages belong to the two users; assign messages to the shard of the user receiving them or maintain consistency by conversation ID.
- For group chats, use the group ID as shard key to localise group message storage and queries.

**Why Shard by User/Conversation ID?**
- Ensures data locality: most read/write operations relate to a single user or conversation.
- Simplifies data retrieval for chat history and online status.
- Reduces cross-shard join complexity.
- Enables even data distribution and load balancing based on hashing.

**Additional Techniques**
- Use consistent hashing to minimise remapping during cluster resizing.
- Combine with time-range partitioning within shards to efficiently purge or archive old messages.
- Careful design to handle hot users by splitting very active chat logs into sub-shards if needed.

**Fault Tolerance  and Replication** - 

**What will happen when a chat server fails?**

**Key Design Considerations**
- **Stateless servers:** Simplify failover without session loss.
- **Shared persistence:** All message and state data stored in databases or queues decoupled from server instances.
- **Heartbeat and health checks:** Fast detection of failure.
- **Client retry logic:** Ensures continuous service with minimal user impact.

1. **Connection Loss Detection:** Clients detect the severed WebSocket or long-poll connection due to server failure via heartbeat timeouts or TCP disconnect.
2. **Failover and Reconnection:** Clients automatically attempt to reconnect, typically routed by load balancers or DNS to other healthy chat server instances.
3. **Session Recovery:** Since chat servers are mostly stateless or hold minimal transient state, the new connected server retrieves the user's context (presence, session info) from shared stores or caches.
4. **Message Resynchronisation:** The client fetches any missed or undelivered messages from persistent storage (message DB or queue) to restore conversation state.
5. **Presence Updates:** User presence status is updated to offline temporarily upon disconnect, then online upon successful reconnection.
6. **Load Redistribution:** Other servers in the cluster absorb the failed server’s load to maintain overall service availability and responsiveness.

**Scaling and Reliability** -

- Horizontally scale app servers; stateless design allows easy addition/replacement
- Use Redis/Kafka for pub/sub to synchronise messages across servers
- Shard databases by user/chat ID for scale
- Replicate data for high availability
- End-to-end encryption for privacy; keys exchanged between client devices
