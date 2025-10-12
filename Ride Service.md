![image.png](https://assets.leetcode.com/users/images/50d7af40-a801-4309-a5df-f5389351692e_1760281247.1325123.png)

**Functional Requirements** - 
- Riders can request rides by specifying pickup and destination.
- Get fare estimate and estimated time of arrival (ETA).
- Match rider to nearest available driver.
- Drivers can accept or decline ride requests.
- Real-time location sharing and tracking for drivers and riders.
- In-app payments and ratings for rides.
- Notifications for ride status updates.

**Non-functional Requirements** - 
- Low latency for real-time updates and matching.
- Scalability to handle millions of concurrent users.
- High availability and fault tolerance.
- Data consistency for ride and payment state.

**Capacity Estimation** - 
- Let’s assume we have 300M customers and 1M drivers with 1M daily active customers and 500K daily active drivers.
- Let’s assume 1M daily rides.
- Let’s assume that all active drivers notify their current location every three seconds. 

**API Design** -
```
POST /fare -> Fare
Body: {
  pickupLocation, 
  destination
}
```
\- returns a Fare object with the estimated fare and eta. We use POST here because we will be creating a new ride object in the database.

```
POST /rides -> Ride
Body: {
  fareId
} 
```
\- riders confirm their ride request after reviewing the estimated fare
	
```
POST /drivers/location -> Success/Error
Body: {
  lat, long
} 
```
\- Note the driverId is present in the session cookie or JWT and not in the body or path params
\- Driver sends current GPS coordinates

```
PATCH /rides/:rideId -> Ride
Body: {
  accept/deny
} 
```
\- allows drivers to accept a ride request

```
POST /payments/charge -> Success/Error
Body: {
  rideId 
} 
```
\- Initiate payment after ride completion using third-party services.

**High Level Architecture** – 
- **Client (Mobile/Web)**: End users searching for rides, drivers accepting/rejecting ride requests
- **API Gateway**:  Entry point for all client-server communication. Handles authentication, routing, throttling, and monitoring.
- **Load Balancer**: Distributes incoming connections to app servers
- **App Servers**: Handle riders, drivers, location, fare estimation, etc.
- **Message Queue / Pub-Sub (Kafka/Redis)**: Async ride requests
- **Database**: Stores location info, user profiles, ride info
- **Segments Producer**: Responsible for communicating with the third-party world map data services (for example, Google Maps). Takes this data and divides the world into smaller regions called segments. The segment producer helps us narrow down the number of places to be searched.
- **QuadTree Servers**: Set of servers that have trees that contain the places in the segments. Finds list of places based on the given radius and the user’s provided location and returns that list to the user. This component mainly aids the search functionality.
- **Aggregators**: QuadTrees accumulate all the places and send them to the aggregators. Aggregators aggregate the results and return the search results to the user.

**Workflows (Step by Step)** –

**Flow-1: Ride Request Workflow**

- Rider inputs pickup and drop-off locations -> sends ride request to ride service.
- Ride Service request -> Third Party Mapping API to calculate the distance and travel time between the pickup and destination locations and then applies the company's pricing model to generate a fare estimate.
- Service then returns the Fare entity to Rider Client so they can make a decision whether to accept the fare and request a ride.
- User confirms their ride request in the client app -> POST request to ride service with the id of the Fare they are accepting.
- Ride Service receives the request -> triggers matching service so that we can assign a driver to the ride.
- Meanwhile, at all times, drivers are sending their current location to the location service, updating QuadTree servers with their latest location lat & long.
- Matching Service determines ranked list of eligible drivers, selects top optimal driver (based on proximity, ETA, preferences).
Notification Service alerts driver about the incoming ride request.
- Driver accepts/declines the request, acceptance is confirmed back to the rider via Notification Service.
- If declined, system will send a notification to the next driver on the list.
- Ride Service receives the request -> updates status of the ride to "accepted" and updates the assigned driver accordingly. Then returns the pickup location coordinates to the Driver Client.
- With the coordinates in hand, the Driver uses on client GPS to navigate to the pickup location.

**Flow-2: Real-Time Ride Tracking Workflow**
- Both rider and driver apps maintain persistent (often WebSocket/Long polling) connections to the backend.
- Location Service receives frequent location updates from both parties.
- Backend pushes live updates to the counterpart (rider sees driver approach, driver sees rider’s pickup point).
- The frontend updates the UI in real time to show vehicle position, ETA, and route on the map.

**Flow-3: Payment Workflow**
- When the ride is marked “completed” by the driver or backend, the Ride Service triggers Payment Service.
- Payment Service interacts with third-party payment providers to process the charge.
- Upon success/failure, the outcome is saved in the database and pushed via Notification Service to both user and driver.
- Receipts are generated, and earnings are updated accordingly.

**Component Deep Dive** -

**Frequent Driver Location Updates**  -

- The existing QuadTree-based Dynamic Grid from the Yelp design struggles with Uber’s real-time driver location updates. The main problems are:
    - **Frequent updates overhead:** Active drivers report locations every 3 seconds. Updating the QuadTree for each change requires finding the driver’s previous grid, removing them if they moved out of it, reinserting into the new grid, and potentially repartitioning if the grid’s capacity is exceeded.
    - **Slow location propagation:** The system must quickly send nearby drivers’ current locations to active customers, and also update the driver and passenger during a ride. While QuadTree accelerates nearby search, it does not guarantee fast updates.
- Instead of updating the QuadTree on every driver location change, we can store the latest positions in a hash table called DriverLocationHT and refresh the QuadTree less frequently (e.g., every 15 seconds). This approach keeps real-time data in the hash table for quick access while reducing the high update overhead on the QuadTree.
- Each driver’s record in the DriverLocationHT stores their ID (3 bytes - 1 million), old latitude (8 bytes), old longitude (8 bytes), new latitude (8 bytes) and new longitude (8 bytes), requiring 35 bytes per driver, excluding hash table overhead. Bandwidth for getting ID and new location (3 + 16) = 19 bytes per 3 seconds.
- Even though the DriverLocationHT could fit on one server, it should be distributed across multiple Driver Location servers using DriverID-based partitioning for scalability and reliability. Each server will:
    - Broadcast driver location updates to all nearby interested customers.
    - Notify the QuadTree server periodically (e.g., every 10 seconds) to refresh the driver’s stored location.
- A Push Model using a publisher/subscriber Notification Service can keep customers updated with drivers’ current locations. When a customer queries for nearby drivers, the server subscribes them to updates from those drivers. The system maintains a subscriber list for each driver, and any location change in DriverLocationHT triggers a broadcast of the updated position to all subscribed customers, ensuring real-time accuracy.
- Implementing the Notification Service can use HTTP long polling or push notifications, but managing dynamic subscriptions for new drivers entering a customer’s view is complex. A simpler approach is a Pull Model, where clients periodically (e.g., every 5 seconds) send their location to the server, which queries the QuadTree for nearby drivers and returns updated positions. Additionally, grids in the QuadTree can handle up to 10% growth or shrinkage beyond their limit before being repartitioned or merged, reducing system load during high traffic.
- Beyond QuadTrees, production systems increasingly use:
    - **Google S2 / Uber H3:** Uber's dispatch system (DISCO) uses Google's S2 library for cell-based geospatial indexing. S2 divides maps into cells with unique IDs, enabling efficient distribution and coverage queries. Uber later developed H3, a hexagonal grid system that handles edge cases better than square grids.​
    - **Geohash:** Geohashing provides fast encoding/decoding and works seamlessly with standard databases. It handles high-frequency writes better than QuadTrees but struggles with uneven density distributions. DoorDash and Uber Eats use geohashing for quick driver updates

**Ride Request Deep Workflow** - 

- The customer put a  request for a ride. 
- One of the Aggregator servers take the request -> asks QuadTree servers to return nearby drivers.
- Aggregator server collects all the results -> sorts them by ratings.
- Aggregator server send a notification to the top (say three) drivers simultaneously, whichever driver accepts the request first will be assigned the ride. The other drivers will receive a cancellation request. If none of the three drivers respond, the Aggregator will request a ride from the next three drivers from the list.
- Once a driver accepts a request, the customer is notified

**Preventing multiple ride requests from being sent to the same driver simultaneously (Consistency)** -

Ride matching requires strong consistency so that each ride request is sent to only one driver at a time, and each driver receives only one request during the 10-second decision window.
- **Problem:** Application-level locks cause coordination issues since there are multiple drivers with their individual locks, inconsistent lock states after crashes, and scalability problems across multiple service instances since coordinating large scale locks becomes expensive. Application-level locks = locking at driver’s app.
- **Database-based locking:** Moves coordination to the database using transactional updates, but still suffers from indefinite locks if the service crashes/restarts before releasing them (timeout). This is a common issue with in-memory timeouts and why they should be avoided when possible. We can use cron job that runs periodically to check for locks that have expired and release them. This works, but adds unecessary complexity and introduces a delay in unlocking the ride request.
- **Final solution (Redis locks):** Use a distributed lock in Redis tied to the driver ID with a TTL of 10 seconds. If the ride is accepted, the lock is released; otherwise, it expires automatically. This resolves coordination and timeout issues, though it introduces dependency on Redis’s availability and performance—an acceptable tradeoff given the short-lived nature of these locks.

**Ensure no ride requests are dropped during peak demand periods** - 
- We could use a distributed message queue system like Kafka, which allows us to commit the offset of the message in the queue only after we have successfully found a match. 
- This way, if the Ride Matching Service goes down, the match request would still be in the queue, and a new instance of the service would pick it up.
- The main problem that can arise is since this is a FIFO queue, requests may get stuck behind a request that is taking a long time to process. This issue is common in FIFO queues and can be solved using priority queues taking into account proximity, driver rating, etc.

**Making system durable if driver fails to respond in a timely manner** - 
- If a driver doesn’t respond within the timeout, the system uses a delay queue mechanism (e.g., Amazon SQS delay queue) to automatically retry the ride request with the next available driver.
- When a request is sent, a delayed message is scheduled for 10 seconds later; if the ride remains unassigned when the message is processed, the system moves to the next driver and reschedules another delay.
- The main challenges are canceling delayed messages when a driver accepts and ensuring proper coordination between the delay queue and the ride matching service to prevent race conditions or incorrect reassignments.
- Using a durable execution framework like Temporal or AWS Step Functions can make ride matching fault-tolerant by persisting workflow state and handling timeouts, retries, and fallbacks even if services crash or restart. The workflow would sequentially send ride requests to drivers with 10-second timeouts, automatically moving to the next driver until one accepts and completes the workflow or all are exhausted—without losing progress.
- The main challenge is the added complexity of integrating and maintaining a workflow orchestration system, but the benefits of guaranteed execution, resilience, and simplified logic often outweigh the cost for mission-critical systems like ride-sharing.

**Further Scaling** - 
- The simplest scaling method, vertical scaling, involves upgrading existing servers by adding more CPU, memory, or storage. 
- While quick and easy, it has major downsides: it is expensive, requires downtime, has scalability limits, and lacks fault tolerance because a single server failure can bring the system down. Adding redundancy here is only a temporary fix.
- A more practical and scalable approach is horizontal scaling, which adds more servers and shards data geographically with read replicas for better throughput and lower latency. 
- Horizontal scaling improves fault tolerance, distributes load, and reduces client-server distance, benefiting all system components like services, message queues, and databases. Proximity searches near shard boundaries may require queries to multiple shards (scatter-gather).
- The main challenge with horizontal scaling is managing data distribution, failure handling, and shard rebalancing complexity. Techniques like consistent hashing can evenly distribute data, and replication strategies improve availability and fault tolerance.
- Overall, horizontal scaling is preferred for large, geo-distributed systems due to its scalability, performance benefits, and fault tolerance, despite higher operational complexity compared to vertical scaling.
