### **Technical Notes: Backend Scaling and Performance Engineering (Part 1)**

---

#### **1. Defining Performance: The User Perspective**

- **The Experience:** Performance is essentially what a user "feels" when interacting with a system—specifically the gap between clicking a button and seeing rendered results (e.g., cards showing up on a screen).
- **The Request Path:** A browser sends a request $\rightarrow$ the server processes it $\rightarrow$ the server interacts with a database or external APIs (e.g., Resend for emails) $\rightarrow$ the server returns JSON $\rightarrow$ the browser parses and renders it.
- **Latency:** The fundamental unit of performance measurement. It is the total time taken for this full interaction cycle.

#### **2. Latency Metrics: Averages vs. Percentiles**

- **The Trap of Averages:** Averages are misleading and often useless for performance because they hide outliers.
  - _Scenario:_ If 99% of requests take 50ms but 1% take 5s, an average might look "good" while 10,000 out of 1 million daily users have a terrible experience.
- **Percentiles (The Industry Standard):**
  - **P50 (Median):** 50% of users experience this latency or less.
  - **P90:** 10% of users experience latency higher than this number.
  - **P99:** Only 1% of users experience latency higher than this. 99% experience less.
- **Why P99/P95 Matter:** These "unhappy" users often represent the most complex business logic (e.g., a high-value customer making a payment vs. just browsing). Focus here targets complex queries and service synchronization.

#### **3. Throughput and Utilization**

- **Throughput:** Measures how many requests a system can handle in a given time (e.g., Requests Per Second/Minute).
  - Essential for planning for spikes like **Black Friday**, email campaigns, or being featured in a podcast.
- **The Relationship:** As throughput increases, latency remains stable initially but eventually spikes dramatically (exponentially).
- **Utilization:** The percentage of system capacity currently in use.
  - **0%:** System is idle.
  - **100%:** System is maxed out and at the brink of collapse.
- **The Headroom Rule:** Never run systems at 100% utilization. Real-world systems usually target **60%–80%**, leaving a 20% **buffer/headroom** to absorb traffic **bursts**.

---

#### **4. Identifying Bottlenecks: Measurement Over Guesswork**

- **The Bottleneck:** The specific part of a system causing slowness.
- **The Mistake:** Jumping to "by the book" solutions like adding caching or upgrading servers without measuring first.
- **Case Study:** An API is slow. The developer adds **Redis** (taking one week), but it remains slow. Actual measurement reveals the database took only 10ms, but a **synchronous remote logging call** (e.g., to Elasticsearch) was taking 500ms.
- **Profiling:** Measures exactly where an application spends its time (CPU-bound tasks).
  - **Flame Graphs:** Visualizes the call stack; wider stacks indicate functions taking more time.
- **Distributed Tracing:** Best for **I/O bound tasks** (database queries, file handling, external API calls), which are the most common causes of backend performance issues.

---

#### **5. Database Optimization Strategies**

**A. The N+1 Query Problem**

- **Definition:** Fetching a list of $N$ items and then performing $N$ individual queries to fetch related details (e.g., fetching 20 blogs, then 20 separate calls for authors = 21 calls).
- **The Cost:** Each query has overhead (network transmission, TCP handshake, query parsing, and execution planning).
- **The Solution:** **Bulk fetching** or using **Joins** to get all data in 1 or 2 queries.

✅ **Shown in video: ORM Primitives for Bulk Fetching**

```javascript
// Examples of avoiding N+1 using different ORMs
// Django: select_related (Foreign Key) or prefetch_related (Many-to-Many)
// Ruby on Rails: .includes()
// Type-safe ORMs (Prisma/Drizzle): Left Join and Select
```

**B. Indexing Strategy**

- **Librarian Analogy:** Searching a library with no catalog requires a **Sequential Scan** (examining every book), which takes days. A catalog (index) allows you to go directly to the shelf.
- **B-Tree Indexes:** The most common type; it maintains a sorted copy of column values with pointers to the actual rows.
- **The Cost of Indexes:** They are not free. They take disk space and slow down **Insert, Update, and Delete** operations because the index must be kept in sync.
- **Composite Index:** Covers multiple columns; the order of columns in the index definition matters for query matching.
- **Covering Index:** An index that contains all the data needed for a query, allowing the database to skip fetching the actual table row.
- **Tool:** Use `EXPLAIN ANALYZE` to see if the database is performing a "Sequential Scan" or an "Index Scan".

**C. Connection Pooling**

- **The Overhead:** Creating a new TCP connection for every query involves a 3-way handshake, authentication, and memory allocation.
- **Connection Limits:** Databases have hard limits (e.g., 500 connections). High traffic can exhaust these and crash the DB.
- **The Solution:** A **Connection Pool** maintains a set of idle, open connections for the server to "borrow" and "return".
  - **Internal Pooling:** Managed by the application driver; risky with horizontal scaling as multiple servers can collectively exceed DB limits.
  - **External Pooling (e.g., PG Bouncer):** A lightweight middleman that manages connections for all server instances, preventing DB exhaustion.

---

#### **6. Caching Patterns and Challenges**

- **Cache Invalidation:** One of the two hardest problems in programming.
  - **Time-based (TTL):** Automatic expiry after a set time.
  - **Event-based:** Explicitly deleting/updating the cache when the underlying data changes.
- **Locations:**
  - **Local Caching:** In-memory (dictionary/map) on the server. Fast but causes **consistency issues** across multiple server instances.
  - **Distributed Caching (Redis/Memcached/Valkey):** Shared across all instances. Avoids inconsistency but adds a network roundtrip (~50ms).
- **Tiered Caching:** Using a local cache for the "hottest" data and a distributed cache as the upstream source.
- **Caching Patterns:**
  1.  **Cache Aside (Lazy Loading):** Most common. Check cache $\rightarrow$ if miss, fetch from DB $\rightarrow$ save in cache $\rightarrow$ return.
  2.  **Write Through:** Update DB and Cache simultaneously. Cache is always fresh, but writes are slower.
  3.  **Write Behind:** Update Cache immediately $\rightarrow$ update DB asynchronously. Fast response, but risk of data inconsistency if the DB write fails.
- **Cache Hit Rate:** The percentage of requests served by the cache. Target is ~90%; low rates suggest poor TTL or misunderstanding of user access patterns.

---

#### **7. Scaling Approaches**

| Feature    | Vertical Scaling (Scaling Up)                                         | Horizontal Scaling (Scaling Out)                                                         |
| :--------- | :-------------------------------------------------------------------- | :--------------------------------------------------------------------------------------- |
| **Method** | Upgrading CPU, RAM, Storage (SSD/NVMe), or Network Cards.             | Adding more instances of the same server.                                                |
| **Pros**   | Simple; no architecture or code changes.                              | Redundancy, no hard limits, and enables geo-distribution.                                |
| **Cons**   | Hard hardware ceilings, single point of failure, no geo-distribution. | Highly complex: requires Load Balancers, state sync, and handling network/node failures. |

- **Load Balancing:** Necessary for horizontal scaling to distribute requests.
- **Distributed Complexity:** Horizontal scaling transforms one set of problems into a new set (e.g., how to keep servers in sync, how to detect dead nodes).

---

🧪 **Generated learning example: Implementing a Cache-Aside Pattern**

```javascript
// Standard pattern for reducing DB load
async function getProduct(id) {
  // 1. Try fetching from Distributed Cache (Redis)
  const cachedProduct = await redis.get(`prod:${id}`);
  if (cachedProduct) return JSON.parse(cachedProduct);

  // 2. Cache Miss: Fetch from DB
  const product = await db.products.findUnique({ where: { id } });

  // 3. Save to Cache with a TTL (e.g., 1 hour)
  if (product) {
    await redis.set(`prod:${id}`, JSON.stringify(product), "EX", 3600);
  }

  return product;
}
```

_Demonstrates the lazy loading logic where the database is only consulted when the cache is empty._
