![image.png](https://assets.leetcode.com/users/images/d3f74020-8ad1-4e30-ac2b-826e457d9893_1759399775.0418859.png)


**Functional Requirements** - 
- Users can search for nearby places (restaurants, gyms, shops, events) or people using their current location.
- Support filters (distance, category, ratings, open status, personalised recommendations).
- View detailed business/place info, photos, reviews, and ratings.
- Add/edit/review places and profiles.
- Bookmark or save favourites.

**Non-functional Requirements** - 
- Millisecond-level search response.
- High scalability: millions of users, tens of millions of locations.
- High availability (99.99% uptime).
- Eventual consistency for new/updated locations/reviews

**Capacity Estimation** - 
- Monthly Active Users (MAU): 50 million. Daily Active Users (DAU): ~10 million (20% of MAU)
- Average Searches/User/Day: 5
- Peak QPS (Queries Per Second): 100,000 QPS
- Reviews Added/Day: ~5 million
- New Businesses/Places Added/Day: 100,000
- Favourites Saved: 100 million total
- 100 million listings; ~2 KB per listing ⇒ Total: ~200 GB
- 1 billion historical reviews; ~1 KB per review ⇒ Total: ~1 TB
- Average 5 photos/business, 200 KB each ⇒ ~100 TB
- 100 million favorites; ~50 bytes each ⇒ 5 GB

**API Design** -
- ```/search?lat&lon&radius&category&open_now``` -  returns sorted nearby locations.
- ```/place/:id``` - get details.
- ```/review/:placeId/add``` - add review.
- ```POST /place/add``` - For business owners to list a new location (includes details like name, address, coordinates, hours, category, description, and initial photos).
- ```PUT /place/:id/edit``` - For owners/admins to update address, hours, category, etc.
- ```POST /place/:id/claim``` - For owners to verify and take control of an existing listing.

**High Level Architecture** – 
- **Client (Mobile/Web):** End users searching, reviewing, and bookmarking places; business owners managing listings.
- **API Gateway:**  Entry point for all client-server communication. Handles authentication, routing, throttling, and monitoring.
- **Cache:** Holds hot searches, popular places, trending zones in-memory for fast retrieval.
- **Load Balancer:** Distributes incoming connections to app servers
- **App Servers:** Handle listing, review, finding, etc.
- **Message Queue / Pub-Sub (Kafka/Redis):** Async update of search indexes
- **Database:** Stores location info, user profiles, business info
- **Media Storage:** Saves shared files in blob storage/CDN
- **Segments Producer:** Responsible for communicating with the third-party world map data services (for example, Google Maps). Takes this data and divides the world into smaller regions called segments. The segment producer helps us narrow down the number of places to be searched.
- **QuadTree Servers:** Set of servers that have trees that contain the places in the segments. Finds list of places based on the given radius and the user’s provided location and returns that list to the user. This component mainly aids the search functionality.
- **Aggregators:** QuadTrees accumulate all the places and send them to the aggregators. Aggregators aggregate the results and return the search results to the user.

**Workflows (Step by Step)** –

**Flow-1: Business/Place Listing Workflow**
- Business owner submits new place details via ```/place/add```.
- Listing Service validates data and stores location (geo-indexed).
- Media (photos) uploaded via Media Service, linked to the place.
- Admin/Moderation Service may review and approve the listing.
- Once approved, listing is searchable and visible to users.

**Flow-2: User Proximity Search Workflow**
- User requests ```/search?lat&lon&radius&filters```.
- API Gateway routes to Geospatial Search Service.
- Search Service -> QuadTree servers to find all the places that fall within the given radius. 
- QuadTree servers -> results to the aggregators to refine them and send them to the user.
- Results are filtered, ranked, and returned (with photos, ratings).

**Flow-3: Write a Review Workflow**
- User submits a review via ```/review/:placeId/add```.
- Review Service validates and stores review; updates average rating.
- Review moderated if flagged, then displayed with the listing.

**Flow-4: Media Upload & Retrieval Workflow**
- User/business uploads photos/media.
- Media Service stores files in object storage; links them to place or user.
- Requests for place details retrieve media URLs, served via CDN.

**Component Deep Dive** -

**Searching** - 

**SQL Solution**
- Stores places with latitude and longitude in separate indexed columns.
- Queries for nearby places use range queries on lat/long.
- Not efficient at large scale (e.g., 500M places) because separate indexes return huge candidate sets with costly intersections.
- Uniform indexing on lat/long fails to capture spatial locality well.

**Grids**
- Divides map into fixed-size grids; each place assigned a GridID.
- Nearby query searches only the grid containing the location plus 8 neighbours.
- Index on GridID reduces query scope drastically versus lat/long alone.
- Grid size typically matches query radius.
- Simple mapping of location to grid is easy.
- However, grid size is static, so dense regions may have large grids with many places, hurting performance.
- Let search radius = 10 KM, total area of earth = 200 million KM => 20 million grids. Need 4 bytes to store grids and 8 bytes for LocationID, we need = (4 * 20M) + (8 * 500M) ~= 4 GB to store index.

​​​​​**​Dynamic Size Grids (QuadTree)**
- Uses a tree structure where each node represents a spatial grid that subdivides when exceeding a place count threshold (e.g., 500).
- QuadTree adapts grid size based on data density—dense areas subdivided many times, sparse areas less so. (QuadTree- a tree with 4 children)
- Leaf nodes hold the place list for their spatial region.
- Enables efficient queries by visiting only relevant tree nodes covering the search area, minimising scanned points.
- Dynamically balances granularity for optimal performance over uneven distributions.

**Building the QuadTree**
- Start with a single root node: This node covers the entire globe or a designated large area.
- Split nodes with >500 locations: When a node exceeds the limit, subdivide into four quadrants, distributing locations based on their coordinates.
- Repeat recursively: Continue splitting each node until all leaf nodes contain fewer than or equal to 500 places.

**Finding the Grid for a Location**
- **Begin at the root node:** Check if the node has children.
- **Descending the tree:** If children exist, navigate down to the child node that contains the location (based on latitude and longitude).
- **Leaf node:** When reaching a node with no children, this is the target grid (leaf node) containing the location.

**Finding Neighbouring Grids**
- **Linked list of leaf nodes:** Connect all leaf nodes with a doubly linked list to traverse neighbours easily.
- **Parent node approach:** Keep parent pointers in each node. To find neighbours, ascending the tree to gather sibling nodes and their neighbours is possible.
- **Expansion of search:** Expand outwards through neighbouring leaf nodes by traversing linked lists or moving up through parent pointers.

**Search Workflow**
- **Locate the grid:** Find the leaf node containing the user's location.
- **Check the number of places:** If sufficient places are present, return results.
- **Expand to neighbours:** If not enough, traverse neighbouring leaf nodes via linked list or parent pointers until reaching the necessary places or max radius.

**Memory Considerations**
- **Location data:** For 500 million places, with 12 bytes per place (LocationID + Lat/Long), approximately 12 GB is needed.
- **Number of grids:** 500M locations/ 500 places = 1 million leaf nodes.
- **Internal nodes:** Estimated at one-third of leaf nodes, requiring around 10 MB. Each internal node will have 4 pointers (for its children) and each pointer is 8 bytes.
- **Total memory:** Roughly 1M * 1/3 * 4 * 8 = 10 MB 12.01 GB, which fits into modern servers.

**Insertion of New Places**
- **Single server case:** Insert directly into the database and into the appropriate leaf node in the QuadTree.
- **Distributed case:** Locate the target grid/server for the new place and insert it accordingly, updating the QuadTree structure to maintain balance and correctness.

**Ranking Search Results** - 

**Storing Popularity in QuadTree**
- Each place has an aggregated popularity score (e.g., average star rating out of ten).
- This popularity score is stored both in the backend database and cached in the QuadTree nodes where places reside.
- When running a search within a radius, each QuadTree partition (leaf node) returns its top 100 places ranked by popularity.

**Aggregating Results**
- The search aggregator collects these top 100 results from multiple QuadTree partitions covering the radius.
- It then merges and selects the top 100 places overall, combining proximity and popularity in the final ranking.

**Managing Popularity Updates**
- Since popularity updates are not frequent, dynamically updating the QuadTree for every rating change would be resource-intensive and could degrade search performance.
- Instead, popularity scores are updated once or twice daily during off-peak hours when system load is low.
- This batch update simplifies maintenance, reduces locking contentions or resource drain on the QuadTree, and is acceptable since eventual consistency on popularity scores is typically not critical for user experience.

**Replication and Fault Tolerance** - 

**Master-Slave Setup for QuadTree Servers**
- **Read distribution:** Slave replicas serve only reads, offloading query traffic from the master.
- **Write path:** All writes (insertions or updates) go through the master first, which then asynchronously replicates to slaves.
- **Eventual consistency:** A slight delay (few milliseconds) in propagation to slaves is acceptable.

**Failure Handling and Failover**
- **Single server failure:** A secondary replica (slave) can take over as master after failover, ensuring system availability with the same QuadTree structure.
- **Simultaneous failure:** If both primary and secondary servers fail, a new server must be provisioned and the QuadTree rebuilt.

**Efficient QuadTree Rebuild with Reverse Index**
- **The challenge:** Without knowing precisely which places belong on a failed server, brute-force scanning of the entire dataset using a hash function is slow and blocks queries.
- **Solution:** Maintain a reverse index mapping between Places and QuadTree servers.
- **Structure:** A dedicated QuadTree Index server holds a HashMap where keys are QuadTree server IDs and values are HashSets of Place IDs (LocationID + Lat/Long) stored on those servers.
- **Benefits:** When rebuilding a server, it requests exact place sets from the Index server for fast recovery.
- **Fast add/remove:** Using HashSets allows quick updates as places are added/removed.
- **Index Server Fault Tolerance:** The QuadTree Index server is itself replicated. If it fails, it can rebuild its index from the main database.

**Data Partitioning** - 

**Sharding Based on Regions**
- **Approach:** Divide places into fixed geographic regions (e.g., zip codes) so that all places in a region are stored on a dedicated server.
- **Advantages:** Simple conceptually; spatial locality preserved as all nearby places are on the same server.
- **Challenges:**
    - **Hot Regions:** Popular or dense regions receive high query traffic that can overwhelm the respective server.
    - **Unequal Data Growth:** Some regions will accumulate more places over time, causing imbalanced storage and processing loads.
- **Mitigations:** Repartitioning the data or applying consistent hashing to spread load when regions become too large or hotspots form.

**Sharding Based on LocationID (Hash-based Sharding)**
- **Approach:** Use a hash function on LocationID to map places to servers, storing places uniformly distributed across servers irrespective of geographic proximity.
- **QuadTree Construction:** Each server builds a QuadTree over its assigned places. Since places are shuffled by hash, the QuadTree structures differ across servers.
- **Querying:** To find nearby places, queries are broadcast to all servers, each returning local nearby places. A central aggregator merges results and returns the best matches.
- **Advantages:** Achieves uniform data distribution automatically, balancing loads across servers.
- **Drawbacks:** Query spreads to all servers, potentially increasing query overhead and latency.
- **Handling Tree Variance:** Different QuadTree structures across partitions won’t cause problems because the search covers all partitions for neighbouring grids within the radius.
