![image.png](https://assets.leetcode.com/users/images/2d0deeed-11d2-4443-a6a0-ec2c75848ce3_1759080742.7426355.png)

**Functional Requirements** - 
- User authentication and registration
- Ability to create, read, update, and delete posts (text, images, video)
- Following/friendship features
- Newsfeed/timeline aggregation
- Likes, comments, shares, notifications
- User search, hashtag search
- Media upload and serving

**Non-functional Requirements** - 
- High availability and reliability
- Scalability (support millions-billions of users)
- Security, privacy, abuse/spam prevention
- Low latency (<1s per feed load)

**Capacity Estimation** –
- Estimate 500 million total users, 1 million DAU
- Each user makes 10 feed views/page requests and 2 posts/Day (total 2 images + 1 video per day)
- Posts can be text (average 300 bytes), image (200 KB), and video (10 MB, but less frequent)
- Photo uploads: 2 million/Day = 2 x 200 GB/day = 400 GB/day
- Video uploads: 1 million/Day = 1 x 10 TB/day = 10 TB/day
- Total space required for 10 years - (400 GB/day + 10 TB/day) x 365 x 10 = ~38 PB

**API Design** -
- ```POST /posts ```- create post
- ```POST /posts``` - create user
- ```GET /feed``` - get user’s newsfeed/timeline
- ```POST /user/follow``` - add follower/friend
- ```POST /user/unfollow``` - remove follower/friend
- ```GET /search``` - user or hashtag search

**High Level Architecture** – 
- **Load Balancer:** Directs traffic to multiple app servers.
- **Microservices:** for modularity and independent scaling
- **Database:** Independent DBs for all microservices(could be sharded).
- **Cache (e.g., Redis):** For feeds, posts, users, etc.
- **Object Storage/CDN:** For media.
- **Rate Limiter:** Prevents spam/abuse

**Workflows (Step by Step)** –

**Flow 1: Create/Update User**
- Fetch user profile or update user details as requested (name, bio, photo).
- Handles privacy setting changes—updates visibility for profile and content.

**Flow 2: Create/Update Post**
- User creates post → validation → save content/meta to DB → trigger media service if media is present.
- Media Service - 
    - Handles media upload: Receives file (or presigned URL), validates, stores in blob storage (e.g., S3).
    - Returns the file URL associated with the post in the DB.
    - Upon retrieval, it serves the file via the CDN, ensuring fast and scalable delivery.
- Edits/deletes handled via update/delete requests.
- Provides details of posts for feeds or timelines.

**Flow 3: Add/Remove Friend/Follower**
- Handles follow/unfollow requests -> Updates relationship graph.
- Responds to “get friends/followers” and mutual friend queries for social features or recommendations.

**Flow 3: Search User/Post** 
- User enters query -> search service indexes and fetches results (users, hashtags, posts) via text search.
- Shows paginated/typed results.

**Component Deep Dive** -

**1. Ranking and Newsfeed Generation -**

**Push Model (Fanout-on-write)**
- When a user (e.g., A) creates a new post, the system identifies all their followers (users B, C, D…) and immediately adds the post reference to each follower’s newsfeed “inbox” or cache.
- Pros: Fast reads, as user feeds are pre-populated.
- Cons: Heavy write amplification for celebrities with millions of followers and large storage use.
- Used in Twitter-like systems for regular users.

**Pull Model (Fanout-on-read)**
- When a user logs in, the feed is generated on-the-fly by aggregating the latest posts from all accounts they follow.
- Pros: Less write overhead and flexible for users with few followers.
- Cons: Slower feed generation for users following many accounts; expensive reads.
- Used by Facebook/Instagram for users with huge follow lists.

**Hybrid / Smart Fanout**
- Use push for “normal” users and pull for high-follower-count users (celebrities).

**Feed Aggregation and Ranking Flow** - 
- **Get Followees:** Retrieve IDs of all people/entities the user is following.
- **Aggregate Posts:** Fetch recent posts by those followees.
- **Rank Posts:** Score each post with:
    a. Recency (freshness)
    b. User affinity and engagement probability
    c. Content type, diversity, global popularity (“trending”)
    d. Cross-platform signals (past interactions, shares, etc.)
- **Apply Filters:** Remove blocked users, hidden/spam content, or ads based on relevance.
- **Paginate & Cache:** Return and cache the top-N posts as the current page of the feed.

**Caching and Pre-Generation**
- Frequently accessed feeds (especially for active users) are cached in memory for fast access.
- As the user scrolls (“infinite scroll”), more posts are pulled, ranked, and streamed.
- If a new post appears (from someone the user follows), the cache is invalidated or instantly updated.

**Optimisation and Scaling**
- **Batch/Trigger Workers:** Periodically refresh popular users’ feeds.
- **Job Queues/Notifications:** Use event queues to signal new content.
- **Sharding:** Distribute feed data and posts by user ID or geographic region for scale.
- **Asynchronous Processing:** Feed generation and ranking run asynchronously; the user receives updates via polling or push notifications.

**2. Database Choices** - 
- Use a relational DB for user data and authentication to ensure strong consistency and complex queries.
- Choose a NoSQL distributed store, like Cassandra or DynamoDB, for posts and feed storage, to handle massive write amplification and horizontal scaling.
- For social graph data, employ a graph DB or wide-column store specialised for relationship traversal.
- A fast caching layer, such as Redis will offload critical low-latency feed read operations.
- Search queries are powered by Elasticsearch or similar.
- Sharding - 
    - Shard primarily by user ID to keep user-related data localised, reducing cross-shard joins.
    - Utilise secondary sharding on post IDs or timestamps for write scalability.
    - Design caches to shard by user ID for load balancing.

**Identifying and Resolving bottlenecks** - 
- Replication for high availability (read replicas)
- Caching (Redis, Memcached) for hot feeds/posts
- Rate limiters, abuse/spam monitoring
- Asynchronous workers for feed generation, notification, etc.
