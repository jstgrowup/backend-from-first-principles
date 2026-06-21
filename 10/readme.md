### **Technical Notes: Handlers, Services, Repositories, Middlewares, and Request Context**

---

#### **1. The Internal Request Life Cycle**

- **Entry Point:** The operating system forwards an HTTP request to a specific port (e.g., 3000, 4000) where the server is listening.
- **Routing:** The server uses routing algorithms to map the request to a specific **Handler** (a predefined function) based on the URL path (e.g., `/users/123`).
- **Design Pattern:** While a single handler could technically perform all tasks, production-grade backends separate responsibilities into three layers—**Handlers, Services, and Repositories**—to ensure the codebase is scalable, maintainable, and easier to debug.

---

#### **2. Handlers and Controllers**

- **Responsibility:** The "control" layer that manages data flow between the client and the server. It is the only layer that should deal with HTTP-specific details like status codes and headers.
- **Inputs:** Handlers typically receive two objects from the runtime: a **Request** object (to read data) and a **Response** object (to send data).
- **Core Tasks:**
  1.  **Data Extraction:** Taking data out of the request object (Query Params for GET; Request Body for POST/PUT/PATCH/DELETE).
  2.  **Binding/Deserialization:** Converting external data formats like **JSON** into the programming language's native format (e.g., a **Go struct**, Python dictionary, or Rust struct).
      - If this fails, the handler sends a **400 Bad Request** and terminates.
  3.  **Validation & Transformation:** Ensuring data follows the expected structure/semantics and making it "convenient" for downstream layers.
      - _Example:_ Setting a default "sort" parameter to "date" if the client leaves it optional.
  4.  **Calling the Service Layer:** Passing validated data and metadata (like User IDs) to the Service.
  5.  **Sending the Response:** Deciding on the appropriate response code (e.g., **200 OK**, **201 Created**, or **204 No Content** for deletes) and returning the final data.

---

#### **3. The Service Layer**

- **Responsibility:** The **"Orchestrator"** where the actual business logic and processing happen.
- **Isolated Logic:** Ideally, a service function should not have any HTTP-related code; it should be isolated and unaware if it is being called by an API or another internal process.
- **Capabilities:**
  - Calling multiple repository methods to merge data.
  - Executing external logic like sending emails or notifications.
  - Making external API calls.

---

#### **4. The Repository Layer**

- **Responsibility:** The **Database Layer** dedicated solely to data persistence.
- **Single Responsibility Principle:** A repository method should do exactly one thing—fetch, insert, or delete a specific kind of data.
  - _Example:_ You should have separate functions for "get all books" and "get book by ID" rather than using one function with optional parameters.
- **Query Construction:** It takes filtered/sorted data from the Service and constructs the actual database query.

🧪 **🧪 Generated learning example: The 3-Layer Pattern (Go)**

```go
// 1. REPOSITORY LAYER: Pure Database Logic
func (r *BookRepo) GetAllBooks(sortBy string) ([]Book, error) {
    // Construct DB Query: SELECT * FROM books ORDER BY sortBy
    return books, nil
}

// 2. SERVICE LAYER: Business Logic/Orchestration
func (s *BookService) FetchBooks(sortParam string) ([]Book, error) {
    // Service doesn't care about HTTP; it just orchestrates
    return s.repo.GetAllBooks(sortParam)
}

// 3. HANDLER/CONTROLLER: HTTP Entry Point
func GetBooksHandler(w http.ResponseWriter, r *http.Request) {
    // Extraction & Transformation
    sort := r.URL.Query().Get("sort")
    if sort == "" { sort = "date" } // Set Default

    // Call Service
    books, err := service.FetchBooks(sort)

    // Send Response
    if err != nil {
        w.WriteHeader(500)
        return
    }
    w.WriteHeader(200)
    json.NewEncoder(w).Encode(books)
}
```

_This demonstrates how a request flows from the Handler (HTTP) to the Service (Logic) to the Repository (DB)._

---

#### **5. Middlewares**

- **Definition:** Optional functions executed in the "middle" of the request life cycle, between the entry point and the final handler.
- **Parameters:** Receives the **Request**, **Response**, and a **Next** function.
- **The Next Function:** Used to pass execution to the next processing context (the next middleware or the handler).
- **Early Termination:** A middleware can send a response and terminate the request immediately if logic fails (e.g., an authentication check).
- **Common Use Cases:**
  - **CORS:** Checks the origin header and adds allowed origin headers to responses.
  - **Logging/Monitoring:** Records the path, method, and parameters of every request for auditing and debugging.
  - **Authentication:** Verifies tokens (JWT/Session ID) and attaches user metadata to the request context.
  - **Rate Limiting:** Tracks IP addresses to prevent abuse; sends **429 Too Many Requests** if thresholds are exceeded.
  - **Global Error Handling:** Usually placed at the **last point** of execution to catch errors from any layer and return a structured JSON error message.
  - **Compression:** Using algorithms like **Gzip** for large JSON responses.
- **Importance of Order:** The order of middlewares is critical. Typically: **CORS (first)** -> Logging -> Authentication -> Handler -> **Error Handling (last)**.

---

#### **6. Request Context**

- **Definition:** A shared storage/state that is **scoped and limited to a single request**.
- **Purpose:** Allows different middlewares and handlers to share information (Key-Value pairs) without being closely coupled.
- **Primary Use Cases:**
  - **Authentication Metadata:** Storing the `user_id` or `role` after a successful auth middleware check.
    - _Security Note:_ Using the `user_id` from the context in the database layer prevents malicious clients from spoofing other users' IDs in the request body.
  - **Tracing:** Storing a **UUID (Request ID)** generated at the start of the request to trace logs across microservices.
  - **Signals:** Sending cancellation signals, abort signals, or deadlines to downstream services to prevent hanging processes.

🧪 **🧪 Generated learning example: Middleware & Request Context (Go)**

```go
func AuthMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")

        // 1. Logic check
        userID, err := verifyToken(token)
        if err != nil {
            // 2. Early Termination
            http.Error(w, "Unauthorized", 401)
            return
        }

        // 3. Set Request Context (Shared State)
        ctx := context.WithValue(r.Context(), "user_id", userID)

        // 4. Call Next
        next.ServeHTTP(w, r.WithContext(ctx))
    }
}
```

_This simulates an authentication middleware that verifies credentials and passes the User ID forward via the request context._
