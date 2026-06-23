### **Technical Notes: Logging, Monitoring, and Observability**

---

#### **1. Core Concepts and the "Spectrum"**

- **The Reality of Implementation:** Logging, monitoring, and observability are not binary rules but a **spectrum of practices**. No company or product follows "all" perfect practices; implementation varies based on needs and team size.
- **Need for Modern Systems:** In modern distributed environments, applications run across different servers and regions with a global user base. These methodologies are essential to keep track of system health, infrastructure, and events.
- **Scope:** While applicable to the frontend, these discussions primarily focus on **backend applications**.

---

#### **2. Definitions and the Three Pillars**

Observability is a modern movement that encompasses and expands upon traditional logging and monitoring.

| Component         | Definition                                                                                          | Parameters Tracked                                                                       |
| :---------------- | :-------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------- |
| **Logging**       | Recording all important, suspicious, or security-related events in an application.                  | User ID, latency, method, function triggered, timestamps, and database queries.          |
| **Monitoring**    | Continuously checking the state, health, and performance of the system to track patterns over time. | CPU usage, memory, requests per second (throughput), and database connection pool state. |
| **Observability** | Determining the internal state of a system by looking at its external outputs.                      | Combines logs, metrics, and traces to explain _what_ is wrong and _exactly why_.         |

**The Three Pillars of Observability:**

1.  **Logs:** A record of important events throughout the application life cycle.
2.  **Metrics:** Concrete numbers (real-time and historical) used to quantify system state (e.g., error rates, throughput, transaction times).
3.  **Traces:** Transactions that track a request's journey across various components (e.g., Load Balancer $\rightarrow$ Handler $\rightarrow$ Validation $\rightarrow$ Service $\rightarrow$ Repository $\rightarrow$ Database).

---

#### **3. Practical Troubleshooting Workflow**

A properly implemented system allows for a seamless debugging path:

1.  **Alert:** An automated message (e.g., via Slack) is triggered when a metric crosses a threshold (e.g., error rate $> 80\%$).
2.  **Metrics Check:** Engineers view dashboards (Grafana/New Relic) to see the trend of failed requests.
3.  **Log Analysis:** From the metric, engineers jump to specific logs (e.g., a "500 Internal Server Error" log).
4.  **Trace Inspection:** By clicking the log, engineers view the **Trace**, seeing exactly which function the request started at and where it failed.

---

#### **4. Logging Deep Dive**

**A. Logging Levels**
Production systems assign levels to logs to filter relevance:

- **Debug:** Used for troubleshooting in development; provides maximum detail. Usually disabled in production.
- **Info:** General application operations and business events (e.g., "to-do created").
- **Warn (Warning):** Non-critical issues or events between info and error (e.g., a user entering a wrong password).
- **Error:** Any operation failure, such as validation errors or database query failures.
- **Fatal:** A serious issue that causes the application to shut down/restart.

**B. Formatting: Structured vs. Unstructured**

- **Unstructured (Console Logs):** Human-readable, colorful plain text used during development for easy spotting of issues in the terminal.
- **Structured (JSON Logging):** Standard for production systems.
  - **Why?** Tools (ELK stack, Loki, New Relic) can easily parse JSON to extract parameters like User ID or Request ID. Parsing plain text is inefficient and error-prone for management tools.

---

#### **5. Modern Implementation Standards**

- **Instrumentation:** The practice of measuring different attributes of a function to make a system observable.
- **OpenTelemetry:** A modern, open standard providing SDKs and tools for all major languages (Node.js, Go, Python) to instrument applications consistently.
- **Real-time Delay:** Traditional monitoring often has a **10–15 second delay** to avoid overwhelming the observability system with constant data.

---

#### **6. Tooling Ecosystem**

The video highlights two paths for implementing these systems:

- **Open Source Route (The "Grafana Stack"):**
  - **Grafana:** Frontend dashboard.
  - **Prometheus:** Builds and stores metrics.
  - **Loki & Promptail:** Log management.
  - **Jaeger:** For distributed traces.
- **Proprietary/One-Stop Route:**
  - **New Relic / DataDog:** Managed solutions that simplify configuration and combine all pillars into one dashboard.

---

#### **7. Infrastructure and Deployment (PaaS)**

- **Sponsor Mention (Seala):** A Platform-as-a-Service (alternative to Netlify, Vercel, Heroku).
- **Capabilities:** Deploy full-stack apps, databases, and observability tools (Grafana, Prometheus) via Docker.
- **Tech Stack:** Uses **Kubernetes** under the hood, **GCP** (Premium Tier), and **Cloudflare’s Edge Network** (260+ points of presence).
- **Features:** Preview deployments (auto-deployed PR domains), Git-bot integration, and automatic static asset caching.

---

### **Code Snippets**

✅ **Shown in Video: Dynamic Log Level Configuration (Go)**

```go
// Function to configure logging based on environment
// Sources
func getLogLevel(env string) string {
    if env == "development" {
        // More verbose logs for local troubleshooting
        return "debug"
    }
    // Standard operational logs for production
    return "info"
}
```

_This logic determines whether the system outputs detailed troubleshooting data or standard business event data._

✅ **Shown in Video: Tracing Middleware Logic**

```go
// Logic described for initializing a request trace
// Sources
func tracingMiddleware(req Request) {
    // 1. Create a new transaction/trace instance
    txn := newRelic.StartTransaction("service_name")

    // 2. Add metadata (Metadata) to the trace for later debugging
    txn.AddAttribute("ip_address", req.IP)
    txn.AddAttribute("user_id", req.UserID)
    txn.AddAttribute("request_id", req.RequestID)

    // 3. Store the transaction in the Context to pass through layers
    ctx := context.WithValue(req.Context(), "txn", txn)

    // 4. Ensure the transaction ends when the request is finished
    defer txn.End()
}
```

_This demonstrates how a "trace" is created at the point of origin and carries metadata across the application's lifecycle._

🧪 **Generated Learning Example: Structured (JSON) Production Log**

```json
{
  "timestamp": "2026-06-23T23:30:32Z",
  "level": "error",
  "message": "database query failed",
  "service": "todo-api",
  "request_id": "req-998877",
  "user_id": "user_123",
  "error_details": {
    "query": "SELECT * FROM todos WHERE id = $1",
    "code": "500"
  },
  "span_id": "span-abc-123"
}
```

_This replicates the production JSON format explained in the video, designed for easy parsing by tools like Loki or New Relic._

🧪 **Generated Learning Example: Service Layer Instrumentation**

```javascript
// Example of adding attributes to a trace within a specific business logic function
async function createTodoService(data, ctx) {
  // 1. Retrieve the existing transaction from the context
  const txn = ctx.getTransaction();

  // 2. Add business-specific attributes (Instrumentation)
  txn.addAttribute("todo_title", data.title);
  txn.addAttribute("priority", data.priority);

  try {
    const result = await db.save(data);
    // Log business event at INFO level
    logger.info(`To-do created successfully with ID: ${result.id}`);
    return result;
  } catch (err) {
    // Add error to the trace so it shows up in the dashboard
    txn.noticeError(err);
    logger.error("Failed to create to-do", { error: err.message });
    throw err;
  }
}
```

_This demonstrates the "Instrumentation" practice described, where specific attributes and errors are manually added to a transaction within the service layer._
