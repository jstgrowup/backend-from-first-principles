### **Technical Notes: Backend Scaling and Performance Engineering (Part 2)**

---

#### **1. Statelessness: The Key to Horizontal Scaling**

- **Definition:** Statelessness means that no individual server instance holds data or information exclusive to itself.
- **Stateful vs. Stateless:**
  - **Stateful:** A server remembers information about a client/user in its local memory (e.g., an in-memory array).
  - **Stateless:** Any instance can handle any request because no instance holds unique data.
- **Centralizing State:** To achieve horizontal scaling, all persistent data must live **outside** the application servers in a centralized location accessible by all instances.
  - **Sessions:** Use an in-memory database like **Redis** instead of local RAM.
  - **File Storage:** Use **Object Storage** (e.g., AWS S3, Cloudflare R2, or MinIO) instead of a server's local SSD.
  - **Databases:** Use centralized instances like **Postgres**, **RDS**, or **Centralized SQLite** instead of local files.

---

#### **2. Load Balancers (LB)**

- **Purpose:** Acts as a mandatory middleman that receives all internet traffic and distributes it among backend instances.
- **The Logic Layer:** The LB uses algorithms to decide which specific instance should handle a request.
- **Load Balancing Algorithms:**
  - **Round Robin:** Requests are sent in a rotating order (A $\rightarrow$ B $\rightarrow$ C $\rightarrow$ A...). Best for similar request types and equal server capacities.
  - **Weighted Round Robin:** Sends more requests to higher-capacity servers (e.g., a server with 8GB RAM gets twice as many requests as a 4GB one).
  - **Least Connections:** The LB checks which server has the fewest **active connections** (requests currently being processed) and routes the new request there.
  - **Least Response Time:** Sends more requests to servers that are returning responses faster.
  - **Resource-based:** Checks real-time CPU or RAM usage and sends traffic to the least-taxed server.
- **Health Checks:**
  - The LB sends periodic "test requests" (e.g., a simple GET every second) to all instances.
  - If a server fails to return a **200 success response**, it is "blacklisted" and removed from the rotation.
  - The LB continues testing blacklisted servers and brings them back online once they respond successfully.

---

#### **3. Database Scaling**

Databases are the "stateful" part of architecture and are harder to scale than code because data must remain consistent across instances.

**A. Read Replicas**

- **Architecture:** One **Primary/Master** instance handles all Write operations (Insert, Update, Delete). Multiple **Secondary/Slave** instances handle Read operations (Select).
- **Benefits:** Reduces load on the primary server and lowers latency by placing replicas closer to users geographically.
- **The Challenge: Replication Lag:**
  - Due to physics (speed of light in fiber optics), there is a delay when syncing data from the Primary to Replicas (e.g., 200ms from US to India).
  - **Consistency Issues:** A user might update their profile (Write to Primary) and immediately refresh. If the Read request hits a Replica before the 200ms sync is finished, they see their old data.
- **Solutions:**
  - Route Read requests for a specific entity to the Primary instance for a short window after a Write.
  - Track replication lag and block Reads until sync is complete.
  - Implement planned latency in the frontend (e.g., waiting 300ms before refetching).

**B. Sharding (Partitioning)**

- **Definition:** Physically dividing a single massive table (e.g., billions of rows) into multiple separate database instances.
- **Mechanism:** Uses a **Sharding Key** (e.g., `order_date`) to decide where data lives.
  - _Example:_ Shard 1 holds Jan–June data; Shard 2 holds July–Dec data.
- **Benefit:** Reduces query latency by searching fewer rows and allows multiple instances to handle more requests per second.

**C. Distributed Databases**

- Modern providers handle sharding, replication, and backups automatically.
- **Examples:** PlanetScale (Vitess/MySQL), Neon (Postgres/Rust), CockroachDB, Yugabyte.

---

#### **4. Global Caching: CDNs and Edge Computing**

- **The Physics Cap:** Light in fiber optics travels at ~200,000 km/s. A round trip from Tokyo to US East (20,000 km) takes a minimum of **100ms**.
- **CDNs (Content Delivery Networks):** Place **Edge Locations** (or POPs - Points of Presence) closer to users (e.g., in Tokyo) to reduce latency from 100ms to ~2ms.
- **Content Types:**
  - **Static:** JS bundles, CSS, HTML, images, videos, fonts.
  - **API Responses:** Product catalogs or blogs (with cache purging/invalidation logic).
- **Security:** CDNs protect against **DDoS attacks** by distributing terabytes of malicious traffic across their global network.
- **Edge Computing:**
  - Processing logic at the edge node (not just serving files).
  - **Use Cases:** Authentication (checking a session at the edge in 2ms instead of 100ms at the main server) and localization (serving content based on a user's region/language).
  - **Constraints:** Edge nodes have limited RAM/CPU compared to primary data centers and often use restricted runtimes (e.g., **V8 Isolates**) that can't access file systems or certain TCP protocols.

---

#### **5. Asynchronous Processing**

- **Goal:** Offload heavy or non-essential tasks out of the request-response cycle to reduce **perceived latency**.
- **Architecture:**
  - **Producer:** The backend server that pushes a "Job" or "Task" into a queue.
  - **Broker (Queue):** Redis or RabbitMQ.
  - **Consumer (Worker):** A separate process that dequeues and executes the task.
- **Use Cases:**
  - **Email/Notifications:** Sending an invite might take 400ms. By offloading it, the user sees success in 100ms.
  - **Video Processing:** Uploading to YouTube triggers background tasks like generating thumbnails, encoding HD, and generating subtitles.
  - **Account Deletion:** Instead of making the user wait 8 seconds to delete 10 years of data, the server responds in 100ms and cleans up in the background.

---

#### **6. Microservices vs. Monoliths**

- **Monolith:** A single deployable unit where all code (auth, payments, notifications) lives together.
- **Microservices:** Dividing an application into independent, deployable services.
- **Why Microservices?**
  - **Scaling Teams:** Allows 500+ developers to work without deployment dependencies.
  - **Independent Scaling:** Scale only the "Payment" service if it’s under load, leaving "Notifications" as is.
  - **Diverse Tech Stacks:** Use **Node.js** for markdown parsing and **Rust/Go** for CPU-bound image manipulation.
- **Drawbacks:** Network latency, failure handling (retries/timeouts), complex debugging (distributed tracing), and data consistency across separate databases.

---

#### **7. Serverless Computing**

- **Traditional (Serverful):** You manage a VM (EC2/Ubuntu), plan capacity (RAM/CPU), and pay 24/7 regardless of traffic.
- **Serverless:** You only manage **Code (Functions)** and **Events**. The provider (AWS Lambda, Cloudflare Workers) handles the machine.
- **The "Cold Start" Problem:** The time it takes for a provider to spin up a new machine instance upon the first request.
  - **Fixes:** Using **Firecracker KVM** or **V8 Isolates** for fast booting, and using interpreted languages (JS/Python) over compiled ones (Java).
- **Ideal Use Cases:** Intermittent tasks like image resizing, video processing, or event-triggered pipelines.

---

### **Code Snippets**

✅ **Shown in video: Least Connections Logic (Pseudo-logic)**

```javascript
// Internal logic described for a Load Balancer algorithm
const servers = [
  { id: "A", activeConnections: 5 },
  { id: "B", activeConnections: 2 },
  { id: "C", activeConnections: 0 },
];

// Algorithm picks the server with 0 connections
const target = servers.reduce((prev, curr) =>
  prev.activeConnections < curr.activeConnections ? prev : curr,
);
```

_Demonstrates how a 'smarter' LB prioritizes idle servers over busy ones._

🧪 **Generated learning example: Implementing a Sharding Router**

```javascript
// Logic for a routing layer to decide which shard to query
function getShardForOrder(orderDate) {
  const month = new Date(orderDate).getMonth();

  if (month < 6) {
    return "shard_primary_jan_june"; // DB Instance 1
  } else {
    return "shard_secondary_july_dec"; // DB Instance 2
  }
}
```

_Mimics the database routing logic explained for partitioning table rows geographically or chronologically._

🧪 **🧪 Generated learning example: Edge Authentication (Cloudflare Workers)**

```javascript
// Example of Edge Computing for Auth
export default {
  async fetch(request) {
    const sessionCookie = request.headers.get("Cookie");

    // 1. Check Auth at the Edge (2ms latency)
    if (!isValidSession(sessionCookie)) {
      return new Response("Unauthorized", { status: 401 });
    }

    // 2. If valid, pass the request to the origin server in US East
    return await fetch("https://primary-api.example.com", request);
  },
};
```

_Illustrates how edge computing reduces round-trip waste by rejecting unauthorized users locally._

---

**Final Thumb Rule:** Always **measure** before you optimize. Prefer **simplicity** (vertical scaling, indexing, monoliths) over complexity (horizontal scaling, microservices) until absolutely necessary.
