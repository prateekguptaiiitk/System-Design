![image.png](https://assets.leetcode.com/users/images/3c8e4f0d-ff78-4b06-adb5-d93082ecf4a2_1758994908.105136.png)

**Functional Requirements** – 
- Convert long URLs to short URLs
- Efficiently redirect short URLs to their long destinations
- Optional analytics
- Possibly allow custom short links and user accounts

**Non-Functional Requirements** – 
- High availability, minimal downtime
- Low latency
- Scalability to millions of URLs and redirects
- Security (prevent abuse or phishing)

**Capacity Estimation** –
- Estimate 100 million stored URLs/Day and 500 million redirects/Day
- Storage: Each mapping ~100 bytes, totalling ~10 GB/Day
- Read/write ratio: Much higher reads than writes (possibly 100:1)

**API Design** -
- ```POST /shorten``` - Accepts long URL, returns new short URL. (optional slug)
- ```GET /{short_url}``` — Redirects to long URL.
- ```GET /analytics/{short_url}``` — Provides stats for the short URL.

**High Level Architecture** – 
- **Load Balancer:** Directs traffic to multiple app servers.
- **App Servers:** Process requests (shortening, redirection, analytics).
- **Database:** Stores URL mappings (could be sharded).
- **Cache (e.g., Redis):** Fast access to popular URLs.
- **Analytics Pipeline:** Tracks clicks asynchronously to avoid bottlenecks.
- **Rate Limiter:** Prevents spam/abuse

**Workflows (Step by Step)** –

**Flow 1: Short URL creation**
- User sends a long URL to the service via API Gateway.
- The URL shortening service receives the request. URL shortening service -> ID generation service.
- The ID Generation Service creates a unique ID (sequential, random, or hash).
- The ID is encoded into a base62 string (short URL code). 
- The mapping of short code → long URL is persisted in URL DB..
- Short URL info is cached in URL cache for fast future lookups.
- The short URL is returned to the user

**Flow 2: URL Redirection**
- User clicks the short URL link.
- Request hits Load Balancer → API Gateway → Redirection service.
- Redirection service first attempts to lookup the short code in the redirection cache.
- On cache miss, it queries the URL DB for the mapping.
- Once the long URL is found, application returns an HTTP 301 redirect response.
- An asynchronous analytics service receives event data for click tracking.

**Flow 3: Analytics and Stats Collection**
- Each redirect or click generates an event that gets sent asynchronously (e.g., to message queues).
- An Analytics Service consumes these events.
- It aggregates and updates statistics like click count, geographic data, referrers in a dedicated analytics database or datastore.

**Component Deep Dive** -

**ID Generation Service** -
1. **Random Allocation (Random String Generation)**
- **Principle:** Instead of sequential IDs, generate a random string using the base62 alphabet (a-z, A-Z, 0-9, so 62 possible characters).
- **Example:** For a 7-char code, randomly select each character, giving ~3.5 trillion options. Generate, check for collisions (if the string is already used), and if unique, save the mapping. If not, retry.
- **Benefits:** Codes are unpredictable (not guessable), distributed generation is easier, and this maximizes the use of the code space.
- **Drawbacks:** Potential (though very low) for collisions as the space fills, adds latency due to collision checks. This can be minimized by using a larger code length, or by pre-generating and reserving unique codes in advance.

2. **Hash Function Techniques**
- **Principle:** Hash the long URL using an algorithm (such as MD5, SHA-256, or MurmurHash). Take as many bytes from the hash as necessary, then encode those bytes with base62 to create a compact short string.
- **Example:** Hash("verylongurl.com") → produces hash; take first/best N characters; convert to base62 string for the short URL.
- **Deterministic vs. Random:** If using just the hash, the same long URL always generates the same short URL (deterministic, good for deduplication). If non-determinism is needed, salt the hash or add random elements.
- **Collision Handling:** If two long URLs produce the same hash slice (possible with short outputs), resolve by either using more hash bits, salting, or appending an incrementing differentiator.
- **Benefits:** Stateless, fast, suitable for distributed environments, and can be deterministic.
- **Drawbacks:** Needs collision handling (though rare), and output code length can’t be too short or collision rates rise.

3. **Distributed Unique ID Generators**
- Use a distributed generator, like Twitter Snowflake, which encodes timestamp, node ID, and sequence into a large unique number, then encode it in base62. This enables massive parallel generation without clashes, at the cost of longer codes and more system complexity.

**Identifying and Resolving bottlenecks** - 
- **Single Points of Failure:** Add replicas for LB, DB, cache.
- Database scaling: Shard by short URL, use read replication and caching for HA.
- **Cache Eviction:** LFU/LRU policies; keep popular URLs in-memory.
- **Monitoring and Alerts:** Detect slow redirects, DB downtime, and abusive traffic.
- **Security:** Validate URLs (no malware/phishing), implement DNS blacklist if needed.
